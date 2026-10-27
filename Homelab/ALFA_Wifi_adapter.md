# Building and Patching the RTL8812AU Driver for an Alfa USB WiFi Adapter

## Overview
An Alfa Network USB WiFi adapter (Realtek RTL8812AU/8821AU chipset) is a standard tool for wireless security work — it supports monitor mode and packet injection, which most built-in laptop WiFi cards don't. The chipset isn't supported by Debian's in-tree kernel driver, so getting it working required building the community-maintained `aircrack-ng/rtl8812au` driver from source via DKMS, and patching a kernel-API incompatibility along the way.

## 1. Diagnosing the Missing Driver
Confirmed the adapter was detected at the USB level but had no interface bound to it:
```
lsusb
# Bus 002 Device 001: ID 0bda:0811 Realtek Semiconductor Corp. RTL8812AU/8821AU 802.11ac WLAN Adapter

ip a
# no wlx... interface present
```
This confirmed a driver gap rather than a hardware or USB fault — the kernel recognized the device descriptor but had no module capable of claiming it.

## 2. Build Environment Setup
Installed the toolchain needed to compile and register an out-of-tree kernel module:
```
sudo apt install dkms git build-essential linux-headers-$(uname -r)
```
- `build-essential` provides the C compiler and `make`
- `linux-headers-$(uname -r)` supplies headers matching the exact running kernel, required for any kernel module compilation
- `dkms` registers the module so it automatically rebuilds against future kernel updates, rather than breaking on the next `apt upgrade`

## 3. Initial Build Failure
```
git clone https://github.com/aircrack-ng/rtl8812au.git
cd rtl8812au
sudo make dkms_install
```
The build failed with a kernel API mismatch:
```
error: initialization of 'int (*)(struct wiphy *, struct net_device *, struct cfg80211_chan_def *)'
from incompatible pointer type 'int (*)(struct wiphy *, struct cfg80211_chan_def *)'
.set_monitor_channel = cfg80211_rtw_set_monitor_channel,
```

**Root cause:** the upstream Linux `cfg80211` API changed the signature of `set_monitor_channel` to include a `struct net_device *` parameter. The driver source already accounted for this, but gated the change behind a version check for kernel `6.13.0`:
```c
static int cfg80211_rtw_set_monitor_channel(struct wiphy *wiphy
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 13, 0))
    , struct net_device *dev
    , struct cfg80211_chan_def *chandef
#elif (LINUX_VERSION_CODE >= KERNEL_VERSION(3, 8, 0))
    , struct cfg80211_chan_def *chandef
...
```
Debian's running kernel (`6.12.101+deb13-amd64`) already carried this upstream API change as a backport, ahead of the plain upstream version numbering the driver's conditional was written against — so the `#if` branch picked the old (incompatible) signature despite the kernel actually requiring the new one.

## 4. Patch
Lowered the version-gate threshold in `os_dep/linux/ioctl_cfg80211.c` to match the kernel's actual ABI rather than its nominal version number:
```c
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0))
```
Cleared the failed build attempt and rebuilt:
```
sudo dkms remove -m 8812au -v 5.6.4.2_35491.20191025 --all
sudo make dkms_install
```
Build completed successfully, module signed and installed:
```
dkms status -m 8812au
# 8812au/5.6.4.2_35491.20191025, 6.12.101+deb13-amd64, x86_64: installed
```

## 5. Validation
```
sudo modprobe 88XXau
ip a
# wlx00c0cab9ac20: ... state DORMANT   ← adapter now recognized
```
Confirmed full monitor-mode capability:
```
sudo airmon-ng check kill
sudo airmon-ng start wlx00c0cab9ac20
iwconfig
# Mode:Monitor confirmed
```
Installed the `aircrack-ng` suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`) and validated the full capture path by monitoring ambient 802.11 traffic with Wireshark on the now-active monitor-mode interface.

## Outcome
A fully functional RTL8812AU driver, built from source, patched for kernel-ABI compatibility, and registered with DKMS so it survives future kernel updates automatically. Adapter confirmed capable of monitor mode and packet capture, validated against the `aircrack-ng` toolchain.

## Skills Demonstrated
Reading and diagnosing C compiler errors against kernel API changes · DKMS-based out-of-tree kernel module management · patching third-party driver source for kernel-version compatibility · wireless adapter monitor-mode validation workflow
