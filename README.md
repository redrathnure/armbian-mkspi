# Unofficial Armbian build for Makerbase [MKS PI](https://github.com/makerbase-mks/MKS-PI) and [MKS SKIPR](https://github.com/makerbase-mks/MKS-SKIPR) and few more boards

TLTR: Unofficial support of Makerbase MKS PI and MKS SKIPR and other boards. Contains `mkspi` board declaration and related kernel and u-boot patches.  
Please note, result Armbian image is not full replacement for mks distributed image. You have to do yourself all OS configuration and and Klipper components installation.  

**WIP, absolutely no guarantees, you do everything at your own risk.**  



## Bit of Liric

The MKS-PI ads look good, especially considering the price and the display (e.g. the size fits the Ghost Flying Bear printer). However, the software and support from the manufacturer is terrible.   

There is no source code (yeah, GPL license, of course), no answer to questions, etc. They provide preconfigured Armbian+Klipper, but the image contains random/outdated components and I have problems with WIFI adapters. And who knows what else is hidden in there.    

So the idea was to build a normal Armbian image using the available information (provided patches, circuitry and information from the native image).   


## Goals

Make Armbian build with `current` and `edge` kernels.     

Please note:
* Klipper, Fluid, and other related components are out of scope of this repo, please refer https://github.com/th33xitus/kiauh if you have interest in these topics.
* original patches were taken from Makerbase repositoy https://github.com/makerbase-mks/armbian-build, this means you have to be mentally ready to see ~~this mess~~ "fast and dirty" approach. I do not have enought knowlage and time to orgonise this mess in right way. 
If you know proper way how to orgonise these patches and right places in armbian sources for them, you are welcome to discussions or just open PRs.
* I do not have access to full MKS PI set and cannot test all features. Main goal is to have worked following component:
  * booting from micro SD
  * MKSPI-TS35 TFT display (better with working touch screen :) )
  * HDMI output
  * USB ports, including USB 3 port
  * ADXL345 (SPI bus)
  * CAN bus for SKIPR board
* If you are interested in any additional features, feel free to step into development? testing or support me by hardware.
* ~~Currently I am focusing only on Ubuntu LTS builds (`current` and `edge` kernels). Feel free to open PRs if you would need non LTS Ubuntu or Debian images.~~ Currently Ubuntu LTS (tested) and Debian Bullseye (non tested) builds are supported.


## Current status

Supported boards:
* [MKS PI](https://github.com/makerbase-mks/MKS-PI) - fully supported
* [MKS SKIPR](https://github.com/makerbase-mks/MKS-SKIPR) - fully supported
* QIDI X-4 and X-6 Mainboards (made by MKS for X-Plus 3 and X-Max 3) - partially supported. See [FreeQIDI](https://openqidi.com/books/build-entire-operating-system-and-klipper-from-scratch/page/freeqidi) and https://github.com/redrathnure/armbian-mkspi/issues/21 for more details.

MKS IPS50 LCD - works starting from `0.3.1-24.2.0-trunk` version.

A `build-essential` and kernel header packages are preinstalled starting from `0.3.2-24.2.0-trunk` version. It should simplify Klipper installation and building custom kernel modules (e.g. WIFI modules like [RTL8188GU (RTL8710B)](https://github.com/McMCCRU/rtl8188gu) one).

Images should be ready for a daily usage. Everything seem to be workes, however please pay attantion:

1. Only Ubuntu builds are actively tested (use Debian ones for your own risk)
2. Some bugs may appear from time to time. Especially it related to edge builds (based on edge 6.x kernel). Please check [release page](https://github.com/redrathnure/armbian-mkspi/releases) for more details.
3. Please double check a [How to Configure CAN Bus](#how-to-configure-can-bus) section if you use [BTT EBB36](https://github.com/bigtreetech/EBB) or similar CAN toolhead


### ADXL345/SPI Usage

TLTR: Do not forget about [klipper_mcu installation](https://www.klipper3d.org/RPi_microcontroller.html).

Full version:

* Step 0: Ensure spidev device exists. e.g. `ls -al /dev/spi*` should show `/dev/spidev0.2` device file
* Step 1: Install Klipper. E.g. 
	```
	cd ~
	git clone https://github.com/th33xitus/kiauh.git
	./kiauh/kiauh.sh
	```
	,  then 1, 1, 1 and so on.
* Step 2: Build and install `klipper_mcu` service
	```
	# Build and install klipper_mcu binary
	cd ~/klipper/
	
	# In the menu, set "Microcontroller Architecture" to "Linux process," then save and exit.
	make menuconfig
	
	# Preapre klipper-mcu service 
	sudo ln -s $PWD/scripts/klipper-mcu.service /etc/systemd/system/
	sudo systemctl daemon-reload
	sudo systemctl enable klipper-mcu.service

	# Stop klipper service and install klipper-mcu binary file
	sudo service klipper stop
	make flash

	# Start klipper_mcu and klipper
	sudo systemctl start klipper-mcu klipper-mcu
	
	# Ensure everything is up and running
	sudo systemctl status klipper-mcu klipper
	```
* Step 3: [Add adxl345 configuration to your `printer.cfg`](https://github.com/makerbase-mks/MKS-PI#adxl345-connection-and-configuration)


### Package Updates via Apt Update

Please double check kernel packages were freezed before running `apt update` command. E.g. run `sudo armbian-config` and check `System` -> `Freeze - Disable Armbian kernel updates` item.


### How to Rotate Screen

Sometimes you need to rotate the image on the screen, for example, when upgrading Flying Bear Ghost 5 printer. To do this you need to change value for `rotate` parameter under `spi_for_lcd@0` section (configuration for the display) and use `touchscreen-inverted-x = <0x01>` and/or `touchscreen-inverted-y = <0x01>` parameters for `spi_for_touch@1` section (configuration for touchscreen) in `/boot/dtb/rockchip/rk3328-roc-cc.dtb` file. Please note, value for `touchscreen-inverted-x = <0x01>` or `touchscreen-inverted-y = <0x01>` does *not* affect anything. To disable e.g. y-inversion, whole parameter should be commented out (`# touchscreen-inverted-y = <0x01>;`). There are few examples:

* 270° (default mode) - `rotate = <270>;` (or `rotate = <0x10e>;` and `touchscreen-inverted-y = <0x01>`
* 90° (flipped horizontally) - `rotate = <90>;` and `touchscreen-inverted-x = <0x01>`

Following commands may be used to perform this configuration:
```
# Backup
sudo cp /boot/dtb/rockchip/rk3328-roc-cc.dtb /boot/dtb/rockchip/rk3328-roc-cc.dtb.$(date +"%Y%m%d_%H%M%S").bak
# Unpack DTB file
sudo dtc -I dtb -O dts -o rk3328-roc-cc.dts /boot/dtb/rockchip/rk3328-roc-cc.dtb
#Make a copy to work with
sudo cp rk3328-roc-cc.dts rk3328-roc-cc-rotated.dts

# Find `rotate = <SOME_VALUE>` and change to `rotate = <NEW_VALUE>`, where NEW_VALUE is a rotation angle, e.g. `rotate = <90>` or `rotate = <270>`
# Then find `touchscreen-inverted-y` attribute and add or replace `touchscreen-inverted-x` one
nano rk3328-roc-cc-rotated.dts

# Or sed -i -e "s/<0x10e>/<90>/g" rk3328-roc-cc-rotated.dts
# and sed -i -e "s/touchscreen-inverted-y/touchscreen-inverted-x/g" rk3328-roc-cc-rotated.dts

# Double check
less rk3328-roc-cc-rotated.dts | grep rotate
less rk3328-roc-cc-rotated.dts | grep touchscreen-inverted

# Pack DTS to DTB
dtc -I dts -O dtb -o rk3328-roc-cc-rotated.dtb rk3328-roc-cc-rotated.dts

# Update rk3328-roc-cc.dtb with new version
sudo cp rk3328-roc-cc-rotated.dtb /boot/dtb/rockchip/rk3328-roc-cc.dtb

# Reboot
sudo reboot
```


### How to Configure CAN Bus

There are two non obvious points which you should be aware to successfully configure CAN bus:

1. MKS SKIPR board must be connected via USB (UART connection does not work for CAN bridge mode)
2. By default latest images/Ubuntu distros do not have `ifconfig` command out of the box, so [Klipper documentation -- USB to CAN bus bridge mode](https://www.klipper3d.org/CANBUS.html#usb-to-can-bus-bridge-mode) will not properly work.

Steps to configure CAN bus:

1. (For SKIPR board) hook MCU via USB cable ("USB to CAN bus bridge" is not supported for UART connection type).
2. (For SKIPR board) [compile Klipper firmware](https://klipper.discourse.group/t/mks-skipr-can-bus/5377/16) and flash MCU. Please specify bitrate/speed which will be used for toolhead. USuallu 500 000 or 1 000 000.
3. Configure `can0` interface on MKSPI: 
   ```
    cat <<-'EOF' | sudo tee -a /etc/network/interfaces.d/can0
	allow-hotplug can0
	iface can0 can static
		bitrate 1000000                                       # Ensure it's the same as selected for MCU firmware.
		# up ifconfig $IFACE txqueuelen 128                   # It will not work because no ifconfig installed by default...
		up ip link set $IFACE txqueuelen 128                  # ... please use this version instead. 
		#up ip link set $IFACE txqueuelen 1024 restart-ms 200 # Bit more aggressive configuration

	EOF
   ```   
4. Reboot. Yes, it's important! 
5. Hook a toolhead board, find connected devices `~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0` and follow [Klipper documentation -- USB to CAN bus bridge mode](https://www.klipper3d.org/CANBUS.html#usb-to-can-bus-bridge-mode) for the rest of configuration.


In case of `MCU 'mcu' shutdown: Timer too close`, `b'Got error -1 in can write: (105)No buffer space available'` or similar troubles, please try following steps:

Step 1. Use `ip -s link show can0` or/and [Moonraker -> Settings -> MCU status](https://www.teamfdm.com/forums/topic/1524-debugging-canbus-and-communication-timeout-while-homingbytes_invalid/#comment-9910) to ensure no error/invalid AND no retransmit packages. In case of error/invalid packages please see `Step 2`. In case of no error but increasing retransmit packages please refer `Step 3`.

Step 2: double check wiring and terminal resistor. A twisted pair for data signals (CAN_L & CAN_H) is almost required for line longer that 20cm.

Step 3. Ensure `sudo systemctl --failed` does not show failed units. Otherwice doble check `/etc/network/interfaces.d/can0` and ensure that `ifconfig` command (from the Klipper documentation) is *not* used. `if@can0` unit must start without any issues.



## How to Build

The new `mkspi` board was declared. Now has support only for `current` and `edge` kernels and Ubuntu Jammy OS (CLI and desktop editions). Build process is pretty usual for Armbain build.


I would advice to read official documentation, however it's short version:

1. Use Ubunut Jammy 22.04 OS (or VM). Ensure you have 15-40GB of free disk and 4-6GB RAM.
2. Clone repo
3. `cd armbian-mkspi`
4. `./compile.sh` and follow instructions... Please do not forget about `BSPFREEZE=yes` build arg (or freezing kernel updates via `sudo armbian-config` right after the first login). A few ready to use commands:
  * Ubuntu Jammy with current kernel:    `./compile.sh  BOARD=mkspi BRANCH=current RELEASE=jammy    BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * Ubuntu Jammy with edge kernel:       `./compile.sh  BOARD=mkspi BRANCH=edge    RELEASE=jammy    BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * Debian bullseye with current kernel: `./compile.sh  BOARD=mkspi BRANCH=current RELEASE=bullseye BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * Debian bullseye with edge kernel:    `./compile.sh  BOARD=mkspi BRANCH=edge    RELEASE=bullseye BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * Debian bookworm with current kernel: `./compile.sh  BOARD=mkspi BRANCH=current RELEASE=bookworm BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * Debian bookworm with edge kernel:    `./compile.sh  BOARD=mkspi BRANCH=edge    RELEASE=bookworm BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * append `INSTALL_HEADERS=yes BUILD_KSRC=yes INSTALL_KSRC=yes` and `BSFREEZE=yes` flags if you will need kernel headers, e.g. to compile custom WiFi drivers
  * if you need to modify the kernel sources: `./compile.sh  BOARD=mkspi BRANCH=current RELEASE=jammy BSPFREEZE=yes BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img kernel-patch`
5. Wait a 20-180 minutes (depends of your hardware, mostly disk system) and check `output\images\` directory



## Some technical Details:
Origina Image:
```
# PLEASE DO NOT EDIT THIS FILE
BOARD=mkspi
BOARD_NAME="mkspi"
BOARDFAMILY=rockchip64
BUILD_REPOSITORY_URL=https://github.com/armbian/build.git
BUILD_REPOSITORY_COMMIT=ed589b248-dirty
VERSION=22.05.0-trunk
LINUXFAMILY=rockchip64
ARCH=arm64
IMAGE_TYPE=user-built
BOARD_TYPE=conf
INITRD_ARCH=arm64
KERNEL_IMAGE_TYPE=Image
BRANCH=edge
/etc/armbian-release (END)
```

https://github.com/makerbase-mks/armbian-build repo contains random crap (half worked patches for legacy 4.4 Kernel and non full armbian integration)

In generally it's not clear what was changed, however looks like MKS guys were not too creative and almost copy rockchip64/Renegade board. Patches include:
1. Changes for `/arch/arm64/boot/dts/rockchip/rk3328-roc-cc.dts` ( redeclaring a few pins, disabling some features and declaring new ones. mostly for MKSPI-TS35 screen)
2. HDMI interface change, seems just to declare mode with 5:3 aspect ration
3. "Patch" for `fbtft/fb_ili9341` driver. Basically redefining screen resolution.
4. patches for SPI support code. 
5. Changes to drm_edid (see `/drivers/gpu/drm/drm_edid.c` and `/drivers/gpu/drm/rockchip/inno_hdmi.c` files). However I have no idea what this about and how to apply this patches to new kernel sources.
6. Kernel v4.4 config. However I am not sure how it's relevant to the current and edge branches.


## See Also
 - https://github.com/makerbase-mks/MKS-PI - MKS delivered image, official instructions and schematic. 
 - https://github.com/makerbase-mks/armbian-build - kind of sources (state of Dec.23 - random unworked crap)





# Original Armbian README.md
<p align="center">
  <a href="#build-framework">
  <img src=".github/armbian-logo.png" alt="Armbian logo" width="144">
  </a><br>
  <strong>Armbian Linux Build Framework</strong><br>
<br>
<a href=https://github.com/armbian/build/graphs/contributors><img alt="GitHub contributors" src="https://img.shields.io/github/contributors-anon/armbian/build?logo=stackexchange&label=Contributors&style=for-the-badge&branch=main&logoColor=white"></a>
<a href=https://github.com/armbian/os><img alt="Artifacts generation" src="https://img.shields.io/github/actions/workflow/status/armbian/os/complete-artifact-matrix-all.yml?logo=dependabot&label=CI%20Build&style=for-the-badge&branch=main&logoColor=white"></a>
<a href=https://github.com/armbian/build/commits/main><img alt="GitHub last commit (branch)" src="https://img.shields.io/github/last-commit/armbian/build/main?logo=github&label=Last%20commit&style=for-the-badge&branch=main&logoColor=white"></a>
</p>

## What does this project do?

- Builds custom **kernel**, **image** or a **distribution** optimized for low-resource hardware,
- Include filesystem generation, low-level control software, kernel image and **bootloader** compilation,
- Provides a **consistent user experience** by keeping system standards across different platforms.

## Getting started

### Requirements for self hosted

- x86_64 / aarch64 machine
- at least 2GB of memory and ~35GB of disk space for VM, container or bare metal installation
- [Armbian / Ubuntu Jammy 22.04.x](https://github.com/armbian/sdk) for native building or any Docker capable Linux for containerised
- Windows 10/11 with WSL2 subsystem running Ubuntu Jammy 22.04.x
- Superuser rights (configured sudo or root access).
- Make sure your system is up-to-date! Outdated Docker binaries, for example, can cause trouble.

For stable branch use `--branch=v24.08`

```bash
apt-get -y install git
git clone --depth=1 --branch=main https://github.com/armbian/build
cd build
./compile.sh
```

<a href="#how-to-build-an-image-or-a-kernel"><img src=".github/README.gif" alt="Armbian logo" width="100%"></a>

- Interactive graphical interface.
- Prepares the workspace by installing the necessary dependencies and sources.
- It guides the entire process and creates a kernel package or a ready-to-use SD card image.

### Build parameter examples

Show work-in-progress areas in interactive mode:

```bash
./compile.sh EXPERT="yes"
```

Build minimal CLI Armbian Jammy for Bananapi M5 with LTS kernel:

```bash
./compile.sh \
BOARD=bananapim5 \
BRANCH=current \
RELEASE=jammy \
BUILD_MINIMAL=yes \
BUILD_DESKTOP=no \
KERNEL_CONFIGURE=no
```

Build with GitHub actions: ([advanced version](https://github.com/armbian/os/blob/main/.github/workflows/complete-artifact-one-by-one.yml))

```
name: "Build Armbian"
on:
  workflow_dispatch:
jobs:
  build-armbian:
    runs-on: ubuntu-latest
    steps:
      - uses: armbian/build@main
        with:
          armbian_token:     "${{ secrets.GITHUB_TOKEN }}"  # GitHub token
          armbian_release:   "jammy"                        # userspace
          armbian_target:    "build"                        # build=image, kernel=kernel
          armbian_board:     "bananapim5"                   # build target
```
Generated image will be uploaded to your repository release. Note: GitHub upload file limit is 2Gb.

## More information:

- [Building Armbian](https://docs.armbian.com/Developer-Guide_Build-Preparation/) (how to start, how to automate)
- [Build options](https://docs.armbian.com/Developer-Guide_Build-Options/) (all build options)
- [User configuration](https://docs.armbian.com/Developer-Guide_User-Configurations/) (how to add packages, patches, and override sources config)

## Download prebuilt images releases

### Point

- [manually released **standard supported** builds](https://www.armbian.com/download/?device_support=Standard%20support) (quarterly)

### Rolling

- [automatically released **staging and standard supported** builds](https://github.com/armbian/os/releases/latest) (daily)
- [automatically released **community maintained** builds](https://github.com/armbian/community/releases/latest) (weekly)

## Compared with industry standards

<details><summary>Expand</summary>
Check similarities, advantages and disadvantages compared with leading industry standard build software.

Function | Armbian | Yocto | Buildroot |
|:--|:--|:--|:--|
| Target | general purpose | embedded | embedded / IOT |
| U-boot and kernel | compiled from sources | compiled from sources | compiled from sources |
| Board support maintenance &nbsp; | complete | outside | outside |
| Root file system | Debian or Ubuntu based| custom | custom |
| Package manager | APT | any | none |
| Configurability | limited | large | large |
| Initramfs support | yes | yes | yes |
| Getting started | quick | very slow | slow |
| Cross compilation | yes | yes | yes |
</details>

## Project structure

<details><summary>Expand</summary>

```text
├── cache                                Work / cache directory
│   ├── aptcache                         Packages
│   ├── ccache                           C/C++ compiler
│   ├── docker                           Docker last pull
│   ├── git-bare                         Minimal Git
│   ├── git-bundles                      Full Git
│   ├── initrd                           Ram disk
│   ├── memoize                          Git status
│   ├── patch                            Kernel drivers patch
│   ├── pip                              Python
│   ├── rootfs                           Compressed userspaces
│   ├── sources                          Kernel, u-boot and other sources
│   ├── tools                            Additional tools like ORAS
│   └── utility
├── config                               Packages repository configurations
│   ├── targets.conf                     Board build target configuration
│   ├── boards                           Board configurations
│   ├── bootenv                          Initial boot loaders environments per family
│   ├── bootscripts                      Initial Boot loaders scripts per family
│   ├── cli                              CLI packages configurations per distribution
│   ├── desktop                          Desktop packages configurations per distribution
│   ├── distributions                    Distributions settings
│   ├── kernel                           Kernel build configurations per family
│   ├── sources                          Kernel and u-boot sources locations and scripts
│   ├── templates                        User configuration templates which populate userpatches
│   └── torrents                         External compiler and rootfs cache torrents
├── extensions                           Extend build system with specific functionality
├── lib                                  Main build framework libraries
│   ├── functions
│   │   ├── artifacts
│   │   ├── bsp
│   │   ├── cli
│   │   ├── compilation
│   │   ├── configuration
│   │   ├── general
│   │   ├── host
│   │   ├── image
│   │   ├── logging
│   │   ├── main
│   │   └── rootfs
│   └── tools
├── output                               Build artifact
│   └── deb                              Deb packages
│   └── images                           Bootable images - RAW or compressed
│   └── debug                            Patch and build logs
│   └── config                           Kernel configuration export location
│   └── patch                            Created patches location
├── packages                             Support scripts, binary blobs, packages
│   ├── blobs                            Wallpapers, various configs, closed source bootloaders
│   ├── bsp-cli                          Automatically added to armbian-bsp-cli package
│   ├── bsp-desktop                      Automatically added to armbian-bsp-desktopo package
│   ├── bsp                              Scripts and configs overlay for rootfs
│   └── extras-buildpkgs                 Optional compilation and packaging engine
├── patch                                Collection of patches
│   ├── atf                              ARM trusted firmware
│   ├── kernel                           Linux kernel patches
|   |   └── family-branch                Per kernel family and branch
│   ├── misc                             Linux kernel packaging patches
│   └── u-boot                           Universal boot loader patches
|       ├── u-boot-board                 For specific board
|       └── u-boot-family                For entire kernel family
├── tools                                Tools for dealing with kernel patches and configs
└── userpatches                          User: configuration patching area
    ├── lib.config                       User: framework common config/override file
    ├── config-default.conf              User: default user config file
    ├── customize-image.sh               User: script will execute just before closing the image
    ├── atf                              User: ARM trusted firmware
    ├── kernel                           User: Linux kernel per kernel family
    ├── misc                             User: various
    └── u-boot                           User: universal boot loader patches
```
</details>

## Contribution

### Want to help?

We always need those volunteering positions:

- [Code reviewer](https://forum.armbian.com/staffapplications/application/23-code-reviewer/)
- [Build framework maintainer](https://forum.armbian.com/staffapplications/application/9-build-framework-maintainer/)
- [Test Automation Engineer](https://forum.armbian.com/staffapplications/application/19-test-automation-engineer/)

Just apply and follow!

## Support

For commercial or prioritized assistance:
 - Book an hour of [professional consultation](https://calendly.com/armbian/consultation)
 - Consider becoming a [project partner](https://forum.armbian.com/subscriptions/)
 - [Contact us](https://armbian.com/contact)!

Free support:

 Find free support via [general project search engine](https://www.armbian.com/search), [documentation](https://docs.armbian.com), [community forums](https://forum.armbian.com/) or [IRC/Discord](https://docs.armbian.com/Community_IRC/). Remember that our awesome community members mainly provide this in a **best-effort** manner, so there are no guaranteed solutions.

## Contact

- [Forums](https://forum.armbian.com) for Participate in Armbian
- IRC: `#armbian` on Libera.chat / oftc.net
- Matrix: [https://forum.armbian.com/topic/40413-enter-the-matrix/](https://forum.armbian.com/topic/40413-enter-the-matrix/)
- Discord: [https://discord.gg/armbian](https://discord.gg/armbian)
- Follow [@armbian](https://twitter.com/armbian) on 𝕏 (formerly known as Twitter), <a rel="me" href="https://fosstodon.org/@armbian">Mastodon</a> or [LinkedIn](https://www.linkedin.com/company/armbian).
- Bugs: [issues](https://github.com/armbian/build/issues) / [JIRA](https://armbian.atlassian.net/jira/dashboards/10000)
- Office hours: [Wednesday, 12 midday, 18 afternoon, CET](https://calendly.com/armbian/office-hours)

## Contributors

Thank you to all the people who already contributed to Armbian!

<a href="https://github.com/armbian/build/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=armbian/build" />
</a>

### Also

- [Current and past contributors](https://github.com/armbian/build/graphs/contributors), our families and friends.
- [Support staff](https://forum.armbian.com/members/2-moderators/) that keeps forums usable.
- [Friends and individuals](https://armbian.com/authors) who support us with resources and their time.
- [The Armbian Community](https://forum.armbian.com/) helps with their ideas, reports and [donations](https://www.armbian.com/donate).

## Armbian Partners

Armbian's partnership program helps to support Armbian and the Armbian community! Please take a moment to familiarize yourself with our Partners:

- [Click here to visit our Partners page!](https://armbian.com/partners)
- [How can I become a Partner?](https://forum.armbian.com/subscriptions)

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=armbian/build&type=Date)](https://star-history.com/#armbian/build&Date)

## License

This software is published under the GPL-2.0 License license.
