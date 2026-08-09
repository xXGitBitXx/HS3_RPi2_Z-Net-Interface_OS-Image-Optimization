# Bare-Metal RaspbianOS Z-Net Setup Guide

This document outlines the manual steps required to convert a fresh RaspbianOS install onto a microSD card into a dedicated, read-only, Z-Net interface.

[Raspbian Lite 2019 Image](http://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2019-04-09/)

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

Choose your parent directory.

Download the `ser2net` binary and `ser2net.conf`:

```bash
sudo wget -O /etc/ser2net.conf https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/etc/ser2net.conf
sudo wget -O /bin/ser2net https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/bin/ser2net
```

```bash
sudo chmod +x /bin/ser2net
```

Download the HomeSeer network registration script:

```bash
sudo wget -O /usr/local/HomeSeer/register_with_find.sh https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/scripts/register_with_find.sh
```

Download the `rc.local` startup script:

```bash
sudo wget -O /etc/rc.local https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/etc/rc.local
```

After downloading, edit `rc.local` and remove / add the following lines:

**Remove**

```bash
15: sleep 5
28 /usr/local/HomeSeer/led.sh yellow
30: /var/www/Main/checkkb
35: /usr/local/HomeSeer/led.sh blue
37: sudo sh /var/www/Main/uzb.sh &
46: echo ""
sudo sh /var/www/Main/uzb.sh &
```

**Add**

```bash
20: # Configure BCM pins 2, 3, 4 (RGB status LED) as outputs
24: # Set initial LED state: pins 3 & 4 HIGH (off, active-low), pin 2 LOW (red on)
36: # echo "HomeSeer is starting..."
```

> **Note:** `/var/www/Main/uzb.sh` detects whether a UZB Z-Wave USB stick is connected and if so, reconfigures /etc/ser2net.conf to point at /dev/ttyACM0 instead of the GPIO header's primary UART interface (/dev/ttyAMA0).
```bash
'2001:raw:0:/dev/ttyACM0:115200 8DATABITS NONE 1STOPBIT -XONXOFF -RTSCTS'
```
> The presence of /var/www/Main/uzb.txt acts as a flag telling the script this has already been done, so it skips re-detection and leaves the existing config alone — avoiding unnecessary network requests and disk writes on reboot.
>
> This deployment removes the line rather than relying on it, since ser2net.conf is already downloaded and configured in the step above — the auto-detection logic and its dependency on a live download from HomeSeer's server aren't needed here.

---

## 3. WiringPi Compatibility (GPIO)

> **Note:** Gordon Henderson deprecated WiringPi in 2019 due to peripheral addressing changes introduced in the Pi 4. Modern distributions use native kernel tools like `gpiod` instead. To maintain compatibility on original RPi 1/2/3 boards without fully reinstalling WiringPi, migrate the legacy dynamically linked C libraries directly.

Copy the legacy libraries into place:

```bash
sudo wget -O /usr/local/bin/gpio https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/bin/gpio
sudo wget -O /usr/lib/libwiringPi.so https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/lib/libwiringPi.so
sudo wget -O /usr/lib/libwiringPiDev.so https://raw.githubusercontent.com/xXGitBitXx/HS3_RPi_Z-Net-Interface_OS-Image-Optimization/main/lib/libwiringPiDev.so
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

# Move high-write system and logging directories into RAM (tmpfs)
tmpfs           /tmp            tmpfs   defaults,noatime,nosuid,size=30M    0       0
tmpfs           /var/tmp        tmpfs   defaults,noatime,nosuid,size=10M     0       0
tmpfs           /var/log        tmpfs   defaults,noatime,nosuid,mode=0755,size=10M  0  0
tmpfs           /var/spool      tmpfs   defaults,noatime,nosuid,size=10M     0       0
```

**Modify `/etc/rc.local`** — since `/var/log` is now a `tmpfs` mount (wiped on every reboot), add the following just before `exit 0` to recreate required log directories and permissions:

```bash
mkdir -p /var/log/apt
chmod 755 /var/log/*
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
