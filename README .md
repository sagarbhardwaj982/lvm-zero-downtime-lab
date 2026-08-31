# Zero-Downtime LVM Disk Swap

## What this project is

A production server has a disk that is old, failing, or too small. The disk
needs to be replaced or grown — but the server must stay online. No
downtime is allowed.

This project shows how to manage LVM storage safely in production: basic
setup, swapping a live disk, growing storage live, and recovering from
LVM corruption — all without stopping the application.

## Why this matters (real-world use case)

- A disk shows early failure warnings (SMART errors) and needs to be
  swapped before it dies.
- An application is running out of disk space and cannot be paused to fix
  it.
- LVM metadata or a filesystem gets corrupted and data needs safe recovery.
- An interviewer asks: "How do you replace a failing disk or fix broken
  LVM in production without downtime?" — this project is the answer.

## What's inside

| File | What it covers |
|---|---|
| [01-basic-lvm-setup.md](01-basic-lvm-setup.md) | LVM basics: PV, VG, LV — creating storage from scratch |
| [02-zero-downtime-disk-swap.md](02-zero-downtime-disk-swap.md) | Swapping a live disk with pvmove, no downtime |
| [03-increase-lvm-size-without-downtime.md](03-increase-lvm-size-without-downtime.md) | Growing LV size live with lvextend + xfs_growfs/resize2fs |
| [04-recover-data-from-corrupt-lvm.md](04-recover-data-from-corrupt-lvm.md) | Recovering from missing PVs, corrupted metadata, and broken filesystems |

## Skills demonstrated

- Physical Volume (PV), Volume Group (VG), Logical Volume (LV) management
- Live data migration with `pvmove`
- Growing storage live with `lvextend`, `xfs_growfs`, `resize2fs`
- Diagnosing and recovering corrupted LVM metadata and filesystems
- Verifying storage health with `pvs`, `vgs`, `lvs`, `df`


