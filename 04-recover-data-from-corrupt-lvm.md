# 04 - Recover Data From Corrupt LVM

## The situation

Something has gone wrong with LVM — a VG won't activate, an LV shows
errors, or metadata looks damaged. The application is down and data needs
to be recovered safely, without making things worse.

This doc covers the common corruption situations and the safe way to
recover from each one. **Golden rule: never write anything to the disk
until you have a backup of the current state.**

## Step 0: Always back up LVM metadata first

Before touching anything:

```bash
vgcfgbackup vg_data
```

This saves the current VG metadata to `/etc/lvm/backup/vg_data`. Even if
things are broken, this backup captures the current state so you can
compare or roll back.

Also copy the raw disk if data is critical and you have space:

```bash
dd if=/dev/sdb of=/root/sdb_backup.img bs=4M status=progress conv=noerror,sync
```

This is slow but protects against making a mistake on the real disk.

## Case A: Volume Group won't activate

Symptom: `vgchange -ay` fails, or `lvs` shows nothing.

Check what LVM sees:

```bash
pvscan
vgscan
lvscan
```

Try activating again:

```bash
vgchange -ay vg_data
```

If the VG is marked `partial` (a PV is missing), see Case B.

## Case B: A disk/PV is missing or unreadable

Symptom: `pvscan` shows the VG as `partial`, one PV is gone.

Check which PV is missing:

```bash
vgs -o +missing_pvs
```

If the missing disk can be reconnected (loose cable, wrong device name),
reconnect it and re-run `pvscan`. If the disk is truly gone and there is
no redundancy, that data is only recoverable from backup — LVM cannot
rebuild data from a missing disk unless mirroring was set up in advance.

## Case C: LVM metadata is corrupted (but disks are fine)

Symptom: `vgs` / `lvs` show errors reading metadata, but the underlying
disks are healthy.

Check for LVM metadata backups on the disk itself:

```bash
vgcfgrestore --list vg_data
```

Restore the last known good metadata:

```bash
vgcfgrestore vg_data
```

Then try activating:

```bash
vgchange -ay vg_data
```

If `/etc/lvm/archive/` still has older metadata files from before the
corruption, restore from a specific one:

```bash
vgcfgrestore -f /etc/lvm/archive/vg_data_00001.vg vg_data
```

## Case D: LV activates, but filesystem inside is corrupted

Symptom: LV is active, but mounting fails or shows filesystem errors.

**Do not run repair tools directly on production data if avoidable. Take
an LVM snapshot first, and repair the snapshot instead**, so the original
data stays untouched if the repair goes wrong.

```bash
lvcreate -L 2G -s -n lv_data_snap /dev/vg_data/lv_data
```

Check/repair the snapshot copy.

For XFS:

```bash
xfs_repair /dev/vg_data/lv_data_snap
```

For ext4:

```bash
fsck -y /dev/vg_data/lv_data_snap
```

Mount the snapshot to verify data is readable:

```bash
mkdir /mnt/recovery_check
mount /dev/vg_data/lv_data_snap /mnt/recovery_check
ls /mnt/recovery_check
```

Only repair the real LV once the fix is confirmed safe on the snapshot.

## Case E: An LV was accidentally deleted

If archiving is enabled (default on most systems), LVM keeps old metadata
in `/etc/lvm/archive/`. Find a version from before the deletion:

```bash
ls -lt /etc/lvm/archive/
```

Restore that version:

```bash
vgcfgrestore -f /etc/lvm/archive/vg_data_000XX.vg vg_data
```

This brings back the LV *definition* (the metadata pointing to the data
blocks). If the blocks have not been overwritten yet, the data is still
there and becomes accessible again once the LV is restored and mounted.

**Important:** act quickly after a deletion — any new writes to the VG
can overwrite the old LV's data blocks and make recovery impossible.

## Quick recovery order (summary)

1. `vgcfgbackup` — save current state before touching anything
2. `pvscan` / `vgscan` / `lvscan` — see what LVM currently detects
3. Missing disk? — reconnect if possible, otherwise check backups
4. Metadata error? — `vgcfgrestore` from `/etc/lvm/archive/`
5. LV active but filesystem broken? — snapshot first, repair the snapshot
6. Deleted LV? — restore old metadata from archive, quickly
