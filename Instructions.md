## Prerequisite - 
Installing all the packages required for compiling Linux kernel and Busybox
```
apt-get install curl libncurses5-dev gcc make qemu git wget
```
Creating master project folder to keep things clean.
```
mkdir custom-linux
cd custom-linux
```
## Linux kernel - 
Get the source code for the linux and applying patches required - 
```
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.7.5.tar.gz
tar -xvf linux-4.7.5.tar.gz
wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=04e36857d6747e4525e68c4292c081b795b48366 -O patch-1
wget https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=474c90156c8dcc2fa815e6716cc9394d7930cb9c -O patch-1
patch -p 2 < patch-1
patch -p 2 < patch-2
```
Compiling the linux at build-linux directory present at master node.
```
make -C linux-4.7.5 ARCH=x86_64 x86_64_defconfig O=../build-linux 
make -C build-linux O=build-linux
```

## Busybox - 
Downloading Busybox source code and configuring it.
```
wget https://busybox.net/downloads/busybox-1.26.2.tar.bz2
tar -xvf busybox-1.26.2.tar.bz2
make -C busybox-x86 O=../build-busybox defconfig
make -C build-busybox O=build-busybox menuconfig
```
There will be a blue pop up window, follow below procedure to make changes required.
1. Press enter on Busybox Settings
2. Press the down arrow until you hit Build BusyBox as a static binary (no shared libs).
3. Press Y

Now Compiling and installing the Busybox at build-busybox directory present at master node.
```
make -C build-busybox O=build-busybox -j2
make -C build-busybox O=build-busybox install
```

## Generating initramfs -
Creating the rootfs required by creating standard folder structure & copying all the binaries from busy box
```
mkdir -pv initramfs/x86-busybox/{bin,dev,sbin,etc,proc,sys/kernel/debug,usr/{bin,sbin},lib,lib64,mnt/root,root}
cp -av build-busybox/_install/* initramfs/x86-busybox/.
cp -av /dev/{null,console,tty,sda1} initramfs/x86-busybox/dev/.
```
Creating a init shell script,which willrun at startup - 
```
echo "#!/bin/sh \
mount -t proc none /proc \
mount -t sysfs none /sys \
mount -t debugfs none /sys/kernel/debug " >> initramfs/x86-busybox/init
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n" >> initramfs/x86-busybox/init
echo "exec /bin/sh" >> initramfs/x86-busybox/init 
```
Giving executable permission to init shell 
```
chmod +x initramfs/x86-busybox/init
```
Generating rootfs into a unix archive cpio file format
```
find initramfs/x86-busybox/. | cpio -H newc -o > initramfs.cpio
```

## Running qemu to boot up your kernel with busybox
Command to start the qemu with bzimage kernel image and cpio rootfs archive.
```
qemu-system-x86_64 \
    -kernel build-linux/arch/x86/boot/bzImage \
    -initrd initramfs.cpio \
    -nographic -append "earlyprintk=serial,ttyS0 console=ttyS0"
```
#### Note - This is flow is tested with ubuntu 18.04 as base OS. 
