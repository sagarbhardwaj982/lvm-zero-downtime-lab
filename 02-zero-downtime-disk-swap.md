# 02 - Zero-Downtime LVM Disk Swap

## The situation

`vg_data` is currently using disk `/dev/sdb`. This disk is old and showing
failure warnings. It must be replaced with a new disk, `/dev/sdc`.

The Logical Volume `lv_data` is mounted at `/mnt/data` and is actively used
by a running application. It cannot be unmounted. There is no maintenance
window.

**Goal:** Move all data from `/dev/sdb` to `/dev/sdc`, then remove
`/dev/sdb` completely — without unmounting, without stopping the
application, without any data loss.

## Step 1: Attach and prepare the new disk

Attach the new disk `/dev/sdc` to the server. Then prepare it for LVM:

```bash
pvcreate /dev/sdc
```

At this point nothing on the live system has changed. This step is safe.

## Step 2: Add the new disk into the Volume Group

```bash
vgextend vg_data /dev/sdc
```

Now `vg_data` has both `/dev/sdb` (old) and `/dev/sdc` (new). The LV is
still fully on the old disk  nothing has moved yet.

Check:

```bash
vgs
pvs
```

## Step 3: Move the data this is the zero-downtime step

```bash
pvmove /dev/sdb /dev/sdc
```

What happens:
- `pvmove` copies every data block from `/dev/sdb` to `/dev/sdc`.
- The filesystem stays mounted the entire time.
- Reads and writes from the application keep working normally during the
  move.
- This can take time on large disks — progress is shown on screen.

Check progress in another terminal:

```bash
pvmove   # shows the move in progress if run with no arguments
```

## Step 4: Confirm the old disk is now empty

```bash
pvs
```

`/dev/sdb` should show 0 used extents all data is now on `/dev/sdc`.

## Step 5: Remove the old disk from the Volume Group

```bash
vgreduce vg_data /dev/sdb
```

This takes `/dev/sdb` out of `vg_data`. It is no longer part of the LVM
setup.

## Step 6: Wipe the old disk's LVM label

```bash
pvremove /dev/sdb
```

The old disk is now free. It can be physically removed from the server.

## Step 7: Final verification

```bash
pvs
vgs
lvs
df -h /mnt/data
```

Confirm:
- `vg_data` only shows `/dev/sdc`.
- `lv_data` is healthy and still mounted at `/mnt/data`.
- The application never lost access to its data.

## Why this works 

`pvmove` operates at the LVM block layer, below the filesystem. The
filesystem and the application never know the underlying disk changed.
That is what makes this a true zero-downtime operation, not just a fast
one.
