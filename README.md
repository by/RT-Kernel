# RT-Kernel
How to compile 64-bit RT-kernel for Raspberry Pi 5 for Debian bookworm onwards incl. trixie (i.e., /boot/firmware/)

## Prepare the environment
```bash
sudo apt install git bc bison flex libssl-dev make
sudo apt install libncurses5-dev
sudo apt install raspberrypi-kernel-headers

mkdir ~/kernel
```
## Clone the git, in this case kernel 6.19, from from https://github.com/raspberrypi/linux/tree/rpi-6.19.y
```bash
cd ~
git clone --depth 1 --branch rpi-6.19.y https://github.com/raspberrypi/linux
```
## *NEW: starting with linux kernel 6.12, the RT-patch is rolled into the mainline codebase for ARM64 architexture (and some others), so no need to apply RT-patches anymore!*

## Update if necessary while scrapping all your local stuff
```bash
git stash
git pull --rebase
#git stash clear
```
P.S.: If resetting and updating your local (git-) environment with the last two steps does not work for any reason, you can always run `sudo rm -rd ~/linux` to start from scratch @ https://github.com/by/RT-Kernel?tab=readme-ov-file#clone-the-git-in-this-case-kernel-619-from-from-httpsgithubcomraspberrypilinuxtreerpi-619
## Or simply pull
```bash
#git pull
```
## Make for Raspberry Pi 5
```bash
make bcm2712_defconfig
```
## Start menuconfig
```bash
make menuconfig
```
## Select General Setup/Preemption Model/Fully Preemptible Kernel (Real-Time)
```bash
## I've made the following changes specifically for my NTP server to also enable kernel PPS:
sudo ~/linux/scripts/diffconfig ~/linux/arch/arm64/configs/bcm2712_defconfig ~/linux/defconfig
-CPU_FREQ_DEFAULT_GOV_ONDEMAND y
-CPU_FREQ_GOV_CONSERVATIVE y
-CPU_FREQ_GOV_POWERSAVE y
-CPU_FREQ_GOV_SCHEDUTIL y
-CPU_FREQ_GOV_USERSPACE y
-IR_GPIO_TX m
-LEDS_TRIGGER_CPU y
-NO_HZ y
-PREEMPT y
 LOCALVERSION "-v8-16k" -> "-v8-16k-NTP"
 PPS_CLIENT_GPIO m -> y
+CPU_FREQ_DEFAULT_GOV_PERFORMANCE y
+EFI_DISABLE_RUNTIME n
+HZ_1000 y
+NTP_PPS y
+PREEMPT_RT y
+RTC_INTF_DEV_UIE_EMUL y
```
See also https://github.com/by/RT-Kernel/blob/main/bcm2712_defconfig_RT_NTP

## Build the kernel using all cores (and try gcc optimization level -O3, if you like)
```bash
make prepare
make CFLAGS='-O3 -march=native' -j6 Image.gz modules dtbs # recommendation is 1.5 times the number of cores (=4), which equals 6 -- if you have enough main memory!
sudo make -j6 modules_install # recommendation is 1.5 times the number of cores (=4), which equals 6
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
The newly built kernel is now also moved into ```/boot/firmware/NTP``` and expects its own ```cmdline.txt``` there, too; upsis is that you can create an RT-kernel-specific ```cmdline.txt``` right here.

## Add this to /boot/firmware/config.txt in order to preserve the standard kernel
```bash
os_prefix=NTP/
kernel=kernel_2712-NTP.img
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
sudo SKIP_KERNEL=1 PRUNE_MODULES=1 rpi-update rpi-6.19.y
```

## Build status for official rpi-6.19.y from https://github.com/raspberrypi/linux:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-6.19.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)

[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-6.19.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)
