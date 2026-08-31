# 01 - Basic LVM Setup

Before doing the disk swap, you must understand basic LVM. This file covers
that.

## What is LVM 

LVM (Logical Volume Manager) lets you combine disks into one pool and
create flexible storage from that pool. You can grow, shrink, or move
storage without deleting data.

Three layers:

- **PV (Physical Volume)** — a raw disk or partition, prepared for LVM.
- **VG (Volume Group)** — a storage pool made from one or more PVs.
- **LV (Logical Volume)** — a piece of storage cut from the VG. This is what
  you format and mount, like a normal partition.

## Step 1: Prepare the disk as a Physical Volume

```bash
pvcreate /dev/sdb
```

This marks `/dev/sdb` as usable by LVM.

Check it:

```bash
pvs
```

## Step 2: Create a Volume Group

```bash
vgcreate vg_data /dev/sdb
```

This creates a pool named `vg_data` using the PV.

Check it:

```bash
vgs
```

## Step 3: Create a Logical Volume

```bash
lvcreate -L 5G -n lv_data vg_data
```

This creates a 5GB volume named `lv_data` inside `vg_data`.

Check it:

```bash
lvs
```

## Step 4: Format the Logical Volume into a Filesystem

```bash
mkfs.xfs /dev/vg_data/lv_data
```

## Step 5: Mount it filesystem to directory

```bash
mkdir /mnt/data
mount /dev/vg_data/lv_data /mnt/data
```

Check it:

```bash
df -h /mnt/data
```

## Step 6: Make the mount permanent (survive reboot)

Add this line to `/etc/fstab`:

```
/dev/vg_data/lv_data   /mnt/data   xfs   defaults   0 0
```

Test it:

```bash
mount -a
```

If no error, the entry is correct.

## Summary

You now have a working LV, mounted and ready to use. This is the base
setup used in the next file, where the disk under this LV gets swapped
live with zero downtime.
