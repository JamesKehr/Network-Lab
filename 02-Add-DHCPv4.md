# DHCPv4 Setup for GEARS-GW

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

These steps walk through the basics of setting up DHCPv4 on the GEARS-GW Ubuntu gateway.

## Prerequisites

Before proceeding, ensure you have completed the [Basic Setup](GEARS-GW-Basic-Setup.md).

## Installation

[Install ISC KEA](https://documentation.ubuntu.com/server/how-to/networking/install-isc-kea/index.html) for DHCPv4 and DHCPv6. The Ubuntu install adds everything you need.

```sh
sudo apt install kea
```

Use the "configure with a random password" option during setup.

## Configuration

Rename the kea-dhcp4.conf file.

```sh
mv /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.original
```

Edit the kea-dhcp4.conf contents below to match your environment. 
- This file is only for a single interface, but can be expanded to handle both interfaces.
- Change `"interfaces": [ "eth1" ]` to `"interfaces": [ "eth1", "eth2" ]` to add both interfaces.
- Add `"interface": "ethX",` entries to the subnet matching eth1 and eth2, after the "id" line. Changing the X in ethX to the appropriate interface number or name.

```javascript
{
  "Dhcp4": {
        "interfaces-config": {
        "interfaces": [ "eth1" ]
        },
        "control-socket": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea4-ctrl-socket"
        },
        "lease-database": {
        "type": "memfile",
        "lfc-interval": 3600
        },
        "valid-lifetime": 600,
        "max-valid-lifetime": 7200,
        "subnet4": [
        {
          "id": 1,
          "subnet": "10.1.0.0/24",
          "pools": [
          {
                "pool": "10.1.0.150 - 10.1.0.200"
          }
        ],
        "option-data": [
        {
                "name": "routers",
                "data": "10.1.0.1"
        },
        {
                "name": "domain-name-servers",
                "data": "192.168.1.1, 1.1.1.1"
        },
        {
                "name": "domain-name",
                "data": "contoso.com"
        }
        ]
        }
        ]
  }
}
```

Create/Edit kea-dhcp4.conf.

```sh
nano /etc/kea/kea-dhcp4.conf
```

Paste the modified conf data from above into the file, then Ctrl+X -> Y -> Enter to save the file.

## Reload Configuration

Run this command to reload the configuration.
- The console will pause with no output.
- Press Ctrl+D.
- The output should contain "Configuration successful."

```sh
kea-shell --host 127.0.0.1 --port 8000 --auth-user kea-api --auth-password $(cat /etc/kea/kea-api-password) --service dhcp4 config-reload
```

## Next Steps

Once DHCPv4 is configured, you can proceed with:
- [IPv6 Setup](GEARS-GW-IPv6-Setup.md) for IPv6-mostly configuration
- [NAT64 Setup](GEARS-GW-NAT64.md) for IPv6 to IPv4 translation
