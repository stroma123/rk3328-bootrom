# Rockchip RK3328 BootROM Study

## Motivation

I’ve been looking for a way to fully boot the RK3328 (U-Boot, Linux kernel, and root filesystem) entirely over USB, without requiring any non-volatile storage (eMMC or SD card).

It turned out to be harder than I expected.

## Rockchip tools

Rockchip do provide open source tools like 
+ [rkbin](https://github.com/rockchip-linux/rkbin)
+ [Rkdeveloptool](https://github.com/rockchip-linux/rkdeveloptool)

These tools allow transferring and executing binary data over USB. However, they are primarily oriented toward eMMC programming.
The simplified boot flow is as follows:
+ After powering-up RK3328 is in so called `Mask ROM` mode, waiting for USB transfers, note only `rkdeveloptool db` command is available at this stage;
+ Running `rkdeveloptool db rk3328_loader_v1.21.250.bin` uploads and executes the initial loader (Rockchip proprietary in this case).
+ After that additional `rkdeveloptool` commands become available, such as:
    + `rl` - read eMMC
    + `wl` - write eMMC
    + `rd` - reset, and more

In other word those commands are supported by `rk3328_loader_v1.21.250.bin`, not by the BootROM itself.

Fortunately, the `rkbin/tools/boot_merger` utility is used to generate the initial loader (`rk3328_loader_v1.21.250.bin`), which makes creating a custom loader feasible.

## The idea

After some consideration, I concluded that I needed to build my own initial loader with support for further USB-based data loading. This would allow me to:
1. Upload a custom loader using `rkdeveloptool`
2. Continue loading the remaining boot stages over USB.

While researching the topic, I found a [blog post](https://www.collabora.com/news-and-blog/blog/2023/07/31/booting-on-radxas-rock-5b-without-any-media-used/) by Eugen Hristevdescribing a similar approach for the newer RK3588 SoC family. Although his work targets a different platform, it provided useful guidance for implementing the same concept on the RK3328.

## The result

After some development work, I successfully added USB gadget support to the U-Boot SPL, along with the ability to download and execute the next-stage bootloader (U-Boot proper) over USB.

For USB transport in SPL, I enabled both:
+ [SDP](https://github.com/ARM-software/u-boot/blob/master/doc/README.sdp) - since I already use it with i.MX6/8 SoCs and have supporting tools.
+ [DFU](https://docs.u-boot.org/en/latest/usage/dfu.html), - as an alternative to SDP.

SPL attempts SDP first and falls back to DFU if necessary. For SDP file transfers, I use the [`imx_usb`](https://github.com/boundarydevices/imx_usb_loader) tool.

To test kernel booting, I prepared the following binaries:
+ `rk3328_uboot_loader.bin` - consisting of `u-boot-tpl.bin` and `u-boot-spl.bin` created with `boot_merger` tool.
+ `u-boot.itb` - a FIT image containing ATF, `u-boot.bin` and STB (mainline U-Boot v2023.10).
+ `kernel.itb` - a FIT image containing `Image.gz` and DTB (mainline Linux kernel v6.12)

I also created a proper SD card image from the above binaries and wrote it to an SD card for testing.

## The issue

For testing, I used the Olimex RK3328-SOM-EVB and RK3318 TV box boards. Both exhibit the same behavior.

The test procedure was:

1. Upload the `rk3328_uboot_loader.bin` via `rkdeveloptool`
2. Inserted the SD card
3. Uploaded `u-boot.itb` via `imx_usb`.
4. Boot the kernel from SD card with:
    ```
    setenv bootargs root=/dev/mmcblk1p2 console=ttyS2,1500000 earlycon ro rootwait; mmc dev 1; load mmc 1 0x800800 kernel.itb; bootm 0x800800
    ```
The kernel begins booting but hangs shortly after:

```
...
[    0.016620] smp: Bringing up secondary CPUs ...
```

This was unexpected, since I had previously booted the board entirely from the SD card, and the kernel started without issues.

I compared the two boot paths:
```
boot path 1: rk3328_uboot_loader.bin (bootrom via OTG)-+
                                                       +- -> uboot.itb (imx_usb) -> kernel.itb (from SD card)
boot path 2: rk3328_uboot_loader.bin (bootrom via SD) -+
```
The results confirmed the issue:
+ Using `path 1` (USB BootROM path), the kernel hangs.
+ Using `path 2` (SD BootROM path), the kernel boots normally.

The most plausible explanation is that some residual SoC state left by the BootROM during USB boot differs from the SD boot path and is not properly reinitialized. This difference likely causes the kernel to hang when bringing up secondary CPUs.

This observation led me to investigate the differences between the RK3328 BootROM USB and SD boot flows, with the goal of identifying and restoring the correct SoC state in SPL before calling U-Boot proper.

## BootROM

Researching the topic, I came across the excellent work of Tom Trebisky on dumping and disassembling the [RK3399 BootROM](https://github.com/trebisky/Rockchip/tree/master/bootrom). His work inspired me to attempt the same for the RK3328. 

This repo is my effort to reverse-engineer and analyze the RK3328 BootROM in the hope of identifying the root cause of the all-USB boot issue described above.

All credit goes to Tom Trebisky. I relied on his tools and reused portions of his RK3399 BootROM annotations in areas where the BootROM code appears to be similar between RK3399 and RK3328.

## Summary

Unfortunately, despite my efforts, I was unable to determine why the USB-only boot process fails, nor did I find a viable workaround.