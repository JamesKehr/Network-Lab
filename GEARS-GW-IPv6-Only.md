# IPv6-Only Configuration

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

This guide covers switching the GEARS-GW Ubuntu gateway between IPv6-only and dual-stack modes.

## Prerequisites

Before proceeding, ensure you have completed:
- [Basic Setup](GEARS-GW-Basic-Setup.md)
- [IPv6 Setup](GEARS-GW-IPv6-Setup.md)
- [NAT64 Setup](GEARS-GW-NAT64.md)
- [IPv6 NCSI Responder](GEARS-GW-NCSI-Responder.md)

## Enable IPv6-only

### Disable DHCPv4

Disable the KEA DHCPv4 service.

```bash
systemctl disable --now kea-dhcp4-server.service
systemctl mask kea-dhcp4-server.service
```

### Remove IPv4 addresses

Remove the IPv4 address on the internal interface (example: eth1).

Open the netplan file.

```bash
nano /etc/netplan/<YAML file>
```

Example:

```bash
nano /etc/netplan/50-cloud-init.yaml
```

Remove the IPv4 addresses from the internal interface in netplan file. 
TIP  : Comment the line and create a modified copy of the line without the IPv4 addresses.
TIP2 : Create a second copy of the dual-stack lines to and remove the IPv6 addresses to create an IPv4-only version of the addresses lines.

Example:

```bash
network:
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: true
    eth1:
#     Dual-stack
#      addresses: [10.1.0.1/24, fd85:8b47:e1fe:1b45::1/64]
#     IP4-only stack
#      addresses: [10.1.0.1/24]
#     IPv6-only stack
      addresses: [fd85:8b47:e1fe:1b45::1/64]
      nameservers:
#     Dual-stack
#        addresses: [192.168.1.60, 192.168.3.61, 2600:1700:5aa0:30cf::60, 2600:1700:5aa0:30ce::61]
#     IP4-only stack
#        addresses: [192.168.1.60, 192.168.3.61]
#     IPv6-only stack
        addresses: [2600:1700:5aa0:30cf::60, 2600:1700:5aa0:30ce::61]
  version: 2
```

### Reboot clients

Reboot the lab clients to reset the network configuration.

## Re-enable IPv4

### Enable DHCPv4

Enable the KEA DHCPv4 service.

```bash
systemctl unmask kea-dhcp4-server.service
systemctl enable --now kea-dhcp4-server.service
```

### Restore IPv4 addresses

Open the netplan file

```bash
nano /etc/netplan/<YAML file>
```

Example:

```bash
nano /etc/netplan/50-cloud-init.yaml
```

Open the netplan file and add the IPv4 address back to the Internal interface (example: eth1). Or, comment/uncomment the approritate lines to enable dual-stack again.

### Reboot

Reboot the lab clients to reset the network configuration.

Reboot the gateway if the DHCP server is not handing out IPv4 addresses.

## Verification

### IPv6-only mode verification

In IPv6-only mode, verify:
- Lab clients receive only IPv6 addresses via DHCPv6
- DNS64 resolves IPv4-only domains to IPv6 addresses
- NAT64 translates IPv6 traffic to IPv4 for external connectivity
- NCSI probe indicates proper connectivity

### Dual-stack mode verification

In dual-stack mode, verify:
- Lab clients receive both IPv4 and IPv6 addresses
- Both protocol stacks are functional
- Connectivity works over both IPv4 and IPv6
