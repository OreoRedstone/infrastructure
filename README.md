# Infrastructure playbook

This repository configures the hosts that form the management and application
platform. It does not provision Proxmox guests, an Immich stack, or the Samba
LXC; those remain separate workloads.

## Managed hosts

| Host | Group | WireGuard address | Purpose |
| --- | --- | --- | --- |
| `proxy-vps` | `vps`, `vpn` | `10.44.0.1` | Public WireGuard hub and Nginx stream proxy |
| `application-vm` | `applications`, `vpn` | `10.44.0.2` | Docker, Komodo, MongoDB, and backend reverse proxy |
| `proxmox-host` | `proxmox`, `vpn` | `10.44.0.3` | Proxmox management peer |
| `proxmox-backup-host` | `proxmox`, `vpn` | `10.44.0.5` | Off-site Proxmox management peer; future backup/Immich guest host |
| Laptop | external peer | `10.44.0.4` | Ansible controller and management client |

The managed WireGuard interface is `wg1`, using UDP port `51821` and network
`10.44.0.0/24`. The VPS forwards traffic between VPN peers only. It is not an
Internet gateway: there is no default route or masquerading for VPN clients.

## Repository layout

- `inventory/hosts.yml` defines host groups and VPN membership.
- `inventory/host_vars/` contains connection addresses and host settings.
- `inventory/group_vars/` contains shared settings and encrypted secrets.
- `playbooks/playbook.yml` applies the common, VPN, firewall, metrics, proxy,
  Docker, and Komodo roles.
- `roles/` contains reusable Ansible roles.
- `examples/` contains the vault template and laptop WireGuard profile.

## Prerequisites

Install the required Ansible collections:

```sh
ansible-galaxy collection install -r requirements.yml
```

Set the bootstrap SSH addresses in `inventory/host_vars/` before provisioning:

- `application-vm`: `192.168.0.100`
- `proxmox-host`: `192.168.0.48`
- `proxmox-backup-host`: `192.168.0.49`
- `proxy-vps`: `217.154.34.64`

Keep the VPS `wireguard_endpoint` as its public address. After bootstrap,
Ansible automatically selects each managed host's WireGuard address.

Add administrators' public SSH keys to
`inventory/group_vars/all/main.yml`:

```yaml
admin_authorized_keys:
  - ssh-ed25519 AAAA... laptop
```

Public keys are safe to commit; private SSH and WireGuard keys are not. Managed
hosts generate their own WireGuard keys under `/etc/wireguard/`; back up those
private keys securely, but never put them in this repository.

Create the encrypted variables file if it does not already exist:

```sh
cp examples/vault.yml inventory/group_vars/all/vault.yml
# Replace every REPLACE_* value, then:
ansible-vault encrypt inventory/group_vars/all/vault.yml
```

The vault contains the laptop public key and Komodo credentials. Supply the
vault password with `--ask-vault-pass` or the normal local vault-password-file
mechanism for every command that loads this inventory.

## Initial provisioning and VPN onboarding

During bootstrap, temporary SSH rules permit access from the LAN (and from the
Internet for the VPS). Hosts in the inventory `bootstrap` group use `root` while
`bootstrap_mode` is true. Once a host's administrator account and management
VPN are working, remove it from that group; this changes only that host.

Run the first VPN deployment against the complete `vpn` group. This is
required because the VPS renders peer configuration from public-key facts
collected from every managed VPN host:

```sh
ansible immich2 -m ping --ask-vault-pass
ansible-playbook playbooks/playbook.yml \
  --limit immich2 \
  --tags common,firewall,node_exporter \
  --ask-vault-pass
```

After the guest is reachable and enrolled in `vpn`, run the WireGuard play for
the complete managed VPN group. The VPS renders peer configuration from
public-key facts collected from every managed VPN host. Inspect generated keys
with:

```sh
ansible-playbook playbooks/show-vpn-public-keys.yml --ask-vault-pass
```

## Laptop VPN

Generate the laptop key pair outside the repository:

```sh
umask 077
wg genkey | tee laptop.key | wg pubkey
```

Put only the printed public key in encrypted
`vault_wireguard_laptop_public_key`. Copy `examples/laptop-wg0.conf` outside
the repository, replace its placeholders, and install it as
`/etc/wireguard/wg1.conf` on the laptop. The example's
`AllowedIPs = 10.44.0.0/24` routes only management traffic; it does not
create a full-tunnel Internet VPN.

After bringing up the profile, verify `10.44.0.1`, `.2`, `.3`, `.5`, and then
`.6` from the laptop. Remove `immich2` from the `bootstrap` group and run:

```sh
ansible-playbook playbooks/playbook.yml --ask-vault-pass
```

This removes temporary bootstrap firewall rules and disables root SSH login.
The Proxmox hosts remain manageable as `oreo` through WireGuard.

## Services and firewall

The firewall defaults to deny incoming and routed traffic and allow outgoing
traffic. It permits the management VPN, the VPS WireGuard endpoint on
`51821/udp`, and configured public stream ports. Proxmox hosts set the routed
policy to `allow` so UFW does not silently break guest bridge traffic; review
that exception if Proxmox's own firewall is enabled.

The VPS exposes HTTPS through its Nginx reverse proxy and forwards the game
stream ports through its Nginx stream proxy:

| Public protocol/port | Backend port | Workload |
| --- | --- | --- |
| TCP `443` | HTTP to selected WireGuard backend | HTTPS/application reverse proxy |
| TCP `25565` | TCP `25565` | Minecraft |
| UDP `27016` | UDP `27016` | Space Engineers |
| UDP `34197` | UDP `34197` | Factorio |

On the VPS, Nginx terminates HTTPS and routes `deploy.74services.net` to
Komodo on `application-vm`, `immich.74services.net` to the staged off-site
Immich guest, and other hostnames to application Traefik at
`10.44.0.2:9999`. The VPS certificate paths are configured in
`inventory/host_vars/proxy-vps.yml` and must exist there. The Immich route is
staged for `10.44.0.6`, while `10.44.0.5` remains assigned to
`proxmox-backup-host`.

On `application-vm`, the local Nginx configuration remains available for
backend-local routing and terminates HTTPS when accessed directly.

The application VM also runs Komodo Core, Periphery, MongoDB, and the optional
authenticated Ntfy wrapper. Komodo Core is published on port `9120` for the
management VPN and configured LAN sources. MongoDB is not published on a host
port. Compose volumes persist MongoDB data, Komodo keys, and Komodo backups.

## Off-site Immich routing

The physical `proxmox-backup-host` is only a managed Proxmox node. Create a
separate Immich VM or LXC and give that guest its own address and documented
networking. Do not reuse `10.44.0.5`, which belongs to the Proxmox host. This
repository currently has no Immich deployment role; manage the guest's Docker
Compose and Traefik configuration separately until that is added.

The VPS now has explicit HTTP hostname routes. Pointing
`immich.74services.net` at the VPS will route it to the staged off-site
address once that guest and its HTTP endpoint are provisioned. Future dynamic
discovery could replace this static map. Sensible designs are:

1. Terminate TLS at the VPS and use an HTTP reverse proxy to send the Immich
   hostname over WireGuard to the guest. Existing application hostnames can
   continue to go to `application-vm`; certificates are centralized.
2. Add SNI-aware TLS passthrough to the VPS Nginx stream configuration using
   `ssl_preread`, with certificates and Traefik on both backends.
3. Publish the Immich guest through a separate off-site public edge and point
   only its DNS record there, if the site has a stable public IP and safe
   inbound forwarding.

For Immich, preserve WebSocket support, allow large request bodies, use long
proxy timeouts, and ensure the photo library and database are included in
off-site backups.

## Validation and operations

```sh
ansible-inventory --graph
ansible-playbook playbooks/playbook.yml --syntax-check --ask-vault-pass
ansible-playbook playbooks/playbook.yml --check --diff --ask-vault-pass
git diff --check
```

After VPN deployment, check on each host:

```sh
sudo wg show wg1
sudo systemctl status wg-quick@wg1
```

The first WireGuard run should not use a narrow `--limit`; later service-only
changes may be limited safely once all peer facts are available.

## Migration notes

The existing VPS remains the public edge and WireGuard hub. Before any VPS
reimage or major cutover, capture its WireGuard, Nginx, and UFW state and take
provider and guest snapshots. Keep the old networking VM available for rollback
until the replacement application platform and public ports have been tested.

The Samba LXC is not recreated by this playbook. Keep it in place until its
mount paths, shares, ACLs, users, and backup requirements are documented. When
migrating its data, preserve ownership, ACLs, and extended attributes with:

```sh
rsync -aHAX --numeric-ids SOURCE/ DESTINATION/
```

Never run two Komodo Cores against the same MongoDB data. Validate application
data, storage mounts, VPN peers, public ports, and backups before retiring old
services.
