# Unofficial Armbian build for [Makerbase MKS PI board](https://github.com/makerbase-mks/MKS-PI). 

TLTR: Unofficial support of Makerbase MKS PI board. Contains `mkspi` board declaration and related kernel and u-boot patches.  
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
* If you are interested in UART, I2C, EMMC etc features, feel free to step into development/testing or support by hardware.
* ~~Currently I am focusing only on Ubuntu LTS builds (`current` and `edge` kernels). Feel free to open PRs if you would need non LTS Ubuntu or Debian images.~~ Currently Ubuntu LTS (tested) and Debian Bullseye (non tested) builds are supported.



## Current status

| Feature | Current (5.15) | Edge (6.1) |
|:--|:--|:--|
| USB 2 | yes             | yes            |
| USB 3 | yes             | yes            |
| USB Type-C (debug serial port) | yes             | yes            |
| HDMI Video | yes             | yes            |
| HDMI Audio | not tested yet  | not tested yet |
| MKSPI-TS35 TFT display | yes | yes            |
| MKSPI-TS35 touch screen | yes | yes |
| Reset button | yes | yes |
| Ethernet | yes | yes |
| WiFi dongles | yes | yes |
| ADXL345 (SPI0 connection) | yes | yes | 
| UART0| yes | yes |
| I2C| not tested yet | not tested yet |




### Known Issues
Edge, Jammy:
* `irq 37: nobody cared` message in boot log and on boot screen
* Works either HDMI out or MKS PI-TS35 display. No dual screen, no reconnection during runtime. Display must be connected before system start and cannot be switched after boot.


Current, Jammy:
* `irq 56: nobody cared` message in boot log
* Works either HDMI out or MKS PI-TS35 display. No dual screen, no reconnection during runtime. Display must be connected before system start and cannot be switched after boot.


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

Sometimes you need to rotate the image on the screen, for example, when upgrading Flying Bear Ghost 5 printer. To do this you need to change value for `rotate` parameter under `spi_for_lcd@0` section in `/boot/dtb/rockchip/rk3328-roc-cc.dtb` file. Following commands may be used to perform this configuration:
```
# Backup
sudo cp /boot/dtb/rockchip/rk3328-roc-cc.dtb /boot/dtb/rockchip/rk3328-roc-cc.dtb.$(date +"%Y%m%d_%H%M%S").bak
# Unpack DTB file
sudo dtc -I dtb -O dts -o rk3328-roc-cc.dts /boot/dtb/rockchip/rk3328-roc-cc.dtb
#Make a copy to work with
sudo cp rk3328-roc-cc.dts rk3328-roc-cc-rotated.dts

# Find `rotate = <SOME_VALUE>` and change to `rotate = <NEW_VALUE>`, where NEW_VALUE is a rotation angle, e.g. `rotate = <90>` or `rotate = 270>`
nano rk3328-roc-cc-rotated.dts

# Or sed -i -e "s/<0x10e>/<90>/g" rk3328-roc-cc-rotated.dts

# Double check
less rk3328-roc-cc-rotated.dts | grep rotate

# Pack DTS to DTB
dtc -I dts -O dtb -o rk3328-roc-cc-rotated.dtb rk3328-roc-cc-rotated.dts

# Update rk3328-roc-cc.dtb with new version
sudo cp rk3328-roc-cc-rotated.dtb /boot/dtb/rockchip/rk3328-roc-cc.dtb

# Reboot
sudo reboot
```


## How to Build

The new `mkspi` board was declared. Now has support only for `current` and `edge` kernels and Ubuntu Jammy OS (CLI and desktop editions). Build process is pretty usual for Armbain build.


I would advice to read official documentation, however it's short version:

1. Use Ubunut Jammy 22.04 OS (or VM). Ensure you have 15-40GB of free disk and 4-6GB RAM.
2. Clone repo
3. `cd armbian-mkspi`
4. `./compile.sh` and follow instructions... Please do not forget to freeze kernel updates via `sudo armbian-config` right after the first login. Or use `BSPFREEZE=yes` CLI arg during image build.
  * ... or `./compile.sh BOARD=mkspi BRANCH=current RELEASE=jammy BSPFREEZE=yes BUILD_MINIMAL=no BUILD_DESKTOP=no BUILD_ONLY=default KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * ... or `./compile.sh BOARD=mkspi BRANCH=current RELEASE=jammy BSPFREEZE=yes BUILD_MINIMAL=no BUILD_DESKTOP=yes KERNEL_CONFIGURE=no DESKTOP_ENVIRONMENT=xfce DESKTOP_ENVIRONMENT_CONFIG_NAME=config_base COMPRESS_OUTPUTIMAGE=sha,gpg,img`
 
  * ... or `./compile.sh BOARD=mkspi BRANCH=edge RELEASE=jammy BSPFREEZE=yes BUILD_MINIMAL=no BUILD_DESKTOP=no BUILD_ONLY=default KERNEL_CONFIGURE=no COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * ... or `./compile.sh BOARD=mkspi BRANCH=edge RELEASE=jammy BSPFREEZE=yes BUILD_MINIMAL=no BUILD_DESKTOP=yes KERNEL_CONFIGURE=no DESKTOP_ENVIRONMENT=xfce DESKTOP_ENVIRONMENT_CONFIG_NAME=config_base COMPRESS_OUTPUTIMAGE=sha,gpg,img`
  * ... feel free to append  ` CREATE_PATCHES=yes` arg if you would like to change U-Boot or Kernel sources.
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
<a href=https://github.com/armbian/os><img alt="Artifacts generation" src="https://img.shields.io/github/actions/workflow/status/armbian/os/complete-artifact-matrix-all.yml?logo=githubactions&label=Build&style=for-the-badge&branch=main&logoColor=white"></a>
</p>

## Table of contents

- [What does this project do?](#what-does-this-project-do)
- [Getting started](#getting-started)
- [Compared with industry standards](#compared-with-industry-standards)
- [Download prebuilt images](#download-prebuilt-images)
- [Project structure](#project-structure)
- [Contribution](#contribution)
- [Support](#support)
- [Contact](#contact)
- [Contributors](#contributors)
- [Partners](#armbian-partners)
- [License](#license)

## What does this project do?

- Builds custom **kernel**, **image** or a **distribution** optimized for low-resource hardware,
- Include filesystem generation, low-level control software, kernel image and **bootloader** compilation,
- Provides a **consistent user experience** by keeping system standards across different platforms.

## Getting started

### Basic requirements

- x86_64 or aarch64 machine with at least 2GB of memory and ~35GB of disk space for a virtual machine, [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install), container or bare metal installation
- Ubuntu Jammy 22.04.x amd64 or aarch64 for native building or any Docker capable amd64 / aarch64 Linux for containerised
- Superuser rights (configured sudo or root access).
- Make sure all your system components are up-to-date. Outdated Docker binaries, for example, can cause trouble.

### Start with the build script

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

Build minimal CLI Armbian Jammy image for Orangepi Zero. Use `current` kernel and write image to the SD card:

```bash
./compile.sh \
BOARD=orangepizero \
BRANCH=current \
RELEASE=jammy \
BUILD_MINIMAL=yes \
BUILD_DESKTOP=no \
KERNEL_CONFIGURE=no \
CARD_DEVICE="/dev/sdX"
```

More information:

- [Building Armbian](https://docs.armbian.com/Developer-Guide_Build-Preparation/) — how to start, how to automate;
- [Build options](https://docs.armbian.com/Developer-Guide_Build-Options/) — all build options;
- [User configuration](https://docs.armbian.com/Developer-Guide_User-Configurations/) — how to add packages, patches, and override sources config;

## Download prebuilt images

- [quarterly released **supported** builds](https://www.armbian.com/download/?device_support=Standard%20support)
- [quarterly released **community maintained** builds](https://www.armbian.com/download/?device_support=Community%20maintained)
- [automatic released **rolling release** builds](https://github.com/armbian/os/releases/latest) (daily or when code changes)

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

### You don't need to be a programmer to help!

The easiest way to help is by "Starring" our repository - it helps more people find our code. 

- [Check out our list of volunteer positions](https://forum.armbian.com/staffapplications/) and choose what you want to do ❤️
- [Mirror our download infrastructure](https://github.com/armbian/mirror/)  ❤️
- [Donate](https://www.armbian.com/donate)!  ❤️

### Want to become a maintainer?

Please review the [Board Maintainers Procedures and Guidelines](https://docs.armbian.com/Board_Maintainers_Procedures_and_Guidelines/) and [apply](https://forum.armbian.com/staffapplications/application/8-single-board-computer-maintainer/) !

### Want to become a developer?

To help with development, review [this document](CONTRIBUTING.md) and move straight to the code:

- [release related tickets](https://www.armbian.com/participate/)
- [review pull requests](https://github.com/armbian/build/pulls?q=is%3Apr+is%3Aopen+review%3Arequired+label%3A%22Needs+review%22)
- ticket dashboard for [junior](https://armbian.atlassian.net/jira/dashboards/10000) and [seniors](https://armbian.atlassian.net/jira/dashboards/10103) developers

## Support

For commercial or prioritized assistance:
 - Book an hour of [professional consultation](https://calendly.com/armbian/consultation)
 - Consider becoming a [project partner](https://forum.armbian.com/subscriptions/)
 - [Contact us](https://armbian.com/contact)! 
 
Free support:

 Find free support via [general project search engine](https://www.armbian.com/search), [documentation](https://docs.armbian.com), [community forums](https://forum.armbian.com/) or [IRC/Discord](https://docs.armbian.com/Community_IRC/). Remember that our awesome community members mainly provide this in a **best-effort** manner, so there are no guaranteed solutions.

## Contact

- [Forums](https://forum.armbian.com) for Participate in Armbian
- IRC: `#armbian` on Libera.chat
- Discord: [https://discord.gg/armbian](https://discord.gg/armbian)
- Follow [@armbian](https://twitter.com/armbian) on X (formerly known as Twitter), [Fosstodon](https://fosstodon.org/@armbian) or [LinkedIn](https://www.linkedin.com/company/armbian).
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
