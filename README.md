# RT-Kernel
How to compile 64-bit RT-kernel for Raspberry Pi 5 for Debian bookworm onwards incl. trixie (i.e., /boot/firmware/)

## Prepare the environment
```bash
sudo apt install git bc bison flex libssl-dev make
sudo apt install libncurses5-dev
sudo apt install raspberrypi-kernel-headers

mkdir ~/kernel
```

## Clone the git, in this case kernel 7.2, from from https://github.com/raspberrypi/linux/tree/rpi-7.2.y
```bash
cd ~
git clone --depth 1 --branch rpi-7.2.y https://github.com/raspberrypi/linux
```

## *NEW: starting with linux kernel 6.12, the RT-patch is rolled into the mainline codebase for ARM64 architexture (and some others), so no need to apply RT-patches anymore!*

## Update if necessary while scrapping all your local stuff
```bash
git stash
git pull --rebase
#git stash clear
```
P.S.: If resetting and updating your local (git-) environment with the last two steps does not work for any reason, you can always run `sudo rm -rd ~/linux` to start from scratch @ https://github.com/by/RT-Kernel?tab=readme-ov-file#clone-the-git-in-this-case-kernel-71-from-from-httpsgithubcomraspberrypilinuxtreerpi-71

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
# === DISABLES: tracing/debugging instrumentation ===
# Removes per-event hooks, ring buffers, and probe points from hot paths.
# Real win for an RT/NTP system: even when "off" at runtime, compiled-in
# tracers add NOPs and conditional branches to scheduler, IRQ, and syscall
# paths that translate to small but measurable jitter.

# CONFIG_BLK_DEV_IO_TRACE is not set
#   Block-layer I/O tracing (blktrace). Adds tracepoints in the block
#   submission path. Removing it cuts hooks from disk I/O — minor on an
#   NTP server (little disk activity), but cost-free to remove.

# CONFIG_FTRACE is not set
#   Master switch for the function tracer infrastructure. The big one.
#   With FTRACE on, every kernel function gets a callable mcount/fentry
#   stub, and there's a global tracing buffer infrastructure. Off, those
#   stubs become true NOPs and the per-CPU trace ring buffers don't exist.
#   Concrete latency win, especially under PREEMPT_RT.

# CONFIG_FTRACE_SYSCALLS is not set
#   Syscall enter/exit tracepoints. Adds a hook to every syscall.
#   Off = small but real reduction in syscall latency.

# CONFIG_FUNCTION_PROFILER is not set
#   Per-function call counting via debugfs. Inert when not active, but
#   no reason to carry it.

# CONFIG_SCHED_TRACER is not set
# CONFIG_STACK_TRACER is not set
#   Scheduler-event tracing and max-stack-depth tracker. Both touch the
#   scheduler, which an RT/NTP system runs constantly. Removing them
#   keeps the scheduler hot path lean.

# === DISABLES: kernel debuggers ===
# Pure code/size removal. No runtime cost when off, but they enable
# entry points (NMI, sysrq) that you don't want on a production server.

# CONFIG_KDB_KEYBOARD is not set
# CONFIG_KGDB is not set
# CONFIG_KGDB_KDB is not set
#   In-kernel debugger and KDB keyboard frontend. Useless on a headless
#   NTP server. Removing them shrinks the image.

# === DISABLES: cpufreq governors you won't use ===
# These are NOT a performance win — they're just dead code if you never
# switch governors. Removing them shrinks the kernel image slightly and
# makes intent explicit. The ONLY one that matters at runtime is the
# DEFAULT, which you set to PERFORMANCE below.

# CONFIG_CPU_FREQ_DEFAULT_GOV_ONDEMAND is not set
#   Stops ONDEMAND from being the boot-time default. Critical: ONDEMAND
#   ramps frequency based on load, which means CPU-frequency transitions
#   during NTP packet handling — a known source of timing jitter.

# CONFIG_CPU_FREQ_GOV_CONSERVATIVE is not set
# CONFIG_CPU_FREQ_GOV_POWERSAVE is not set
# CONFIG_CPU_FREQ_GOV_SCHEDUTIL is not set
# CONFIG_CPU_FREQ_GOV_USERSPACE is not set
#   Removes the *option* to switch to these. Pure cleanup, no perf delta.

# === DISABLES: drivers you don't have ===
# Image-size cleanup. Zero runtime impact.

# CONFIG_IR_GPIO_TX is not set
#   Infrared LED transmit driver. No IR hardware on an NTP server.

# CONFIG_LEDS_TRIGGER_CPU is not set
#   Drives LEDs based on CPU activity. Adds a hook in the idle path.
#   Removing it eliminates one tiny idle-path side trip — negligible
#   but cost-free.

# CONFIG_NET_ACT_CTINFO is not set
#   tc action that copies conntrack info to packet metadata. Nobody uses
#   this on a stratum-1 NTP server. Removing it shrinks the network
#   action module list.

# === DISABLES: virtualization ===
# Real win: removes a substantial code path entirely.

# CONFIG_KVM is not set
# CONFIG_VIRTUALIZATION is not set
#   Removes the KVM hypervisor and virt support. On bare-metal arm64,
#   this enables some VHE/EL2 handling on every interrupt and exception
#   even when no VMs are running. Off = a cleaner exception path. Modest
#   win in absolute terms, but it's continuous, so it adds up.

# === DISABLES: misc instrumentation ===

# CONFIG_LATENCYTOP is not set
#   Records latency sources in scheduler decisions. Ironic: the tool to
#   measure latency adds latency. Adds work to context switches —
#   exactly the path RT cares most about. Real win.

# CONFIG_CONTEXT_TRACKING_USER_FORCE is not set
#   Forces kernel/user transition tracking on all CPUs unconditionally.
#   Used by NO_HZ_FULL; useless when NO_HZ_IDLE is also off (HZ_PERIODIC).
#   Removing it avoids per-syscall bookkeeping. Small win, fits the
#   "predictable timing over power saving" theme.

# CONFIG_NO_HZ_IDLE is not set
#   This is the headline NTP-specific item. NTP_PPS (kernel hardpps)
#   requires !NO_HZ_COMMON, which means NO_HZ_IDLE must be off and the
#   kernel runs HZ_PERIODIC (constant 1000 Hz timer tick). Trade-off:
#   slightly higher idle power consumption in exchange for kernel-level
#   PPS discipline of the system clock. For a wall-powered NTP server,
#   this is the right call.

# === ENABLES: NTP-specific timing core ===

CONFIG_HZ_1000=y
#   1000 Hz scheduler tick (1ms granularity). Default on the Pi defconfig
#   is often 250 Hz. 1000 Hz means the scheduler can preempt within 1ms
#   of an event becoming runnable, which directly improves PPS jitter
#   floor and chrony's ability to discipline within a sub-ms window.
#   Real, measurable win for NTP precision.

CONFIG_PREEMPT_RT=y
#   The big one. Converts most kernel spinlocks to rt_mutexes, makes
#   nearly all interrupt handlers preemptible (threaded IRQs), and
#   makes most code paths preemptible by higher-priority tasks. For NTP,
#   the value is in worst-case latency: a non-RT kernel can have
#   multi-millisecond latency spikes from non-preemptible sections that
#   blow up PPS measurement variance. RT compresses the long tail of
#   the latency distribution. The mean latency may actually be slightly
#   *worse* under RT (more locking overhead), but the 99.99th percentile
#   is dramatically better — and that's what NTP timing depends on.

CONFIG_PPS_CLIENT_GPIO=y
#   Built-in (not module) GPIO PPS input driver. Reads the 1 PPS pulse
#   from your GPS receiver via a Pi GPIO pin. =y instead of =m means
#   it's available before module loading during boot, so PPS works from
#   the earliest userspace.

CONFIG_NTP_PPS=y
#   Enables hardpps() — kernel-level PPS discipline of CLOCK_REALTIME.
#   Allows ntp_adjtime() with PPS flags to feed the system clock
#   directly from the PPS source, bypassing some userspace round-trip.
#   Whether you actually use this depends on your chrony/ntpd
#   configuration; modern chrony does excellent userspace PPS handling.
#   But having the kernel path available costs nothing.

# === ENABLES: governor + accounting ===

CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y
#   Sets PERFORMANCE as the default cpufreq governor at boot. PERFORMANCE
#   pins the CPU at maximum frequency — no DVFS transitions, no entry
#   into deep idle states for frequency reasons. This is the single
#   biggest cpufreq-related jitter reduction available. Trade: higher
#   idle power, slightly higher temperature. For an always-on NTP
#   server, worth it.

CONFIG_VIRT_CPU_ACCOUNTING_GEN=y
#   Higher-resolution CPU time accounting (uses sched_clock instead of
#   tick-based sampling). Improves the accuracy of /proc/stat,
#   getrusage(), and per-task time accounting. Honest take: this is
#   nice-to-have for monitoring, NOT a measurable win for NTP precision
#   per se. It does add a tiny per-context-switch cost. You could drop
#   it and not measure a difference. Keep if you want accurate
#   per-process CPU stats from monitoring.

# === ENABLES: useful operational features (no perf benefit) ===
# Honest disclosure: these don't help NTP precision. They're for
# you, the operator. Including them in the fragment is a values
# choice — debuggability vs. minimalism.

CONFIG_KPROBES=y
#   Dynamic instrumentation: lets you attach probes to kernel functions
#   at runtime (e.g., via bpftrace) for diagnostics. Useful when chasing
#   a weird latency spike, useless if you never debug. The infrastructure
#   is dormant when no probes are active — no measurable runtime cost.

CONFIG_MAGIC_SYSRQ=y
#   Emergency keyboard/serial commands (Alt-SysRq-B reboots, etc.).
#   For a remote NTP server, accessible via serial console — invaluable
#   when something is hung. Zero runtime cost. Pure operability.

CONFIG_RELAY=y
#   Kernel→userspace high-bandwidth relay channels. Used by some tracing
#   tools (blktrace, ftrace's snapshot mode). Honest take: with FTRACE
#   off, RELAY has very few consumers left in your kernel. You could
#   probably drop this; I left it because Pi userspace tooling
#   sometimes expects it. Negligible code size, zero runtime cost.

# === Identification ===

CONFIG_LOCALVERSION="-v8-16k-NTP"
#   Appends "-NTP" to `uname -r`. Lets you see at a glance whether you
#   booted the RT/NTP kernel vs. stock. No functional impact — pure
#   ergonomics.
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
sudo mkinitramfs -o /boot/firmware/NTP/initramfs_2712-NTP $(make -s kernelrelease)
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
sudo SKIP_KERNEL=1 PRUNE_MODULES=1 rpi-update rpi-7.2.y
```
and if it tells you about potential issues with using custom ```ìnitramfs```, then just  regenerate yours gain (see https://github.com/by/RT-Kernel/edit/main/README.md#regenerate-iniramfs-for-your-custom-kernel).

## Build status for official rpi-7.2.y from https://github.com/raspberrypi/linux:
[![Pi kernel build tests](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml/badge.svg?branch=rpi-7.2.y)](https://github.com/raspberrypi/linux/actions/workflows/kernel-build.yml)

[![dtoverlaycheck](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml/badge.svg?branch=rpi-7.2.y)](https://github.com/raspberrypi/linux/actions/workflows/dtoverlaycheck.yml)

## A short note on recent PPS-performance enhancing kernel commits

~~You may have noticed that PPS is not performing stelarly on your Pi5; part of the reason is that there is no longer a direct connection to the UART but only via the new RP1 chip, which adds a bit of latency, but also to a bug in the custom kernel code which prevents RP1 GPIO IRQ to follow a given smp_affinity; you can read more about it here https://github.com/raspberrypi/linux/issues/7301 and find a propsoed fix here: https://github.com/raspberrypi/linux/pull/7302. – I hope that my commit will ultimately make it into the Raspberry custom kernel.~~
Fixes are now available in 7.0ff.: https://github.com/raspberrypi/linux/commit/30f29f86ebc8343b049361187109133a83135b11 and https://github.com/raspberrypi/linux/commit/8d5acfef4c6dd1c38ca609e353bbd9fc10f0a166

On the other hand, the performance of kernel PPS can be significantly enhanced when running under PREEMPT_RT, as there is unnecessary jitter introduce with the current implementation; I've proposed a kernel patch upstream, you can find it here until successfully merged: [https://github.com/torvalds/linux/compare/master...by:linux-PPS:pps-rt-v3](https://github.com/by/linux-PPS/tree/pps-rt-v7-clean) – and with a bit of luck, we'll see it in one of the next kernel versions (https://lore.kernel.org/lkml/20260602063650.LWatIBPk@linutronix.de/)

Once one or both patches are merged, I will delete the respective line item(s).
