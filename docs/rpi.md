# Raspberry Pi Overview

Pi's are flashed with raspberry pi imager, and a few modifications are made.

I run both raspberry pi 5's and 4's. The 4's are the controlplane nodes, and the 5's are all dedicated to rook/ceph.

For the pi4's, a custom kernel is used to avoid having to recompile envoy for envoy gateway.

There are two basic setups for the pi's.

The pi 4's have a single boot drive, using an nvme hat. These are ~120g optane drives, with very good performance.

The pi 5's have a usb uas nvme and boot from 16g optane drives. (Previouslly sd cards were dying, and I couldn't/didn't care to figure out why.)

With the limited space on the boot drive, things like rancher and local-path provisioner are configured to use part of one of the ssds on the penta sata hat.

The "local" partition uses LVM and has a partition for local data, and 2 partitions for ceph metadata devices to speed up the spinning osds.

It looks something like this

```bash
sda - boot (uas nvme optane) 16g
sdb - spinning rust osd
sdc - ssd osd (1t, or 4t)
sdd - ssd osd (1t or 4t)
sde - lvm
  - local-md0 (metadata for 1st spinning osd)
  - local-md1 (metadata for 2nd spinning osd)
  - local-local - mounted to /local, used by k3s (data dir) and local-path-provisioner)
```

## Configure the network interfaces.

I hate nmcli, give me netplan

```bash
sudo nmcli con delete "Wired connection 1"
sudo nmcli connection add type ethernet ifname eth0 con-name "eth0" autoconnect yes
sudo nmcli con modify eth0 ipv4.address "192.168.1.122/24" ipv4.gateway "192.168.1.1" ipv4.dns "192.168.1.1" ipv4.method manual
sudo nmcli con delete preconfigured
```

For using the pi's onboard ethernet connection:

```bash
sudo nmcli con delete "Wired connection 1" || \
sudo nmcli con delete "Wired connection 2" || \
sudo nmcli connection add type ethernet ifname eth0 con-name "eth0" autoconnect yes || \
sudo nmcli con modify eth0 ipv4.address "192.168.1.123/24" ipv4.gateway "192.168.1.1" ipv4.dns "192.168.1.1" ipv4.method manual || \
sudo nmcli con delete preconfigured
```


Recreate ethernet interface
```bash
name=eth0
ip=192.168.1.120
nm=24
sudo nmcli conn delete $name
sudo nmcli connection add type ethernet ifname $name con-name "$name" autoconnect yes
sudo nmcli con modify eth0 ipv4.address "$ip/$nm" ipv4.gateway "192.168.1.1" ipv4.dns "192.168.1.1" ipv4.method manual
sudo nmcli con up $name
```


We will also create a separate storage network, (assuming you have a second nic / switch/vlan).

```bash
name=eth1
ip=192.168.100.115
nm=24
sudo nmcli conn delete $name
sudo nmcli connection add type ethernet ifname $name con-name "$name" autoconnect yes
sudo nmcli con modify $name ipv4.address "$ip/$nm" ipv4.method manual
sudo nmcli con up $name
```

## Update and install some tools

```bash
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install -y \
  lsof \
  arping \
  arp-scan \
  lvm2 \
  smartmontools
```

## Nic service

The usb ethernet adapters that I use for the storage network seem to fail after reboots. This is a simple service that resets the usb devices as a oneshot after a reboot.

```bash
sudo cat << EOF | sudo tee /etc/systemd/system/usbfix.service
[Unit]
Description=Fixes some usb issues

[Service]
ExecStart=/usr/sbin/usb_modeswitch -v 0bda -p 8151 -R
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable usbfix
```

```
echo "[Time]
NTP=192.168.1.1" | sudo tee /etc/systemd/timesyncd.conf
```

## Enable penta sata hat, configure the first drive

```bash
echo "dtoverlay=disable-wifi
dtoverlay=disable-bt
hdmi_safe:0=1
hdmi_safe:1=1
gpu_mem=32
dtparam=pciex1
dtparam=pciex1_gen=3
BOOT_ORDER=0x4
usb_max_current_enable=1
kernel=kernel8.img
" | sudo tee -a /boot/firmware/config.txt
```

```
kernel=kernel8.img
```

^ this is needed for running (most?) postgres images.

## add cgroups

cgroups are disabled by default

```bash
echo -n " cgroup_memory=1 cgroup_enable=memory" | sudo tee -a /boot/firmware/cmdline.txt
```

## Create local storage drive

Check what drives you have:

```bash
~ $ sudo lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 110.3G  0 disk
├─sda1        8:1    0   100M  0 part
├─sda2        8:2    0     1M  0 part
├─sda3        8:3    0  1000M  0 part
├─sda4        8:4    0     1M  0 part
├─sda5        8:5    0   100M  0 part
└─sda6        8:6    0    42M  0 part
mmcblk0     179:0    0 115.2G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 114.7G  0 part /
```

Optionally, wipe a drive (replace XX with the drive letter - don't wipe your root.):

```bash
sudo sgdisk --zap-all /dev/sdXX
```

## LVM

### commands to create the volume for metadata and local.

```bash
sudo mkdir /local
sudo chattr +i /local
```

```bash
sudo pvcreate /dev/sdb
sudo vgcreate local /dev/sdb
sudo lvcreate -L 120G -n md0 local
sudo lvcreate -L 120G -n local local
sudo mkfs.ext4 /dev/local/local
# mount the device with /etc/fstab
echo "/dev/local/local /local ext4 defaults 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```

### create a metadata device

```bash

```

Then format the drive:

```bash
sudo mkfs -t ext4 /dev/sda
sudo mkdir /local
sudo chattr +i /local
echo "/dev/sda /local ext4 defaults 0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a

```

## swapfile

```bash
sudo cat << EOF | sudo tee /etc/dphys-swapfile
# /etc/dphys-swapfile - user settings for dphys-swapfile package
# author Neil Franklin, last modification 2010.05.05
# copyright ETH Zuerich Physics Departement
#   use under either modified/non-advertising BSD or GPL license

# this file is sourced with . so full normal sh syntax applies

# the default settings are added as commented out CONF_*=* lines


# where we want the swapfile to be, this is the default
CONF_SWAPFILE=/local/swap

# set size to absolute value, leaving empty (default) then uses computed value
#   you most likely don't want this, unless you have an special disk situation
CONF_SWAPSIZE=16000

# set size to computed value, this times RAM size, dynamically adapts,
#   guarantees that there is enough swap without wasting disk space on excess
#CONF_SWAPFACTOR=2

# restrict size (computed and absolute!) to maximally this limit
#   can be set to empty for no limit, but beware of filled partitions!
#   this is/was a (outdated?) 32bit kernel limit (in MBytes), do not overrun it
#   but is also sensible on 64bit to prevent filling /var or even / partition
CONF_MAXSWAP=16000
EOF

sudo systemctl restart dphys-swapfile
```

## hostname setup

who needs dns anyways

```bash
echo "192.168.1.111 k1
192.168.1.112 k2
192.168.1.113 k3
192.168.1.114 k4
192.168.1.115 k5
192.168.1.116 k6
192.168.1.117 k7
192.168.1.118 k8
192.168.1.119 k9
192.168.1.120 k10
192.168.1.121 k11
192.168.1.122 k12
192.168.1.123 k13
192.168.1.124 k14" | sudo tee -a /etc/hosts
```

echo "192.168.1.119 k9" | sudo tee -a /etc/hosts
