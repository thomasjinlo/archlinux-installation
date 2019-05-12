# archlinux-installation
A guide for installing archlinux in UEFI mode with systemd boot.

---

## Partitioning
Create the partitions using [gdisk](https://wiki.archlinux.org/index.php/Gdisk). Use `lsblk` to list out your drives/devices. Usually `sda` will be your main drive you want to partition. I'll use `sda` as the main drive for this example.

* run `gdisk /dev/sda`
* enter `o` to clear previous partition table and create a new one. **this will wipe out your drive entirely**
* enter `n` to create a new partition.
  * press enter to use default partition number.
    * press enter for the first sector. This determines where you want to start the file partition. By default it should be 2048M.
      * This is where you enter your partition size. Format is `+` + `size of partition` + `unit`. e.g. `+50M` would mean 50 megabytes of space.
        * Lastly you will need to enter your partition type by entering a Hex code or GUID.
	  * optional, but you can name your partition by pressing `c` and the partition number.

	  #### desired partition
	  Number | Size | Code | Name
	  -------|----- | -----|-------
	  1 | 550.0 MiB | EF00 | esp
	  2 | 40.0 GiB | 8304 | root
	  3 | 8 GiB (ram 2x) | 8200 | swap
	  4 | rest | EF02 | home

* after the desired table is created enter `w` to write it permanently

---

## Create File System
After each partition has been created, we need to create the filesystem that'll be used for each partition (excluding the swap partition). You can check the current file types by running `lsblk -f`. If they're not in the desired format listed before, we'll have to set it up manually.

* For `sda1` or our `esp` partition, run `mkfs.fat -F32 /dev/sda1`
* For `sda2/sda4` or our `root/home` partitions, run `mkfs.ext4 /dev/sda2` and `mkfs.ext4 /dev/sda4`.

#### desired file types

NAME | FSTYPE
-----|-------
sda1 | vfat
sda2 | ext4
sda3 | swap
sda4 | ext4

---

## Mount the partitions
The last remaining step before we start our install process is to mount our partitions to the associated directories. If there already is no `/mnt`, `/mnt/efi`, `/mnt/home` directories, create them. `/mnt` should most likely be available, but the other two nested directories are likely not there. Once they've been created we're ready to start the mounting process.

* `mount /dev/sda2 /mnt`
* `mount /dev/sda1 /mnt/efi`
* `mount /dev/sda4 /mnt/home`

Additionally we'll have to setup the swap partition, which doesn't require mounting.

* `mkswap /dev/sda3`
* `swapon /dev/sda3`
