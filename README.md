# "Hello world"? We prefer "Hello Bao!"

Welcome to the Bao Hypervisor! Get ready for an interactive journey as we explore the world of Bao
together. Whether you're a experience Bao user or a newcomer, this tour is designed to give you a
practical introduction to the Bao hypervisor.

If you're already familiar with Bao or want to dive into specific setups provided by our team, feel
free to skip ahead to the [Bao demos repository](https://github.com/bao-project/bao-demos).

In this guide, we will take a tour of the different components required to build a setup using the
Bao hypervisor and learn how the different components interact. We share two different setups, one
targeting Arm's aarch64 architecture and another targeting RISC-V's riscv64 architecture. For this
purpose, the guide contains the following topics:

- A **getting started** section that helps users to prepare the environment where we will build the
  target setups. We also provide extra detailed documentation notes regarding some implementation
  aspects of the hypervisor;
- An **initial setup** section that explores the different components of the system and gets the
  first practical example of this guide;
- An **interactive tutorial on changing the guests** running on top of Bao;
- A **practical example** of changing the setup running;
- An example of **how different guests can coexist and interact** with each other;

## 1. Getting Started

In this section, we'll guide you through preparing a development environment to build Bao. Don't
worry; we'll provide you with helpful pointers to Bao's implementation details in case you want to
explore any further.

### 1.1 Recommended Operating System: Linux (e.g., Ubuntu 22.04)
We recommend using a Linux-based operating system to make the most of this tutorial and the Bao
hypervisor. While the instructions may work on other platforms, this guide is set up on top of a
Linux-based machine, specifically Ubuntu 22.04. This will ensure compatibility and an optimal
experience throughout the tour.

### 1.2 Installing Required Dependencies
Before we can dive into the world of Bao, we need to install several dependencies to enable a
seamless experience. Open your terminal and run the following command to install the necessary
packages:

```sh
sudo apt install build-essential bison flex git libssl-dev ninja-build u-boot-tools pandoc \
    libslirp-dev pkg-config libglib2.0-dev libpixman-1-dev gettext-base curl xterm cmake \
    python3-pip
```

This command will install essential tools and libraries required for building and running Bao.
Next, we need to install some Python packages. Execute the following command to do so:

```sh
pip3 install pykwalify packaging pyelftools
```

### 1.3 Download and Setup the Toolchain

#### 1.3.1. Choosing the Right Toolchain

[arm-toolchains]:
  https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
[riscv-toolchains]:
  https://github.com/sifive/freedom-tools/releases

Before we delve deeper, let's ensure you have the right tools at your disposal. We'll guide you
through obtaining and configuring the appropriate cross-compile toolchain for your target
architecture.

|  Architecture  | Toolchain Name       | Download Link                    |
|----------------|:--------------------:|:--------------------------------:|
| Armv8 Aarch64  | aarch64-none-elf-    | [Arm Developer][arm-toolchains]  |
| Armv7/8 Aarch32| arm-none-eabi-       | [Arm Developer][arm-toolchains]  |
| RISC-V         | riscv64-unknown-elf- | [SiFive Tools][riscv-toolchains] |



#### 1.3.2. Installing and Configuring the Toolchain

Download the pre-build binary packages of the appropriate toolchain for your target architecture.
Then, set the **CROSS_COMPILE** environment variable with the reference toolchain prefix path:

```sh
export CROSS_COMPILE=/path/to/toolchain/install/dir/bin/your-toolchain-prefix-
```

### 1.4 Ensuring Enough Free Space

Please note that having sufficient free space is important for a smooth experience, particularly
because of the Linux image that will be built for the Linux guest VM. To prevent any space-related
issues, we suggest having a minimum of **13 GiB of free space** available on your system. With your
environment set up and all the dependencies installed, you are now prepared to explore the Bao
hypervisor.

| Component            | Required Space | Percentage of space Required |
|----------------------|----------------|------------------------------|
| Bao                  | 155.8 MiB      | 1.23%                        |
| Guest (Linux)        | 10.5 GiB       | 84.74%                       |
| Guest (freeRTOS)     | 24.8 MiB       | 0.20%                        |
| Guest (baremetal)    | 4.2 MiB        | 0.03%                        |
| Tools (qemu-aarch64) | 1.7 GiB        | 13.74%                       |
| Tools (qemu-riscv64) | 1.3 GiB        | 10.74%                       |
| Tools (u-boot)       | 237 MiB        | 1.87%                        |
| Tools (ATF)          | 52 MiB         | 0.41%                        |
| Tools (OpenSBI)      | 114 MiB        | 0.92%                        |

---

## 2. Initial setup - Taking the First Steps! {.tabset}

Now that you're geared up, it's time to take the first steps on this tour. In this section, we'll
explore the different components of the system and walk you through a practical example to give you
a solid foundation.

We'll begin by configuring a development environment and establishing a directory tree to hold the
several components needed. Open up your terminal, select the target architecture (`aarch64` or
`riscv64`) through the `PLATFORM` variable and execute the following commands:

> [!IMPORTANT]
> Specific details for each architecture setup are collapsed under each `Arm (aarch64)` and `RISC-V
> riscv64` sections. Open to navigate each step. Generic steps are outside those collapsed
> sections.

<details>
<summary>Arm (aarch64)</summary>
<br>

```sh
export PLAT=aarch64
```
</details>
<br>
<details>
<summary>RISC-V (riscv64)</summary>
<br>

```sh
export PLAT=riscv64
```
</details>
<br>

```sh
export ROOT_DIR=$(realpath .)
export SETUP_BUILD=$ROOT_DIR/bin
export PATCHES_DIR=$ROOT_DIR/patches

export BUILD_GUESTS_DIR=$SETUP_BUILD/guests
export BUILD_BAO_DIR=$SETUP_BUILD/bao
export BUILD_FIRMWARE_DIR=$SETUP_BUILD/firmware

mkdir -p $BUILD_GUESTS_DIR
mkdir -p $BUILD_BAO_DIR
mkdir -p $BUILD_FIRMWARE_DIR
mkdir -p $TOOLS_DIR/bin
```

Upon completing these commands, your directory tree should look like this:

``` sh
├── bin
│   ├── bao
│   ├── firmware
│   └── guests
├── configs
│   ├── aarch64
│   └── riscv64
├── img
│  ├──...
└──README.md
```

### 2.1. Building Your First Bao Guest

[bao-demos-platforms]: https://github.com/bao-project/bao-demos#appendix-i

Let's start the journey of building your first Bao guest. Here, you'll acquire hands-on experience
in creating a Baremetal guest. Before we move on to the practical aspects, let's first understand
the setup we're building. Our goal is to deploy a baremetal guest on top of the Bao hypervisor, as
shown in the figure below:

![Init Setup](/img/baremetal-setup.svg)

> [!NOTE]
> For the sake of simplicity and accessibility, we'll use the QEMU emulator (don't worry, we'll
> guide you through its installation later in the tutorial). However, remember that you can apply
> these steps to various [other platforms][bao-demos-platforms].

To start, let's define an environment variable for the baremetal app source
code:

```sh
export BAREMETAL_SRCS=$ROOT_DIR/baremetal
```

Then, clone the Bao baremetal guest application we've prepared (you can skip
this step if you already have your own baremetal source):
```sh
git clone https://github.com/bao-project/bao-baremetal-guest.git --branch demo $BAREMETAL_SRCS
```

And now, let's compile it (for simplicity, our example includes a Makefile to
compile the baremetal compilation):

```sh
git -C $BAREMETAL_SRCS apply $PATCHES_DIR/baremetal.patch
make -C $BAREMETAL_SRCS PLATFORM=qemu-$(PLAT)-virt
```

Upon completing these steps, you'll find a binary file in the
``BAREMETAL_SRCS`` directory. If you followed our provided Makefile, the binary
takes the name ``baremetal.bin``. Now, move the binary file to your build
directory (``BUILD_GUESTS_DIR``):

```sh
mkdir -p $BUILD_GUESTS_DIR/baremetal-setup
cp $BAREMETAL_SRCS/build/qemu-$(PLAT)-virt/baremetal.bin \
    $BUILD_GUESTS_DIR/baremetal-setup/baremetal.bin
```

### 2.2. Building Bao
Next, we'll guide you through building the Bao Hypervisor. The first step involves configuring the
hypervisor using Bao's configuration file. For this specific setup, we're providing you the
configuration file (`configs/aarch64/baremetal.c` or `configs/riscv64/baremetal.c`) to ease the
process. If you're curious to explore different configuration options, our detailed Bao config
documentation is [here](https://github.com/bao-project/bao-docs/tree/wip/bao-classic_config) to
help.

#### 2.2.1. Cloning the Bao Hypervisor
Your gateway to seamless virtualization begins with cloning the Bao Hypervisor repository. Execute
the following commands in your terminal to initiate this crucial step:

```sh
export BAO_SRCS=$ROOT_DIR/bao
git clone https://github.com/bao-project/bao-hypervisor $BAO_SRCS --branch demo
```

#### 2.2.2. Copying Your Configuration

Now, let's ensure that the configuration is copied to the working directory
with the following commands:

```sh
mkdir -p $mkdir -p $BUILD_BAO_DIR/config
mkdir -p $BUILD_BAO_DIR/config
```

#### 2.2.3. Compiling Bao Hypervisor
We are all set! It's time to bring the Bao Hypervisor to life. Now, we just
need to compile it!

```sh
make -C $BAO_SRCS\
    PLATFORM=qemu-$(PLAT)-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal\
    CONFIG_BUILTIN=y\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

Upon completing these steps, you'll find a binary file in the `BAO_SRCS` directory called
`bao.bin`. Now, let's move the binary file to your build directory:

```sh
cp $BAO_SRCS/bin/qemu-$(PLAT)-virt/baremetal/bao.bin $BUILD_BAO_DIR/bao.bin
```

## 3. Build Firmware - Powering Up Your Setup

Firmware is essential to power your virtual world. That's why we're here to assist you in obtaining
the necessary firmware tailored to your target platform (you can find the instructions to build the
firmware for other platforms
[here](https://github.com/bao-project/bao-demos#b5-build-firmware-and-deploy)).

### 3.1 Getting Started with the QEMU Platform

QEMU provides a convenient alternative to a hardware platform. If you haven't installed it yet,
don't worry. We're here to walk you through the process of building and installing it.

If you already have qemu-system-aarch64 or qemu-system-riscv64, or if you'd prefer to install it
directly using a package manager or another method, just make sure you're working with version
7.2.0 or higher. In that case, feel free to move on to the [next step](#32-now-you-need-u-boot).

To install QEMU, simply run the following commands:

```sh
export QEMU_DIR=$ROOT_DIR/tools/qemu-$(PLAT)
export TOOLS_DIR=$ROOT_DIR/tools/bin
mkdir -p $TOOLS_DIR
git clone https://github.com/qemu/qemu.git $QEMU_DIR --depth 1\
   --branch v7.2.0
cd $QEMU_DIR
./configure --target-list=$(PLAT)-softmmu --enable-slirp
make -j$(nproc)
sudo make install
```

<details>
<summary>Arm (aarch64)</summary>

### 3.2 Installing U-Boot

To get U-Boot up and running, simply execute the following commands:

```sh
export UBOOT_DIR=$ROOT_DIR/tools/u-boot
git clone https://github.com/u-boot/u-boot.git $UBOOT_DIR --depth 1 --branch v2022.10

cd $UBOOT_DIR
make qemu_arm64_defconfig

echo "CONFIG_TFABOOT=y" >> .config
echo "CONFIG_SYS_TEXT_BASE=0x60000000" >> .config

make -j$(nproc)

cp $UBOOT_DIR/u-boot.bin $TOOLS_DIR
```

### 3.3 Let's Build the TrustedFirmware-A

We're almost there! Now, let's go ahead and build TF-A:

```sh
export ATF_DIR=$ROOT_DIR/tools/arm-trusted-firmware
git clone https://github.com/bao-project/arm-trusted-firmware.git $ATF_DIR --branch bao/demo\
   --depth 1
cd $ATF_DIR
make PLAT=qemu bl1 fip BL33=$TOOLS_DIR/u-boot.bin QEMU_USE_GIC_DRIVER=QEMU_GICV3
dd if=$ATF_DIR/build/qemu/release/bl1.bin of=$TOOLS_DIR/flash.bin
dd if=$ATF_DIR/build/qemu/release/fip.bin of=$TOOLS_DIR/flash.bin seek=64 bs=4096 conv=notrunc
```
</details>
<br>
<details>
<summary>RISC-V (riscv64)</summary>

### 3.2 Clone OpenSBI

To get OpenSBI up and running, simply execute the following commands:

```sh
export OPENSBI_DIR=$ROOT_DIR/tools/OpenSBI
git clone https://github.com/bao-project/opensbi.git $OPENSBI_DIR\
    --depth 1 --branch bao/demo
```
</details>
<br>

## 4. Let's Try It Out! - Unleash the Power

Now that everything is set up, let's revise all the steps performed yet:

:white_check_mark: Build guest (baremetal)

:white_check_mark: Build bao hypervisor

:white_check_mark: Build firmware (qemu)

Once everything is in place, we'll proceed with the QEMU launch. This is the command to run:

<details>
<summary>Arm (aarch64)</summary>

```sh
qemu-system-aarch64 -nographic\
   -M virt,secure=on,virtualization=on,gic-version=3 \
   -cpu cortex-a53 -smp 4 -m 4G\
   -bios $TOOLS_DIR/flash.bin \
   -device loader,file="$BUILD_BAO_DIR/bao.bin",addr=0x50000000,force-raw=on\
   -device virtio-net-device,netdev=net0 \
   -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22\
   -device virtio-serial-device -chardev pty,id=serial3 \
   -device virtconsole,chardev=serial3
```

Now, you should see TF-A and U-Boot initialization log. Now, let's set up connections and jump into
Bao. QEMU will reveal the pseudo-terminals that where placed in the virtio serial. Here's an
example:

```sh
char device redirected to /dev/pts/4 (label serial3)
```

To make the connection, open a fresh terminal window and establish a connection to the specified
pseudoterminal. Here's how:

```sh
screen /dev/pts/4
```

Finally, make U-Boot jump to where the bao image was loaded:
```sh
go 0x50000000
```
And you should have an output as follows (video [here](https://asciinema.org/a/613609)):

![baremetal](/img/.gif/aarch64/baremetal.gif)

To exit QEMU hold press `Ctrl-A` then `X`.

</details>

<details>
<summary>RISC-V (riscv64)</summary>

```sh
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

Now, you should see OpenSBI initialization log. Now, let's set up connections and jump into Bao.
Here's an example:

And you should have an output as follows (video
[here](https://asciinema.org/a/620788)):

![baremetal](/img/.gif/riscv64/baremetal_1vCPU.gif)

To exit QEMU hold press `Ctrl-A` then `X`.

</details>
<br>


## 5. Adjusting Your Setup

As we progress, let's focus on fine-tuning your Bao setup. Having the ability to modify your
virtual environment can be quite useful. In the upcoming sections, we'll provide you with detailed,
step-by-step instructions to implement various changes to your guests. By the end of this segment,
you'll have gained a deeper insight into how the different components interact, and you'll be
skilful at making the necessary adjustments to meet your requirements.

### 5.1 Modify the baremetal VM

Let's begin by modifying the baremetal VM. We'll increase the number of vCPUs assigned to the
baremetal, as presented in the following figure:

![Init-mod Setup](/img/baremetal-mod-setup.svg)

#### 5.1.1 Update the VM configuration

Let's start by changing the number of vCPUs assigned to the VM (changes applied in the
corresponding `baremetal_mod.c` configuration file:

```diff
-       .cpu_num = 1,                                       | line 17
+       .cpu_num = 4,                                       | line 17
```
Then, recompile Bao:

```sh
make -C $BAO_SRCS\
    PLATFORM=qemu-$(PLAT)-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal_mod\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR

cp $BAO_SRCS/bin/qemu-$(PLAT)-virt/baremetal/bao.bin $BUILD_BAO_DIR/bao.bin
```

#### 5.1.2 Run the setup

To run this setup just use the following commands:

<details>
<summary>Arm (aarch64)</summary>

```sh
qemu-system-aarch64 -nographic \
  -M virt,secure=on,virtualization=on,gic-version=3 \
  -cpu cortex-a53 -smp 4 -m 4G \
  -bios $TOOLS_DIR/flash.bin \
  -device loader,file="$BUILD_BAO_DIR/bao.bin",addr=0x50000000,force-raw=on \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22 \
  -device virtio-serial-device -chardev pty,id=serial3 \
  -device virtconsole,chardev=serial3
```

</details>
<br>
<details>
<summary>RISC-V (riscv64)</summary>

```sh
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

Now, you should see OpenSBI initialization log. Now, let's set up connections and jump into Bao.
Here's an example:

And you should have an output as follows (video [here](https://asciinema.org/a/620789)):

![baremetal](/img/.gif/riscv64/baremetal_4vCPU.gif)

</details>
<br>

### 5.2 Add a second guest - freeRTOS

In this section, we'll delve into various scenarios and demonstrate how to configure specific
environments using Bao. Let's kick things off by incorporating a second VM running FreeRTOS.

Let's kick things off by incorporating a second VM running FreeRTOS.

![Init Setup](/img/baremetal-freertos-setup.svg)

First, we can use the baremetal compiled from the first setup:

```sh
mkdir -p $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup
cp $BAREMETAL_SRCS/build/qemu-$(PLAT)-virt/baremetal.bin \
    $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup/baremetal.bin
```

#### 5.2.1. Compile freeRTOS
Then, let's compile our new guest:

```sh
export FREERTOS_SRCS=$ROOT_DIR/freertos
export FREERTOS_PARAMS="STD_ADDR_SPACE=y"

git clone --recursive --shallow-submodules https://github.com/bao-project/freertos-over-bao.git\
    $FREERTOS_SRCS --branch demo
git -C $FREERTOS_SRCS apply $PATCHES_DIR/freeRTOS.patch
make -C $FREERTOS_SRCS PLATFORM=qemu-$(PLAT)-virt $FREERTOS_PARAMS
```

Upon completing these steps, you'll find a binary file in the `FREERTOS_SRCS` directory, called
`freertos.bin`. Move the binary file to your build directory (`BUILD_GUESTS_DIR`):

```sh
cp $FREERTOS_SRCS/build/qemu-$(PLAT)-virt/freertos.bin \
    $BUILD_GUESTS_DIR/baremetal-freeRTOS-setup/free-rtos.bin
```

#### 5.2.2. Integrating the new guest

Now, we have both guests compiled and ready for our dual-guest setup. However, there are some steps
required to fit the two VMs on our platform. Let's understand the differences between the
`baremetal.c` and  `baremetal-freeRTOS.c` configuration files.

First of all, we need to add the second VM image:

```diff
- VM_IMAGE(baremetal_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-setup/baremetal.bin));
+ VM_IMAGE(baremetal_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-freeRTOS-setup/baremetal.bin));
+ VM_IMAGE(freertos_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-freeRTOS-setup/freertos.bin));
```

Also, since we now have 2 VMs, we need to change the `vm_list_size` in our configuration:

```diff
- .vmlist_size = 1,
+ .vmlist_size = 2,
```

Next, we need to think about resources. In the first setup, we assigned 4 vCPUs to the baremetal.
But this time, we need to split the vCPUs between the two VMs. For the baremetal, we will use 3
CPUs:

```diff
- .cpu_num = 4,
+ .cpu_num = 3,
```

While for the freeRTOS VM, we will assign only one CPU:
```diff
+ .cpu_num = 1,
```

Additionally, we need to include all the configurations of the second VM.
(Details are omitted for simplicity but you can check further details in the
[configuration file](configs/baremetal-freeRTOS.c)):

```diff
+        {
+            .image = {
+                .base_addr = 0x0,
+                .load_addr = VM_IMAGE_OFFSET(freertos_image),
+                .size = VM_IMAGE_SIZE(freertos_image)
+            },
+
+            ...        // omitted for simplicity
+        },
```

#### 5.2.3. Let's rebuild Bao!

As we've seen, changing the guests includes changing the configuration file. Therefore, we need to
repeat the process of building Bao. Please note that the flag `CONFIG` defines the configuration
file to be used on the compilation of Bao. To compile it, use the following command:

```sh
make -C $BAO_SRCS\
    PLATFORM=qemu-$(PLAT)-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-freeRTOS\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

Upon completing these steps, you'll find a binary file in the `BAO_SRCS` directory, called
`bao.bin`. Move the binary file to your build directory (`BUILD_BAO_DIR`):

```sh
cp $BAO_SRCS/bin/qemu-$(PLAT)-virt/baremetal-freeRTOS/bao.bin \
    $BUILD_BAO_DIR/bao.bin
```

#### 5.2.4. Ready for launch!

Now, you have everything configured for testing your new setup! Just run the following command:

<details>
<summary>Arm (aarch64)</summary>

```sh
qemu-system-aarch64 -nographic \
  -M virt,secure=on,virtualization=on,gic-version=3 \
  -cpu cortex-a53 -smp 4 -m 4G \
  -bios $TOOLS_DIR/flash.bin \
  -device loader,file="$BUILD_BAO_DIR/bao.bin",addr=0x50000000,force-raw=on \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22 \
  -device virtio-serial-device -chardev pty,id=serial3 \
  -device virtconsole,chardev=serial3
```

Finally, make U-Boot jump to where the bao image was loaded:
```sh
go 0x50000000
```

Now, you should have an output as follows (video [here](https://asciinema.org/a/613622)):

![baremetal-freeRTOS](/img/.gif/aarch64/baremetal_freeRTOS.gif)

</details>
<br>
<details>
<summary>RISC-V (riscv64)</summary>

```sh
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
    $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

Now, you should have an output as follows (video
[here](https://asciinema.org/a/620790)):

![baremetal-freeRTOS](/img/.gif/riscv64/baremetal_freeRTOS.gif)

</details>
<br>

### 5.3 Adding Linux to the Mix

Now, let's introduce a third VM running the Linux OS.

![Init Setup](/img/baremetal-linux-setup.svg)

First, we can re-use our guests from the previous setup:

```sh
mkdir -p $BUILD_GUESTS_DIR/baremetal-linux-setup
cp $BAREMETAL_SRCS/build/qemu-aarch64-virt/baremetal.bin \
    $BUILD_GUESTS_DIR/baremetal-linux-setup/baremetal.bin
```

#### 5.3.1 Build Linux Guest

Now let's start by building our linux guest. Setup linux environment variables:

```sh
export LINUX_DIR=$ROOT_DIR/linux
export LINUX_REPO=https://github.com/torvalds/linux.git
export LINUX_VERSION=v6.1

export LINUX_SRCS=$LINUX_DIR/linux-$LINUX_VERSION

mkdir -p $LINUX_DIR/linux-$LINUX_VERSION
mkdir -p $LINUX_DIR/linux-build

git clone $LINUX_REPO $LINUX_SRCS --depth 1 --branch $LINUX_VERSION
cd $LINUX_SRCS
git apply $ROOT_DIR/srcs/patches/$LINUX_VERSION/*.patch
```

Setup an environment variable pointing to the target architecture and platform specific config to
be used by buildroot:

```sh
export LINUX_CFG_FRAG=$(ls $ROOT_DIR/srcs/configs/base.config\
    $ROOT_DIR/srcs/configs/$(PLAT).config\
    $ROOT_DIR/srcs/configs/qemu-$(PLAT)-virt.config 2> /dev/null)
```

Setup buildroot environment variables:
```sh
export BUILDROOT_SRCS=$LINUX_DIR/buildroot-$(PLAT)-$LINUX_VERSION
export BUILDROOT_DEFCFG=$ROOT_DIR/srcs/buildroot/$(PLAT).config
export LINUX_OVERRIDE_SRCDIR=$LINUX_SRCS
```

Clone the latest buildroot at the latest stable version
```sh
git clone https://github.com/buildroot/buildroot.git $BUILDROOT_SRCS --depth 1 --branch 2022.11
cd $BUILDROOT_SRCS
```

Use our provided buildroot defconfig, which points to the a Linux kernel defconfig and patches and
build.

```sh
make defconfig BR2_DEFCONFIG=$BUILDROOT_DEFCFG
make linux-reconfigure all

mv $BUILDROOT_SRCS/output/images/Image\
    $BUILDROOT_SRCS/output/images/Image-qemu-$(PLAT)-virt
```

The device tree for this setup is available in `srcs/devicetrees/qemu-aarch64-virt` or
`srcs/devicetrees/qemu-riscv64-virt`. For a device tree file named `linux.dts` define a environment
variable and build:

```sh
export LINUX_VM=linux
dtc $ROOT_DIR/srcs/devicetrees/qemu-$(PLAT)-virt/$LINUX_VM.dts >\
    $LINUX_DIR/linux-build/$LINUX_VM.dtb
```

Wrap the kernel image and device tree blob in a single binary:

```sh
make -j $(nproc) -C $ROOT_DIR/srcs/lloader\
    ARCH=$(PLAT)\
    IMAGE=$BUILDROOT_SRCS/output/images/Image-qemu-$(PLAT)-virt\
    DTB=$LINUX_DIR/linux-build/$LINUX_VM.dtb\
    TARGET=$LINUX_DIR/linux-build/$LINUX_VM
```

Finaly, copy the binary file to the (compiled) guests folder:

```sh
cp $LINUX_DIR/linux-build/$LINUX_VM.bin \
    $BUILD_GUESTS_DIR/baremetal-linux-setup/linux.bin
```

#### 5.3.2 Welcome our new guest!

After building our new guest, it's time to integrate it into our setup.

First, we need to load our guests:

```diff
- VM_IMAGE(baremetal_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-freeRTOS-setup/baremetal.bin));
+ VM_IMAGE(baremetal_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-linux-setup/baremetal.bin));
+ VM_IMAGE(linux_image, XSTR(BUILD_GUESTS_DIR/guests/baremetal-linux-setup/linux.bin));
```

Let's now update our VM list size to integrate our new guest:

```diff
-    .vmlist_size = 1,
+    .vmlist_size = 2,
```

Then, we need to rearrange the number of vCPUs:

```diff
    // baremetal configuration
    {
-       .cpu_num = 2,
+       .cpu_num = 1,
        ...
    },
    // linux configuration
    {
+       .cpu_num = 3,
    }
```

Additionally, you have the option to configure the Linux VM to integrate various devices and even
memory regions. For specific details regarding this setup, refer to the the [configuration
file](/configs/baremetal-linux.c).

#### 5.3.3. Let's rebuild Bao!

As we've seen, changing the guests includes changing the configuration file. Therefore, we need to
repeat the process of building Bao using the following command:

Compile bao and move the binary file to your build directory (`BUILD_BAO_DIR`):

```sh
make -C $BAO_SRCS\
    PLATFORM=qemu-$(PLAT)-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-linux\
    CONFIG_BUILTIN=y\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR

cp $BAO_SRCS/bin/qemu-$(PLAT)-virt/baremetal-linux/bao.bin $BUILD_BAO_DIR/bao.bin
```

#### 5.3.4. Ready to Launch!

With all the necessary components in place, it's time to launch QEMU and see the results of your
work. Let's proceed:

<details>
<summary>Arm (aarch64)</summary>

```sh
qemu-system-aarch64 -nographic \
  -M virt,secure=on,virtualization=on,gic-version=3 \
  -cpu cortex-a53 -smp 4 -m 4G \
  -bios $TOOLS_DIR/flash.bin \
  -device loader,file="$BUILD_BAO_DIR/bao.bin",addr=0x50000000,force-raw=on \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22 \
  -device virtio-serial-device -chardev pty,id=serial3 \
  -device virtconsole,chardev=serial3
```

To make the connection, open a fresh terminal window and establish a connection to the specified
pseudoterminal. Here's how:

```sh
screen /dev/pts/4
```

> [!WARNING]
> Please be aware that the pts port may vary for each user. To find the correct pts port, kindly
> refer to the qemu output console.

After all, you should see an output as follows (video [here](https://asciinema.org/a/616290)):

![baremetal-linux](/img/.gif/aarch64/baremetal_linux.gif)
</details>
<br>
<details>
<summary>RISC-V (riscv64)</summary>

```sh
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

To make the connection, open a fresh terminal window and establish a connection to the specified
pseudoterminal. Here's how:

```sh
pyserial-miniterm --filter=direct /dev/pts/4
```

> [!WARNING]
> Please be aware that the pts port may vary for each user. To find the correct pts port, kindly
> refer to the qemu output console.

After all, you should see an output as follows (video [here](https://asciinema.org/a/620804)):

![baremetal-linux](/img/.gif/riscv64/baremetal_linux.gif)
</details>
<br>

## 5.4 Facilitating Guest Interaction

In specific scenarios, it's important for guests to establish a communication channel. To achieve
this, we'll make use of a shared memory object and Inter-Process Communication (IPC) mechanisms,
enabling the Linux VM to seamlessly interact with the system.

![Init Setup](/img/baremetal-linux-ipc-setup.svg)

### 5.4.1. Add Shared Memory and IPC to our guest

Let's start by changin our baremetal to handle messages sent via IPC. We've prepared a patch that
can be applied directly to the baremetal source code. To apply the changes, use the following
command:

```sh
git -C $BAREMETAL_SRCS apply $PATCHES_DIR/baremetal_shmem.patch
```

Next, rebuild the baremetal:

```sh
make -C $BAREMETAL_SRCS PLATFORM=qemu-$(PLAT)-virt

mkdir -p $BUILD_GUESTS_DIR/baremetal-linux-shmem-setup
cp $BAREMETAL_SRCS/build/qemu-$(PLAT)-virt/baremetal.bin \
    $BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/baremetal.bin
```

Now, let's integrate an IPC into Linux. For simplicity, the `baremetal-linux-shmem.c` configuration
file already includes the following changes.

```diff
+    bao-ipc@f0000000 {
+        compatible = "bao,ipcshmem";
+        reg = <0x0 0xf0000000 0x0 0x00010000>;
+		read-channel = <0x0 0x2000>;
+		write-channel = <0x2000 0x2000>;
+        interrupts = <0 52 1>;
+		id = <0>;
+    };
```

Now, let's generate the updated device tree:

```sh
export LINUX_VM=linux-shmem
dtc $ROOT_DIR/srcs/devicetrees/qemu-$(PLAT)-virt/$LINUX_VM.dts >\
    $BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM.dtb
```

> [!WARNING]
> To correctly introduce these changes, you need to ensure that you applied the patch to Linux, as
> described [before](#521-build-linux-guest).

Bundle the kernel image and device tree blob into a single binary:

```sh
make -j $(nproc) -C $ROOT_DIR/srcs/lloader\
    ARCH=$(PLAT)\
    IMAGE=$PRE_BUILT_IMGS/guests/baremetal-linux-shmem-setup/Image-qemu-riscv64-virt\
    DTB=$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM.dtb\
    TARGET=$BUILD_GUESTS_DIR/baremetal-linux-shmem-setup/$LINUX_VM
```

### 5.4.2. Rebuild Bao

Given that you've modified one of the guests, it's now essential to rebuild Bao. Thus, compile it
using the following command:

```sh
make -C $BAO_SRCS\
    PLATFORM=qemu-$(PLAT)-virt\
    CONFIG_REPO=$ROOT_DIR/configs\
    CONFIG=baremetal-linux-shmem\
    CPPFLAGS=-DBAO_WRKDIR_IMGS=$BUILD_GUESTS_DIR
```

Upon successful completion, you'll locate a binary file named bao.bin in the ``BAO_SRCS``
directory. Move it to your build directory (``BUILD_BAO_DIR``):

```sh
cp $BAO_SRCS/bin/qemu-$(PLAT)-virt/baremetal-linux/bao.bin $BUILD_BAO_DIR/bao.bin
```

### 5.4.3. Run Our Setup

Now, you're ready to execute the final setup:

<details>
<summary>Arm aarch64</summary>

```sh
qemu-system-aarch64 -nographic \
  -M virt,secure=on,virtualization=on,gic-version=3 \
  -cpu cortex-a53 -smp 4 -m 4G \
  -bios $TOOLS_DIR/flash.bin \
  -device loader,file="$BUILD_BAO_DIR/bao.bin",addr=0x50000000,force-raw=on \
  -device virtio-net-device,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22 \
  -device virtio-serial-device -chardev pty,id=serial3 \
  -device virtconsole,chardev=serial3
```

If all went according to plan, you should be able to spot the IPC on Linux by running the following
command:
```sh
ls /dev
```

You'll see your IPC as follows (video [here](https://asciinema.org/a/616289)):

![baremetal-linux-shmem](/img/.gif/aarch64/baremetal_linux_shmem.gif)
</details>
<br>
<details>
<summary>RISC-V riscv64</summary>

```sh
make -C $TOOLS_DIR/OpenSBI PLATFORM=generic \
    FW_PAYLOAD=y \
    FW_PAYLOAD_FDT_ADDR=0x80100000\
    FW_PAYLOAD_PATH=$BUILD_BAO_DIR/bao.bin

cp $TOOLS_DIR/OpenSBI/build/platform/generic/firmware/fw_payload.elf \
    $TOOLS_DIR/bin/opensbi.elf

qemu-system-riscv64 -nographic\
    -M virt -cpu rv64 -m 4G -smp 4\
    -bios $TOOLS_DIR/bin/opensbi.elf\
    -device virtio-net-device,netdev=net0 \
    -netdev user,id=net0,net=192.168.42.0/24,hostfwd=tcp:127.0.0.1:5555-:22\
    -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```

If all went according to plan, you should be able to spot the IPC on Linux by running the following
command:
```sh
ls /dev
```

You'll see your IPC as follows (video [here](https://asciinema.org/a/620803)):

![baremetal-linux-shmem](/img/.gif/riscv64/baremetal_linux_shmem.gif)
</details>
<br>

From here, you can employ the IPC on Linux to dispatch messages to the Baremetal by writing to
``/dev/baoipc0``:

```sh
echo "Hello, Bao!" > /dev/baoipc0
```
