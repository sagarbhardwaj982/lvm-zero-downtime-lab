# 03 - Increase LVM Size Without Downtime

## The situation

`lv_data` is running low on space. The application using it cannot be
stopped. You need to grow the storage live, while it stays mounted and in
use.

This is different from the disk swap project — here you are not replacing
a disk, you are adding more space to the existing setup.

## Case A: The Volume Group already has free space

Check first:

```bash
vgs
```

Look at the `VFree` column. If there is free space in `vg_data`, you can
grow directly.

### Step 1: Extend the Logical Volume

```bash
lvextend -L +5G /dev/vg_data/lv_data
```

This adds 5GB to `lv_data`. The filesystem is still the old size at this
point — only the LV is bigger.

### Step 2: Grow the filesystem to use the new space

For **XFS**:

```bash
xfs_growfs /mnt/data
```

For **ext4**:

```bash
resize2fs /dev/vg_data/lv_data
```

Both commands work live, on a mounted filesystem. No unmount, no
downtime.

### Step 3: Verify

```bash
df -h /mnt/data
lvs
```

The new size should show immediately, with no interruption to the
application.

## Case B: The Volume Group has no free space (need a new disk)

If `vgs` shows `VFree` as 0, you must add a new disk first.

### Step 1: Prepare the new disk

```bash
pvcreate /dev/sdd
```

### Step 2: Add it to the Volume Group

```bash
vgextend vg_data /dev/sdd
```

### Step 3: Extend the Logical Volume using the new space

```bash
lvextend -L +10G /dev/vg_data/lv_data
```

### Step 4: Grow the filesystem

```bash
xfs_growfs /mnt/data
# or for ext4:
resize2fs /dev/vg_data/lv_data
```

### Step 5: Verify

```bash
df -h /mnt/data
pvs
vgs
lvs
```

## Shortcut: extend using all free space in the VG

Instead of picking a size, you can use all remaining free space:

```bash
lvextend -l +100%FREE /dev/vg_data/lv_data
xfs_growfs /mnt/data
```

## Why this works (the key idea)

`lvextend` only changes the size at the LVM layer — it is instant and
safe. `xfs_growfs` / `resize2fs` then grow the filesystem live to fill
that new space. Neither step needs the filesystem to be unmounted, so the
application keeps running the whole time.

## Important note

- Growing storage is always safe and simple this way.
- **Shrinking** is a different and riskier operation — XFS cannot be
  shrunk at all, and ext4 needs the filesystem unmounted first. That is a
  separate topic, not covered here.
