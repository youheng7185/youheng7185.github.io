# NXP IMX6SL DDR Bringup Notes

## Introduction

Here is the record  of bringup of uboot on kindle 8th generation e-ink reader. This device is using nxp imx6sl soc, most likely would be MCIMX6L8DVN10AB, as it is the only one with EPD peripheral in the family. Other than that, this device uses a 512MB LPDDR2 running at 400MHz, but its actually DDR3-792 due to pll scallings.

## IMX6SL source code on mainline u-boot

[https://github.com/u-boot/u-boot/blob/master/board/nxp/mx6slevk/mx6slevk.c](https://github.com/u-boot/u-boot/blob/master/board/nxp/mx6slevk/mx6slevk.c)

By inspecting its defconfig file, CONFIG_XPL_BUILD is turned off. So lets follow the evk method to init our ddr, The ddr init is actually in [https://github.com/u-boot/u-boot/blob/master/board/nxp/mx6slevk/imximage.cfg](https://github.com/u-boot/u-boot/blob/master/board/nxp/mx6slevk/imximage.cfg). I was shocked as its just bunch of register addresses and value, this would be impossible for me to refer to datasheet and fill in one by one.

## NXP DDR Stress Test Tool

After searching on the internet, people uses this official tool to generate the register table. [https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/i-MX-6-7-DDR-Stress-Test-Tool/ta-p/1108221?profile.language=en](https://community.nxp.com/t5/i-MX-Processors-Knowledge-Base/i-MX-6-7-DDR-Stress-Test-Tool/ta-p/1108221?profile.language=en)

They also gave an excel file to fill in ddr value and it would generate .inc file. After filling in proper ddr configuration:

```
=============================================================================			
// DDR Controller Registers			
//=============================================================================			
// Manufacturer:	Samsung		
// Device Part Number:	K4P8G304EB-AGC1		
// Clock Freq.: 	400MHz		
// Density per CS in Gb: 	4		
// Chip Selects used:	1		
// Total DRAM density (Gb)	4		
// Number of Banks:	8		
// Row address:    	14		
// Column address: 	10		
// Data bus width	32		
//=============================================================================			
```

Run the calibration option once, it will give MPRDDLCTL and MPWRDLCTL value, just replace with the default one in .inc file.

Until here, I used LLM to convert that .inc to the style in imximage.cfg, which is `DATA 4 register_addr register_value`. Pasting it in and rebuild, 

```
export CROSS_COMPILE=arm-linux-gnu-
export ARCH=arm

make clean
make distclean
make mx6sl_kindle_defconfig
```

it works in the first trial:

```
U-Boot 2026.07-rc3-g7bf90b0bc4b4-dirty (May 29 2026 - 13:25:46 +0800)

CPU:   Freescale i.MX6SL rev1.3 996 MHz (running at 792 MHz)
CPU:   Commercial temperature grade (0C to 95C) at 41C
Reset cause: POR
Model: Freescale i.MX6 SoloLite EVK Board
Board: MX6SLEVK
DRAM:  512 MiB
Core:  83 devices, 20 uclasses, devicetree: separate
WDT:   Started watchdog@20bc000 with servicing every 1000ms (128s timeout)
MMC:   FSL_SDHC: 0, FSL_SDHC: 1, FSL_SDHC: 2
Loading Environment from MMC... MMC: no card present
*** Warning - No block device, using default environment

In:    serial@2020000
Out:   serial@2020000
Err:   serial@2020000
Net:   Could not get PHY for FEC0: addr -2
No ethernet found.

Hit any key to stop autoboot: 0
MMC: no card present
MMC: no card present
Booting from net ...
```