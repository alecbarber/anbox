# Anbox

Anbox is a container-based approach to boot a full Android system on a
regular GNU/Linux system like Ubuntu. In other words: Anbox will let
you run Android on your Linux system without the slowness of
virtualization. For full details, see the mainline distribution's documentation:
<https://github.com/anbox/anbox>.

This fork implements webcam access from Anbox. While the webcam is accessible
from Anbox, this project isn't ready to be merged into the main repository
&ndash; see the "limitations" section below.

## Installation instructions

The only reliable way to use this fork is to build it from source. Before
building the Anbox runtime from source, you need an Android image. This can
either be downloaded from [here](https://drive.google.com/file/d/1zc3uVuu8p2i1DdNYgz6vHlaYn1qLa5Qb/view?usp=sharing)
or built from source using the instructions below.

These instructions are copied almost verbatim from the main repository

### Requirements

To build the Anbox runtime itself there is nothing special to know. We're using
cmake as build system. A few build dependencies need to be present on your host
system:

 * libdbus
 * google-mock
 * google-test
 * libboost
 * libboost-filesystem
 * libboost-log
 * libboost-iostreams
 * libboost-program-options
 * libboost-system
 * libboost-test
 * libboost-thread
 * libcap
 * libexpat1-dev
 * libsystemd
 * mesa (libegl1, libgles2)
 * libglm
 * libsdl2
 * libprotobuf
 * protobuf-compiler
 * python3
 * lxc (>= 3.0)

On an Ubuntu system you can install all build dependencies with the following
command:

```
$ sudo apt install build-essential cmake cmake-data debhelper dbus google-mock \
    libboost-dev libboost-filesystem-dev libboost-log-dev libboost-iostreams-dev \
    libboost-program-options-dev libboost-system-dev libboost-test-dev \
    libboost-thread-dev libcap-dev libexpat1-dev libsystemd-dev libegl1-mesa-dev \
    libgles2-mesa-dev libglm-dev libgtest-dev liblxc1 \
    libproperties-cpp-dev libprotobuf-dev libsdl2-dev libsdl2-image-dev lxc-dev \
    pkg-config protobuf-compiler python3-minimal
```
It's recommended that you use Ubuntu 20.04 (focal) as your build environment.


### Build

Afterwards you can build Anbox with

```
$ git clone https://github.com/alecbarber/anbox.git --recurse-submodules
$ cd anbox
$ mkdir build
$ cd build
$ cmake ..
$ make
```

A simple

```
$ sudo make install
```

will install the necessary bits into your system.

## Run Anbox

There are several steps to run Anbox:

Setup the network bridge:
```
$ sudo scripts/anbox-bridge.sh start
```

Start the container manager:
```
$ sudo mkdir /dev/binderfs
$ sudo mount -t binder binder /dev/binderfs
$ datadir=$HOME/anbox-data
$ mkdir -p $datadir/rootfs
$ sudo anbox container-manager \
    --privileged \
    --daemon \
    --android-image=/path/to/android.img \
    --data-path=$datadir
```

In a separate terminal session, run the session manager:
```
$ anbox session-manager
```

Finally, launch the Application Manager:
```
$ anbox launch --package=org.anbox.appmgr --component=org.anbox.appmgr.AppViewActivity
```

### Webcam setup

The session manager uses environment variables at startup to determine which
cameras (if any) to initialise. The front-facing camera is controlled by
`ANBOX_HW_CAMERA_FRONT` and the rear camera is defined by
`ANBOX_HW_CAMERA_BACK`. To connect `/dev/videoNN` to Anbox, set the appropriate
environment variable to `webcamNN`. For example, the following commands will
launch the session manager with `/dev/video2` as the front-facing camera:

```
$ export ANBOX_HW_CAMERA_FRONT=webcam2
$ anbox session-manager
```

Note that the session manager needs to be restarted every time the camera
configuration changes. If you need to access `/dev/videoNN` where `NN >= 16`,
define `ANBOX_HW_CAMERA_MAX_CAMERA` to be at least `NN + 1`.

### Copy screen to emulated camera

If you want to use the screen as an emulated camera input (e.g. to scan a QR
code), start by installing [v4l2loopback](https://github.com/umlaeute/v4l2loopback/).

You can then use `ffmpeg` to stream a portion of your screen to the emulated
device, using something like:

```
$ sudo modprobe v4l2loopback
$ ffmpeg -f x11grab -r 2 -video_size 800x600 -i :0.0+850,80 -f v4l2 /dev/LOOPBACK_DEVICE
```

### Common issues

* If the session manager fails to start with an error mentioning EGL, try `export EGL_PLATFORM=x11` 
* If build fails with an error about tests, comment out line 141 in [CMakeLists.txt](https://github.com/alecbarber/anbox/blob/master/CMakeLists.txt#L141)

## Building Android

Building Android from source is not a small undertaking. You need significant
computational resources available &ndash; 100GB of disk space and 16GB of
RAM are sensible minimum values. The full build will take a couple of hours;
if you encounter issues, expect troubleshooting to be time-consuming. These
instructions are based on the instructions [here](https://github.com/anbox/anbox/blob/master/docs/build-android.md).
More detailed information on building Android (including required packages etc)
can be found at the [Android Open Source Project](https://source.android.com/setup/build/requirements).

First, get the sources
```
$ repo init --depth=1 -u https://github.com/alecbarber/platform_manifests.git -b anbox
$ repo sync  -f --force-sync --no-clone-bundle --no-tags -j8
```

Then initialise the environment
```
$ . build/envsetup.sh
$ lunch anbox_x86_64-userdebug
```

And build!
```
$ make -j8
$ cd vendor/anbox
$ scripts/create-package.sh \
    $PWD/../../out/target/product/x86_64/ramdisk.img \
    $PWD/../../out/target/product/x86_64/system.img
```

### Common issues

If you encounter out-of-memory errors during the build, try increasing your swap
memory.

If the build fails with an assertion about locales failing, try
`export LC_ALL=C` and re-running `make`.

## Implementation details

On the host side, all this fork does is incorporate QEMU's host-side camera
implementation into the Anbox session manager. There are no substantive changes
to QEMU's code, but it has been refactored from C to be closer to the Anbox
C++ coding style.

There are a couple of minor changes to the Android source. The original QEMU
client implementation tries to read each camera frame in a single `read()`
call, and fails if it doesn't read the whole frame. This causes issues because
1024x720 video uses about 2MB per frame! The QEMU client has been
modified to respect `EAGAIN` when reading frames.

## Limitations

The main limitations are:

* The camera stream has heavy lag
* The camera doesn't work when running in a snap

The camera stream has about 50-100ms of lag; this is mainly because the QEMU
host camera implementation is extremely inefficient ([source](https://github.com/alecbarber/anbox/blob/master/src/anbox/camera/camera_format_converters.cpp#L419)).
The lag seems to trigger Whatsapp's "poor connection" detection; video calls are
quickly dropped. Major improvements should be possible by replacing the linked
code with GL shaders to do the format conversion.

When running Anbox as a snap, I've had little success enabling the camera. In
particular, I haven't been able to find the session manager logs, which has made
it difficult to debug the camera connection.

I'm actively working on both of these issues &ndash; contributions are more than
welcome!