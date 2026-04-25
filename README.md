# RT-Kernel
How to compile 64-bit RT-kernel for Raspberry Pi 5 for Debian bookworm onwards incl. trixie (i.e., /boot/firmware/)

## Prepare the environment
```bash
sudo apt install git bc bison flex libssl-dev make
sudo apt install libncurses5-dev
sudo apt install raspberrypi-kernel-headers

mkdir ~/kernel
```

## Clone the git, in this case kernel 7.0, from from https://github.com/raspberrypi/linux/tree/rpi-7.0.y
```bash
cd ~
git clone --depth 1 --branch rpi-7.0.y https://github.com/raspberrypi/linux
```

## *NEW: starting with linux kernel 6.12, the RT-patch is rolled into the mainline codebase for ARM64 architexture (and some others), so no need to apply RT-patches anymore!*

## Update if necessary while scrapping all your local stuff
```bash
git stash
git pull --rebase
#git stash clear
```
P.S.: If resetting and updating your local (git-) environment with the last two steps does not work for any reason, you can always run `sudo rm -rd ~/linux` to start from scratch @ https://github.com/by/RT-Kernel?tab=readme-ov-file#clone-the-git-in-this-case-kernel-70-from-from-httpsgithubcomraspberrypilinuxtreerpi-70

## Or simply pull
```bash
#git pull
```

## Make for Raspberry Pi 5
```bash
make bcm2712_defconfig
```

## Select General Setup/Preemption Model/Fully Preemptible Kernel (Real-Time) with some optimizations (getting rid of some tracer and debugging)
```bash
## I've made the following kernel patches specifically for my NTP server to optimize performance and also enable kernel PPS (in fragment form!):
CONFIG_LOCALVERSION="-v8-16k-NTP"
CONFIG_PPS_CLIENT_GPIO=y
CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y
CONFIG_HZ_1000=y
CONFIG_NTP_PPS=y
CONFIG_PREEMPT_RT=y
```
For more optimization, please see  https://github.com/by/RT-Kernel/blob/main/RT_NTP.config

## Copy the patched fragment from https://github.com/by/RT-Kernel/blob/main/RT_NTP.config and overlay onto the existing .config
```bash
cp ~/bcm2712_defconfig_RT_NTP_diff ~/linux/kernel/configs/RT_NTP.config
make RT_NTP.config
```

## Start menuconfig for further manual patches
```bash
make menuconfig
# Enter whatever you deem appropriate via the menuconfig UI
```

## Build the kernel using all cores
```bash
make prepare
make -j6 Image.gz modules dtbs # recommendation is 1.5 times the number of cores (=4), which equals 6 -- if you have enough main memory!
sudo make -j6 modules_install  # recommendation is 1.5 times the number of cores (=4), which equals 6
```

## Create the required directories once
```bash
sudo mkdir /boot/firmware/NTP
sudo mkdir /boot/firmware/NTP/overlays
```

## Create a copy of the kernel-specific parameter file cmdline.txt
```bash
sudo cp -v /boot/firmware/cmdline.txt /boot/firmware/NTP/cmdline.txt
```
The newly built kernel is now also moved into ```/boot/firmware/NTP``` and expects its own ```cmdline.txt``` there, too; upside is that you can create an RT-kernel-specific ```cmdline.txt``` right here.

## Regenerate ```iniramfs``` for your custom kernel (still in ~/linux)
```bash
sudo mkinitramfs -o /boot/firmware/NTP/initramfs_2712-NTP "$(make -s kernelrelease)"
```
and ignore the warning about not being able to check availability of zstd compression support (```CONFIG_RD_ZSTD```) due to missing kernel configuration ```/boot/config-$(uname -r)```; here, only a copy of the file in this very directory is missing, but ``ìnitramfs```correct assumes it to be available (see the respective warning message).

## Add this once to /boot/firmware/config.txt in order to preserve the standard kernel
```bash
os_prefix=NTP/
kernel=kernel_2712-NTP.img
```
and add to enable ```initramfs``` for your custom kernel
```bash
#auto_initramfs=1 as it apparently does not properly follow the changed directory os_prefix
initramfs initramfs_2712-NTP followkernel
```

## Copy the file ino the right directories
```bash
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/NTP/; sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/NTP/overlays/; sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/NTP/overlays/; sudo cp arch/arm64/boot/Image.gz /boot/firmware/NTP/kernel_2712-NTP.img
```
## Reboot to activate the kernel
```bash
sudo reboot now
```
## Update the firmware (but not the standard kernel)
```bash
sudo SKIP_KERNEL=1 PRUNE_MODULES=1 rpi-update rpi-7.0.y
```
and if it tells you about potential issues with using custom ```ìnitramfs```, then just  regenerate yours gain (see https://github.com/by/RT-Kernel/edit/main/README.md#regenerate-iniramfs-for-your-custom-kernel).

## Build status for official rpi-7.0.y from https://github.com/raspberrypi/linux:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-7.0.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)

[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-7.0.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)

## A short note on recent PPS-performance enhancing kernel commits

~~You may have noticed that PPS is not performing stelarly on your Pi5; part of the reason is that there is no longer a direct connection to the UART but only via the new RP1 chip, which adds a bit of latency, but also to a bug in the custom kernel code which prevents RP1 GPIO IRQ to follow a given smp_affinity; you can read more about it here https://github.com/raspberrypi/linux/issues/7301 and find a propsoed fix here: https://github.com/raspberrypi/linux/pull/7302. – I hope that my commit will ultimately make it into the Raspberry custom kernel.~~
Fixes are now available in 7.0ff.: https://github.com/raspberrypi/linux/commit/30f29f86ebc8343b049361187109133a83135b11 and https://github.com/raspberrypi/linux/commit/8d5acfef4c6dd1c38ca609e353bbd9fc10f0a166

On the other hand, the performance of kernel PPS can be significantly enhanced when running under PREEMPT_RT, as there is unnecessary jitter introduce with the current implementation; I've proposed a kernel patch upstream, you can find it here until successfully merged: https://github.com/torvalds/linux/compare/master...by:linux-PPS:pps-rt-v3

Once one or both patches are merged, I will delete the respective line item(s).
