# RT-Kernel
How to compile 64-bit RT-kernel for Raspberry Pi 5 for Debian bookworm (i.e., /boot/firmware/)

## Prepare the environment
```bash
sudo apt install git bc bison flex libssl-dev make
sudo apt install libncurses5-dev
sudo apt install raspberrypi-kernel-headers

mkdir ~/kernel
```
## Clone the git, in this case kernel 6.11, from from https://github.com/raspberrypi/linux/tree/rpi-6.11.y
```bash
cd ~
git clone --depth 1 --branch rpi-6.11.y https://github.com/raspberrypi/linux
```
## Get the latest RT-patch from https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/, in this case RT3 for kernel 6.11-rc3, from https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.11/ respectively
```bash
cd ~/kernel
wget -c https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.11/patch-6.11-rc3-rt3.patch.xz
xz -d patch-6.11-rc3-rt3.patch.xz
```
## Go back into the cloned linux
```bash
cd ~/linux
```
## Undo prior patch, if necessary, in this case the one for 6.11-rc3-rt2 – please perform this only, if you've applied a patch before and the respective kernel patch is still available in ~/kernel!
```bash
#patch -R -p1 < ~/kernel/patch-6.11-rc3-rt2.patch
```
## Update if necessary while scrapping all your local stuff
```bash
git stash
git pull --rebase
#git stash clear
```
P.S.: If resetting and updating your local (git-) environment with the last two steps does not work for any reason, you can always run `sudo rm -rd ~/linux` to start from scratch @ https://github.com/by/RT-Kernel?tab=readme-ov-file#clone-the-git-in-this-case-kernel-610-from-from-httpsgithubcomraspberrypilinuxtreerpi-610y
## Or simply pull
```bash
#git pull
```
## Patch the kernel
```bash
patch -p1 < ~/kernel/patch-6.11-rc3-rt3.patch
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
-CPU_FREQ_DEFAULT_GOV_POWERSAVE y
-CPU_FREQ_GOV_CONSERVATIVE y
-CPU_FREQ_GOV_ONDEMAND y
-CPU_FREQ_GOV_PERFORMANCE y
-CPU_FREQ_GOV_SCHEDUTIL y
-CPU_FREQ_GOV_USERSPACE y
-LEDS_TRIGGER_CPU y
-PREEMPT y
 LOCALVERSION "-v8-16k" -> "-v8-16k-NTP"
 PPS_CLIENT_GPIO m -> y
+CPU_FREQ_DEFAULT_GOV_PERFORMANCE y
+EFI_DISABLE_RUNTIME n
+HZ_1000 y
+HZ_PERIODIC y
+NTP_PPS y
+PREEMPT_RT y
+RTC_INTF_DEV_UIE_EMUL y
+STRICT_DEVMEM n
```
See also https://github.com/by/RT-Kernel/blob/main/bcm2712_defconfig_RT_NTP

## Build the kernel using all 4 cores (and try gcc optimization level -O3, if you like)
```bash
make prepare
make -j4 Image.gz modules dtbs
make CFLAGS='-O3 -march=native' -j4 Image.gz modules dtbs
sudo make modules_install
```
## Create the required directories once
```bash
sudo mkdir /boot/firmware/NTP
sudo mkdir /boot/firmware/NTP/overlays-NTP
```
## Add this to /boot/firmware/config.txt in order to preserve the standard kernel
```bash
os_prefix=NTP/
overlay_prefix=overlays-NTP/
kernel=/kernel_2712-NTP.img
```
## Copy the file ino the right directories
```bash
sudo cp arch/arm64/boot/dts/broadcom/*.dtb /boot/firmware/NTP/; sudo cp arch/arm64/boot/dts/overlays/*.dtb* /boot/firmware/NTP/overlays-NTP/; sudo cp arch/arm64/boot/dts/overlays/README /boot/firmware/NTP/overlays-NTP/; sudo cp arch/arm64/boot/Image.gz /boot/firmware/kernel_2712-NTP.img
```
## Reboot to activate the kernel
```bash
sudo reboot now
```
## Update the firmware (but not the standard kernel)
```bash
sudo SKIP_KERNEL=1 PRUNE_MODULES=1 rpi-update rpi-6.11.y
```
