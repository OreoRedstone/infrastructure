# Important data backups

## Decided mechanism

Use PVE-native `vzdump` backups between the two existing Proxmox hosts:

- `host1` backs up the data disk from VM 101 `application-plane` to backup
  storage exported by `host2`.
- `host2` backs up the Immich data disks from VM 100 `immich2` to backup
  storage exported by `host1`.
- Use NFS over the existing WireGuard connection for the peer backup storage.
- Store backups on dedicated filesystems, register each NFS export as PVE
  storage with `content backup`, and run scheduled backup jobs with
  `compress=zstd` and retention rules.

No operating-system disks are included. They are recreated with Ansible.

## Confirmed live layout

### `host1` / VM 101 `application-plane`

- `scsi0`: 96 GB OS disk — exclude from backup.
- `scsi1`: 75 GB `local-lvm` disk, mounted in the guest as
  `/mnt/docker-data` — include in backup.
- The disk is currently about 59 GB used.

### `host2` / VM 100 `immich2`

- `scsi0`: 32 GB OS disk — exclude from backup.
- `scsi1`: 32 GB `local-lvm` disk, mounted as
  `/srv/immich2/appdata` — include because it contains Immich application and
  database data.
- `scsi2`: 32 GB `local-lvm` disk, mounted as
  `/srv/immich2/photos` and mapped by the Immich container to `/data` — include
  because it contains the photo library.
- The photos filesystem is currently about 36 MB used. The Immich PostgreSQL
  data is on `scsi1` under `/srv/immich2/appdata/postgres`.

All included disks are Proxmox-managed virtual disks. Neither VM uses disk
passthrough, and none of the disks currently has `backup=0`.

## Required work

- [ ] Add or designate a sufficiently large dedicated backup filesystem on
  `host2`. Its current PVE `local` storage has only about 62 GB free, which is
  not enough to safely retain VM 101's 75 GB data disk backup plus generations.
- [ ] Designate the existing `host1` `main-nas` ZFS pool, or a dedicated
  dataset on it, as the backup target for VM 100. It currently has about 1.26
  TB free.
- [ ] Export the `host2` backup filesystem via NFS, restricted to `host1`'s
  management address. Export the `host1` backup dataset via NFS, restricted to
  `host2`'s management address. Permit access only across the existing
  WireGuard-connected path.
- [ ] Add both NFS exports to the appropriate PVE storage configuration with
  `content backup`.
- [ ] Set the VM disk backup flags so only the required data disks are backed
  up: VM 101 `scsi1`; VM 100 `scsi1` and `scsi2`. Set `backup=0` on both VMs'
  `scsi0` OS disks.
- [ ] Create reciprocal scheduled `vzdump` jobs:
  - VM 101 on `host1` to the NFS backup storage on `host2`.
  - VM 100 on `host2` to the NFS backup storage on `host1`.
- [ ] Set `compress=zstd` explicitly. The installed PVE `vzdump` defaults to
  `compress=0`; backups are not compressed unless this is set. The resulting
  VM backup files use `.vma.zst`.
- [ ] Measure the actual VM 101 backup before relying on the current `host2`
  capacity. After setting VM 101 `scsi0` to `backup=0`, run one ordinary
  `vzdump` with `compress=zstd` to a temporary directory on `host1`'s `main-nas`
  and record the resulting `.vma.zst` size. PVE has no size-only dry-run; this
  one-off backup is the accurate test and `host1` currently has about 1.26 TB
  free.
- [ ] Configure retention and failure notifications in PVE. Limit backup
  bandwidth so the WireGuard link and running services remain usable.
- [ ] Run one backup in each direction and confirm the archive contains only
  the selected data disks.
- [ ] Restore each selected disk into a temporary VM and verify:
  `/mnt/docker-data`, Immich app/database data, and the Immich photo library.
- [ ] Document the Ansible rebuild sequence followed by attaching/restoring the
  selected data disks, then mark the jobs operational.

## Repository references

- VM and host inventory: `inventory/hosts.yml` and
  `inventory/host_vars/{proxmox-host,proxmox-backup-host}.yml`
- Application VM storage: `../stacks` persistent mounts under
  `/mnt/docker-data`
- Immich mounts: `../stacks/immich2/compose.yaml`
- PVE backup configuration: the Proxmox Datacenter **Backup** job settings or
  the equivalent `vzdump` configuration on the two nodes.
