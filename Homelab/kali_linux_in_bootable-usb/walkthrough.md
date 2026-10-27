# Building a Persistent Kali Linux Live USB

## Overview
To pair with a stable Debian daily-driver (documented separately), I built a dedicated **Kali Linux live USB with a persistent storage partition** — keeping the pentest/CTF toolset isolated from the main OS while still retaining installed tools, configs, and captured data across reboots.

## 1. Image Acquisition & Verification
Downloaded `kali-linux-2026.2-live-amd64.iso` directly from `cdimage.kali.org`. When direct HTTP access intermittently failed, fell back to the **official torrent**, confirming first that torrenting the file itself is legitimate — Kali officially distributes images this way, and the BitTorrent protocol's per-piece hash verification means any peer that sent tampered or corrupted data would simply be rejected and the piece re-fetched from elsewhere. The protocol's legality is independent of content; downloading an image the publisher explicitly offers via torrent carries no more risk than an HTTP mirror.

Verified file integrity before use:
```
sha256sum kali-linux-2026.2-live-amd64.iso
```
Compared the output against Kali's published `SHA256SUMS` to confirm a byte-for-byte match with the official release before writing anything to physical media.

## 2. Flashing the Image
Used `dd` rather than a GUI imaging tool, since this was done from an existing Linux host:
```
sudo dd if=kali-linux-2026.2-live-amd64.iso of=/dev/sdb bs=4M status=progress conv=fsync
```
- Targeted the raw block device (`/dev/sdb`, not a partition like `/dev/sdb1`), since the ISO carries its own partition table that needs to sit at the start of the disk
- `conv=fsync` ensured all data was physically flushed before considering the write complete — critical to avoid silent corruption from pulling the drive too early
- Confirmed target device identity via `lsblk` immediately before running the command, given `dd`'s lack of built-in confirmation prompts

## 3. Building the Persistence Partition
The live image's own partitions (boot + small flag partition) left the remainder of the 128GB USB (~110GB usable) unallocated. Used `parted` to carve out a dedicated persistence partition in the free space:
```
sudo parted /dev/sdb
(parted) mkpart primary ext4 5532MB 100%
(parted) quit
```
Formatted and correctly labeled the new partition — Kali's live-boot process requires the **exact label `persistence`** to recognize and mount it:
```
sudo mkfs.ext4 -L persistence /dev/sdb3
```
Mounted it temporarily to write the required configuration file that activates the overlay filesystem:
```
sudo mount /dev/sdb3 /mnt/kali-persist
echo "/ union" | sudo tee /mnt/kali-persist/persistence.conf
sudo umount /mnt/kali-persist
```

## 4. Debugging: Persistence Silently Failing
On first boot into the "Live system (persistence)" option, `mount | grep persistence` returned nothing — the partition wasn't active, despite the correct boot flag (`persistence`) appearing in `/proc/cmdline`.

Root-caused by checking the partition label directly:
```
sudo blkid /dev/sdb3
```
Output showed `LABEL="persistance"` — a one-character typo from the initial `mkfs.ext4` command. Kali's boot scripts match the label string exactly and silently skip any partition that doesn't match `persistence`, with no error surfaced to the user — making this a straightforward typo invisible without directly inspecting the partition metadata.

**Fix:**
```
sudo e2label /dev/sdb3 persistence
```
Rebooted, reselected the persistence boot option, and confirmed via `mount | grep persistence` that `/dev/sdb3` was now correctly mounted under the live overlay (`upperdir=/run/live/persistence/sdb3/rw`).

## 5. Validation
Created a test file, rebooted into persistence mode, and confirmed the file survived:
```
mkdir ~/test-persistence
echo "persistence check" > ~/test-persistence/note.txt
# reboot into persistence mode
cat ~/test-persistence/note.txt   # confirmed intact
```

## Outcome
A fully functional, persistent Kali Linux live USB, verified end-to-end: authenticated source image, correctly partitioned and labeled persistence volume, and confirmed data retention across reboots — isolated from the host Debian install by design.

## Skills Demonstrated
Image integrity verification (checksum validation against publisher-signed hashes) · low-level disk imaging with `dd` · manual partitioning and filesystem labeling with `parted`/`mkfs` · Linux live-boot/overlay filesystem internals · methodical root-cause debugging from a silent failure down to a single metadata field
