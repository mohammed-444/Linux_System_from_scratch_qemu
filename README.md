# Linux System run on qemu-system-arm

this project was built on ubuntu Ubuntu 22.04.4 LTS x86_64.
you can build the linux system if you download the above files and do step 1 & 6 only .

## steps 
1. install dependencies for the project
2. build u-boot
3. build kernel
4. generate the needed bin from busybox
5. build rootfilesystem
6. build linux system 
  

## 1. install dependencies for the project
First of all need to install dependencies for u-boot 
 
```
sudo apt -y install gcc-arm-linux-gnueabihf binutils-arm-linux-gnueabihf qemu-system-arm
sudo apt-get -y install bison flex bc libssl-dev make gcc 
```
## 2. u-boot stage 

then go through download source code 

`
git clone https://github.com/u-boot/u-boot.git
`
then go to directory 

` cd u-boot
`

you can find all default config under configs folder 
we are going to search for in defconfig for qemu arm architecture

```
find . -name "*qemu*arm*defconfig*"

```
Set the defconfig 
---
```
make  ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- qemu_arm_defconfig

```

adjust your settings through menuconfig 
---

```
make menuconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```

Then run make to build the binary 
---
`-j` after it put  the number of core to use for the run
```
make -j8 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

```
we can find the the build output in the same directory `u-boot.bin`
---
---
---
---
## 3.Kernel stage 

go to `https://www.kernel.org/` and download the kernel version you want to use as  `filename.tar.xz`
Extract downloaded archives:
```
tar xf filename.tar.xz
# then enter the directory
cd filename
```
you can find deconfig for arm under `./arch/arm/configs/`

Set deconfig for qemu arm to the basic defconfig :
---
`make` will know to use the deconfig under arm architecture 
```
  make defconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

```
change in configuration if needed 
---
```

make menuconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

```

Compile actual kernel
---
takes some time depend on you machine
```

make -j8 zImage  ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-

```



the output zImage you can found under `./arch/arm/boot/zImage`
---
---
---
---
## 3. BusyBox 
------------------------------------------------------------------

Download repo
			
------------------------------------------------------------------

```

git clone git://busybox.net/busybox.git --branch=1_33_0 --depth=1
cd BusyBox

````
you can find deconfig for arm under `./arch/arm/configs/`

Set deconfig for arm :
---

```
  make defconfig ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```
change in configuration 
---
```

make menuconfig

Settings -> Build static binary 	(no shared libraries) 	Enable
Settings -> Cross compiler prefix 	arm-Linux-gnueabihf-
Settings -> Destination path for ‘make install’ 	

make -j8 
make install

```
it will be install under `./_install`
after this step we succeed to build the binary for the rootfs


## 5. build rootfilesystem
What do we want for our minimalistic Linux system? Only a few things:

   - init script – Kernel needs to run something as first process in the system.
   - Busybox – it will contain basic shell and utilities (like cd, cp, ls, echo etc)
  

let's create the folder structure first for rootfs:
---
```
mkdir -pv rootfs/{bin,sbin,etc,proc,sys,usr/{bin,sbin}}

# copy the binary from under the busybox to this folder 
cp -av busybox/_install/* rootfs/

cd rootfs # every thing under it 
```

make the init script 
---
```
# generate a simbolic link to busybox and it will handle the init process for you

ln -sf bin/busybox init 

chmod +x init
```
creat rcS to run in init process
---
it will run to mount the folder to the kernel `/proc` `/sys` and make the special folder in `/dev`
```
mkdir etc/init.d
touch etc/init.d/rcS
chmod +x etc/init.d/rcS
```
```
Add the following entries to etc/init.d/rcS:

#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys

echo /sbin/mdev > /proc/sys/kernel/hotplug 
mdev -s  # -s	"Scan /sys and populate /dev"

echo "you are in user space"

```
That’s all. We will use contents of this directory as the init ram disk so we need to create cpio archive and compress it with gzip:
```
$ cd rootfs
$ find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz
```
## 6. build linux system 

```
 qemu-system-arm -M virt -kernel {your_path}/zImage  -initrd {your_path}/rootfs.cpio.gz -bios {your_path}/u-boot.bin -nographic
```