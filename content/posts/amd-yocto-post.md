---
title: Building Yocto Based Linux Image for AMD Ryzen Embedded v1000 Series SoCs with AWS Agents installed.
date: 2022-10-30T17:25:08-05:00
tags: ["Linux", "Yocto", "OS", "Operating Systems"]
draft: false
---

This guide outlines the steps involved to build a linux distribution using Yocto for AMD Ryzen Embedded v1000 Series SoCs with AWS SDKs, services, and software from the [meta-aws](https://github.com/aws4embeddedlinux/meta-aws) layer.

## Prerequisites:

To follow the steps image build section, you will need:
- **A Supported Linux Distribution**: You should have a reasonably current Linux-based host system. You will have the best results with a recent release of Fedora, openSUSE, Debian, Ubuntu, or CentOS as these releases are frequently tested against the Yocto Project® and officially supported. I’ve tried building this with both Ubuntu 20.04 and Pop_OS! 22.04.

- **Required Packages for the Build Host**:
    - Installing the required packages on a Debian based distribution(Ubuntu, Pop_OS!,etc):
    ```bash
    sudo apt install -y gawk wget git diffstat unzip texinfo gcc \ 
    build-essential chrpath socat cpio python3 python3-pip \ 
    python3-pexpect xz-utils debianutils iputils-ping python3-git \ 
    python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 xterm \ 
    python3-subunit mesa-common-dev
    ```
- Other steps you might need to do:
    - Because the Yocto Project tools rely on the “python” command, you will likely need to alias `python` to `python3`. Edit your `.bashrc` or `.zshrc` file and add:
        - `alias python=python3`
        - restart your terminal or run `source ~/.bashrc || source ~/.zshrc`
        - run `python --version` and checkout if you are running the python3 version

## The Layers we’ll be using:

### Poky reference distribution layer:

- The [poky reference distribution layer](https://layers.openembedded.org/layerindex/branch/master/layer/meta-poky/) is the common base for all yocto base builds.

- You will want your poky layer branch to match the branches of all other third-party layers you download. You can view the available poky release names here: https://wiki.yoctoproject.org/wiki/Releases. We’re going to have all layers be on the `kirkstone` branch.

### The meta-openembedded Layer:

- This layer is going to help us to get features like networking , python, a filesystem, a desktop environment etc. below are the list of layers in [meta-openembedded](http://cgit.openembedded.org/meta-openembedded/tree/):
```bash
❯ git clone -b kirkstone git://git.openembedded.org/meta-openembedded
❯ cd meta-openembedded && ll
total 52K
drwxr-xr-x  2 vasuper domain^users 4.0K Sep 27 08:27 contrib
-rw-r--r--  1 vasuper domain^users 1.1K Sep 27 08:27 COPYING.MIT
drwxr-xr-x  6 vasuper domain^users 4.0K Sep 27 08:27 meta-filesystems
drwxr-xr-x 10 vasuper domain^users 4.0K Sep 27 08:27 meta-gnome
drwxr-xr-x  8 vasuper domain^users 4.0K Sep 27 08:27 meta-initramfs
drwxr-xr-x 10 vasuper domain^users 4.0K Sep 27 08:27 meta-multimedia
drwxr-xr-x 17 vasuper domain^users 4.0K Sep 27 08:27 meta-networking
drwxr-xr-x 25 vasuper domain^users 4.0K Sep 27 08:27 meta-oe
drwxr-xr-x  5 vasuper domain^users 4.0K Sep 27 08:27 meta-perl
drwxr-xr-x  9 vasuper domain^users 4.0K Sep 27 08:27 meta-python
drwxr-xr-x  9 vasuper domain^users 4.0K Sep 27 08:27 meta-webserver
drwxr-xr-x 13 vasuper domain^users 4.0K Sep 27 08:27 meta-xfce
-rw-r--r--  1 vasuper domain^users  297 Sep 27 08:27 README
```

- We are going to be using only specific layers from the meta-openembedded layer as we don’t need all these yet. We’ll be using(the layers we choose here are detemined from reading the required dependencies from the layers used above this like meta-amd and meta-aws):
    - **meta-networking**
    - **meta-python**
    - **meta-filesystem**
    - **meta-oe**

### The meta-amd Layer:
- The [meta-amd](https://git.yoctoproject.org/meta-amd/tree/) layer contains Board Support Packages (and a distro) for ONLY selected AMD x86 boards. The list of supported features on each of these supported boards is available here: https://git.yoctoproject.org/cgit/cgit.cgi/meta-amd/tree/FEATURES.md.

- You can learn about the amd architectures that are supported in this path `root/meta-amd-bsp/conf/machine` in the `meta-amd` layer.

### The meta-aws Layer:

- [meta-aws](https://github.com/aws4embeddedlinux/meta-aws) adds useful aws bits for AWS IoT:
    - awscli
    - aws iot device client
    - aws iot GGv2
    - aws sdk for python
    - aws iot sdk for python
    - aws cloudwatch publisher
    - aws firecracker and more......

### Other layers that you might need or can add:
- [**meta-rtlwifi**](https://layers.openembedded.org/layerindex/branch/master/layer/meta-rtlwifi/) for realtek wifi cards
- [**meta-virtualization**](https://layers.openembedded.org/layerindex/branch/master/layer/meta-virtualization/) for building Xen, KVM, Libvirt, and associated packages necessary for constructing OE-based virtualized solutions.
- [**meta-dpdk**](https://layers.openembedded.org/layerindex/branch/master/layer/meta-dpdk/): Support layer for DPDK - a set of libraries and drivers for fast packet processing

## Setup:

Make sure you have the build dependencies installed from the [prerequisites section](#prerequisites) mentioned above.

Select the yocto branch (we’ll use the latest long term supported branch `kirkstone`)

```bash
$ YOCTO_BRANCH="kirkstone"
```

### Clone the git repos:

```bash
$ git clone --single-branch --branch "${YOCTO_BRANCH}" \
    "git://git.yoctoproject.org/poky" "poky-amd-${YOCTO_BRANCH}"
$ cd poky-amd-${YOCTO_BRANCH}
$ git clone --single-branch --branch "${YOCTO_BRANCH}" \
    "git://git.openembedded.org/meta-openembedded"
$ git clone --single-branch --branch "${YOCTO_BRANCH}" \
    "git://git.yoctoproject.org/meta-dpdk"
$  git clone --single-branch --branch "${YOCTO_BRANCH}" \
    "https://github.com/aws4embeddedlinux/meta-aws"
$ git clone --single-branch --branch "${YOCTO_BRANCH}" \
    "git://git.yoctoproject.org/meta-amd"
```

## Build:

### Select a target machine:

You can find the supported platform types under the link below or the by traversing to the same path in the meta-amd-bsp git repo locally.

[https://git.yoctoproject.org/meta-amd/tree/meta-amd-bsp/conf/machine?h=kirkstone](https://git.yoctoproject.org/meta-amd/tree/meta-amd-bsp/conf/machine?h=kirkstone)

Here we'll be using `v1000`.

This might support boards like too, but are not tested yet:
- AMD Ryzen Embedded V1000 (V-Series APU)
- AMD Ryzen Embedded R1000 (R-Series APU)
- AMD EPYC Embedded E3000 (E-Series CPU)

You might ask, what are these machine codes? These are the code names AMD uses for their CPU architecture and each code distinguishes between stuff like Zen, Zen+, EPYC(milan/rome), Ryzen Embedded(v1000)etc, and for yocto we’d need to define these as part of the build.

### Setup the build environment and build for the selected machine:

- Source the **oe-init-build-env** script:

```bash
$ source ./oe-init-build-env build-amd-v1000-awsiot-kirkstone
```

- Replace or edit accordingly the local.conf file from the one seen in this repo: https://github.com/Vasu77df/v1000-AMD-AWS-Yocto-Image 

- add the required layers to the build configuration:
```bash 
$ bitbake-layers add-layer ../meta-openembedded/meta-oe
$ bitbake-layers add-layer ../meta-openembedded/meta-python
$ bitbake-layers add-layer ../meta-openembedded/meta-networking
$ bitbake-layers add-layer ../meta-dpdk
$ bitbake-layers add-layer ../meta-aws
$ bitbake-layers add-layer ../meta-amd/meta-amd-distro
$ bitbake-layers add-layer ../meta-amd/meta-amd-bsp
```
- Start the build, by building one of the supported images `core-image-base` or  `core-image-sato` . More on image types here:
    - [https://docs.yoctoproject.org/ref-manual/images.html#images](https://docs.yoctoproject.org/ref-manual/images.html#images)
    - I selected `core-image-sato` that uncludes x11 with the sato theme.
```
bitbake core-image-sato -k
```
- the `k || --continue` flag will continue as much as possible after an error. While the target that failed and anything depending on it cannot be built, as much as possible will be built before stopping.

## Deploying an image to the target

After building an image following the section above, we can deploy it to the
target machine using a USB Flash Drive or a CD/DVD. The built images can be
found in the `<build-dir>/tmp/deploy/images/<machine-name>` directory to
which we will refer to as the **"Image Deploy Directory"** in this doc.

**Note**:
- Change the `<machine-name>` and `<image-name>` placeholders in the
following instructions according to the selected machine and the image 
built in following the section above.

This directory contains `.wic` and `.iso` images for USB and CD/DVD
respectively. Follow the instructions below to make a bootable
USB Flash Drive or a CD/DVD by writing/burning the image to it:

### Deploy using a USB Flash Drive

We can use **bmaptool** (from *bmap-tools* package) or **dd** to write
the `<image-name>-<machine-name>.wic` image located in the
Image Deploy Directory to a USB Flash Drive:

##### Using bmaptool *(recommended)*
```bash
$ sudo bmaptool copy <image-name>-<machine-name>.wic /dev/<dev-node>
```

###### where `<dev-node>` is to be replaced with the device node of the USB Flash Drive.
###### (e.g. `sda`, `sdb` or `sdc` etc.)

### Booting the target

Insert the bootable USB or CD/DVD (created in above steps) into the
target machine and power ON the machine.

The grub boot menu should appear at this point where you will see
options to `boot` or `install` this image:

* Select the `boot` option to boot up the target machine.

* Select the `install` option to install the image onto the target
machine's hard drive. Follow the instructions there to complete the
installation process, and reboot the machine and boot from the
hard drive you selected during the installation process.

You will be presented with a console (serial or graphical) or a
graphical user interface depending on the image and the target machine.

If your t640 has a serial port, you can watch the boot messages. Eventually Linux will start a console on this device (i.e. the serial port). You can use a terminal emulator (such as`minicom`) to monitor the messages coming out of the serial port of your t640 on your host build machine.In my case I plugged in a serial-to-USB dongle into the t640 and my Linux host. On my Linux host the USB console device appears at `/dev/ttyUSB2`. Using `/dev/ttyUSB2` as the port in `minicom`, and setting the communication parameters to `115200 8N1`.
