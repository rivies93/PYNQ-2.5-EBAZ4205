# PYNQ-2.5-EBAZ4205

PYNQ 2.5 port for the **EBAZ4205 Zynq-7020** board, including a custom
boot flow that configures the programmable logic (PL) before Linux
starts and enables Ethernet through the Zynq PS Ethernet MAC using
**EMIO**.

The project combines:

-   PYNQ 2.5
-   Xilinx Zynq-7020 / XC7Z020
-   PetaLinux 2019.1
-   Ethernet through PS GEM0 using EMIO
-   A modified device tree for the EBAZ4205 Ethernet PHY
-   A custom `ethernet_ebaz` PYNQ `sdbuild` package for the Ethernet
    interface configuration
-   A custom `BOOT.BIN` containing the PetaLinux FSBL, FPGA bitstream,
    and the PYNQ U-Boot
-   PYNQ's `image.ub` for the Linux/PYNQ environment

## Hardware

-   EBAZ4205 board
-   Xilinx Zynq-7020 (XC7Z020)
-   Ethernet connection through the board's RJ45 interface

## Software

The build environment used for this project is based on:

-   Ubuntu Linux
-   Vivado / Xilinx SDK 2019.1
-   PetaLinux 2019.1
-   PYNQ 2.5

## 1. Prepare the EBAZ4205

Start by preparing the EBAZ4205 hardware project and obtaining the
hardware design files.

References:

-   EBAZ4205 hardware introduction:
    https://gitee.com/actionchen/ebaz4205_hw/blob/master/Doc/ebaz4205_introduce.md
-   FPGA Zero to Hero -- EBAZ4205:
    https://www.codeembedded.com/blog/fpga_zero_to_hero_vol_5/

The hardware design provides the basis for generating the Xilinx
hardware platform used by PetaLinux.

## 2. Install Vivado, SDK and PetaLinux 2019.1

Install the Xilinx tools required to build the hardware and PetaLinux
components.

References:

-   Vivado / SDK 2019.2 installation guide for Ubuntu 18.04:
    https://www.hackster.io/news/vivado-vitis-2019-2-install-on-ubuntu-18-04-lts-93242be6c9eb
-   PetaLinux 2019.1 installation:
    https://www.fpgadeveloper.com/how-to-install-petalinux-2019.1/

> This project uses the 2019.1 toolchain. Make sure the Vivado, SDK and
> PetaLinux versions are compatible with the project.

## 3. Generate the hardware files and BSP

Generate the hardware platform from the Vivado design and use it to
create the PetaLinux BSP. 
    https://webuiltawallwebuiltthepyramids.blogspot.com/2021/01/ebaz4205-petalinux-installation.html

The EBAZ4205 Ethernet interface is connected to the Zynq PS Ethernet MAC
through **EMIO** rather than the standard Ethernet MIO pins.

The Linux device tree therefore needs to be modified so that the PS GEM
interface is enabled and configured for the external PHY.

The Ethernet interface is configured as MII where required by the
EBAZ4205 hardware design.

The device-tree modification is included in the PetaLinux/BSP side of
the project.

Before generating the BSP you can modify `{your_BSPproject}/project-spec/meta-user/recipes-bsp/device-tree/files/system_user.dtsi` with:
```make
/include/ "system-conf.dtsi"

/ {

amba {

	ethernet@e000b000 {

	     status = "okay";

	     phy-mode = "mii";

	     phy-handle = <&phy0>;

	     local-mac-address = [00 0a 35 00 00 00];

	     phy0: phy@0 {

		 

		 reg = <0>;

		 device_type = "ethernet-phy";

		 xlnx,phy-type = <5>;

	     };

	};

};

};
```
Or just modify the device tree file and re-generate the BSP

Some commands:
```make
 source /tools/Xilinx/SDK/2019.1/settings64.sh
 source /tools/Xilinx/PetaLinux/2019.1/settings.sh
 petalinux-util --webtalk off
 petalinux-create --type project --template zynq --name ebaz
 petalinux-config --get-hw-description=/yourHW_sdk
```
Modify BPS configuration as the reference and then:
```make
petalinux-build
```
After Modify Device-Tree generate BSP:

```make
petalinux-package --bsp -p /home/ric/Petalinux/ebaz --output ebaz --force
petalinux-package --boot --format BIN --fsbl ./images/linux/zynq_fsbl.elf --fpga ./images/linux/system.bit --u-boot --force
```


Reference:

https://webuiltawallwebuiltthepyramids.blogspot.com/2021/01/ebaz4205-petalinux-installation.html

The resulting BSP is used as the basis for the PYNQ image build.


## 4. Configure Ethernet with the `ethernet_ebaz` package

The project includes a custom PYNQ `sdbuild/package` package named:

``` text
ethernet_ebaz
```
`sdbuild/package/ethernet_ebaz` Includes a `pre.sh` copy of the one already included in the original pynq packages and a `eth0` file with:

```make
auto eth0
iface eth0 inet static
	address 192.168.2.99
	netmask 255.255.255.0
```
(Adds IP configuration)

This package configures the Linux Ethernet interface for the EBAZ4205.

The important distinction is:

-   **Device tree:** enables and describes the Ethernet hardware so
    Linux can use the PS GEM through EMIO.
-   **`ethernet_ebaz` package:** configures the Linux network interface.

The package is added to the EBAZ4205 board configuration through
`STAGE4_PACKAGES`.

### Package order

The order of the Stage 4 packages is important.

`ethernet_ebaz` must be listed **before** packages such as `pynq`:

``` make
STAGE4_PACKAGES_EBAZ4205 := ethernet_ebaz pynq
```
`.spec` file includes:

```make
ARCH_ebaz := arm
BSP_ebaz := ebaz.bsp
BITSTREAM_ebaz := base/ebaz.bit
STAGE4_PACKAGES_ebaz := ethernet_ip pynq
```

This is required because the PYNQ package processing can otherwise
prevent the EBAZ Ethernet package configuration from being present in
the final image.

The package installs the Ethernet configuration into the generated Linux
filesystem.

## 5. Build the PYNQ image

Build the PYNQ image using the PYNQ 2.5 `sdbuild` flow and the Bionic
base image.

PYNQ 2.5 SD-card/image build documentation:

https://pynq.readthedocs.io/en/v2.5/pynq_sd_card.html#pynq-sd-card

The resulting image contains the PYNQ Linux environment together with
the EBAZ4205-specific Ethernet configuration.

```make
cd /home/{user}/PYNQ/sdbuild/
source /tools/Xilinx/Vivado/2019.1/settings64.sh
source /tools/Xilinx/SDK/2019.1/settings64.sh
source /tools/Xilinx/PetaLinux/2019.1/settings.sh
petalinux-util --webtalk off
sudo make clean
make BOARDDIR={your_boards_dir} PREBUILT=~/bionic.arm.2.5.img
```

`{your_boards_dir}` where /ebaz is and `bionic.arm.2.5.img` is the prebuilt image for ZYNQ devices. 


After building the image, the filesystem can be inspected by mounting
the Linux partition of the generated `.img` file and verifying the
Ethernet configuration installed by `ethernet_ebaz`.

## 6. Generate the custom `BOOT.BIN`

The EBAZ4205 requires a custom boot image because the FPGA programmable
logic must be configured **before Linux starts**.

The custom `BOOT.BIN` combines:

1.  PetaLinux FSBL
2.  FPGA bitstream (`system.bit`)
3.  PYNQ U-Boot

The PetaLinux-generated FSBL and bitstream come from the EBAZ4205
PetaLinux BSP, while the U-Boot executable is taken from the PYNQ
image/build.

A `.bif` file is used to create the boot image.

Example structure:

``` text
the_ROM_image:
{
    [bootloader] fsbl.elf
    system.bit
    u-boot.elf
}
```

The `[bootloader]` attribute is important: it identifies the FSBL as the
bootloader and ensures the expected Zynq boot flow.

```make
source /tools/Xilinx/Vivado/2019.1/settings64.sh
source /tools/Xilinx/SDK/2019.1/settings64.sh
source /tools/Xilinx/PetaLinux/2019.1/settings.sh
petalinux-util --webtalk off
bootgen -image bitstream.bif -arch zynq -process_bitstream bin
```

The resulting boot sequence is:

``` text
FSBL
  |
  +--> Configure PL with system.bit
  |
  +--> Start U-Boot
          |
          +--> Load PYNQ image.ub
                    |
                    +--> Linux
```

Loading the PL before Linux is essential for the EBAZ4205 Ethernet
design because the Ethernet signals are routed through EMIO.

## 7. Prepare the SD card

Write the generated PYNQ image to the SD card.

PYNQ documentation:

-   Writing the PYNQ SD-card image:
    https://pynq.readthedocs.io/en/v2.5/appendix.html#writing-the-sd-card-image
-   PYNQ getting started:
    https://pynq.readthedocs.io/en/v2.5/getting_started.html

After writing the image, use the generated/custom `BOOT.BIN` and the
PYNQ `image.ub` as required by the final boot partition.

The important boot files are:

``` text
BOOT.BIN
image.ub
```

`BOOT.BIN` contains the EBAZ-specific FSBL and FPGA bitstream together
with the PYNQ U-Boot.

`image.ub` comes from the PYNQ 2.5 build.

## 9. Ethernet configuration

The EBAZ4205 image is configured to use a static IPv4 address.

Default configuration:

``` text
Interface:  eth0
IP address: 192.168.2.99
Netmask:    255.255.255.0
```

After booting PYNQ, the Ethernet interface can be checked with:

``` bash
ip addr show eth0
```

The interface can also be tested with:

``` bash
ping 192.168.2.99
```

Once Ethernet is working, the PYNQ Jupyter environment can be accessed
through the board's IP address.

## Boot and build flow

The complete process can be summarized as:

``` text
EBAZ4205 hardware design
          |
          v
      Vivado/XSA
          |
          v
   PetaLinux 2019.1
          |
          +---- modified device tree
          |
          +---- FSBL + system.bit
          |
          v
        BSP
          |
          v
      PYNQ 2.5 sdbuild
          |
          +---- ethernet_ebaz
          |
          +---- pynq
          |
          v
      PYNQ image
          |
          +---- image.ub
          |
          v
     Custom BOOT.BIN
          |
          +---- FSBL
          +---- system.bit
          +---- PYNQ U-Boot
          |
          v
        SD card
          |
          v
   EBAZ4205 boots PYNQ
          |
          v
 PL configured before Linux
          |
          v
 Ethernet available through EMIO
```

## References

-   EBAZ4205 hardware:
    https://gitee.com/actionchen/ebaz4205_hw/blob/master/Doc/ebaz4205_introduce.md
-   FPGA Zero to Hero:
    https://www.codeembedded.com/blog/fpga_zero_to_hero_vol_5/
-   Vivado / SDK installation:
    https://www.hackster.io/news/vivado-vitis-2019-2-install-on-ubuntu-18-04-lts-93242be6c9eb
-   PetaLinux 2019.1 installation:
    https://www.fpgadeveloper.com/how-to-install-petalinux-2019.1/
-   EBAZ4205 PetaLinux installation:
    https://webuiltawallwebuiltthepyramids.blogspot.com/2021/01/ebaz4205-petalinux-installation.html
-   PYNQ 2.5 SD-card build:
    https://pynq.readthedocs.io/en/v2.5/pynq_sd_card.html#pynq-sd-card
-   PYNQ 2.5 SD-card preparation:
    https://pynq.readthedocs.io/en/v2.5/appendix.html#writing-the-sd-card-image
-   PYNQ 2.5 getting started:
    https://pynq.readthedocs.io/en/v2.5/getting_started.html

## Status

This project provides a PYNQ 2.5 environment for the EBAZ4205 with
Ethernet working through the Zynq PS GEM and EMIO, using a custom boot
flow that configures the programmable logic before Linux starts.
