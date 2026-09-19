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

## Recurrence on 19 September 2026

The same failure recurred on 19 September 2026. The first recorded
`e1000e` hardware hang was at 09:46:53 BST, matching the reported loss of
network access around 09:45. The error then repeated approximately every two
seconds until the host was shut down.

The host remained sufficiently functional to shut down the guest cleanly. The
relevant sequence was:

```text
09:46:53  First recorded `eno1: Detected Hardware Unit Hang`
11:09:52  `pvestatd` began timing out while sending metrics to 10.44.0.2
11:10:22  `systemd-logind`: `Power key pressed short`; host began powering off
11:10:25  Application VM: `hypervisor initiated shutdown`
11:11:40  Proxmox boot completed
11:11:50  `eno1`: NIC link up at 1000 Mbps full duplex
11:12:11  Application VM boot completed
```

There was no evidence of a kernel panic, OOM event, or storage failure. The
shutdown was clean and was initiated by a power-button/ACPI event, rather than
by an automatic kernel recovery. The power-button event may have been the
manual intervention that ended the network outage.

The broader offload workaround has now been applied to `eno1`:

```bash
sudo ethtool -K eno1 gso off gro off tso off tx off rx off
```

The workaround was applied after the recurrence; its persistence across a
future reboot should be verified. Continue monitoring for another
`Detected Hardware Unit Hang` before treating the issue as resolved.

## Kernel status

The host was updated and rebooted. It is now running:

```text
7.0.14-16-pve
```

The hang was originally observed on `7.0.14-8-pve`. Installed kernels also
include `6.17.13-21-pve`. The kernel update is useful, but does not guarantee
that the I219-LM/e1000e failure will not recur.

## Recommended workaround

The broader offload workaround has now been applied. Changing the active
physical interface can interrupt SSH and VM traffic, so apply or reapply it
from a local console or maintenance session:

```bash
sudo ethtool -K eno1 gso off gro off tso off tx off rx off
```

For a lower-impact initial test on another system, TSO/GSO alone may be
tested first:

```bash
sudo ethtool -K eno1 tso off gso off
```

Check the resulting state with:

```bash
sudo ethtool -k eno1
```

If the host remains stable, make the selected workaround persistent in the
`iface eno1` stanza in `/etc/network/interfaces`:

```text
post-up /sbin/ethtool -K eno1 gso off gro off tso off tx off rx off
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
