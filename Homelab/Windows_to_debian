
# Refurbished ThinkPad T470p: Security Audit & Debian 13 + KDE Plasma Deployment

## Overview
A refurbished Lenovo ThinkPad T470p arrived with a previous owner's account still active and no audit trail on the refurbishment process. Rather than trust an unknown configuration, I treated it as a potentially compromised endpoint and performed a full wipe-and-rebuild onto a hardened, purpose-built Linux environment for home-lab and CTF work.

**Hardware**
- Lenovo ThinkPad T470p
- Intel Core i5-7440HQ
- Intel HD Graphics 630 (integrated) + NVIDIA GeForce 940MX (discrete)
- 16GB RAM, 256GB SSD (both independently upgradeable — dual SODIMM slots, M.2 + 2.5" SATA bay)
- Intel Dual Band Wireless-AC 8265
- Synaptics/Validity fingerprint reader

## 1. Pre-Install Security Posture
The laptop booted directly into an already-configured Windows account. Given no provenance on the refurbishment process, the safer control was a **clean reinstall** rather than an in-place audit of an unknown OS state — auditing can miss a persistence mechanism; a full wipe cannot.

A secondary finding during initial power-on: WiFi showed no available networks. Diagnosis ruled out a missing or blacklisted card (confirmed via Device Manager that the Intel AC 8265 was present and legitimate, not a swapped/unauthorized part) and traced the issue to the WiFi radio toggle — a common refurb shipping state, not a hardware fault.

## 2. Distribution & Desktop Environment Selection
Target use case: daily-driver and home-lab host, with a separate live/persistent Kali USB handling active pentest tooling (documented separately).

**Decision matrix considered:** Arch, Debian, Parrot OS
- **Arch** — maximal learning value, but rolling-release instability is a poor fit for a machine that needs to reliably support coursework and lab VMs
- **Parrot OS** — Debian-stable base with pre-bundled security tooling; a strong option but heavier by default
- **Debian 13 (Trixie)** — selected. Rock-stable, well-documented, and the upstream base both Kali and Parrot build from — working directly in Debian builds transferable knowledge of that shared foundation

**Desktop environment:** KDE Plasma, chosen over GNOME (heavier, more opinionated workflow) and weighed against XFCE (lighter but less customizable). With 16GB RAM earmarked for running lab VMs, Plasma's ~700MB–1GB idle footprint was an acceptable tradeoff for a materially better daily-use experience.

## 3. Installation
Used the **DVD-1 ISO** rather than netinst — the Alfa AWUS USB WiFi adapter's Realtek chipset was a known risk for not being recognized by the installer's default driver set, so an installer with zero network dependency removed that failure point entirely.

Key install decisions:
- Guided partitioning, single partition, targeting the internal SSD
- `linux-image-amd64` metapackage (tracks the stable kernel automatically)
- Generic kernel flavor (not hardware-targeted)
- KDE Plasma Desktop selected in place of default GNOME
- Full package set installed cleanly from DVD-1 with no network mirror required

## 4. Post-Install Recovery: Missing User Account
First boot login failed — the installer had captured a hostname but never finalized a user/password pair. Resolved via standard Linux recovery flow rather than reinstalling from scratch:

1. Interrupted GRUB at boot, edited the boot entry, and appended `init=/bin/bash` to drop into a root shell pre-login
2. Remounted root read-write: `mount -o remount,rw /`
3. Confirmed no standard user existed (`grep 1000 /etc/passwd` returned nothing — only system accounts present)
4. Created a proper account and granted sudo:
   ```
   adduser netrunner
   usermod -aG sudo netrunner
   exec /sbin/init
   ```
5. Note: the display manager (SDDM) blocks direct root login by default even with a valid root password — confirming the need for a standard sudo-capable account rather than relying on root for daily use.

## 5. Post-Install Fix: Package Repositories
`apt update`/`apt install` reported every package as missing. Root cause: `/etc/apt/sources.list` contained only a **commented-out DVD entry** — zero active repositories, so apt had nothing to query.

Resolved by backing up the original file and replacing it with the standard Debian 13 mirror set:
```
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
```

## Outcome
- Debian 13 + KDE Plasma running as the daily driver, fully patched
- Internal WiFi (Intel AC 8265) functional and connected
- Standard sudo user account configured correctly
- Online repositories active and verified

## Skills Demonstrated
Threat-conscious handling of unverified hardware · Linux installation and partitioning · GRUB-level recovery and root-shell troubleshooting · APT package management diagnostics · informed OS/DE selection based on operational requirements rather than default assumptions
