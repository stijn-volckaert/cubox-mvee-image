# Installing an IP-MON-capable OS on the SolidRun CuBox-i

## Preparation

You will need:
* A microSD card reader
* An x86 pc with the ARM cross-compilation toolchains installed (On Ubuntu: `sudo apt-get install gcc-arm-linux-gnu-gnueabi`)

## Installing the base image

Download the latest CuBox-i OS image here:
https://images.solid-build.xyz/IMX6/Debian/

Or simply:
```
wget https://images.solid-build.xyz/IMX6/Debian/sr-imx6-debian-jessie-cli-20171108.img.xz
```

Now look up the device name for the microSD card. You do *NOT* have to mount the card to do this. Instead, simply look up the device using `sudo gparted` or fdisk. On my pc, the device name is /dev/sdc/

Now, flash the image onto the card using:
```
# replace /dev/sdX by the device name for your microSD card.
# make sure you double check the device name as this will erase the contents of the device.
xzcat /path/to/image/sr-imx6-debian-jessie-cli-20171108.img.xz | dd of=/dev/sdX
```

Replace X by the device number.

## Mounting the microSD card

Now, mount the microSD card into your file system. This time, you will need the partition number too:
```
mkdir -p /path/to/sdcard/
# replace sdXX by the partition name. If your device name was /dev/sdc, 
# then the partition name is probably /dev/sdc1. Once again, use gparted or fdisk to double check
mount -t ext4 /dev/sdXX /path/to/sdcard/
```

## Getting the updated kernel sources and preparing for cross-compilation

Get the latest CuBox-i kernel sources directly from GitHub using:
```
git clone https://github.com/SolidRun/linux-fslc
```

Next, set up a setupenv.sh script that looks like this:

```
export ARCH="arm"
export CROSS_COMPILE="/usr/bin/arm-linux-gnueabi-"
export INSTALL_PATH="/path/to/sdcard/boot"
export INSTALL_MOD_PATH="/path/to/sdcard"
```

INSTALL_MOD_PATH should be the root of the microSD card's file system.
INSTALL_PATH should be the boot folder within that root file system.

Next, open arch/arm/boot/install.sh and comment out the INSTALLKERNEL invocations.
In my script, the lines I commented out were:
```
# User may have a custom install script
if [ -x ~/bin/${INSTALLKERNEL} ]; then exec ~/bin/${INSTALLKERNEL} "$@"; fi
if [ -x /sbin/${INSTALLKERNEL} ]; then exec /sbin/${INSTALLKERNEL} "$@"; fi
```

We need to comment these lines out because otherwise your distro will try to package up the kernel we're going to compile.
It might also try to update your boot loader.
We don't want any of that because we're going to install the kernel directly onto the microSD card.

## Compiling the kernel

Now, go into the kernel source folder and do the following:
```
# This should be the path to the setupenv.sh file you prepared in the previous step.
# Note that you have to use source to execute the script.
# Otherwise, the exported environment variables will not persist
source ../setupenv.sh
make imx_v7_cbi_hb_defconfig
make -j 8 zImage imx6q-cubox-i.dtb imx6dl-cubox-i.dtb imx6dl-hummingboard.dtb imx6q-hummingboard.dtb
make -j 8 modules
```

## Installing the kernel

I recommend doing this in a sudo shell using:
```
sudo su -
```

Now, go back to your kernel sources folder and re-source the setupenv.sh script:
```
source ../setupenv.sh
```

Now, you can install the zImage using:
```
make zinstall
make modules_install
make dtbs_install
```

These commands install the following files:
* /path/to/sdcard/boot/vmlinux-<kernelver>
* /path/to/sdcard/boot/config-<kernelver>
* /path/to/sdcard/boot/System.map-<kernelver>
* /path/to/sdcard/boot/dtbs/<kernelver>
* /path/to/sdcard/lib/modules/*
* /path/to/sdcard/lib/firmware/*

## Rebuilding the initrd

Now, we need to rebuild the initrd, which is used to load the fs drivers and mount the root file system.
We begin by grabbing the existing initrd and copying into a tmp folder:

```
mkdir -p /path/to/tmpinitrd/
cp /path/to/sdcard/boot/initrd /path/to/tmpinitrd/
cd /path/to/tmpinitrd/
```

The initrd file should be a gzip-compressed ASCII cpio archive.
Verify this using:
```
file initrd
```

The output should look like:
```
# file initrd 
initrd: gzip compressed data, from Unix, last modified: Tue Feb 6 14:09:47 2018
```

Unfortunately, gunzip refuses to unpack anything without a .gz extension, so we have to rename the file:
```
mv initrd initrd.gz
```

Now, unpack the file:
```
gunzip initrd.gz
```

Next, verify that it is indeed a cpio archive:
```
# file initrd 
initrd: ASCII cpio archive (SVR4 with no CRC)
```

OK! Time to unpack:
```
cpio -id < initrd
```

Now we can delete the old file, and the old drivers included in the initrd.
```
rm initrd
rm -rf lib/firmware lib/modules
```

Next, grab the new drivers and copy them into the initrd:
```
cp -R /path/to/sdcard/lib/modules lib/
cp -R /path/to/sdcard/lib/firmware lib/
```

Now, we can create the new initrd:
```
find . | cpio --create --format='newc' > ../newinitrd
cd ..
gzip newinitrd
```

Finally, copy the new initrd to the microSD card:
```
cp newinitrd.gz /path/to/sdcard/boot/initrd.img-<kernelver>
```

## Setting up the symlinks

All that remains now is to set up the new symlinks on the sdcard:
```
cd /path/to/sdcard/boot/
unlink zImage 
unlink initrd 
unlink dtb 
unlink dtb-dir
ln -s vmlinuz-<kernelver> zImage
ln -s initrd.img-<kernelver> initrd
ln -s dtbs/<kernelver> dtb
ln -s dtbs/<kernelver> dtb-dir
```

## And finally

Now, just unmount the sdcard, pop it into the CuBox-i and everything should just work!

