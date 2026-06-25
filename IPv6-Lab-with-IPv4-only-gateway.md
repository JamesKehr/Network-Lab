Setup the gateway following the steps in:

- [00 Gateway VM Setup](00-Gateway-VM-Setup.md) (tcgui steps can be ignored)

- [01 Gateway Basic Setup](01-Gateway-Basic-Setup.md)

- [02 Add DHCPv4](02-Add-DHCPv4.md)

- [03 Gateway IPv6 Setup](03-Gateway-IPv6-Setup.md) with `OPTION 1: Configure for NAT64 (Recommended)` enabled.

Make the following changes to the netplan file:
- eth0 (WAN/external) only has an IPv4 address. This can be static or dynamic.
- `accept-ra` must be `false` when SLAAC (IPv6 router advertisements) is enabled on the local network.
- eth1 (LAN/internal) must have both an IPv4 and IPv6 address.
- Point the name servers to the internal addresses.

Example netplan file:

```yaml
network:
  renderer: networkd
  ethernets:
    eth0:
#     Turns on/off Router Advertisements, which disables IPv6 SLAAC
      accept-ra: false
#     IPv4 Only
      addresses: [192.168.0.26/23]
    eth1:
#     Dual-stack - ULA
        addresses: [10.1.0.1/24, fd85:8b47:e1fe:1b45::1/64]
        nameservers:
#     Dual-stack - ULA
          addresses: [10.1.0.1, fd85:8b47:e1fe:1b45::1]
  version: 2
```

Apply the update: `netplan try`


Reboot the internal VM to reset the network stack.

IPv6 only connectivity will fail because the gateway has no way of working without a 4to6 tunnel. Use Loops of Zen, which is an IPv6 only site, to confirm: https://loopsofzen.uk/

IPv4 only and dual-stack will work.
