# HS3 RPi2 OS Image Optimization
## Stripped Raspbian 9 OS for Z-Net Interface

> **Note:** This procedure was engineered because official Raspbian 8 recovery images are no longer provided, requiring custom optimization for legacy Z-Net-only hardware.
> For rapid deployment, the necessary files within each folder of this repository can be copied directly to their intended system directories.

---

## Target Device
**1GB USB Deployment for Raspberry Pi 2**

---

# Prerequisites

- **MicroSD Card**
- **Linux Environment**
- **CloneZilla Live Media**
- **HomeSeer Rasp-Pi Image:**  
  http://www.homeseer.com/updates3/hs3pi3_image_070319.zip

---

# Phase 1: OS Debloating & Preparation

### Goal
Strip the Raspbian 9 environment to its absolute essentials to fit a 1GB physical constraint.

---

## 1.1 Software Purge

### Remove High-Capacity Suites

```bash
sudo apt-get purge wolfram-engine
sudo apt-get purge libreoffice*
sudo apt-get purge mono-xsp4
sudo apt purge "mono-*" -y
sudo rm -f /etc/apt/sources.list.d/mono-official-stable.list
sudo apt autoremove --purge -y
```

### Dependency Cleanup

```bash
sudo apt-get autoremove
```

---

## Kernel & Module Slimming

execute:

```bash
uname -r
```

to identify the active kernel.

Delete all inactive kernel versions in:

```text
/lib/modules/[version]
```

---

## Localization & Documentation

Delete:

```text
/usr/share/doc
/usr/share/man
```

keeping only English manuals.

```bash
find /usr/share/doc -depth -type d -empty -exec rmdir {} \;
```

```bash
find /usr/share/man -mindepth 1 -maxdepth 1 -type d ! -name 'man*' ! -name 'en*' -exec rm -rf {} +
```

Strip unused language packs in:

```text
/usr/share/locale
```

```bash
sudo find /usr/share/locale -mindepth 1 -maxdepth 1 -type d ! -name 'en*' ! -name 'default' -exec rm -rf {} +
```

Purge cache and temporary files:

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*
```

---

## Remove Legacy Hardware Source Demos

Wipe the proprietary VideoCore GPU code samples. Because this is a headless appliance that functions strictly as a network serial bridge, multimedia test assets and raw H.264 video streams are dead weight.

```bash
sudo rm -rf /opt/vc/src/hello_pi
```

## 1.2 Service Management

### Mask Services

Disable the following to reduce I/O and interference.

### avahi-daemon

```bash
sudo systemctl mask avahi-daemon
```

### ModemManager

Prevents serial port interference.

```bash
sudo systemctl mask ModemManager
```

### apt-daily (Update Downloader)

```bash
sudo systemctl mask apt-daily.service
sudo systemctl mask apt-daily.timer
```

### apt-daily-upgrade (Upgrade Installer)

```bash
sudo systemctl mask apt-daily-upgrade.service
sudo systemctl mask apt-daily-upgrade.timer
```
### rsyslog (System Logging Daemon)
Prevents heavy syslog write cycles from thrashing flash storage, freezing the kernel, or causing prolonged `sync` lockups.

**Option A: Live System Mask**

If the system is currently booted:

```bash
sudo systemctl mask rsyslog
```

**Option B: Host System Mask (Offline Method)**

Mount the USB data partition to a host Linux environment to apply a structural mask before booting:

```bash
# Mount the USB root partition to your host system
sudo mount /dev/sdX# /mnt/target

# Force systemd to point rsyslog straight to a dead end (/dev/null)
sudo ln -sf /dev/null /mnt/target/etc/systemd/system/rsyslog.service

# Clean up legacy auto-start symlinks to ensure a clean boot path
sudo rm -f /mnt/target/etc/systemd/system/multi-user.target.wants/rsyslog.service
sudo rm -f /mnt/target/etc/rc*.d/S*rsyslog

# Commit the updates to the silicon and unmount cleanly
sync
sudo umount /mnt/target

```

---

## 1.3 Binary & Script Handshakes

### ser2net Setup

Copy the attached R8 Pi 2 binary file to:

```text
/usr/local/sbin/
```

Rename it to:

```text
ser2net
```

Set permissions:

```bash
chmod +x /usr/local/sbin/ser2net
```

---

## Configure the Serial-to-Network Bridge

Edit:

```text
/etc/rc.local
```

Add these lines just before the `exit 0` line:

```bash
sudo sh /var/www/Main/uzb.sh &

/usr/local/sbin/ser2net -c /etc/ser2net.conf -n > /dev/null 2>&1 &

/usr/local/HomeSeer/register_with_find.sh > /dev/null 2>&1 &
```

Enable execution permissions for hardware initialization:

```bash
sudo chmod +x /var/www/Main/uzb.sh
```

---

## /etc/ser2net.conf

Map the Z-Wave board on `/dev/ttyAMA0` to TCP port `2001` by uncommenting or adding this line:

```text
2001:raw:0:/dev/ttyAMA0:115200 8DATABITS NONE 1STOPBIT -XONXOFF -RTSCTS
```

This maps the Z-Wave board on `/dev/ttyAMA0` to TCP port `2001`.

---

## Z-Wave Radio Logic

Modify:

```text
/etc/rc.local
```

Add the following above *printf "Setting audio output to analog...\n"*:

```bash
modprobe ftdi_sio vendor=0x0403 product=0xc07f
```

### Explanation

`modprobe ftdi_sio...`

This manually forces the kernel to load the USB-to-serial driver for a specific device:

- Vendor: `0403`
- Product: `c07f`

This ensures the OS recognizes the Z-Wave radio hardware even if the default Raspbian kernel does not automatically map it to the serial driver.

```bash
sudo sh /var/www/Main/uzb.sh &
```

Launches the discovery script in the background to avoid hanging the boot process.

---

Create:

```text
/var/www/Main/uzb.txt
```

as a state flag to prevent redundant network requests.

This file tells the `/etc/rc.local` script:

> “The Z-Wave radio has already been found and configured; do not overwrite /etc/ser2net.conf.”

By checking for this file first, the system avoids unnecessary network requests (`wget`) and disk writes during reboot, helping preserve USB drive lifespan.

---

# Phase 2: Filesystem & Partition

### Goal
Shrink the logical filesystem and re-align partitions for the 1GB target.

---

## 2.1 Filesystem Compression & Tuning

> ⚠ USB must be unmounted to avoid corruption ⚠

### Disable Journaling

This frees hidden data blocks before imaging (Phase 3).

```bash
sudo tune2fs -O ^has_journal /dev/sdX#
```

### Set Reserved Blocks

Reduces root-reserved space from 5% to 1% to maximize usable capacity on a 1GB drive.

```bash
sudo tune2fs -m 1 /dev/sdX#
```

### Force Check

```bash
sudo e2fsck -f /dev/sdX#
```

### Shrink Filesystem

This utility shrinks the filesystem to the smallest possible size.

```bash
sudo resize2fs -M /dev/sdX#
```

---

## 2.2 Partition Table Re-alignment (fdisk)

### Delete Partition

Delete the EXT4 partition.

### Recreate Partition

Create a new logical partition.

### Signature Prompt

Choose:

```text
No
```

when asked to remove the ext4 signature.

---

# Phase 3: Cloning & Boot Redirection

### Goal
Transfer the data and update the bootloader to recognize the USB drive.

---

## 3.1 CloneZilla Backup

### Choose Expert Mode

Only back up the EXT4 partition. This prevents restoring the NOOBS architecture during restoration.

### Active Flags

```text
-icds
-k
-c
```

### Disabled Flags

```text
-j2
-e1 auto
-e2
-r
```

---

## 3.2 CloneZilla Restoration

### Re-enable Journaling

After the image restore completes, enter:

```bash
sudo tune2fs -j /dev/sdX#

sudo resize2fs /dev/sdX#
```

### Verify UUID

execute:

```bash
blkid /dev/sdX#
```

to retrieve the new `PARTUUID`.

---

## 3.3 Bootloader Configuration

### Update `/boot/cmdline.txt`

Remove:

```text
console=serial0,115200
```

from both the microSD and USB configurations.

Update:

```text
root=PARTUUID=[new-uuid]
```

### Update `/etc/fstab`

Map the root mount point to the new `PARTUUID`.

---

# Phase 4: Post-Boot Optimization

### Goal
Finalize the zero-swap state and logging constraints.

---

## 4.1 Swap Management

```bash
sudo dphys-swapfile swapoff

sudo systemctl disable dphys-swapfile

sudo rm /var/swap
```

Set:

```text
CONF_SWAPSIZE=0
```

in:

```text
/etc/dphys-swapfile
```

---

## 4.2 Logging Constraints (/etc/systemd/journald.conf)

Prevent logs from consuming excessive storage space.

```ini
[Journal]
Storage=persistent
SystemMaxUse=20M
SystemMaxFileSize=4M
RuntimeMaxUse=16M
RuntimeMaxFileSize=4M
```

---

# Phase 5: Verification

### **Goal:** Confirm the system is running within the 1GB physical constraint with Zero-Swap.

Once booted

execute: 

```bash
netstat -tulpn | grep 2001
```

If you see ser2net listening on that port, the bridge is active.

execute:

```bash
`df -h`, `lsblk`, and `free -h` 
```

to verify the filesystem footprint and memory management.

## [Terminal Verification] ##

**Key Metrics to confirm:**
* **Filesystem Size:** (fits on 1GB USB).
* **Swap:** 0B (confirmed disabled to preserve flash memory).
* **Memory Usage:** Minimal footprint.

<img width="809" height="400" alt="image" src="https://github.com/user-attachments/assets/f17d1d1f-1fb8-4f8e-8941-4e010274f483" />
