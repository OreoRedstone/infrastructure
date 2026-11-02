# Version audit — 2026-10-27

## Reviewed dependencies

| Dependency | Repository selection | Evidence and compatibility |
| --- | --- | --- |
| Ansible Core | `2.21.5` | Installed controller version; requires controller Python >=3.12. |
| `community.general` | `13.5.0` | Installed version; requires Ansible >=2.18. |
| `community.docker` | `5.4.0` | Installed version; requires Ansible >=2.17; Compose support includes version 5 output handling. |
| `ansible.posix` | `2.2.2` | Installed version; requires Ansible >=2.16. |
| Komodo Core and Periphery | `2.3.4` | Latest upstream release verified through GitHub; both image manifests include linux/amd64 and linux/arm64. One shared role default prevents host overrides drifting apart. |
| MongoDB | `8.3.13` | Published image manifest verified for linux/amd64 and linux/arm64. Stays within the existing database's 8.3 release series. |
| Ntfy wrapper | `foxxmd/komodo-ntfy-alerter:b008eb5` | Existing commit-tagged image; manifest verified for linux/amd64 and linux/arm64. Retained without an unverified upgrade. End-to-end notification delivery was not tested. |

All container image checks used `docker manifest inspect` on the application
host. No images were pulled or containers recreated during this audit.
Exact release tags make updates explicit; unlike digest references, they do
not protect against an upstream publisher replacing an existing tag.

## Host compatibility

Read-only SSH checks found:

| Host | Distribution | Kernel | Python |
| --- | --- | --- | --- |
| `proxy-vps` | Ubuntu 26.04.1 | `7.0.0-38-generic` | 3.14.4 |
| `application-vm` | Ubuntu 26.04.1 | `7.0.0-38-generic` | Not separately queried during this audit |
| `proxmox-host` | Debian 13.7 | `7.0.14-16-pve` | 3.13.5 |
| `proxmox-backup-host` | Debian 13.5 | `7.0.2-6-pve` | 3.13.5 |
| `immich2` | Debian 13.7 | `6.12.107+deb13-amd64` | 3.13.5 |

The application host runs Docker Engine 29.9.2 and Compose 5.6.1.
Docker's official repository supports Ubuntu Resolute and Debian Trixie;
the role selects the host's distribution and codename rather than a fixed
Ubuntu repository. Docker packages use `state: present`, so applying the
role does not automatically upgrade an already installed package.

Nginx and `libnginx-mod-stream` on the VPS both report
`1.28.3-2ubuntu1.12`, avoiding a module/package version mismatch. Node Exporter
is 1.10.2 on the VPS and 1.9.0 on the Debian hosts; these are distribution
packages and need not have identical versions across distributions. UFW and
WireGuard tools likewise remain distribution-managed. No kernel, Proxmox,
VPN, or distribution package version changes are introduced by this audit.

## MongoDB on Ubuntu's 7.0 kernel

The application host's `/proc/version_signature` reports upstream 7.0.14,
although `uname -r` reports 7.0.0-38-generic. MongoDB's startup guard incorrectly
rejects that release string. Its Ubuntu detection fix is assigned to 8.3.14,
which was not published in the official image list at audit time.

The application host alone sets `komodo_mongo_glibc_tunables` to
`glibc.pthread.rseq=1`. This workaround avoids the TCMalloc per-CPU cache path
that triggers the guard and can affect allocator performance. After the user
deployed this workaround, read-only checks found normal MongoDB checkpoints
and HTTP 200 from Komodo on localhost:9120. The deployment was still using
floating `:2` Core and Periphery references; exact-tag deployment remains to
be verified after applying the repository changes.

When 8.3.14 or a newer compatible 8.3 patch is published, verify its registry
manifest and release notes, update the shared MongoDB pin, remove the host
workaround, and verify database startup and Komodo HTTP readiness. Do not
downgrade the existing database volume to MongoDB 7 or 8.2 to bypass the guard.

## Sources and limits

- [Komodo 2.3.4 release](https://github.com/moghtech/komodo/releases/tag/v2.3.4)
- [Upstream Komodo MongoDB Compose deployment](https://github.com/moghtech/komodo/blob/main/compose/mongo.compose.yaml)
- [Official MongoDB image tags](https://github.com/docker-library/official-images/blob/master/library/mongo)
- [MongoDB 8.3 release notes](https://www.mongodb.com/docs/manual/release-notes/8.3/)
- [Ubuntu kernel detection fix, SERVER-131779](https://jira.mongodb.org/browse/SERVER-131779)
- [MongoDB 8.3.13 startup check](https://github.com/mongodb/mongo/blob/r8.3.13/src/mongo/db/startup_check_rseq.cpp)
- [rseq workaround reports in the image repository](https://github.com/docker-library/mongo/discussions/748)
- [Docker Ubuntu installation support](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Debian installation support](https://docs.docker.com/engine/install/debian/)
- [Ansible Docker Compose module requirements](https://docs.ansible.com/projects/ansible/latest/collections/community/docker/docker_compose_v2_module.html)

Collection versions and minimum Ansible versions were read from the installed
collection manifests and `meta/runtime.yml`. The audit covers dependencies
declared by this repository. Application stacks managed separately through
Komodo, including Immich and its database/vector extension images, are not
declared here and were not audited for compatibility. The encrypted vault
was not decrypted. Registry availability and documented compatibility do not
replace a deployment check or end-to-end application and notification tests.

Validation passed: playbook syntax using a local-only inventory; parsing all
34 non-vault YAML files; isolated Ansible renders with dummy secrets for the
application host, agent-only host, and role defaults; assertions for matching
Komodo image versions, service sets, and workaround scope; Docker Compose
5.6.1 schema validation of the rendered application configuration; and
`git diff --check`. Full inventory validation requires the vault password.
