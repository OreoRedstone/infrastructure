# Proxmox automation options

Research reviewed 3 November 2026. These are recommendations, not deployed
configuration. No live Proxmox API or host inspection was performed.

## Recommendation for this repository

Extend the existing Ansible setup first. Add a separate, explicitly invoked
Proxmox API playbook for storage, backup jobs, and later guest creation;
retain SSH roles for host operating-system configuration. Introduce OpenTofu
only if guest lifecycle management becomes large enough to justify state and a
second workflow. This recommendation follows the existing two-node scope and
the already-decided reciprocal NFS backup plan in [TODO.md](TODO.md).

The repository currently configures Proxmox administrator access, packages,
WireGuard, UFW, and node exporter through [playbooks/playbook.yml](playbooks/playbook.yml).
It intentionally preserves node hostnames, permits routed bridge traffic, and
does not create guests, storage mounts, or network interfaces. The installed
collection requirements do not include `community.proxmox`. These observations
come from the repository, not a claim about the hosts' complete configuration.

## What each tool should own

| Layer | Suggested owner | Examples |
| --- | --- | --- |
| Physical installation and recovery access | Installer/manual runbook initially | Disk layout, boot recovery, console, initial management network |
| Host operating system | Existing Ansible over SSH | Packages, accounts, WireGuard, NFS exports, narrowly scoped NIC workaround |
| Proxmox API objects | Ansible initially; optionally OpenTofu later | Storage registration, backup schedules, VM hardware, templates, ACLs |
| Guest first boot | Cloud-init | Initial address, administrator public key, minimal prerequisites |
| Guest ongoing configuration | Existing Ansible roles | Docker, Komodo, proxy, VPN and monitoring |
| Application data | Backup/restore procedures | Databases, photo library, persistent volumes |

Choose one owner for each resource or setting; do not let Ansible and OpenTofu
both reconcile the same VM disks, networking, or storage entries.

## Ansible can cover more than guest software

The community collection currently documents VM/LXC management, disks, NICs,
storage, backup execution and schedules, users/ACLs, firewall, networking, and
information modules. It is community-maintained, rather than a Proxmox vendor
product. Add and pin a tested collection release instead of assuming the
existing `community.general` dependency supplies this interface.
See the [collection module index](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/index.html).

For guest provisioning, `community.proxmox.proxmox_kvm` supports cloning and
cloud-init options; `community.proxmox.proxmox` handles containers. The KVM
module requires `proxmoxer >= 2.3` and `requests` on the executing host, so API
tasks can run on the controller rather than installing those libraries on every
hypervisor. Verify update behavior for each field before adopting existing
guests; creation succeeding does not prove full reconciliation.
See the [KVM module](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/proxmox_kvm_module.html).

Host API coverage is specific. The `proxmox_node` module manages certificates
and power states; it is not a complete Debian/PVE host configuration role.
The node-network module can stage bridges, bonds, VLANs and interface settings,
then requires an explicit `state=apply`. Keep remote management-network changes
in a separate maintenance workflow with working console access. Use SSH tasks
for settings outside the API/module schema, rather than replacing entire
Proxmox-managed configuration files indiscriminately.
See [node](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/proxmox_node_module.html)
and [node network](https://docs.ansible.com/projects/ansible/latest/collections/community/proxmox/proxmox_node_network_module.html).

## OpenTofu with bpg/proxmox

This is a credible optional guest-lifecycle layer: the provider supports
Terraform/OpenTofu and publishes to both registries. Most operations use the
PVE API; some, including snippet uploads, require SSH. It is an independent
community provider, with version 0.x compatibility caveats. Check the selected
release against the actual PVE versions and pin it plus the dependency lock
file. Upstream currently targets PVE 9, provides limited PVE 8 support, and
does not support PVE 7. See the [provider's own compatibility and security
notes](https://github.com/bpg/terraform-provider-proxmox).

Its practical benefit is a reviewable lifecycle plan. Start with a disposable
VM; import existing resources only after writing matching configuration and
reviewing the resulting plan. Regular `tofu plan` can reveal externally made
changes to managed objects; keep refresh enabled. This does not discover or
manage every host setting automatically. See [OpenTofu planning](https://opentofu.org/docs/cli/commands/plan/).

The current common role disables root SSH after bootstrap. Any provider
operation needing SSH must use an explicitly configured supported sudo account;
an API token alone does not cover those operations. Review the upstream SSH
privilege requirements before granting access.

## Cloud-init as the handoff

Prepare one tested Debian template, clone it with a unique VMID/address, and
provide an administrator public key and networking before first boot. Include
the SSH/Python/sudo prerequisites expected by this repository. Cloud-init must
be installed inside the guest; PVE supplies its configuration through a virtual
CD-ROM. Custom snippets require suitable storage and availability on every
potential migration destination. Keep ongoing role configuration in Ansible.
See [Proxmox's cloud-init documentation source](https://github.com/proxmox/pve-docs/blob/master/qm-cloud-init.adoc).

## Backups, credentials, and verification

Implement [TODO.md](TODO.md)'s reciprocal NFS/vzdump design first: dedicated
capacity, exports limited to the peer over WireGuard, `content backup` storage,
explicit zstd compression, retention, bandwidth limits and failure notification.
Ansible's storage and backup-schedule modules are candidates for the API side;
Ansible SSH roles can configure the NFS servers. PVE documents file-level/NFS
backup storage as an alternative to PBS. PBS is a later option if deduplication
and its additional recovery features justify changing the chosen design.
See [Proxmox backup documentation](https://github.com/proxmox/pve-docs/blob/master/vzdump.adoc).

Excluding OS disks remains conditional on a demonstrated rebuild and separate
protection for secrets, certificates, VPN identities, workload definitions and
host configuration. Restore the selected data disks into temporary guests and
validate application/database recovery before declaring the jobs operational.
Code and VM snapshots alone do not establish that recovery works.

Use a dedicated API user and privilege-separated token with the ACLs required
by the specific endpoints. Token permissions intersect with its user's
permissions; do not default to unrestricted root credentials. Keep TLS
verification enabled and trust the intended CA. See [Proxmox API token
permissions](https://github.com/proxmox/pve-docs/blob/master/pveum.adoc).

Store credentials in the existing encrypted Ansible vault or inject them from
the controller environment. Protect secret-bearing task output. If OpenTofu is
introduced, keep state and saved plans out of Git, use protected storage with
locking and backups, and configure encryption. State can contain resource
secrets and local state is plaintext by default. See [OpenTofu sensitive
state](https://opentofu.org/docs/language/state/sensitive-data/).

## Proposed sequence

1. Record actual node names/PVE versions, storage IDs, network topology, guest
   IDs and disk ownership. Preserve existing guests while completing the
   reciprocal backups and restore drills.
2. Add a dedicated Ansible Proxmox host/API role boundary and pinned controller
   dependencies. Start with read-only information tasks and the backup/storage
   configuration. Keep new tasks outside routine application deployments.
3. Automate persistence of the documented `eno1` offload workaround as a
   host-specific setting after reviewing the active interface configuration;
   verify it after reboot. Follow [the incident record](PROXMOX-NIC-HARDWARE-HANG.md)
   rather than applying the workaround to both hosts indiscriminately.
4. Provision one disposable Debian cloud-init guest, run existing Ansible
   onboarding, rerun for idempotence, and prove destroy/rebuild plus data restore.
5. Add OpenTofu only if explicit VM lifecycle plans/imports are valuable enough
   to warrant the extra state workflow; migrate resource ownership deliberately.
