---
title: Linux Kernel Compile And Debug 
date: 2023-04-07 18:32:00 +0800
categories: [Blogging, Learning]
tags: [OS, linux]
---

> For linux v4.19 as an example, also works for other versions.

```bash
git clone https://github.com/torvalds/linux
git checkout -b v4.19-dev v4.19
```

## Preparation

- `sudo apt install libncurses5-dev libssl-dev bison flex libelf-dev gcc make openssl libc6-dev`
- note: should use gcc-8 (or older gcc) for compiling `v4.19`, or else you may need to add some compile options to disable some features.

## Compile the kernel

### Configurations: `make menuconfig`

- turn on:
  - Kernel hacking -> Kernel debugging
  - Kernel hacking -> KGDB:kernel debugger
  - Kernel hacking -> Compile time checks and compiler options -> Provide GDB scripts for kernel debugging
- turn off:
  - Kernel hacking -> Compile time checks and compiler options -> Reduce debugging information

> for linux v5.x, you need to turn on `Kernel hacking -> Compile-time checks and compiler options -> Debug information (Generate DWARF Version 5 debuginfo) -> DWARF Version 5 debuginfo` to enable debug info.

### Compiling

Run `make -j$(nproc)`.

After about 30 minutes, the kernel will be compiled. The `vmlinux` file will be generated in the current directory, which is the compiled raw kernel file, and `arch/x86/boot/bzImage` is the compressed kernel file.

### ERRORs

- [New address family defined, please update secclass_map.](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=760f8522ce08)
- [error: ‘-mindirect-branch’ and ‘-fcf-protection’ are not compatible](https://gcc.gnu.org/onlinedocs/gcc-9.3.0/gcc/Instrumentation-Options.html#Instrumentation-Options) use gcc-8 or older gcc; or you can add `MITIGATION_CFLAGS += -fcf-protection=none` to turn off this feature.
- thunk_64.o: warning: objtool: missing symbol table --> add `CONFIG_PREEMPT=y`.
- No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list':
  - find `CONFIG_SYSTEM_TRUSTED_KEYS` and set `CONFIG_SYSTEM_TRUSTED_KEYS=""`
  - OPT: find `CONFIG_SYSTEM_REVOCATION_KEYS` and set `CONFIG_SYSTEM_REVOCATION_KEYS=""`.

## make rootfs

In Linux, the filesystem provides a structural way to store and access our files from the system storage. Linux follows the Filesystem Hierarchy Standard (FHS) as a reference. The FHS defines the structure of the filesystem. In other words, it defines the main directories and their contents.

The root filesystem is at the top of the hierarchical file tree (also known as ‘/’). The Linux kernel directly mounts rootfs through the configuration argument ‘root=‘. The root filesystem also has mount points where we can mount other filesystems as well in order to connect them to this filesystem hierarchy. It has a number of directories containing files critical for booting the system and system operations.

### compile -m32 on ubuntu

```bash
sudo apt -y install build-essential binutils
sudo apt install gcc-multilib

sudo ln -s /usr/include/asm-generic/ /usr/include/asm
```

### Build an initrd-based rootfs

First create a simple c program as the init program.

```c
#include <stdio.h>
int main()
{
    printf("hello world!\n");
    printf("hello linux!\n");
    fflush(stdout);
    while(1);
    return 0;
}
```

```sh
gcc --static -o fakeinit fakeinit.c -m32
# use cpio to create initrd_rootfs.img
echo fakeinit | cpio -o --format=newc > initrd_rootfs.img
```

### Build a rootfs with busybox

We can use busybox to build a rootfs. Busybox is a single piece of software that integrates hundreds of common Linux commands and tools, which is very convenient when testing the kernel.

#### Download and compile busybox

First, download the [busybox source code](https://busybox.net/downloads/) and unzip it.

```sh
tar -xvf busybox-1.32.1.tar.bz2
cd busybox-1.32.1
make menuconfig
```

Make sure the following options are selected:

- Settings -> Build Options -> Build static binary (no shared libs)

If you need to compile 32-bit programs, you need to add `-m32` to the following options:

- Settings -> Build Options -> Additional CFLAGS
- Settings -> Build Options -> Additional LDFLAGS

Then, run `sudo make install CONFIG_PREFIX=../_install_32` to compile for 32-bit.

Or run `sudo make install  CONFIG_PREFIX=../_install` to compile for 64-bit.

#### Creat rootfs with busybox

First create an empty disk image file:

```sh
dd if=/dev/zero of=./busybox_rootfs_ext3_m64.img bs=1M count=10
mkfs.ext3 ./busybox_rootfs_ext3_m64.img
mkdir rootfs
sudo mount -t ext3 -o loop ./busybox_rootfs_ext3_m64.img ./rootfs
```

Then copy the busybox files to the rootfs:

```sh
sudo cp -r _install/* rootfs
# for 32-bit, run:
# sudo cp -r _install_32/* rootfs

# or make install CONFIG_PREFIX=/path/to/rootfs/
```

Finally, configure busybox init and uninstall rootfs.
  
```sh
mkdir rootfs/proc
mkdir rootfs/dev
mkdir rootfs/etc
sudo cp -r busybox-1.32.1/examples/bootfloppy/etc/* rootfs/etc/
sudo umount rootfs
```

### Build a rootfs with Ubuntu

You can also build a rootfs with Ubuntu/Debian/Archlinux and any other Linux distributions.

#### Download and extract the rootfs

I choose to use [Ubuntu Base 22.04.2 LTS](https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/) as the rootfs.

```sh
cd rootfs/by_disk_image
wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04/release/ubuntu-base-22.04-base-amd64.tar.gz

dd if=/dev/zero of=./ubuntu_rootfs_ext4_m64.img bs=1M count=500
mkfs.ext4 ubuntu_rootfs_ext4_m64.img
mkdir rootfs_mount
sudo mount -t ext4 -o loop ./ubuntu_rootfs_ext4_m64.img ./rootfs_mount
sudo tar -xpf ubuntu-base-22.04-base-amd64.tar.gz -C rootfs_mount
sudo umount rootfs_mount
```

#### Configure the rootfs

```sh
# mount the rootfs
sudo mount -t ext4 -o loop ./ubuntu_rootfs_ext4_m64.img ./rootfs_mount
sudo mount -v --bind /dev rootfs_mount/dev
sudo mount -vt devpts devpts rootfs_mount/dev/pts -o gid=5,mode=620
sudo mount -vt sysfs sysfs rootfs_mount/sys
sudo mount -vt proc proc rootfs_mount/proc
sudo mount -vt tmpfs tmpfs rootfs_mount/run
sudo cp /etc/resolv.conf rootfs_mount/etc/resolv.conf
# chroot into the rootfs
sudo chroot rootfs_mount
# configure the rootfs
passwd root

apt update
apt install -y vim # or other packages

# default shell is dash, change it to bash
rm /bin/sh
ln -s /bin/bash /bin/sh
# use systemd as the init program
apt install -y systemd
ln -s /lib/systemd/systemd /sbin/init

exit
# umount the rootfs
sudo umount rootfs_mount/run
sudo umount rootfs_mount/proc
sudo umount rootfs_mount/sys
sudo umount rootfs_mount/dev/pts
sudo umount rootfs_mount/dev
sudo umount rootfs_mount

# check & resize the rootfs
e2fsck -p -f ubuntu_rootfs_ext4_m64.img
resize2fs  -M ubuntu_rootfs_ext4_m64.img
```

## Debug the kernel

### Boot the kernel with Qemu

Go to the linux source code root directory,

For ubuntu_rootfs_ext4_m64.img:

```sh
qemu-system-x86_64 \
 -kernel ./arch/x86/boot/bzImage \
 -hda  ../rootfs/by_disk_image/ubuntu_rootfs_ext4_m64.img \
 -serial stdio \
 -append "root=/dev/sda rw console=ttyS0 nokaslr"
```

If you are following the above steps, you can successfully boot the kernel and login to the rootfs.

If your init program is not systemd but `/bin/sh`, you might get a kernel panic. The reason is that `/bin/sh` can not work with `nokaslr` parameter. You can remove `nokaslr` parameter and boot the kernel again.

For busybox_rootfs_ext3_m64.img:

```sh
qemu-system-x86_64 \
 -kernel ./arch/x86/boot/bzImage \ # --> kernel image file
 -hda  ./rootfs/by_disk_image/busybox_rootfs_ext3_m64.img \ # --> rootfs image file
 -serial stdio \ # --> use stdio as the serial port
 -append "root=/dev/sda console=ttyS0 nokaslr" \ # --> kernel parameters
 -s -S # --> enable gdb debug
```

For initrd_rootfs.img:

```sh
qemu-system-x86_64 \
 -kernel ./arch/x86/boot/bzImage \ # --> kernel image file
 -initrd  ../rootfs/by_initrd/initrd_rootfs.img \
 -serial stdio \
 -append "root=/dev/ram rdinit=/fakeinit console=ttyS0 nokaslr"
```

### Connect GDB to Qemu

Note: You need to start qemu with `-s -S` to enable gdb debug.

```sh
gdb ./vmlinux
add-auto-load-safe-path ...path_to_linux
gef-remote --qemu-user localhost 1234 # target remote localhost:1234
b start_kernel
c
```

ref:

- [The Linux Kernel](https://www.kernel.org/doc/html/next/index.html)
- [https://www.sobyte.net/post/2022-02/debug-linux-kernel-with-qemu-and-gdb/](https://www.sobyte.net/post/2022-02/debug-linux-kernel-with-qemu-and-gdb/)
- [https://www.josehu.com/memo/2021/01/02/linux-kernel-build-debug.html](https://www.josehu.com/memo/2021/01/02/linux-kernel-build-debug.html)
