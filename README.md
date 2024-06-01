# Netboot Imager Tutorial
A tutorial about how you can create a tiny Linux distro to image computers of a fog using iventoy netbooting.

## What?

Imagine that you have a "mother" PC and you want to clone its operating system on many other computers. One of the basic options which you have is to just boot everything else using a USB disk and use software like [Partclone](https://partclone.org/download/) or [Clonezilla](https://clonezilla.org/) in order to do mass cloning.

But what if you wanted more control on what you are doing? Or maybe you want to just experience it yourself. In this case, you can use this tutorial to create a versatile bootable ISO from scratch (or [Alpine Linux](https://www.alpinelinux.org/)) in order to do it yourself.

At last, we will use [iventoy](https://www.iventoy.com) in order to mass deploy our ISO to multiple computers and image them.

## Steps

### The Kernel

The first step to create a bootable ISO is to get the kernel file. In this case you have two options:

1. Compile the kernel yourself
2. Get it from a distro

#### Compiling the Kernel

In this case, at first, grab the latest kernel version from [kernel.org](https://www.kernel.org/). The use the following command to install the required dependencies for compiling the Linux kernel:

```bash
apt install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

Now you have two options which are explained below:

1. Use the default configuration which the Linux gives you
2. Use your distro's configuration file

Compile your kernel with `make -j $(nproc)`. In the end, the compiled Linux kernel is located under `arch/x86/boot/bzImage` and the kernel module files are located in the corresponding directory in the source code tree. They have `.ko` extension.

You can also set `INSTALL_MOD_PATH` environment variable before executing `make modules_install` to gather all modules files in a specific directory.

##### Default Configuration

You can get the default configuration by just using the following command in the kernel's source code directory:

```bash
make defconfig
```

This will replace your configuration with a default and minimal Linux configuration. This configuration is sufficient to run your operating system under a VM. However, you might and probably will run into a lot of driver issues as soon as you try to run this kernel under real hardware.

One easy way to find out which drivers (and modules) are needed under your machine is to run a distro such as Ubuntu or Alpine from a USB stick and use `lsmod` command to list every loaded module. Then you can go to kernel configuration and simply enable every single module which your distro was using.

You can either enable drivers as modules or simply embed them inside your kernel. If you are building drivers as modules you need to load them inside your `init` script. I'll talk about this in the next section.

> [!IMPORTANT]
> You probably want to always enable the "packet socket" option (CONFIG_PACKET) in the kernel's configuration. This will enable you to use DHCP inside the operating system. You can either mark this as a compiled module or directly build it inside the kernel.

> [!WARNING]  
> Some of the drivers might simply NOT work if they are embedded in your kernel. One instance that I've personally encountered is the `r8169` driver which is labeled as "Realtek 8169/8168/8101/8125 ethernet support" under Linux configuration (CONFIG_R8169). The OS simply did not recognized my ethernet driver unless the module was loaded manually. I personally recommend avoiding built-in hardware drivers as much as you can.

##### Using Your Distro's Configuration

Another option which I recommend is using a distribution's configuration file. This configuration file is located under `/boot` directory and usually starts with `config-`. This file must be copied to Linux's source code tree with the name of `.config`. After copying this file, make sure to run `make menuconfig` to fill the missing configurations with default values.You can also double check every module, driver and features.

> [!TIP]
> You might run into an error such as `*** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.`. In this case, open the `.config` file using a text editor and search for values containing `.pem` file. Just empty those values and try to make the kernel again.

#### Using a Distro's Kernel

This is a simpler way to get started. You just need to get the ISO of a distro. I personally recommend [Alpine Linux](https://www.alpinelinux.org/downloads/) standard edition ISO. As you probably know, there are two very main components when you are booting Linux: The kernel and initramfs. In most ISO files like Alpine, the kernel is located at `/boot/vmlinuz` and initramfs is located as `/boot/initramfs`. Names may vary; for instance, `/boot/bzImage` is also another common name for Linux kernel. You can check the files using the `file` command.

However, one very important note that you should take is that you HAVE TO use the compiled kernel modules included in your distribution and you CANNOT compile them yourself because of the mismatch between the compilers and the kernel version. In alpine the kernel modules are located in the `/boot/modloop-lts` file. This file can be extracted using 7z.

### Busybox

### Loading Modules

### Imager

### Making the ISO

### iventoy