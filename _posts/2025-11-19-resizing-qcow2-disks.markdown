---
layout: post
title: "Resizing QCOW2 Disks"
date: 2025-11-19 11:06:31 +08:00
categories: linux tutorial reference cheatsheet
published: true
---

Sometimes I need to resize a VM's disk, and I always forget how to do it. This post is simply a cheat sheet.

I'm assuming that you're using a KVM+QEMU-based VM, therefore a .qcow2 is probably the disk in question.

If this gets out of date, use [Arch's instructions](https://wiki.archlinux.org/title/Resizing_LVM-on-LUKS) instead.

## The QCOW2 (a.k.a. the easy part)
```bash
qemu-img resize mydisk.qcow2 +20G #Add 20 Gigabytes to disk size
```

## The LVM-on-LUKS-on-GPT partition (a.k.a. the not-so-easy part)
This part requires booting from a live-install medium (if you can't unmount the partition in question, i.e. it's the root partition, otherwise, you can do this after booting into your normal desktop environment).

### virt-manager settings for booting
1. Boot Options->Boot device order, set it so the VM boots from SATA CDROM 1 with the highest priority. If you don't have a CDROM device, add one.
2. SATA CDROM 1->Source path, set this to your OS iso.
3. Apply and boot VM (enter Arch Install Medium)

### Inside the live CD
Remember to adjust disk names (like `/dev/vda2` and `/dev/vda3`) accordingly!

#### If you have a new hard drive
```bash
# Create a new partition (/dev/vda3)
cfdisk
# Copy old data to new partition
dd if=/dev/vda2 of=/dev/vda3 bs=4M
# Open new crypt partition
cryptsetup luksOpen /dev/vda3 cryptdisk
```

#### If you are expanding the old partition
```bash
# Resize the current partition
cfdisk
# Open new crypt partition
cryptsetup luksOpen /dev/vda2 cryptdisk
```

#### Perform the resize
```bash
# review the old PV size
pvdisplay -m
# Physical extent 0 to 4859:
#   Logical volume    /dev/ArchinstallVg/root
# Total PE * PE Size = hard disk size
pvresize /dev/mapper/cryptdisk
# review the new physical segments to add
pvdisplay -m
# Physical extent 4859 to 9978:
#   FREE
# Extend the LV
lvresize -L +$(((9978-4859+1) * 4))M /dev/ArchinstallVg/root
# review the new physical extents
pvdisplay -m
# Physical extent 0 to 9978:
#   Logical volume    /dev/ArchinstallVg/root
# Always end with an fs check
btrfsck /dev/ArchinstallVg/root
```
Finally, boot into your existing OS.
```bash
# Check current file system size
df -h
# Filesystem                      Size  Used  Avail Use%  Mounted on
# /dev/mapper/ArchinstallVg-root   19G   15G     4G  79%  /
# Resize btrfs (as root)
btrfs filesystem resize max /
# Check file system size
df -h
# Filesystem                      Size  Used  Avail Use%  Mounted on
# /dev/mapper/ArchinstallVg-root   39G   15G    24G  39%  /
```
And voila.

# Shrinking (shrinkwrapping) a QCOW2
Note that I'm doing this with a rocky10 VM, so I'll mention multiple ways to handle shrinking.

Firstly, in the **guest**, clear unused space on the disk:
```bash
fstrim -av
```
OR (if fstrim isn't available)
```bash
dd if=/dev/zero of=/tempfile
rm -f /tempfile
```

Then shutdown the guest and make a backup of the disk
```bash
cp old.qcow2 old.qcow2.bkp
```

Finally shrink the disk
```bash
qemu-img convert -O qcow2 old.qcow2 new.qcow2
```

That's it!
