---
title: Installing drivers for Tenda W311MI AX300 WiFi USB Adapter on (Any?) Linux
date: 2023-12-29 18:39:26+11:00
categories: [linux, guides]
tags: [linux, wifi, drivers, tenda, w311mi, w311miv6, usb, arm, kernel, debian, ubuntu, mint, arch, archlinux]
image: /assets/img/2023-12/tenda-W311MI.webp
---

Tenda provides Debian DEB packages for their W311MIv6.0 adapter [here](https://www.tendacn.com/download/detail-4785.html), but you'll find these packages no longer work on recent kernels. The fix is quite easy though. This post will cover how to install the kernel module on a generic Linux distribution. I will be using Arch Linux.

## Unpacking

Start by downloading `W311MIV6_Linux_Driver_20230612_English.zip`. In case it is no longer available, I have mirrored it [here](https://github.com/selenologist/blog-file-mirrors/raw/main/2023-12/W311MIV6_Linux_Driver_20230612_English.zip).

When you extract the ZIP, you should see these files:
```bash
[luna@rainbow tenda]$ unzip ~/Downloads/W311MIV6_Linux_Driver_20230612_English.zip
Archive:  /home/luna/Downloads/W311MIV6_Linux_Driver_20230612_English.zip
   creating: W311MIV6_Linux_Driver_20230612_English/
   creating: W311MIV6_Linux_Driver_20230612_English/ARM/
 extracting: W311MIV6_Linux_Driver_20230612_English/ARM/w311miv6-pkg-v1.0.0-arm64.deb
  inflating: W311MIV6_Linux_Driver_20230612_English/W311MIV6.0╬▐╧▀═°┐иLinux╧╡═│╙в╬─░▓╫░╓╕─╧.pdf
   creating: W311MIV6_Linux_Driver_20230612_English/X86/
 extracting: W311MIV6_Linux_Driver_20230612_English/X86/w311miv6-pkg-v1.0.0-amd64.deb
```

Despite separate "Debian binary packages" for ARM and X86, these packages only differ in the label in their Debian control section. The packages actually compile kernel modules and binaries from source code. I will not cover installing dependencies needed to compile kernel modules in this post. You will need a compiler and kernel headers.

Debian binary packages are Unix 'ar' archives that include control and data 'tar' archives, as well as a small marker file containing the debian package format version.
Let's unpack it and have a look at the control archive to see the main part of how it works.
```bash
[luna@rainbow tenda]$ cd W311MIV6_Linux_Driver_20230612_English/X86/
[luna@rainbow X86]$ ar vx w311miv6-pkg-v1.0.0-amd64.deb
x - debian-binary
x - control.tar.gz
x - data.tar.gz
[luna@rainbow X86]$ mkdir control
[luna@rainbow X86]$ cd control
[luna@rainbow control]$ tar xavf ../control.tar.gz
./
./control
./postinst
./postrm
./preinst
./prerm
[luna@rainbow control]$ cat control
package:W311MIV6-pkg
version:1.0.0
architecture:amd64
maintainer:liliu
description:aic8800 wifi driver deb package
```

In `control` you can see the architecture field I mentioned before. In the "ARM" package it says armhf or something similar - the data archive is identical. We're going to ignore this file and just follow the `postinst` script ourselves.

## Attempt to install...

```bash
[luna@rainbow control]$ cat postinst
#!/bin/bash
Main_version=`uname -r |awk -F'.' '{print $1}'`
Minor_version=`uname -r |awk -F'.' '{print $2}'`
cp /AIC8800/aic.rules /etc/udev/rules.d
udevadm trigger
udevadm control --reload
echo "udev done"
if [ -L /dev/aicudisk ]; then
echo "device exist"
eject /dev/aicudisk
else
echo "device not exist"
fi
cd /AIC8800/fw/
rm -rf /lib/firmware/aic8800DC
cp -rf /AIC8800/fw/aic8800DC /lib/firmware/
echo "cp fw done"
cd /AIC8800/drivers/aic8800/
make
make install
modprobe cfg80211w
insmod /AIC8800/drivers/aic8800/aic_load_fw/aic_load_fw.ko
insmod /AIC8800/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko
echo "insmod done"
cd /AIC8800/aicrf_test/
make
make install
echo "Install aic8800 wifi driver successful!!!!!"
exit 0
```

`postinst` is where everything really happens. Firstly it obtains your kernel version numbers from `uname`, but I'm not sure why since it never reads these values again later. Then it copies a udev rule to your system udev rules directory, before reloading udev to apply the rule. This rule just turns off that silly Windows driver install disk when the device is plugged - which is also what the next few lines of the script do (So much for driverless, huh?). It's up to you whether or not you run these lines.

But wait, where is /AIC8800/? When Debian packages are installed they execute in a chroot containing files from the `data.tar.gz` archive. Let's extract that into a new directory now, then we'll just remove the leading "/" from the start of everything in the script. We don't need any files from the control directory anymore, so we can leave it.

```bash
[luna@rainbow control]$ cd ..
[luna@rainbow X86]$ mkdir data
[luna@rainbow X86]$ cd data
[luna@rainbow data]$ tar xavf ../data.tar.gz
./
./AIC8800/
./AIC8800/aic.rules
./AIC8800/aicrf_test/
./AIC8800/aicrf_test/.gitignore
./AIC8800/aicrf_test/Android.mk
./AIC8800/aicrf_test/Makefile
./AIC8800/aicrf_test/aicrf_test.c
./AIC8800/aicrf_test/bt_test.c
./AIC8800/aicrf_test/wifi_test.c
...
lots of files
...
```

We'll continue at the `rm -rf /lib/firmware/aic8800DC` line of the postinst script. You'll need to perform these next two commands as root. They remove any previous firmware files, then copy over new ones.
```bash
[luna@rainbow data]$ sudo rm -rf /lib/firmware/aic8800DC
[luna@rainbow data]$ sudo cp -rf AIC8800/fw/aic8800DC /lib/firmware/
```

Now we're going to try to compile the kernel driver. We'll try it unmodified on a recent kernel, just to show you what happens.
```bash
[luna@rainbow data]$ cd AIC8800/drivers/aic8800/
[luna@rainbow aic8800]$ make
```

## Something's wrong! The Fix:

After a bit of a wait, you should eventually see an error message complaining about an undefined reference to `REGULATORY_IGNORE_STALE_KICKOFF` on line 1711 of `aic8800_fdrv/rwnx_mod_params.c`. This flag has been removed in recent kernels.

> While I did try to find out what exactly went down... I didn't have much luck. So far as I can tell there's no equivalent and it's OK to simply comment out that line, as the code that would have read the flag elsewhere has been removed. Please let me know using the comment system below if that's not the case.
{: .prompt-warning }

For now, let's just patch out that line:

```patch
[luna@rainbow aic8800]$ cat << EOF | patch aic8800_fdrv/rwnx_mod_params.c
--- aic8800_fdrv/rwnx_mod_params.c          2023-12-29 19:23:09.424648079 +1100
+++ aic8800_fdrv/rwnx_mod_params.c.patched  2023-12-29 19:23:09.484651875 +1100
@@ -1708,7 +1708,7 @@
     if (!rwnx_hw->mod_params->custregd)
         return;

-    wiphy->regulatory_flags |= REGULATORY_IGNORE_STALE_KICKOFF;
+    //wiphy->regulatory_flags |= REGULATORY_IGNORE_STALE_KICKOFF;
     wiphy->regulatory_flags |= REGULATORY_WIPHY_SELF_MANAGED;

     rtnl_lock();
EOF
```

and from then on, we should be able to follow along with the original script.

## Finishing up

```bash
[luna@rainbow aic8800]$ make
[luna@rainbow aic8800]$ sudo make install
[luna@rainbow aic8800]$ modprobe cfg80211w
[luna@rainbow aic8800]$ insmod aic_load_fw/aic_load_fw.ko
[luna@rainbow aic8800]$ insmod aic8800_fdrv/aic8800_fdrv.ko
```

It's up to you whether you also compile and install the test binaries as per the rest of the script. They're not needed to use the device normally.

The network link should now be visible with `ip link`. It should now be able to be configured like any other wireless network interface. If yes, congrats. If not, try unplugging it and replugging it, or rebooting.

## What Tenda could have done differently

For a start, working on upstreaming your driver so that it's supported by the mainline kernel and doesn't require driver installation would be the best option.

But failing that, the script needs better error handling. When `make` fails, the script continues on without reporting anything to the user, especially with typical graphical frontends. The user has literally no feedback that it didn't work on Ubuntu. I had to manually run things to find out it went wrong. If I didn't know how the sausage was made, I wouldn't have known why nothing was working.