# Building TensorFlow for Raspberry Pi: a Step-By-Step Guide

_[Back to readme](README.md)_

## What You Need

* Raspberry Pi 2 or 3 Model B
* An SD card running Raspbian with several GB of free space
	* An 8 GB card with a fresh install of Raspbian **does not** have enough space. A 16 GB SD card minimum is recommended.
	* These instructions may work on Linux distributions other than Raspbian
* Internet connection to the Raspberry Pi
* A USB memory drive that can be installed as swap memory (if it is a flash drive, make sure you don't care about the drive). Anything over 1 GB should be fine
* A fair amount of time

## Overview

These instructions were crafted for a [Raspberry Pi 3 Model B](https://www.raspberrypi.org/products/raspberry-pi-3-model-b/) running a vanilla copy of Raspbian 8.0 (jessie). It appears to work on Raspberry Pi 2, but [there are some kinks that are being worked out](https://github.com/tensorflow/tensorflow/issues/445#issuecomment-196021885). If these instructions work for different distributions, let me know!

Here's the basic plan: build a 32-bit version of [Protobuf](https://github.com/google/protobuf), use that to build a RPi-friendly version of [Bazel](https://github.com/bazelbuild/bazel), and finally use Bazel to build TensorFlow.

### Contents

1. [Install basic dependencies](#1-install-basic-dependencies)
2. [Build Protobuf](#2-build-protobuf)
3. [Install USB Memory as Swap](#3-install-a-memory-drive-as-swap-for-compiling)
4. [Build Bazel](#4-build-bazel)
5. [Compiling TensorFlow](#5-compiling-tensorflow)
6. [Cleaning Up](#6-cleaning-up)
7. [References](#references)

## The Build

### 1. Install basic dependencies

First, update apt-get to make sure it knows where to download everything.

```shell
sudo apt-get update
```

Next, install some base dependencies and tools we'll need later.

For Protobuf:

```
sudo apt-get install autoconf automake libtool maven
```

For Bazel:

```shell
sudo apt-get install pkg-config zip g++ zlib1g-dev unzip openjdk-8-jdk
# Select the java-8-openjdk option for the update-alternatives command
sudo update-alternatives --config java
```

For TensorFlow:

```
# For Python 2.7
sudo apt-get install python-pip python-numpy swig python-dev
pip install --user wheel

# For Python 3.3+
sudo apt-get install python3-pip python3-numpy swig python3-dev
pip3 install --user wheel
```

To be able to take advantage of certain optimization flags:

```
sudo apt-get install gcc-4.8 g++-4.8
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 100
```

Finally, for cleanliness, make a directory that will hold the Protobuf, Bazel, and TensorFlow repositories.

```shell
mkdir tf
cd tf
```

### 2. Build Protobuf

Clone the Protobuf repository.

```shell
git clone https://github.com/google/protobuf.git
```

Now move into the new `protobuf` directory, configure it, and `make` it. _Note: this takes a little while._

```shell
cd protobuf
git checkout v3.1.0
./autogen.sh
./configure
make -j 4
sudo make install
sudo ldconfig
```

Great! You should now have `protoc` installed in `/usr/local/bin`, and should be on your `PATH`. Check to make sure it's working correctly:

```shell
protoc --version
```

Now that we have the `protoc` compiler, let's install a USB stick as additional swap memory.

```
cd ..
```

### 3. Install a Memory Drive as Swap for Compiling

In order to succesfully build TensorFlow, your Raspberry Pi needs a little bit more memory to fall back on. Fortunately, this process is pretty straightforward. Grab a USB storage drive that has at least 1GB of memory. I used a flash drive I could live without that carried no important data. That said, we're only going to be using the drive as swap while we compile, so this process shouldn't do too much damage to a relatively new USB drive.

First, put insert your USB drive, and find the `/dev/XXX` path for the device.

```shell
sudo blkid
```

As an example, my drive's path was `/dev/sda1`

Once you've found your device, unmount it by using the `umount` command.

```shell
sudo umount /dev/XXX
```

Then format your device to be swap:

```shell
sudo mkswap /dev/XXX
```

If the previous command outputted an alphanumeric UUID, copy that now. Otherwise, find the UUID by running `blkid` again. Copy the UUID associated with `/dev/XXX`

```shell
sudo blkid
```

Now edit your `/etc/fstab` file to register your swap file. (I'm a Vim guy, but Nano is installed by default)

```shell
sudo nano /etc/fstab
```

On a separate line, enter the following information. Replace the X's with the UUID (without quotes)

```bash
UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX none swap sw,pri=5 0 0
```

Save `/etc/fstab`, exit your text editor, and run the following command:

```shell
sudo swapon -a
```

If you get an error claiming it can't find your UUID, go back and edit `/etc/fstab`. Replace the `UUID=XXX..` bit with the original `/dev/XXX` information.

```shell
sudo nano /etc/fstab
```

```bash
# Replace the UUID with /dev/XXX
/dev/XXX none swap sw,pri=5 0 0
```

Alright! You've got swap! Don't throw out the `/dev/XXX` information yet- you'll need it to remove the device safely later on.

### 4. Build Bazel

To build [Bazel](https://github.com/bazelbuild/bazel), we're going to need to download a zip file containing a distribution archive. Let's do that now and extract it into a new directory called `bazel`:

```shell
wget https://github.com/bazelbuild/bazel/releases/download/0.4.3/bazel-0.4.3-dist.zip
unzip -d bazel bazel-0.4.3-dist.zip
```

Once it's done downloading and extracting, we can move into the directory to make a few changes:

```shell
cd bazel
```

Before building Bazel, we need to set the `javac` maximum heap size for this job, or else we'll get an OutOfMemoryError. To do this, we need to make a small addition to `bazel/scripts/bootstrap/compile.sh`. (Shout-out to @SangManLINUX for [pointing this out.](https://github.com/samjabrahams/tensorflow-on-raspberry-pi/issues/5#issuecomment-210965695).

```shell
nano scripts/bootstrap/compile.sh
```

Move down to line 137, where you'll see the following block of code:

```shell
run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      -d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      -encoding UTF-8 "@${paramfile}"
```

At the end of this block, add in the `-J-Xmx500M` flag, which sets the maximum size of the Java heap to 500 MB:

```
run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      -d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      -encoding UTF-8 "@${paramfile}" -J-Xmx500M
```

Finally, we have to add one thing to `tools/cpp/cc_configure.bzl` - open it up for editing:

```shell
nano tools/cpp/cc_configure.bzl
```

Place the line `return "arm"` around line 133 (at the beginning of the `_get_cpu_value` function):

```shell
...
"""Compute the cpu_value based on the OS name."""
return "arm"
...
```

Now we can build Bazel! _Note: this also takes some time._

```shell
sudo ./compile.sh
```

When the build finishes, you end up with a new binary, `output/bazel`. Copy that to your `/usr/local/bin` directory.

```shell
sudo cp output/bazel /usr/local/bin/bazel
```

To make sure it's working properly, run `bazel` on the command line and verify it prints help text. Note: this may take 15-30 seconds to run, so be patient!

```shell
bazel

Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  mobile-install      Installs targets to mobile devices.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.
```

Move out of the `bazel` directory, and we'll move onto the next step.

```shell
cd ..
```

### 5. Compiling TensorFlow

First things first, clone the TensorFlow repository and move into the newly created directory.

```shell
git clone --recurse-submodules https://github.com/tensorflow/tensorflow.git
cd tensorflow
```

_Note: if you're looking to build to a specific version or commit of TensorFlow (as opposed to the HEAD at master), you should `git checkout` it now._

Once in the directory, we have to write a nifty one-liner that is incredibly important. The next line goes through all files and changes references of 64-bit program implementations (which we don't have access to) to 32-bit implementations. Neat!

```shell
grep -Rl 'lib64' | xargs sed -i 's/lib64/lib/g'
```

Next, we need to delete a particular line in `tensorflow/core/platform/platform.h`. Open up the file in your favorite text editor:

```shell
sudo nano tensorflow/core/platform/platform.h
```

Now, scroll down toward the bottom and delete the following line containing `#define IS_MOBILE_PLATFORM` (around line 48):

```cpp
#elif defined(__arm__)
#define PLATFORM_POSIX
...
#define IS_MOBILE_PLATFORM   <----- DELETE THIS LINE
```

This keeps our Raspberry Pi device (which has an ARM CPU) from being recognized as a mobile device.

Finally, we have to adjust the protocol to access the Numeric JS library- for some reason the Cloudflare security certificates don't work properly over `https`. We'll need to fix this in the Bazel `WORKSPACE` file:

```shell
sudo nano WORKSPACE
```

Around line 283, change `https` to `http`:

```
http_file(
  name = "numericjs_numeric_min_js",
  url = "http://cdnjs.cloudflare.com/ajax/libs/numeric/1.2.6/numeric.min.js",
)
```

Now let's configure the build:

```shell
./configure

Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python
Do you wish to build TensorFlow with Google Cloud Platform support? [y/N] N
Do you wish to build TensorFlow with GPU support? [y/N] N
Do you wish to build TensorFlow with Hadoop File System support? [y/N] N
Please input the desired Python library path to use. Default is [/usr/local/lib/python2.7/dist-packages]
Do you wish to build TensorFlow with OpenCL support? [y/N] N
Do you wish to build TensorFlow with GPU support? [y/N] 
```

_Note: if you want to build for Python 3, specify `/usr/bin/python3` for Python's location and `/usr/local/lib/python3.4/dist-packages` for the Python library path._

Now we can use it to build TensorFlow! **Warning: This takes a really, really long time. Several hours.**

```shell
bazel build -c opt --copt="-mfpu=neon-vfpv4" --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --copt="-fomit-frame-pointer" --local_resources 1024,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package
```

_Note: I toyed around with telling Bazel to use all four cores in the Raspberry Pi, but that seemed to make compiling more prone to completely locking up. This process takes a long time regardless, so I'm sticking with the more reliable options here. If you want to be bold, try using `--local_resources 1024,2.0,1.0` or `--local_resources 1024,4.0,1.0`_

When you wake up the next morning and it's finished compiling, you're in the home stretch! Use the built binary file to create a Python wheel.

```shell
bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

And then install it!

```shell
pip install --user /tmp/tensorflow_pkg/tensorflow-1.0.1-cp27-none-linux_armv7l.whl
```

### 6. Cleaning Up

There's one last bit of house-cleaning we need to do before we're done: remove the USB drive that we've been using as swap.

First, turn off your drive as swap:

```shell
sudo swapoff /dev/XXX
```

Finally, remove the line you wrote in `/etc/fstab` referencing the device

```
sudo nano /etc/fstab
```

Then reboot your Raspberry Pi.

**And you're done!** You deserve a break.

## References

1. [Building TensorFlow for Jetson TK1](http://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html) (an update to this post is available [here](http://cudamusing.blogspot.com/2016/06/tensorflow-08-on-jetson-tk1.html))
2. [Turning a USB Drive into swap](http://askubuntu.com/questions/173676/how-to-make-a-usb-stick-swap-disk)
3. [Safely removing USB swap space](http://askubuntu.com/questions/616437/is-it-safe-to-delete-a-swap-partition-on-a-usb-install)

---

_[Back to top](#building-tensorflow-for-raspberry-pi-a-step-by-step-guide)_
