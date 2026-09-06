# Infrastructure playbook

This Ansible repository builds a management VPN and application platform:

- `proxy-vps` is the public WireGuard hub and Nginx TCP/UDP stream proxy.
- `application-vm` runs Docker, Komodo Core, Komodo Periphery, and MongoDB.
- `proxmox-host` joins WireGuard for management. It is treated as Debian, and
  its hostname and routed VM traffic are deliberately left intact.
- the controller laptop is an Arch Linux host at `10.13.13.4`; it can manage
  every node through the VPS without sending normal Internet traffic through
  the VPN. During migration, the laptop can still reach Proxmox directly over
  the existing LAN.

## Address plan

| Node | WireGuard address | Role |
| --- | --- | --- |
| VPS | `10.13.13.1` | Public hub and stream proxy |
| Applications | `10.13.13.2` | Docker and Komodo |
| Proxmox | `10.13.13.3` | VPN-only management peer |
| Laptop | `10.13.13.4` | Ansible controller |

The VPS enables forwarding only between `wg0` peers. There is no default route,
masquerading, or Internet full tunnel.

## First run

1. Install Ansible dependencies:

   ```sh
   ansible-galaxy collection install -r requirements.yml
   ```

2. Replace the `CHANGE_ME_*` SSH addresses in `inventory/host_vars`. Leave the
   VPS endpoint in `wireguard_endpoint` public even if its later `ansible_host`
   changes to `10.13.13.1`.

3. Add one or more public keys to `admin_authorized_keys` in
   `inventory/group_vars/all/main.yml`. Root key login remains available while
   `bootstrap_mode` is true; passwords are disabled immediately.

   The default bootstrap creates `oreo` with locked password authentication but
   grants it passwordless sudo, so Ansible does not require a become password.
   If your policy requires password-protected sudo, set
   `admin_sudo_nopasswd: false`, provide a sudo password securely, and run with
   `ansible-playbook -K`.

4. Create the encrypted variables file:

   ```sh
   cp examples/vault.yml inventory/group_vars/all/vault.yml
   ansible-vault encrypt inventory/group_vars/all/vault.yml
   ```

   Replace all example values first. Generate random Komodo values with a
   password manager or `openssl rand -base64 48`.

5. Check connectivity and deploy the managed VPN peers together:

   ```sh
   ansible all -m ping
   ansible-playbook playbooks/playbook.yml --tags common,wireguard,firewall
   ansible-playbook playbooks/show-vpn-public-keys.yml
   ```

Managed servers generate their own WireGuard keys on first use. Back up
`/etc/wireguard/wg0.key` from each node securely; it is never stored in this
repository.

## Join the laptop

Generate a laptop key pair outside this repository:

```sh
umask 077
wg genkey | tee laptop.key | wg pubkey
```

Put the printed public key in `vault_wireguard_laptop_public_key`, rerun the
WireGuard play, then copy `examples/laptop-wg0.conf` outside the repository.
Insert the laptop private key and the VPS public key reported by
`show-vpn-public-keys.yml`. On Arch, install `wireguard-tools`, copy the profile
to `/etc/wireguard/wg0.conf`, and run `sudo wg-quick up wg0`.

After confirming `ping 10.13.13.1`, `.2`, and `.3` from the laptop:

1. Set `bootstrap_mode: false` in `inventory/group_vars/all/main.yml`. This
   removes temporary Internet/LAN SSH and Proxmox UI rules and disables root
   login.
2. Ensure each host's `ansible_user` is set to `oreo` where applicable. The
   inventory now selects each host's WireGuard address automatically whenever
   `bootstrap_mode` is false; it uses the recorded bootstrap address while the
   mode is true.
3. Run the complete playbook.

   ```sh
   ansible-playbook playbooks/playbook.yml
   ```

## Services and firewall

UFW denies incoming and routed traffic by default, permits the management VPN,
opens `51820/udp` on the VPS, and opens the configured public stream ports. The
Proxmox routed policy remains `allow` so enabling UFW does not stop guest bridge
traffic. Review that exception if Proxmox's own firewall is already enabled.

Nginx forwards these public ports to `10.13.13.2`: TCP `443`, TCP `25565`, UDP
`27016`, and UDP `34197`. It does not terminate TLS.

Komodo binds directly to `10.13.13.2:9120` and `192.168.0.100:9120`; no external
Docker `nginx` network is required. Its Core and Periphery containers share
their generated authentication keys, and Periphery connects outbound to Core.
MongoDB is not published on a host port. MongoDB and Komodo keys remain in
Docker named volumes (`mongo-data`, `mongo-config`, and `keys`), matching the
existing Compose deployment. The deployment follows Komodo's supported MongoDB
Compose layout:

- <https://komo.do/docs/setup/mongo>
- <https://komo.do/docs/setup/connect-servers>

## Useful commands

```sh
ansible-inventory --graph
ansible-playbook playbooks/playbook.yml --syntax-check
# Use check mode after the managed hosts have generated their WireGuard keys.
ansible-playbook playbooks/playbook.yml --check --diff
```

Do not use `--limit` for the first WireGuard run: public-key facts are collected
from the complete `vpn` group before peer configurations are rendered.

## Migration runbook (reuse the current VPS)

The migration should be performed in this order. Do not reimage the VPS until
the Applications VM and Proxmox peer have been prepared, because the VPS is
currently the WireGuard endpoint for the existing networking VM.

1. **Capture state and stop writes.** Save the current VPS WireGuard/Nginx/UFW
   configuration, the networking VM's `/etc/komodo`, its Komodo Compose file,
   MongoDB Docker volume names, and the Applications VM's Docker state. Stop
   Komodo schedules and application writes during the final database copy.
2. **Snapshot everything.** Take provider snapshots of the VPS, Proxmox VM/LXC
   snapshots, and an off-host backup of the application disk and Samba data.
   Keep the networking VM powered off—not deleted—for rollback.
3. **Prepare Applications in place.** Keep `192.168.0.100` and its application
   virtual disk. Install Docker and the new WireGuard client with Ansible. Do
   not enable the new Komodo Compose project yet.
4. **Move Komodo state.** Stop the old Core on the networking VM. Back up and
   restore the existing MongoDB Docker volumes and Core/Periphery key volume.
   Keep the Compose project name `komodo` so its named volumes are reused; if
   the old project used different names, set explicit volume names before the
   first deployment. Validate ownership, database credentials, and key files
   before starting Compose. Never run both Cores against the same MongoDB data.
   The current Compose file does not mount a persistent Core/Periphery key
   volume, so capture the running containers' Komodo authentication state and
   expect to re-onboard Periphery if those keys cannot be recovered.
   Copy non-secret settings from the old `.env` into `komodo_extra_env`; set
   the existing database credentials in the dedicated encrypted vault fields.
   The Ntfy wrapper is disabled for this migration and can be enabled later by
   supplying its `ntfy.env` contents.
5. **Move Periphery and validate locally.** Start Core and Periphery on
   Applications, connect Periphery to Core, then validate Komodo login and
   every application stack using the existing application repository.
6. **Add WireGuard peers.** Configure Applications (`10.13.13.2`) and Proxmox
   (`10.13.13.3`) as peers. Before applying the VPS role, put the current
   networking VM peer in `wireguard_transition_peers`; this keeps the old
   client connected while Applications is tested. Verify laptop-to-Proxmox and
   laptop-to-Applications access while the old path is still available.
   First compare the old networking peer's VPN address with the new address
   plan. If it already uses `10.13.13.2` (or another address reserved above),
   stage Applications on a temporary unused address such as `10.13.13.10` and
   change the Nginx upstream only after the old peer is removed. Never run two
   peers with the same `AllowedIPs`.
7. **Reconfigure the VPS in place.** Use the provider console or rescue mode,
   not only SSH. Preserve the public IP, install the Ansible-managed WireGuard
   hub, and change Nginx upstreams to `10.13.13.2`. Keep the VPS snapshot until
   the public ports have passed external tests.
8. **Cut over and retire the networking VM.** Remove
   `wireguard_transition_peers`, then confirm all four stream ports,
   Komodo, storage mounts, and application health. Then stop the networking VM.
   Leave it and its snapshot available for at least 72 hours before deletion.
9. **Close bootstrap access.** Once the Arch laptop VPN is working, set
   `bootstrap_mode: false`, switch Ansible users to `oreo`, and optionally use
   the WireGuard addresses as `ansible_host` values.

The Samba LXC is not automatically recreated by this playbook yet. Keep it in
place during the application cutover. Once its mount path, share names, ACLs,
and Samba users are documented, add it as a separate Ansible-managed storage
host. If applications consume this share, migrate it before moving their
containers and preserve ACLs/xattrs with an `rsync -aHAX --numeric-ids` copy.
