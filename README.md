Ansible role: manage-mdraid1
============================

The ansible role that manages the software RAID-1 configuration as follows:

1. mkdir /boot/efi and /boot/efi2 directories.
2. mount /dev/sda2 and /dev/sdb2 onto /boot/efi and /boot/efi2, respectively.
3. execute the grub-install command if the directory is empty (not configured.)
4. update /etc/fstab entries.

These /boot/efi, /dev/sda2, and the "ubuntu" EFI label can be modified by default variables.
Please check the "Role Variables" section.

Backgrounds
-----------

The Debian/Ubuntu preseed system is the great kitting mechanism.
However, the system doesn't support to prepare the EFI system partition on second disk.

The aim of this ansible role is to setup the secondary EFI partition.

Requirements
------------

This module expects the following system configuration:

1. The system has two disks, /dev/sda and /dev/sdb, that configured as the software RAID-1.
2. The system uses the EFI boot mode, not legacy (MBR) boot mode.

### Ansible

- Version 2.8

### Distributions

- Ubuntu 18.04.2

Role Variables
--------------

### Defaults

    yamdraid1_targets:
	  - { device: "/dev/sda2", point: "/boot/efi", grublabel: "ubuntu" }
	  - { device: "/dev/sdb2", point: "/boot/efi2", grublabel: "ubuntu" }

Note: The "grublabel" key should be the fixed value, "ubuntu". If you change the string, the corrensponding EFI system will be installed into "/EFI/$grublabel". However, the grub system will still look for the "grub.cfg" file from "/EFI/ubuntu/" directory.

Dependencies
------------

N/A

Example Playbook
----------------

    - name: testing new ansible role
      hosts: all
      remote_user: ubuntu
      become: True
      vars:
      roles:
       - ansible-manage-mdraid1

#### How updating the /etc/fstab file

After executing this role, the /etc/fstab file will be updated as follows.

```diff
$ diff -u /etc/fstab.20190522.1253 /etc/fstab
--- /etc/fstab.20190522.1253    2019-05-21 15:52:33.663026747 +0900
+++ /etc/fstab  2019-05-22 12:53:42.591229910 +0900
@@ -8,4 +8,5 @@
 # / was on /dev/md0 during installation
 UUID=b1081827-06b0-460b-8fa8-84656b73a46d /               ext4    errors=remount-ro 0       1
 # /boot/efi was on /dev/sda2 during installation
-UUID=ED67-067F  /boot/efi       vfat    umask=0077      0       1
+/dev/sda2 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
+/dev/sdb2 /boot/efi2 vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
```

The efibootmgr command returns the following results;

```bash
$ efibootmgr -v
BootCurrent: 0000
Timeout: 1 seconds
BootOrder: 0002,0000,0001
Boot0000* ubuntu        HD(2,GPT,a7866e0e-0516-46f5-9d16-74f2572cb667,0x7c4,0x5f5e2)/File(\EFI\UBUNTU\SHIMX64.EFI)
Boot0001* UEFI: HL-DT-ST DVDROM DUD0N   PciRoot(0x0)/Pci(0x17,0x0)/Sata(5,65535,0)/CDROM(1,0x448,0x1340)..BO
Boot0002* ubuntu-2nd    HD(2,GPT,4d76e3cd-78ed-443b-baef-a8471fc78990,0x7c4,0x5f5e2)/File(\EFI\ubuntu-2nd\shimx64.efi)

```

### How to fix the system when the primary disk broken

If the primary /boot/efi partition or its disk, /dev/sda in this example, is broken, the system boot process will be suspended.
You will see the following message on the console:

```text
You are in emergency mode. After ....
....
Press Enter for maintenance
(or press Control-D to continue):
```

After entering the maintenance mode, comment out the missing /dev/sdb2 device in /etc/fstab as follows:

```text
# /boot/efi was on /dev/sda2 during installation
/dev/sda2 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
# /dev/sdb2 /boot/efi2 vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
```

The previous /boot/efi2 of /dev/sdb2 is now mounted on the /boot/efi.
The system suspends the boot process due to the missing /dev/sdb2, former /dev/sda2, device.

If you would like to replace the /dev/sda, please update the /etc/fstab carefully.
Before shutting down the system, update the /etc/fstab as follows:

```text
## There is no partitions on the bare /dev/sda.
## The /boot/efi must be pointing the healthy partition.

# /boot/efi was on /dev/sda2 during installation
# /dev/sda2 /boot/efi2 vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
/dev/sdb2 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
```

Replace the /dev/sda and boot up the system, then fixing up the partition table as follows:

```bash
## check the healthy partition table
$ sudo parted /dev/sdb
(parted) print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sdb: 20480MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start     End       Size      File system  Name                  Flags
 1      1.00MiB   20027MiB  20026MiB                                     raid
 2      20027MiB  20479MiB  452MiB    fat32        EFI System Partition  boot, esp


## parepare the partion on new disk
$ sudo parted /dev/sda
(paretd) mkpart parimary 1 20027
(parted) set 1 raid
(parted) mkpart ESP fat32 20027 20479
(parted) set 2 boot on
(parted) print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 20480MiB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start     End       Size      File system  Name     Flags
 1      1.00MiB   20027MiB  20026MiB               primary  raid
 2      20027MiB  20479MiB  452MiB    fat32        ESP      boot, esp
```

Then, run the rsync command to update the empty /boot/efi2:

```bash
$ sudo mount /dev/sda2 /boot/efi2
$ sudo rsync -av /boot/efi/. /boot/efi2/.
```

Now both EFI partiions are healthy, so that update the /etc/fstab as follows:

```text
# /boot/efi was on /dev/sda2 during installation
/dev/sda2 /boot/efi vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
/dev/sdb2 /boot/efi2 vfat rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro 0 0
```

And also, update the software raid by the mdadm command.

```bash
$ sudo mdadm --manage /dev/md0 --add /dev/sda1
```

License
-------

Apache License 2.0

Author Information
------------------

[Yasuhiro ABE](http://www.yasundial.org/foaf.xml)
