# Infrastructure TODO

## immich2 off-site deployment

### Current state

- `proxmox-backup-host` is the friend's off-site Proxmox hypervisor. Keep it
  outside Docker and outside the application routing layer.
- Create a separate guest named `immich2` for the photo backup service.
- The VPS already terminates HTTPS and routes `immich.74services.net` to the
  staged backend `10.44.0.6:2283`.
- The VPS certificate has been copied into the configured `/etc/letsencrypt`
  paths. ACME renewal automation remains future work.
- `10.44.0.5` remains reserved for `proxmox-backup-host`; do not assign it to
  `immich2`.

### Guest and disk design

- [ ] Provision `immich2` as a VM, preferred over LXC for Docker isolation,
      normal kernel behaviour, portability, and recovery simplicity.
- [ ] Use an SSD-backed OS virtual disk for the guest OS, Docker metadata,
      PostgreSQL, Redis/Valkey, and Immich machine-learning model cache.
- [ ] Use a separate virtual data disk for the Immich media library. Mount it
      inside the guest at `/srv/immich2/library` using a filesystem UUID.
- [ ] Initially place the data virtual disk on SSD storage if the HDD is not
      available yet.
- [ ] When the HDD arrives, move that virtual disk to Proxmox HDD storage,
      preserving the guest disk identity and mount path. Stop Immich and verify
      the filesystem and data before bringing it back up.
- [ ] Do not put PostgreSQL on a network share. Immich recommends local SSD
      storage for the database; use `DB_DATA_LOCATION` on the OS SSD.
- [ ] Keep the backup repository separate from the live photo-library disk if
      possible. A second disk or independent backup target is preferred.

### Networking and access

- [x] Add `immich2` as a separate inventory host with reserved WireGuard
      address `10.44.0.6` and bootstrap address `192.168.68.107`.
- [ ] Before bootstrap, deploy the VPS peer route and update the laptop profile
      with `192.168.68.107/32`; the off-site Proxmox host must forward that route
      to its LAN and provide a return route (or narrowly scoped SNAT).
- [ ] Bootstrap only `immich2` while it is in the inventory `bootstrap` group,
      install WireGuard, then enroll it in the managed `vpn` inventory group
      and remove it from `bootstrap` to replace the temporary LAN path with
      `10.44.0.6`.
- [ ] Add the VPS as the WireGuard peer and verify bidirectional connectivity.
- [ ] Permit Immich's internal HTTP endpoint on port `2283` only from the VPS
      WireGuard address/network; do not expose it directly to the Internet.
- [ ] Permit SSH and management access only through the intended management
      network.
- [ ] Verify the off-site link's MTU and upload performance before importing a
      large photo library.

### Immich Compose deployment

- [ ] Add a dedicated `immich2` stack in the stacks repository and keep it
      separate from the application VM stacks.
- [ ] Use Immich's official Docker Compose deployment, pinning a tested
      `IMMICH_VERSION` rather than tracking an unbounded latest image.
- [ ] Set `UPLOAD_LOCATION=/srv/immich2/library`.
- [ ] Set `DB_DATA_LOCATION` to a local SSD path such as
      `/var/lib/immich2/postgres`.
- [ ] Keep the database, Redis/Valkey, and machine-learning services private
      to the Compose network; publish only Immich HTTP/2283 to the WireGuard
      interface or host firewall scope needed by the VPS.
- [ ] Configure hardware acceleration only after the CPU deployment is stable
      and the off-site host's hardware is known.
- [ ] Configure long request timeouts, WebSockets, and a large request body at
      the VPS reverse proxy for photo/video uploads.
- [ ] Use a dedicated Immich database password and secrets; do not commit
      them to the stacks repository.
- [ ] Register the guest with Komodo Periphery and create a separate Git-backed
      Stack resource with webhook-based redeployment.

### VPS routing and TLS

- [x] Deploy VPS HTTPS hostname routing for `immich.74services.net`.
- [x] Install the manually managed VPS certificate and key.
- [ ] Confirm the certificate covers `immich.74services.net` and
      `deploy.74services.net`.
- [ ] Update `10.44.0.6:2283` in `inventory/host_vars/proxy-vps.yml` if the
      final WireGuard allocation changes.
- [ ] After the guest is live, test the route with SNI/Host and verify that
      `X-Forwarded-Proto`, WebSockets, uploads, and long-running requests work.
- [ ] Add ACME certificate issuance and renewal on the VPS later.

### Data migration and operations

- [ ] Create the filesystem, mount point, ownership, and directory structure
      before starting the stack.
- [ ] If an existing library is being imported, stop Immich during the final
      copy and verify file counts, permissions, and available capacity.
- [ ] Enable monitoring for disk usage, database health, container health,
      WireGuard reachability, and backup freshness.
- [ ] Document the recovery procedure for rebuilding the guest from the
      Compose files, secrets, database backup, and media disk.

### Encrypted off-site recovery

- [ ] Define what the primary application VM sends to `immich2`, and what
      `immich2` sends back to the primary site.
- [ ] Use application-aware database dumps before copying database backups.
- [ ] Use Restic or Borg with separate repository encryption keys and retention
      policies for each direction.
- [ ] Run backup jobs outbound over WireGuard; do not expose backup services
      through the public Nginx proxy.
- [ ] Keep recovery keys outside both hosts and document the emergency key
      access process.
- [ ] Add pruning, alerting, and periodic restore tests.
- [ ] Test recovery of both the application VM and the Immich database/media
      from the off-site copies.

### Architecture boundary

- [ ] Keep `immich2` as an independent Docker Compose host initially. Do not
      introduce a cross-site Swarm merely for this one stateful application.
- [ ] Reconsider adding other workloads to the off-site host only after
      capacity, storage isolation, backup impact, and routing ownership are
      documented.
- [ ] If more off-site applications are added, choose an explicit discovery
      model before adding more public hostnames: static inventory routes,
      `traefik-kop`/Redis, or a deliberately designed Swarm cluster.
