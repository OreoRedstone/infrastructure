# Infrastructure playbook

This Ansible repository builds a management VPN and application platform:

- `proxy-vps` is the public WireGuard hub and Nginx TCP/UDP stream proxy.
- `application-vm` runs Docker, Komodo Core, Komodo Periphery, and MongoDB.
- `proxmox-host` joins WireGuard for management. It is treated as Debian, and
  its hostname and routed VM traffic are deliberately left intact.
- the controller laptop is `10.13.13.4`; it can manage every node through the
  VPS without sending normal Internet traffic through the VPN.

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
`show-vpn-public-keys.yml`. Import that profile into WireGuard, or on Linux run
`sudo wg-quick up <profile>`.

After confirming `ping 10.13.13.1`, `.2`, and `.3` from the laptop:

1. Set `bootstrap_mode: false` in `inventory/group_vars/all/main.yml`. This
   removes temporary Internet/LAN SSH and Proxmox UI rules and disables root
   login.
2. Change each host's `ansible_user` to `oreo`. Change `ansible_host` to its
   `10.13.13.x` address if management should be strictly VPN-only.
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

Komodo binds only to `10.13.13.2:9120`. Its Core and Periphery containers share
their generated authentication keys, and Periphery connects outbound to Core.
MongoDB is not published on a host port. The deployment follows Komodo's
supported MongoDB Compose layout:

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
