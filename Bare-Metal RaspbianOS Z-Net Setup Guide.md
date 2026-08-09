# Bare-Metal RaspbianOS Z-Net Setup Guide

This document outlines the manual steps required to convert a fresh RaspbianOS install onto a microSD card into a dedicated, read-only, Z-Net interface.

---

## 1. Initial Mount & SSH Configuration

Mount the root and boot partitions of the target microSD/USB drive, then enable SSH on boot if wanted:

```bash
sudo touch /ssh
```

---

## 2. Download Required Files

Create the HomeSeer and web directories:

```bash
sudo mkdir -p /usr/local/HomeSeer
sudo mkdir -p /var/www/Main
```

Download the `ser2net` binary and configuration:

```bash
sudo wget -O /etc/ser2net.conf https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/etc/ser2net.conf
sudo wget -O /bin/ser2net https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/bin/ser2net
sudo chmod +x /bin/ser2net
```

Download the HomeSeer network registration script:

```bash
sudo wget -O /usr/local/HomeSeer/register_with_find.sh https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/scripts/register_with_find.sh
```

Download the `rc.local` startup script:

```bash
sudo wget -O /etc/rc.local https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/etc/rc.local
```

After downloading, edit `rc.local` and **remove** the following line:

```bash
sudo sh /var/www/Main/uzb.sh &
```

> **Note:** `uzb.sh detects whether a UZB Z-Wave USB stick is connected and, if so, reconfigures /etc/ser2net.conf to point at /dev/ttyACM0. The presence of /var/www/Main/uzb.txt acts as a flag telling the script this has already been done, so it skips re-detection and leaves the existing config alone — avoiding unnecessary network requests and disk writes on reboot.

---

## 3. WiringPi Compatibility (GPIO)

> **Note:** Gordon Henderson deprecated WiringPi in 2019 due to peripheral addressing changes introduced in the Pi 4. Modern distributions use native kernel tools like `gpiod` instead. To maintain compatibility on original RPi 1/2/3 boards without fully reinstalling WiringPi, migrate the legacy dynamically linked C libraries directly.

Copy the legacy libraries into place:

```bash
sudo wget -O /usr/local/bin/gpio https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/bin/gpio
sudo wget -O /usr/lib/libwiringPi.so https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/lib/libwiringPi.so
sudo wget -O /usr/lib/libwiringPiDev.so https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi2_Z-Net-Interface_OS-Image-Optimization/main/lib/libwiringPiDev.so
```

**Verify after boot** — confirm the dynamic linking succeeded and that `libwiringPi.so` opens correctly:

```bash
sudo ldconfig
gpio readall
```

✅ Success is confirmed when the matrix outputs without throwing a shared object file error.

---

## 4. Boot Configuration & Read-Only Filesystem

Update the boot parameters to lock the filesystem and prevent unwanted expansion.

**Modify `/boot/cmdline.txt`** — edit the following:

```
root=PARTUUID=<EDIT>-02 ro, rootwait, noswap
```

**Modify `/etc/fstab`** — enforce a read-only (`ro`) mount for the root filesystem:

```
PARTUUID=<EDITED>-02 /    ext4    ro,noatime
```

---

## 5. System Pruning & Service Masking

Edit system configurations and mask unnecessary services to reduce overhead and prevent startup errors on a read-only filesystem.

Revise Journald configuration:

```bash
sudo nano /etc/systemd/journald.conf
```

Set the following values to keep logs in memory and cap their size if wanted (helps avoid unnecessary writes on a read-only filesystem):

```ini
Storage=volatile
SystemMaxUse=20M
SystemMaxFileSize=4M
RuntimeMaxUse=16M
RuntimeMaxFileSize=4M
```

Mask unneeded services:

```bash
sudo systemctl mask avahi-daemon
sudo systemctl mask avahi-daemon.socket
sudo systemctl mask apt-daily.service
sudo systemctl mask apt-daily.timer
sudo systemctl mask apt-daily-upgrade.service
sudo systemctl mask apt-daily-upgrade.timer
sudo systemctl mask rsyslog
sudo systemctl mask console-setup.service
```

> **Note:** `console-setup.service` is masked due to service startup issues on read-only filesystems and general redundancy.
