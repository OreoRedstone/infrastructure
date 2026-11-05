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
| `immich2` | `offsite_applications`, `vpn` | `10.44.0.6` | Off-site Komodo Periphery and separately managed Immich workload |
| Laptop | external peer | `10.44.0.4` | Ansible controller and management client |

The managed WireGuard interface is `wg1`, using UDP port `51821` and network
`10.44.0.0/24`. The VPS forwards traffic between VPN peers only. It is not an
Internet gateway: there is no default route or masquerading for VPN clients.

## Repository layout

See [Proxmox automation options](PROXMOX-AUTOMATION.md) for researched options
and a proposed sequence for managing storage, backups, and guest provisioning.

- `inventory/hosts.yml` defines host groups and VPN membership.
- `inventory/host_vars/` contains connection addresses and host settings.
- `inventory/group_vars/` contains shared settings and encrypted secrets.
- `playbooks/playbook.yml` applies the common, VPN, firewall, metrics, proxy,
  Docker, and Komodo roles.
- `roles/` contains reusable Ansible roles.
- `examples/` contains the vault template and laptop WireGuard profile.

## Prerequisites

The reviewed controller uses Ansible Core 2.21.5, which requires Python 3.12
or newer. Install it inside your controller's Python virtual environment,
or use an equivalent distribution package:

```sh
python3 -m pip install -r requirements-controller.txt
```

Install the pinned Ansible collections:

```sh
ansible-galaxy collection install -r requirements.yml
```

Container versions are shared in `roles/komodo/defaults/main.yml`: Core and
all Periphery agents use the same exact release. Update them together using
`--limit application-vm,immich2 --tags komodo`. This also runs the Docker role
on those hosts. Review [VERSION-AUDIT.md](VERSION-AUDIT.md) before changing
MongoDB or removing the application host's temporary rseq workaround.

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

## Create a new host and enable SSH

Create the VM or install the physical machine manually; this repository does
not create guests, partitions, storage mounts, or network interfaces. For a
new application guest, use a minimal Debian 13 installation with a unique
hostname and LAN address, working DNS/default gateway, and the Debian security
repository enabled. Keep console access available through onboarding.

During installation, create the administrator `oreo` (matching `admin_user` in
`inventory/group_vars/all/main.yml`), set a strong local password, and select
SSH server and standard system utilities. On Debian, leaving the installer root
password blank normally enables sudo for the first user. Verify this rather
than assuming it. If the account was not created by the installer, run the
following as root from the console; skip `adduser` if `oreo` already exists:

```sh
apt update
apt install openssh-server python3 sudo ca-certificates
adduser oreo
usermod -aG sudo oreo
systemctl enable --now ssh
hostname -I
```

Apply OS security updates and reboot if required before exposing workloads.
Log out and back in after changing group membership. From an `oreo` console
session, confirm `sudo -v` succeeds using the local account password.

Install the controller's **public** SSH key using the console. As `oreo`, run:

```sh
install -d -m 700 ~/.ssh
# Paste the controller's public key as one line into this file:
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

The key should match an entry in `admin_authorized_keys`; never copy the
controller's private key to a server. If editing as root, ensure `/home/oreo/.ssh`
and its files belong to `oreo:oreo`. Allow TCP/22 from the controller's actual
source address in any provider, hypervisor, or existing host firewall.

From the controller, replace `NEW_LAN_IP` with the address shown at the console:

```sh
ssh oreo@NEW_LAN_IP
sudo -v
```

Verify the SSH host-key fingerprint against the console before accepting it.
For a replacement at a reused address, verify the new fingerprint before
removing the old known-host entry. Do not disable host-key checking.

### Register the host in Ansible

Add the host to the appropriate workload group and to `bootstrap` in
`inventory/hosts.yml`. Use `offsite_applications` for a Periphery-only guest;
`applications` deploys Komodo Core and MongoDB as well. These workload groups
already inherit membership in `vpn`. Proxmox nodes belong in `proxmox` and
should retain `manage_hostname: false`.

Create `inventory/host_vars/NEW_HOST.yml` following the existing hosts. Replace
all example addresses and names before running Ansible:

```yaml
---
ansible_host_bootstrap: NEW_LAN_IP
ansible_host: "{{ ansible_host_bootstrap if (bootstrap_mode | bool) else (wireguard_address.split('/')[0]) }}"
ansible_user: "{{ admin_user }}"
ansible_become: true
hostname: NEW_HOST
wireguard_address: 10.44.0.7/24  # Example only: verify it is unused.
wireguard_peers:
  - name: VPS hub
    host: proxy-vps
    endpoint: "{{ hostvars['proxy-vps']['wireguard_endpoint'] }}:{{ wireguard_port }}"
    allowed_ips:
      - "{{ wireguard_vpn_network }}"
    persistent_keepalive: "{{ wireguard_persistent_keepalive }}"
```

Initially supply the account's sudo password with `--ask-become-pass`, or use a
host-specific encrypted vault variable as the existing application hosts do.
The common role installs passwordless sudo by default; subsequent runs no
longer need the initial password. Bootstrap membership selects the connection
address and temporary firewall rules; host variables select the SSH user.

Add the new peer to `proxy-vps.yml` with its unique VPN address as a `/32`.
For a remote LAN, arrange routing and SSH firewall access before onboarding;
copying the existing off-site host's address will not create a route. Override
`ufw_bootstrap_rules` if the shared LAN source does not include the controller.
For Periphery-only guests, also copy the relevant `komodo_deploy_core: false`
and onboarding settings from `immich2.yml` and supply its vault onboarding key.

Replace `NEW_HOST` in these commands with the inventory name:

```sh
ansible NEW_HOST -m ping --ask-vault-pass --ask-become-pass
ansible-playbook playbooks/playbook.yml --limit NEW_HOST --tags common --ask-vault-pass --ask-become-pass
ansible-playbook playbooks/playbook.yml --limit vpn --tags wireguard --ask-vault-pass
ansible-playbook playbooks/playbook.yml --limit NEW_HOST --ask-vault-pass
```

The VPN run needs all managed peers reachable to collect their public keys.
Before the full host run, prepare any required storage mounts, certificates,
and workload-specific vault settings. Confirm key-based administrator SSH over
the new VPN address, then remove the host from `bootstrap` and rerun the full
host command. This switches management to WireGuard and removes temporary SSH
rules. Verify access again before closing the console.

## Initial provisioning and VPN onboarding

During bootstrap, temporary SSH rules permit access from the LAN (and from the
Internet for the VPS). The inventory `bootstrap` group enables `bootstrap_mode` for its members;
it does not change the SSH user. Set `ansible_user` to an existing key-enabled
sudo account (or root during initial console-assisted onboarding). Once a
host's administrator account and management VPN work, remove it from that group.

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

The guest is already enrolled in `vpn` in the current inventory. After it is reachable, run the WireGuard play for
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
Komodo on `application-vm`, `photos-2.74services.net` to the staged off-site
Immich guest, and other hostnames to application Traefik at
`10.44.0.2:9999`. The VPS certificate paths are configured in
`inventory/host_vars/proxy-vps.yml`; the enabled Certbot tasks create them. The Immich route targets
HTTP port `2283` at `10.44.0.6`, while `10.44.0.5` remains assigned to
`proxmox-backup-host`.

The VPS obtains and renews `*.74services.net` with Certbot using Cloudflare
DNS-01 validation. Before applying the proxy role, add
`vault_certbot_email` and a narrowly scoped `vault_cloudflare_dns_api_token`
to `inventory/group_vars/all/vault.yml`, then encrypt the file. The token needs
Zone:Read and DNS:Edit access to the `74services.net` zone. The role installs
the Certbot Cloudflare plugin, requests the wildcard certificate, and enables
`certbot.timer`; a Certbot deploy hook tests and reloads Nginx after renewals.
Check renewal readiness on the VPS with `sudo certbot renew --dry-run` and
`systemctl status certbot.timer`. No inbound HTTP challenge port is needed.

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

The VPS has explicit HTTP hostname routes. Pointing
`photos-2.74services.net` at the VPS will route it to the staged off-site
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

## Debian application VM migration

Debian 13 (trixie) is the preferred fresh application VM target; Debian 12
(bookworm) is also accepted by the Docker role. Docker repositories and keys
are selected from gathered distribution facts, not from the inventory hostname.
See [Docker's Debian requirements](https://docs.docker.com/engine/install/debian/).
The MongoDB rseq workaround is applied only to Ubuntu hosts.

Prepare a minimal Debian installation with systemd, working DNS, time sync,
the Debian security APT repository, OpenSSH server, Python 3, and sudo. Apply
security updates and boot the updated kernel before exposing services. The
playbook installs application packages but does not upgrade the OS/kernel.
For the first connection use a key-enabled root account or an existing sudo
administrator; a minimal installation may require Python/sudo installation
from the console before Ansible can run. Set ansible_user explicitly to that
account when onboarding; membership in bootstrap does not select the SSH user.

Build the replacement on a distinct LAN address. Add application-vm to the
bootstrap inventory group and set its bootstrap address to the replacement.
Keep the old guest off the production VPN while the new guest uses its identity;
never run two guests with 10.44.0.2 simultaneously. Back up the old WireGuard
private key securely and restore it on the replacement before VPN configuration,
or collect all VPN peer facts and update the hub with the new public key.
Run common provisioning first, then the VPN plays against the full vpn group:

```sh
ansible-playbook playbooks/playbook.yml --limit application-vm --tags common --ask-vault-pass
ansible-playbook playbooks/playbook.yml --limit vpn --tags wireguard --ask-vault-pass
ansible-playbook playbooks/playbook.yml --limit application-vm --ask-vault-pass
```

Before the final command, restore /etc/letsencrypt (including symlink targets)
for the application host's local HTTPS proxy, or arrange replacement certificates.
Restore application mounts, ownership/ACLs, Compose definitions and secrets,
Komodo keys/backups, and consistent database backups before starting workloads.
This repository does not recreate arbitrary Komodo stacks, GPU drivers or
passthrough, NAS mounts, or external application data. Inventory those separately.
Do not copy a running MongoDB data directory; stop writes for the final sync or
use a supported database backup/restore procedure.

After provisioning, verify VPN handshakes, mounts, database contents, Komodo,
HTTPS routing, certificate renewal, monitoring, and GPU workloads. Remove the
host from bootstrap only after administrator SSH and VPN access work. Keep the
old VM powered off for rollback; avoid concurrent database writers. A dry run
on an empty guest cannot fully validate packages, generated VPN keys or services;
perform a real staging deployment before production cutover.

### Role task structure

The reverse-proxy role installs Nginx, imports `certificates.yml` for optional
Cloudflare/Certbot lifecycle management, then imports `proxy.yml` for routes
and service configuration. The WireGuard role imports `keys.yml` before
`configure.yml`; the two VPN plays still collect every peer key before rendering
any peer configuration. Static imports preserve role tags and task conditions.

The firewall role intentionally retains an ordered list of policy and workload
rules in one file; its shared defaults and host inventory define the exceptions.
