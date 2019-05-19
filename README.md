# What is this guide?
Archlinux is a lightweight linux distribution that is so minimal it requires a lot of manual configuration during the installation process. There is a very in-depth [installation guide](https://wiki.archlinux.org/index.php/installation_guide) available, however, it covers various different ways for installing arch (depending on your system), and can get quite confusing during the install process.

This guide is an extracted version of the arch wiki installation guide, with the primary focus on the necessary configurations needed to install arch in UEFI mode with [systemd boot](https://wiki.archlinux.org/index.php/Systemd-boot).

---

# Partition setup
## [Creating the partitions](https://wiki.archlinux.org/index.php/GPT_fdisk#Create_a_partition_table_and_partitions)
Create the partitions using [gdisk](https://wiki.archlinux.org/index.php/Gdisk). Use `lsblk` to list out your drives/devices. Usually `sda` will be your main drive you want to partition. I'll use `sda` as the main drive for this example.

* `gdisk /dev/sda`, to enter an interactive session.
* enter `o` to clear the previous partition table and create a new one. **this will wipe out your drive entirely**
* enter `n` to create a new partition.
  * It'll ask for your partition number, type enter to use the next available number.
  * Next prompt will ask for the starting position, type enter to use default.
  * This prompt will ask for the ending position, which also determines the size of the partition.
    * You want the sizes to follow the desired partition table below, but I'll list out the steps required to achieve the desired format.
    * For example command to give the partition a size of 550Mib (EFI System Partition) is `+550M`.
      * `+` as it sounds, declares we want to add a set amount from the starting position.
      * `50` here is the size of the partition, and `M` is the unit, in this case `MiB`.
  * After inputting the size, we need to enter the correct code for the partition listed below.
* after the desired table is achieved enter `w` to write it permanently.

#### desired partition table (press p to print current table)
**Note:** *size of swap file is usually double your ram. Since I have 4gb of ram I'm giving it `+8G`.
Number | Size | Code | Name
-------|----- | -----|-------
1 | 550.0 MiB | EF00 | EFI System
2 | 80.0 GiB | 8304 | Linux x86-64 root(/)
3 | 8 GiB | 8200 | Linux swap
4 | rest | 8302 | Linux /home

## [Formatting the filesystem](https://wiki.archlinux.org/index.php/File_systems#Create_a_file_system)
After each partition has been created, we need to format them with the appropriate filesystem (excluding the swap partition). You can check the current file types by running `lsblk -f`. If they're not in the desired format listed before, we'll have to set it up manually.

* `mkfs.fat -F32 /dev/sda1` - configures the EFI System Partition to use vfat.
* `mkfs.ext4 /dev/sda2` - configures the root partition to use ext4.
* `mkfs.ext4 /dev/sda4` - configures the home partition to use ext4.
* `mkswap /dev/sda3`
* `swapon /dev/sda3`

#### desired file types
NAME | FSTYPE
-----|-------
sda1 | vfat
sda2 | ext4
sda3 | swap
sda4 | ext4

## [Mounting the partitions](https://wiki.archlinux.org/index.php/File_systems#Mount_a_file_system)
The last remaining step before we start our install process is to mount our partitions to the associated directories. If there already is no `/mnt`, `/mnt/boot`, `/mnt/home` directories, create them. `/mnt` should most likely be available, but the other two nested directories are likely not there. Once they've been created we're ready to start the mounting process.

* `mount /dev/sda2 /mnt`
* `mount /dev/sda1 /mnt/boot`
* `mount /dev/sda4 /mnt/home`

---

# Installation of base packages
Now that we have the partitions setup and ready to go, we can go ahead and start installing the base packages for arch. I'll include what I think are the basic essentials necessary for a minimal arch setup.

### Check if you have internet access
#### Running ethernet
If you're connected through an ethernet port, I believe all you need to do is enable `dhcpcd` by running `dhcpcd`.

#### Running wireless
If you're using wireless for this install, run `wifi-menu` to find your wireless network and connect to it. If you're seeing connection failure when connecting, make sure your wireless driver isn't set to `UP`. To set it to down, first find your interface name with `iw dev`. Then set it to `DOWN` with `ip link set interface_name down` e.g. `ip link set wlp3s0 down` where `wlp3s0` is my interface name.

#### Final check for connection
To check if you have connection, run `ping google.com`, you should see that requests are being made to google. Type `ctrl-c` to exit the loop.

### Install the packages
`pacstrap /mnt base base-devel intel-ucode dialog wpa_supplicant neovim`

---

# Required environment configurations

#### Update the system clock
`timedatectl set-ntp true`
Verify with `timedatectl status`

#### Generate Filesystem Table
`genfstab -U /mnt >> /mnt/etc/fstab`

#### Change root
`arch-chroot /mnt`

#### Setup timezone and localization
`ln -sf /usr/share/zoneinfo/<Your Region>/<Your City> /etc/localtime` e.g. `/usr/share/zoneinfo/America/Chicago ...`
`hwclock --systohc`

Uncomment `en_US.UTF-8 UTF-8` within `/etc/locale.gen`, generate with `locale-gen`.

Create a new file `/etc/locale.conf` and add
```
LC_ALL=en_US.UTF-8
LC_CTYPE=en_US.UTF-8
LANG=en_US.UTF-8
```

#### Set hostname
Create a new file `/etc/hostname`.
```
*myhostname*
```

---

# [Setup bootloader](https://wiki.archlinux.org/index.php/Systemd-boot)
For our installation we'll be using systemd boot as the bootloader. A bootloader is necessary as part of the booting process for loading the kernel and initial RAM disk with specific paramaters.

## [Install and update bootloader](https://wiki.archlinux.org/index.php/Systemd-boot#Installation)
`bootctl --path=/boot install`
`bootctl update`

Add this to `/boot/loader/loader.conf`

## [Configure](https://wiki.archlinux.org/index.php/Systemd-boot#Configuration)
Create a loader config `/boot/laoder/loader.conf`.
```
default archlinux
```
Create an entry config that matches the name listed in `loader.conf` (archlinux), `/boot/loader/entries/archlinux.conf`
```
title archlinux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=PARTUUID=<fill your part uuid for root partition> rw
```
You can get the root partition uuid by running `blkid | grep root`. **Note:** Do not paste in the string values,
```
/dev/sda2: UUID=...PARTUUID="abcde123a-f234-9eda0-9k7nb3d194b"
```

example format
```
title archlinux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=PARTUUID=abcde123a-f234-9eda0-9k7nb3d194b rw
```
