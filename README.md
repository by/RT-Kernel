# RT-Kernel
How to compile 64-bit RT-kernel for Raspberry Pi 4 for Debian bookworm (i.e., /boot/firmware/)

## Prepare the environment
```bash
sudo apt install git bc bison flex libssl-dev make
sudo apt install libncurses5-dev
sudo apt install raspberrypi-kernel-headers
```
## Clone the git, in this case kernel 6.7, from from https://github.com/raspberrypi/linux/tree/rpi-6.7.y
```bash
cd ~
git clone --depth 1 --branch rpi-6.7.y https://github.com/raspberrypi/linux
```
## Get the RT-patch, in this case RT1 for kernel 6.7, from https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.7/
```bash
cd ~/kernel
wget -c https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.7/patch-6.7-rc2-rt1.patch.xz
xz -d patch-6.7-rc2-rt1.patch.xz
```
## Go back into the cloned linux
```bash
cd linux
```
## Update if necessary while scrapping all your local stuff
```bash
git stash
git pull --rebase
```
## Or simply pull
```bash
#git pull
```
## Patch the kernel
```bash
patch -p1 < ~/kernel/patch-6.7-rc2-rt1.patch
```
## Undo prior patch if necessary
```bash
#patch -R -p1 < ~/kernel/patch-6.7-rc2-rt1.patch
```
## Make for Raspberry Pi 4
```bash
make bcm2711_defconfig
```
## Start menuconfig
```bash
make menuconfig
```
## Select General Setup/Preemption Model/Fully Preemptible Kernel (Real-Time)
```bash
## I've made the following changes specifically for my NTP server to also enable kernel PPS:
##-CPU_FREQ_DEFAULT_GOV_POWERSAVE y
##-CPU_FREQ_GOV_CONSERVATIVE y
##-CPU_FREQ_GOV_ONDEMAND y
##-CPU_FREQ_GOV_PERFORMANCE y
##-CPU_FREQ_GOV_SCHEDUTIL y
##-CPU_FREQ_GOV_USERSPACE y
##-LEDS_TRIGGER_CPU y
##-NO_HZ y
##-PPS_CLIENT_LDISC m
##-PREEMPT y
## LOCALVERSION "-v8" -> "-v8-NTP"
## PPS_CLIENT_GPIO m -> y
##+CONTEXT_TRACKING_USER_FORCE n
##+CPU_FREQ_DEFAULT_GOV_PERFORMANCE y
##+HZ_1000 y
##+NTP_PPS y
##+PREEMPT_RT y
##+RTC_INTF_DEV_UIE_EMUL y
##+VIRT_CPU_ACCOUNTING_GEN y
```
See also https://github.com/by/RT-Kernel/blob/main/bcm2711_defconfig_RT_NTP

## Build the kernel using all 4 cores (and try gcc optimization level -O3, if you like)
```bash
make prepare
make -j4 Image.gz modules dtbs
make CFLAGS='-O3 -march=armv8-a+crc -mtune=cortex-a72' -j4 Image.gz modules dtbs
sudo make modules_install
```
## Create the required directories once
```bash
sudo mkdir /boot/firmware/NTP
sudo mkdir /boot/NTP/firmware/overlays-NTP
```
## Add this to /boot/config.txt in order to preserve the standard kernel
```bash
os_prefix=NTP/
overlay_prefix=overlays-NTP/
kernel=/kernel8-ntp.img
```
## Copy the file ino the right directories
```bash
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/NTP/; sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/NTP/overlays-NTP/; sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/NTP/overlays-NTP/; sudo cp arch/arm64/boot/Image.gz /boot/firmware/kernel8-NTP.img
```
## Reboot to activate the kernel
```bash
#sudo reboot now
```
