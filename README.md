# Netboot Imager Tutorial
A tutorial about how you can create a tiny Linux distro to image computers of a fog using iventoy netbooting.

## What?

Imagine that you have a "mother" PC and you want to clone its operating system on many other computers. One of the basic options which you have is to just boot everything else using a USB disk and use software like [Partclone](https://partclone.org/download/) or [Clonezilla](https://clonezilla.org/) in order to do mass cloning.

But what if you wanted more control on what you are doing? Or maybe you want to just experience it yourself. In this case, you can use this tutorial to create a versatile bootable ISO from scratch (or [Alpine Linux](https://www.alpinelinux.org/)) in order to do it yourself.

At last, we will use [iventoy](https://www.iventoy.com) in order to mass deploy our ISO to multiple computers and image them.

## Step 1 - The Kernel

The first step to create a bootable ISO is to get the kernel file. In this case you have two options:

1. Compile the kernel yourself
2. Get it from a distro

### Compiling the Kernel

In this case, at first, grab the latest kernel version from [kernel.org](https://www.kernel.org/). The use the following command to install the required dependencies for compiling the Linux kernel:

```bash
apt install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

Now you have two options which are explained below:

1. Use the default configuration which the Linux gives you
2. Use your distro's configuration file

Compile your kernel with `make -j $(nproc)`. In the end, the compiled Linux kernel is located under `arch/x86/boot/bzImage` and the kernel module files are located in the corresponding directory in the source code tree. They have `.ko` extension.

You can also set `INSTALL_MOD_PATH` environment variable before executing `make modules_install` to gather all modules files in a specific directory.

#### Default Configuration

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

#### Using Your Distro's Configuration

Another option which I recommend is using a distribution's configuration file. This configuration file is located under `/boot` directory and usually starts with `config-`. This file must be copied to Linux's source code tree with the name of `.config`. After copying this file, make sure to run `make menuconfig` to fill the missing configurations with default values.You can also double check every module, driver and features.

> [!TIP]
> You might run into an error such as `*** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.`. In this case, open the `.config` file using a text editor and search for values containing `.pem` file. Just empty those values and try to make the kernel again.

### Kernel Modules

One thing which you should probably do when you are compiling the Linux yourself is enabling the modules which are needed to run on the fog. For instance, a network driver might be needed in order to use the LAN. Or a SCSI driver might be needed in order to detect the disk or many other reasons.

In case that you are building based on the default config you have to explicitly enable modules which are needed for your target hardware (such as disk or network drivers). However, if your config file is based on a distro's config file you generally do not need to worry about choosing the modules. You just have to worry about loading them in the boot time (which we will get to later).

In case that the computers are running on low ram, you might want to enable the module compression in the configuration, under the "Enable loadable module support" section. I personally prefer gzip because it is fast and not resource intensive while keeping an acceptable compression ratio.

To export kernel modules in a folder use the following commands:

```bash
export INSTALL_MOD_PATH=/tmp/modules
export INSTALL_MOD_STRIP=1
make modules_install
```

`INSTALL_MOD_PATH` defines the place which the modules will be put while `INSTALL_MOD_STRIP` will strip the modules from the debugging symbols, reducing the size of them.

### Using a Distro's Kernel

This is a simpler way to get started. You just need to get the ISO of a distro. I personally recommend [Alpine Linux](https://www.alpinelinux.org/downloads/) standard edition ISO. As you probably know, there are two very main components when you are booting Linux: The kernel and initramfs. In most ISO files like Alpine, the kernel is located at `/boot/vmlinuz` and initramfs is located as `/boot/initramfs`. Names may vary; for instance, `/boot/bzImage` is also another common name for Linux kernel. You can check the files using the `file` command.

However, one very important note that you should take is that you HAVE TO use the compiled kernel modules included in your distribution and you CANNOT compile them yourself because of the mismatch between the compilers and the kernel version. In alpine the kernel modules are located in the `/boot/modloop-lts` file. This file can be extracted using 7z.

## Step 2 - initramfs

The the other component which is needed to boot Linux beside the kernel is the `initrd` or `initramfs`. This is the file system the Linux kernel sees when it boots up. Usually this file system is very light and only contains a limited set of tools. For us who want to only image the disk, [BusyBox](https://busybox.net/) is completely sufficient because it provides us with tools such as `dd`, `wget` and `udhcpc`.

### Compiling Busybox

The first step to create the initramfs is to compile the BusyBox itself. You have to grab the latest source code from their [website](https://busybox.net/). Then run `make menuconfig` to create a configuration file. In the Settings section, search for "Build static binary (no shared libs)" and enable it. This will create a statically linked executable and enables to create the initramfs without compiling and including glibc.

Then simply execute `make` and then `make install`. Busybox will create a folder named `_install` and copy all files in there.

> [!TIP]
> Use `file busybox` to check if the executable is linked statically.

### /init

`/init` is the first file which the Linux kernel executes. This file does stuff such as mounting system file systems, loading drivers and such. But first before creating the `init` script, in the `_install` folder execute this command to create the required folders for system:

```bash
mkdir dev proc sys
```

With that, your `_install` must look like this:

```
.
├── bin
├── dev
├── proc
├── sbin
├── sys
└── usr
    ├── bin
    └── sbin
```

Now create a file called `init` and make it executable with `chmod +x init`. Put the following content in it:

```bash
#!/bin/sh
# Mount system stuff
mount -t devtmpfs none /dev
mount -t proc none /proc
mount -t sysfs none /sys
# Create the loopback interface
ifconfig lo 127.0.0.1
# Shut up kernel logs
dmesg -n 1
echo "Welcome to Flasher OS!"
# Run shell with cttyhack to enable signals
exec setsid cttyhack /bin/sh
```

In nutshell, this scripts mounts the special file systems such as `/dev`, `/proc` and `/sys` and enables other applications to use them which is quite necessary for your OS to actually function. Next, we create the loopback interface which is needed for networking in general. The `dmesg -n 1` log disables the non fatal kernel logs to be printed to console. The message logs can be accessed again in the console using `dmesg` function. At last, we run `/bin/sh` with `setsid` and `cttyhack`. Both of these commands enables us to use signals and quit from running applications with CTRL + C and such while not killing the `/bin/sh`.

However as you might have guessed, this script only gives you a shell and does not even enable networking. However, I personally think that this is the barebones of all init scripts and everything must be built on top of something like this.

#### Networking

The first thing which is needed is networking. Because you are netbooting, you are probably using a LAN cable. You also have a DHCP server and every guest gets its IP from it. To enable networking with DHCP, at first you need to write a script to handle the DHCP messages. One example can be found at [BusyBox source code](https://github.com/mirror/busybox/blob/master/examples/udhcp/sample.bound). If you have a more predictable environment you can also create one yourself. For example, I used the script which I included in [dhcp-sample.script](https://github.com/HirbodBehnam/Netboot-Imager-Tutorial/blob/master/dhcp-sample.script).

Next, just after configuring the loopback driver run these two lines in order to start the interface and the DHCP client.

```bash
ifconfig eth0 up
udhcpc -i eth0 -s /usr/share/udhcpc/default.script
```

> [!NOTE]
> Make sure to tune the interface name and the location of the script in the `init` script. The script must be executable (with `chmod +x`).

#### Bind Shell

Another feature which I find very useful is a bind shell which can be used to get shell from all remote computers from any other remote computer. However, this feature must only be used __ONLY AND ONLY__ if your network is a local network and __NOTHING__ else can access it. These shells can accessed without username as password with root privileges. Be careful if you enable them.

To enable a bind shell on port 12345 use this line after network initialization and driver loading in `init` script:

```bash
nc -lvnp 12345 -e /bin/sh &
```

#### Loading Modules

#### Imager

#### Full init

A full `init` script can be found at [init-sample](https://github.com/HirbodBehnam/Netboot-Imager-Tutorial/blob/master/init-sample). This is the script which I used to netboot bunch of PCs.

### Packing initramfs

## Step 4 - Making the ISO

## Step 5 - Imaging the Mother

## Step 6 - iventoy

## References

* https://medium.com/@ThyCrow/compiling-the-linux-kernel-and-creating-a-bootable-iso-from-it-6afb8d23ba22
* https://phoenixnap.com/kb/build-linux-kernel
* https://stackoverflow.com/a/51044120/4213397
* https://superuser.com/q/705121/940438
* https://www.hackingtutorials.org/networking/hacking-netcat-part-2-bind-reverse-shells/

## Thanks

* Amirhasan Jafar Abadi
* Amirmahdi Namjoo
* Arman Tahmasebi
* Mehdi Mohammadi
* ArshiA Akhavan
* Kamran Bavar