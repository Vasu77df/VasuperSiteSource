---
title: Build your own Customized Live Debian Distro using Debootstrap
date: 2023-01-02T08:23:29-06:00
tags: ["Linux", "Debian", "OS", "Operating Systems"]
draft: false
---

In this guide, we'll be going over the steps involved to build a LiveCD Debian distribution using debootstrap with both legacy BIOS and UEFI boot support using GRUB2. Why build your own distribution, you ask? 
There are scenarios and use cases that necessitate packages, certificates or other artifacts are already installed in the OS Image that's distributed in the machines that are shipped, this guide will go over the steps involved to build that image.

## Prerequisites

Let's begin by installing the build tools.

```
sudo apt install \
    debootstrap \
    squashfs-tools \
    xorriso \
    isolinux \
    syslinux-efi \
    grub-pc-bin \
    grub-efi-amd64-bin \
    grub-efi-ia32-bin \
    mtools \
    dosfstools \
    parted \
    debian-archive-keyring
```

Create a directory where we will store all of the files we create throughout this guide. 

```
mkdir -p $HOME/LIVE_BOOT
```

## Bootstrap and Configure Debian

 Set up the base Debian environment. I'm using `bullseye` for my distribution and `amd64` for the architecture. Consult the list of [debian mirrors](https://www.debian.org/mirror/list). 
*Please change the URL in this command if there is a mirror that is closer to you.*
```
sudo debootstrap \
    --arch=amd64 \
    --variant=minbase \
    bullseye \
    $HOME/LIVE_BOOT/edit \
    http://ftp.us.debian.org/debian/
```

### Copy resolver settings 
We're copying the DNS resolver settings from the host to the root filesystem of the image we are building so that we can resolve DNS request while within the [chroot](https://en.wikipedia.org/wiki/Chroot) that we'll be pivoting into in the next step.
```
sudo mkdir -p $HOME/LIVE_BOOT/edit/run/systemd/resolve
sudo cp -f /etc/resolv.conf $HOME/LIVE_BOOT/edit/run/systemd/resolve/stub-resolv.conf
```

### Prepare and chroot
Chroot to the Debian environment we just bootstrapped and mounts directories linux expects to handles processes and block devices
```
sudo mount -t proc /proc $HOME/LIVE_BOOT/edit/proc
sudo mount -t sysfs /sys $HOME/LIVE_BOOT/edit/sys
sudo mount -o bind /dev $HOME/LIVE_BOOT/edit/dev
sudo mount -o bind /dev/pts $HOME/LIVE_BOOT/edit/dev/pts
sudo chroot $HOME/LIVE_BOOT/edit /bin/bash
export HOME=/root
export LC_ALL=C
```
****[chroot]**** Set a custom hostname for your Debian environment. 
```
echo "custom-debian-machine" > /etc/hostname
```
****[chroot]**** add hostname to hosts file and update the nameservers

```
cat > /etc/hosts << EOF
127.0.0.1 localhost
127.0.1.1 $(cat /etc/hostname)

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF

cat << EOF > /run/systemd/resolve/stub-resolv.conf 
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF
```

****[chroot]**** Install a Linux kernel of your choosing. I chose the image `linux-image-amd64`. We also install `live-boot` which provides the necessary hooks for LiveCD system. We install `systemd-sysv` (or an equivalent) as it's necessary for an init system. Let's install some other packages we might need in our distribution. You can also add packages to this list to get installed as part of the image you are building, allowing you to further customize the image to include software you want to distribute with your distro.
```
apt update && \
apt install --no-install-recommends \
 linux-image-amd64 \
 live-boot \
 systemd-sysv \
 vim \
 less \
 dnsmasq \
 network-manager \
 net-tools \
 wireless-tools \
 curl \
 openssh-client \
 procps \
 wget \
 gnupg2 \
sudo \
 ca-certificates \
iputils-ping
```

**[chroot]** Installing mosquitto
This is a MQTT agent that facilitates a lot IoT based interactions with cloud-services or your local MQTT based IoT platform. This step can be ignores if you don't have this requirement.

```
wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
apt-key add mosquitto-repo.gpg.key

apt-get update
apt-get install mosquitto
apt-get install mosquitto-clients
```

**[chroot] Downloading the AmazonRootCA.pem**
Cert for various AWS IoT communications, can be ignored if not needed for your use case.

```
wget https://www.amazontrust.com/repository/AmazonRootCA1.pem -P /etc/pki
# check contents with.
openssl x509 -in /etc/pki/AmazonRootCA1.pem -text -noout
```


**[chroot]** Cleaning up

```
apt-get clean
apt-get autoclean
apt-get autoremove
# remove bash history
rm -rf /tmp/* ~/.bash_history
```

**[chroot]** Adding a user and setting  password for root**
Make sure to change this and not use the default.
You can generated salted passwords using this command
```
openssl passwd -6 -salt xyz password
```
```
# password is password
useradd -G sudo -m -U -s /bin/bash -p \
$6$pass$75vTrf3kWE4ncL.vF1TixnLx7qs34pt3tlgm/6uXOhccJXsb9LrY.d3izKxAYnzlRvRwkolgdRlipd.eNrE8g. \ 
user
```

**[chroot]** Exit chroot and un-mount all special directories

```
exit
sudo umount $HOME/LIVE_BOOT/edit/proc
sudo umount $HOME/LIVE_BOOT/edit/sys
sudo umount $HOME/LIVE_BOOT/edit/dev/pts
sudo umount $HOME/LIVE_BOOT/edit/dev
```
## Building the image with GRUB2

* * *
 Create directories that will contain files for our live environment and scratch files. 

```
mkdir -p $HOME/LIVE_BOOT/{scratch,image/live}
```

Compress the chroot environment into a Squash filesystem.

```
sudo mksquashfs \
    $HOME/LIVE_BOOT/edit \
    $HOME/LIVE_BOOT/image/live/filesystem.squashfs \
    -e boot
```

Copy the kernel and initramfs from inside the chroot to the live directory.

```
cp $HOME/LIVE_BOOT/chroot/boot/vmlinuz-* \
    $HOME/LIVE_BOOT/image/vmlinuz && \
cp $HOME/LIVE_BOOT/chroot/boot/initrd.img-* \
    $HOME/LIVE_BOOT/image/initrd
```

Create a menu configuration file for grub.

This config instructs Grub to use the search command to infer which device contains our live environment. This seems like the most portable solution considering the various ways we may write our live environment to bootable media.

```
cat <<'EOF' >$HOME/LIVE_BOOT/scratch/grub.cfg

insmod all_video

search --set=root --file /ZTP_IMAGE

set default="0"
set timeout=30

menuentry "Debian Live" {
    linux /vmlinuz boot=live quiet nomodeset
    initrd /initrd
}
EOF
```

Create a special file in image named `DEBIAN_IMAGE`. This file will be used to help Grub figure out which device contains our live filesystem. This file name must be unique and must match the file name in our grub.cfg config.

```
touch $HOME/LIVE_BOOT/image/DEBIAN_IMAGE 
```

### Adding BIOS and UEFI boot support

Create a grub UEFI image.

```
grub-mkstandalone \
    --format=x86_64-efi \
    --output=$HOME/LIVE_BOOT/scratch/bootx64.efi \
    --locales="" \
    --fonts="" \
    "boot/grub/grub.cfg=$HOME/LIVE_BOOT/scratch/grub.cfg"
```

Create a FAT16 UEFI boot disk image containing the EFI bootloader. Note the use of the `mmd` and `mcopy` commands to copy our UEFI boot loader named `bootx64.efi`


```
(cd $HOME/LIVE_BOOT/scratch && \
    dd if=/dev/zero of=efiboot.img bs=1M count=10 && \
    sudo mkfs.vfat efiboot.img && \
    mmd -i efiboot.img efi efi/boot && \
    mcopy -i efiboot.img ./bootx64.efi ::efi/boot/
)
```

Create a grub BIOS image.

```
grub-mkstandalone \
    --format=i386-pc \
    --output=$HOME/LIVE_BOOT/scratch/core.img \
    --install-modules="linux normal iso9660 biosdisk memdisk search tar ls" \
    --modules="linux normal iso9660 biosdisk search" \
    --locales="" \
    --fonts="" \
    "boot/grub/grub.cfg=$HOME/LIVE_BOOT/scratch/grub.cfg"
```

Use `cat` to combine a bootable Grub `cdboot.img` bootloader with our boot image.


```
cat \
    /usr/lib/grub/i386-pc/cdboot.img \
    $HOME/LIVE_BOOT/scratch/core.img \
> $HOME/LIVE_BOOT/scratch/bios.img
```

Generate the ISO file.


```
xorriso \
    -as mkisofs \
    -iso-level 3 \
    -full-iso9660-filenames \
    -volid "DEBIAN_IMAGE" \
    -eltorito-boot \
        boot/grub/bios.img \
        -no-emul-boot \
        -boot-load-size 4 \
        -boot-info-table \
        --eltorito-catalog boot/grub/boot.cat \
    --grub2-boot-info \
    --grub2-mbr /usr/lib/grub/i386-pc/boot_hybrid.img \
    -eltorito-alt-boot \
        -e EFI/efiboot.img \
        -no-emul-boot \
    -append_partition 2 0xef ${HOME}/LIVE_BOOT/scratch/efiboot.img \
    -output "${HOME}/LIVE_BOOT/custom-debain-distro.iso" \
    -graft-points \
        "${HOME}/LIVE_BOOT/image" \
        /boot/grub/bios.img=$HOME/LIVE_BOOT/scratch/bios.img \
        /EFI/efiboot.img=$HOME/LIVE_BOOT/scratch/efiboot.img
```

## Making the raw disk image

We would need a raw disk image to provide to vendors that they `dd` as a lot of manufacturers expect the image to be in this format for imaging clients on the factory floor

We already have built an iso file. We can use that to create our raw disk image.

### Create a new sparse disk image

```
sudo dd if=/dev/zero of=custom-debian.img bs=1 count=0 seek=400M 
```

### Partition the disk

```
sudo parted -s custom-debian.img mklabel gpt 
sudo parted --script --align=optimal custom-debian.img mkpart ESP fat32 1MiB 100MiB 
sudo parted --script --align=optimal custom-debian.img mkpart ext4 100MiB 100%
sudo parted --script custom-debian.img set 1 boot on
# checkout the partitions
sudo parted -s custom-debian.img print
```

### Check which loop device we can use

```
sudo losetup --show -f custom-debian.img
# example output
#/dev/loop15
```

### Create loop device

```
sudo losetup -P /dev/loop15 custom-debian.img
```

### Write iso to loop device, which writes to raw disk

```
sudo dd if=custom-debian-distro.iso of=/dev/loop15 
```

### Cleanup the loop device

```
sudo losetup -d /dev/loop15
```

now you should have have a bootable raw disk image `custom-debian.img` , you can learn more about the disk running this command

```
fdisk -l custom-debian.img
```

### Some Tips and Tricks if you are building this on an EC2 instance

1. You need to run a `sudo apt update && apt upgrade` before installing the build tools on first EC2 provisioning or as the package has not been setup: 
2. You need to provide your own resolv.conf as copying the resolv.conf from the host machine i.e the EC2 instance points to internal ec2 nameservers and wouldn’t work externally.
3. A better instance type might help in speed of building the squashfs filesystem. The t2.micro takes about a solid  2mins.
4. On an ec2 instance you cannot use the `mkfs.vfat` unless you are root, I updated the command with sudo prepended. A detail to consider.
5. The debian AMI ec2 instance doesn't have `parted` command installed. I've added `parted` to the prereqs build tools.


## References
- https://www.willhaley.com/blog/custom-debian-live-environment-grub-only/ (major thanks to Will Haley for this guide)