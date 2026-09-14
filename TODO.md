# Infrastructure TODO

## Off-site photo backup and recovery architecture

### Context

- `proxmox-backup-host` is an off-site server provided by a friend to run a
  subscription-free photo backup service, currently Immich.
- It should be managed through this infrastructure repository and exposed via
  the existing domain.
- The off-site host is also a useful second location for encrypted backups of
  important data, with the primary and off-site locations able to exchange
  recovery copies.
- Immich is currently the only planned application at that location, although
  the host may later carry other isolated workloads.

### Architectural findings

- Keep the Proxmox host itself outside Docker Swarm. It remains a hypervisor
  and should not become a general application node.
- Provision a dedicated Immich VM or LXC on the off-site Proxmox host. Give it
  its own WireGuard identity and address; do not reuse `10.44.0.5`, which is
  assigned to `proxmox-backup-host`.
- The VPS should remain the public edge. Terminate HTTPS there and forward
  traffic privately over WireGuard to the Immich guest. Certificates are now
  configured on the VPS and are expected to be installed manually until ACME
  automation is added. Do not copy TLS private keys to the off-site host.
- The manually installed VPS certificate must cover every public hostname,
  including `immich.74services.net` (for example through a SAN or wildcard).
- Keep the Immich deployment separate from the main application deployment,
  both operationally and in the stacks repository. A separate Komodo Stack
  resource can deploy the off-site Git-backed stack.
- Treat the off-site location primarily as a storage and recovery boundary,
  not as a normal compute pool. The photo library, Immich database, thumbnails,
  and machine-learning data should remain pinned to storage local to the
  Immich guest.
- Prefer a VM for Immich. It gives Docker a normal kernel, stronger isolation
  from the Proxmox host, and simpler disk migration/recovery. An unprivileged
  LXC is lighter, but Docker-in-LXC usually needs nesting/feature exceptions
  and creates a less portable security and troubleshooting boundary. Use LXC
  only if the host's established operational model makes its trade-offs
  worthwhile.
- Use two virtual disks in the Immich guest: an SSD-backed OS disk and a
  separate data disk for the photo library. Keep the OS, Docker metadata,
  PostgreSQL database, and model cache on the SSD. Put Immich's
  `UPLOAD_LOCATION` on the data disk, mounted at a stable path such as
  `/srv/immich/library`.
- When the HDD arrives, attach it as a separate Proxmox virtual disk backed by
  HDD storage, format it in the guest with a Linux filesystem, mount it by
  UUID, and migrate the library before changing `UPLOAD_LOCATION`. Do not use
  an anonymous Docker volume for the photo library. Keep PostgreSQL on local
  SSD unless capacity forces a change; Immich recommends local SSD for its
  database and does not support a network share for it.
- Keep the recovery repository separate from the live Immich library where
  possible. A second disk or an independent backup target is preferable to
  storing the only backup copy on the same HDD as the photos.
- Do not introduce a cross-site Swarm solely for one Immich installation. A
  Swarm overlay across the WireGuard/WAN link adds control-plane, MTU,
  firewall, quorum, and persistent-storage concerns without providing useful
  failover for a stateful photo service.
- If a Swarm is adopted for the main application platform, keep the Immich
  guest as a worker or as an independent Compose host initially. Do not make
  the Immich workload a Swarm manager merely to obtain routing.
- If a future multi-site Swarm becomes justified, use a separate small manager
  VM at the off-site location rather than granting the Immich workload
  cluster-manager privileges. Three managers are needed for useful quorum
  tolerance; two managers do not provide fault tolerance for manager loss.

### Routing direction

Preferred initial design:

```text
Internet
  -> VPS edge proxy / TLS termination
  -> WireGuard
  -> Immich guest reverse proxy or Immich HTTP endpoint
  -> Immich services and local storage
```

The first implementation can use an explicit VPS route for the Immich
hostname. If more applications are later deployed off-site, revisit dynamic
discovery using one of these approaches:

1. A local Traefik on the Immich guest plus a small, authenticated route
   synchronizer/aggregator on the VPS.
2. `traefik-kop` and Redis if several independent Docker hosts need to publish
   ordinary Compose labels to one central Traefik.
3. Docker Swarm with native Traefik discovery if the main application hosts
   are deliberately converted into one cluster.

The first option is appropriate for the current single-application scope, but
the aggregator plugins found so far are small community projects. Do not make
the off-site backup service dependent on an untested plugin before validating
failure and stale-route behaviour.

### Encrypted cross-site backups

- Define a backup matrix for the application VM and Immich guest, including
  databases, application configuration, credentials/config metadata, and
  photo data.
- Use application-aware database dumps or backup procedures before copying
  database data.
- Use a repository-based encrypted backup tool such as Restic or Borg, with
  separate repository encryption keys and retention policies for each site.
- Prefer outbound, authenticated backup jobs over WireGuard rather than
  exposing backup services through the public proxy.
- Keep backup keys outside the backup data and store recovery material in a
  separately protected location.
- Add pruning, retention, monitoring, and periodic restore tests. A backup is
  not complete until a representative restore has been verified.
- Consider immutable or append-only protection on at least one copy so a
  compromised host cannot immediately erase both locations.

### Implementation TODO

- [ ] Decide whether the off-site guest should be a VM or LXC; document the
      Docker and storage trade-offs.
- [ ] Prefer a VM with an SSD OS disk and a separate HDD-backed data disk;
      record the chosen VM/LXC decision before provisioning.
- [ ] Create a new inventory host for the Immich guest, separate from
      `proxmox-backup-host`.
- [ ] Allocate a new WireGuard address and peer configuration for the guest.
- [x] Implement VPS HTTPS hostname routing for `immich.74services.net` and
      `deploy.74services.net`; replace the staged Immich address if the final
      allocation differs.
- [ ] Install the manually managed VPS certificate and key at the paths
      configured in `inventory/host_vars/proxy-vps.yml`.
- [ ] Define certificate renewal/ACME automation on the VPS later.
- [ ] Provision storage, filesystem layout, and backup capacity for Immich.
- [ ] Add the guest as a Komodo Server/Periphery target and create a separate
      Git-backed Immich Stack resource.
- [ ] Document the required Immich request size, timeout, WebSocket, and
      upload settings at the VPS edge.
- [ ] Decide whether the main application platform will adopt Swarm. Treat
      that as independent of the off-site Immich deployment.
- [ ] If multiple off-site applications are added, define an explicit routing
      discovery strategy before adding more public hostnames.
- [ ] Implement encrypted bidirectional backup jobs between the application
      VM and off-site guest.
- [ ] Test restoration of the application VM and Immich data using only the
      off-site recovery copies.
