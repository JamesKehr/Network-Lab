# GEARS-GW Setup Documentation

This documentation provides step-by-step instructions for setting up an Ubuntu gateway (GEARS-GW) for Windows network administrators in a lab environment.

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

## Overview

The GEARS-GW is an Ubuntu-based network gateway designed for learning and testing network configurations. It supports:
- Multi-homed networking with NAT
- IPv4 and IPv6 dual-stack or IPv6-only configurations
- DHCP services for both IPv4 and IPv6
- DNS with DNS64 for IPv6-mostly networks
- NAT64 for IPv6-to-IPv4 translation
- Network emulation capabilities
- Hyper-V optimization

## Setup Guides

Follow the guides below in order to build your GEARS-GW gateway:

### 1. [Basic Setup](GEARS-GW-Basic-Setup.md)
**Start here!** This is the foundation for all other configurations.

Covers:
- Ubuntu Server installation and initial configuration
- Multi-homed network setup (eth0, eth1, eth2)
- NAT4 (MASQUERADE) configuration
- Hyper-V network tuning
- tcgui for WAN emulation
- Basic connectivity testing

### 2. [DHCPv4 Setup](GEARS-GW-DHCPv4.md) *(Optional)*
Instructions for configuring ISC KEA DHCP server for IPv4 address assignment on your lab network.

### 3. [IPv6 Setup](GEARS-GW-IPv6-Setup.md) *(Optional)*
Comprehensive guide for IPv6-mostly networking, including:
- KEA DHCPv6 for ULA address assignment
- radvd for Router Advertisements
- BIND9 DNS server with DNS64

### 4. [NAT64 Setup](GEARS-GW-NAT64.md) *(Optional)*
Instructions for setting up jool to enable NAT64 translation, allowing IPv6-only clients to access IPv4 resources.

**Prerequisite:** Requires IPv6 Setup with DNS64 configured.

### 5. [IPv6 NCSI Responder](GEARS-GW-NCSI-Responder.md) *(Optional)*
Set up an internal Network Connectivity Status Indicator (NCSI) responder using nginx for proper network detection on Windows clients.

### 6. [IPv6-Only Configuration](GEARS-GW-IPv6-Only.md) *(Optional)*
Instructions for switching between IPv6-only and dual-stack modes, including:
- Disabling/enabling DHCPv4
- Removing/restoring IPv4 addresses
- Verification steps

**Prerequisites:** Requires IPv6 Setup, NAT64 Setup, and NCSI Responder.

### 7. [Certbot Setup](GEARS-GW-Certbot.md) *(Future)*
Instructions for setting up certbot to generate certificates for DNS over HTTPS (DoH).

**Status:** Under development.

## Quick Start

For a basic dual-stack gateway with IPv4 and IPv6:

1. Complete [Basic Setup](GEARS-GW-Basic-Setup.md)
2. Complete [DHCPv4 Setup](GEARS-GW-DHCPv4.md)
3. Complete [IPv6 Setup](GEARS-GW-IPv6-Setup.md)

For an IPv6-only gateway:

1. Complete [Basic Setup](GEARS-GW-Basic-Setup.md)
2. Complete [DHCPv4 Setup](GEARS-GW-DHCPv4.md) *(needed initially)*
3. Complete [IPv6 Setup](GEARS-GW-IPv6-Setup.md)
4. Complete [NAT64 Setup](GEARS-GW-NAT64.md)
5. Complete [IPv6 NCSI Responder](GEARS-GW-NCSI-Responder.md)
6. Complete [IPv6-Only Configuration](GEARS-GW-IPv6-Only.md)

## Network Topology

The typical GEARS-GW setup uses three network interfaces:

- **eth0**: WAN/External network (DHCP)
- **eth1**: RED network (10.1.0.1/24 or IPv6 ULA)
- **eth2**: BLUE network (10.2.0.1/24 or IPv6 ULA)

Lab clients connect to eth1 (RED) or eth2 (BLUE) networks and route through the gateway for internet access.

## Support and References

- [Ubuntu Server Documentation](https://documentation.ubuntu.com/server/)
- [ISC KEA Documentation](https://documentation.ubuntu.com/server/how-to/networking/install-isc-kea/)
- [BIND9 Documentation](https://bind9.readthedocs.io/)
- [Jool NAT64 Documentation](https://nicmx.github.io/Jool/)

## Related Tools

- [Initialize-NetworkLab PowerShell Script](https://github.com/JamesKehr/Initialize-NetworkLab) - Automated Windows Server network lab configuration
- [Get-AzPrivateIPv6Subnet](https://github.com/JamesKehr/Azure) - ULA IPv6 address space generator

## Source

This documentation was originally part of the [Initialize-NetworkLab repository](https://github.com/JamesKehr/Initialize-NetworkLab) and has been split into separate topic-based files for easier navigation and maintenance.
