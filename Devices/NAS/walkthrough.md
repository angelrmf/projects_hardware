# Building a Home Media NAS on a Raspberry Pi 4 with OpenMediaVault

## Overview
Built a small-footprint network-attached storage server for home media storage, using a Raspberry Pi 4 (4GB RAM) paired with a 1TB USB hard drive and **OpenMediaVault (OMV)** as the storage/management OS. Goal was a low-power, always-on box for centralizing and serving media files to other devices on the local network.

**Hardware**
- Raspberry Pi 4 Model B, 4GB RAM
- microSD card (boot/OS drive)
- 1TB USB hard drive (bulk storage)

## 1. OS Installation
Flashed OpenMediaVault's Raspberry Pi image to the microSD card and booted the Pi headless, completing initial setup through OMV's web-based admin interface rather than a monitor/keyboard directly on the device — standard for this kind of appliance-style deployment.

## 2. Network Connectivity Troubleshooting
Initial WiFi connection was unreliable during setup — intermittent drops/failures to associate, requiring repeated reconfiguration through OMV's network settings before the Pi held a stable connection to the local network.

## 3. Locating the Device for SSH Access
With no monitor attached and the Pi's assigned IP unknown, standard discovery methods (router admin page, hostname resolution) didn't immediately surface it. Used `nmap` to sweep the local subnet and identify the Pi by its open ports/MAC vendor signature:
```
nmap -sn 192.168.1.0/24
```
Once the correct IP was identified, SSH access was established normally:
```
ssh pi@<discovered-ip>
```
This is a common real-world first step for headless device setup when DHCP hands out an address you don't yet know — confirming the device is live and finding it is often the actual first troubleshooting step, before any configuration issues even come into play.

## 4. Storage Configuration
Connected the 1TB USB HDD and brought it under OMV's management:
- Wiped and initialized the drive through OMV's **Storage → Disks** interface
- Created a filesystem and mounted it as a managed storage volume through **Storage → File Systems**
- Configured the mounted volume as a **Shared Folder** in OMV, the logical unit that services (SMB) are then pointed at

## 5. Services Configured
- **SMB/CIFS** — enabled and configured to share the media volume across the local network, allowing other devices (laptops, smart TVs, media players) to browse and stream directly from the NAS
- **SSH** — enabled for remote administration of the Pi itself, separate from the file-sharing service, allowing command-line management without needing a monitor/keyboard attached

## 6. Network Scope
Currently configured for **local-network access only** — no port forwarding or VPN exposure to the public internet, keeping the attack surface minimal while serving its intended purpose as an in-home media store.

## Outcome
A functioning, low-power home NAS serving media over SMB to local devices, administered via SSH, built on inexpensive single-board-computer hardware rather than a dedicated NAS appliance.

## Skills Demonstrated
Headless Linux device bring-up and network discovery (`nmap` subnet scanning) · WiFi connectivity troubleshooting on embedded hardware · storage provisioning and shared-folder configuration in OpenMediaVault · SMB and SSH service configuration · scoped network exposure (local-only) as a deliberate security boundary
