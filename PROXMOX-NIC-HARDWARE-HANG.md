# Proxmox `e1000e` NIC hardware hang

## Incident summary

On 12 September 2026, the Proxmox host became unreachable externally after the
onboard Ethernet controller stopped transmitting. The application VM
continued collecting metrics over its local Docker network, which indicates
that the VM and host compute/storage paths were still operating. External VM
access and Proxmox metrics stopped because they depend on the host's physical
network path.

The failure was initially suspected to be a host outage, but the logs show a
network-controller failure rather than CPU, memory, or storage exhaustion.

## Confirmed hardware and topology

```text
Host:       host1
Interface:  eno1
Bridge:     vmbr0
Hardware:   Intel Ethernet Connection (7) I219-LM
PCI:        0000:00:1f.6
Driver:     e1000e
Firmware:   0.5-4
MAC:        a4:bb:6d:b5:2d:b2
```

`eno1` is the physical interface behind `vmbr0`, carrying host management and
external VM traffic.

## Evidence

The previous boot logged repeated transmit-ring hangs:

```text
e1000e 0000:00:1f.6 eno1: Detected Hardware Unit Hang
```

The error repeated approximately every two seconds. During a later occurrence,
`pvestatd` remained alive but could not send metrics to InfluxDB:

```text
pvestatd: metrics send error 'InfluxDB': 500 Can't connect to 10.44.0.2:8086
(Connection timed out)
```

This is consistent with the host remaining powered on and running while its
physical network interface was wedged. The application VM's internal Docker
traffic was unaffected, while traffic crossing `eno1` was unavailable.

## Kernel status

The host was updated and rebooted. It is now running:

```text
7.0.14-16-pve
```

The hang was originally observed on `7.0.14-8-pve`. Installed kernels also
include `6.17.13-21-pve`. The kernel update is useful, but does not guarantee
that the I219-LM/e1000e failure will not recur.

## Recommended workaround

First test the minimal offload change from a local console or maintenance
session, since changing the active physical interface can interrupt SSH and VM
traffic:

```bash
sudo ethtool -K eno1 tso off gso off
```

If the hang continues, test the broader set discussed in the Proxmox forum:

```bash
sudo ethtool -K eno1 gso off gro off tso off tx off rx off
```

Check the resulting state with:

```bash
sudo ethtool -k eno1
```

If the host remains stable, make the selected workaround persistent in the
`iface eno1` stanza in `/etc/network/interfaces`:

```text
post-up /sbin/ethtool -K eno1 tso off gso off
```

Apply this to `eno1`, not `vmbr0`. Disabling offloads may increase CPU use and
reduce peak network throughput. Keep the broader workaround as a fallback if
TSO/GSO alone is insufficient.

## Longer-term mitigation

If the error recurs after the kernel update and offload changes:

1. Move Proxmox management and VM uplinks to a separate physical NIC.
2. Prefer a PCIe NIC over the onboard I219-LM controller.
3. Test a different cable and switch port, and inspect switch logs.
4. Update system BIOS/UEFI and relevant firmware.
5. Add independent monitoring for the Proxmox API and application VM.
6. Use IPMI/BMC, a serial console, or a managed PDU if available.

## References

- [First2Host: How To Fix Proxmox Detected Hardware Unit Hang On Intel NICs](https://first2host.co.uk/blog/how-to-fix-proxmox-detected-hardware-unit-hang/)
- [Proxmox forum: Intel NIC e1000e hardware unit hang](https://forum.proxmox.com/threads/intel-nic-e1000e-hardware-unit-hang.106001/)
