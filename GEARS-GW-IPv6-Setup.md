# IPv6-mostly Setup (DHCPv6 + DNS64 + NAT64)

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

This guide covers setting up IPv6-mostly networking on the GEARS-GW Ubuntu gateway, including:
- KEA DHCPv6 for ULA address assignment
- radvd for Router Advertisements
- BIND9 DNS server with DNS64

## Prerequisites

Before proceeding, ensure you have completed the [Basic Setup](GEARS-GW-Basic-Setup.md).

## Use KEA DHCPv6 for ULA address assignment

Use a tool to generate a ULA IPv6 prefix.
- I have [built a tool](https://github.com/JamesKehr/Azure) for creating ULA address spaces.
- Run this command in PowerShell to generate an IPv6 address space. This works in PowerShell for Linux, too.
  
```powershell
$ipv6 = iwr https://raw.githubusercontent.com/JamesKehr/Azure/main/Get-AzPrivateIPv6Subnet.ps1 | iex
```

- Then use this command to get the subnet CIDR syntax that can be used in the config below.

```powershell
$ipv6.GetAzSubnet(0)
```

- Like the DHCP4 conf file, additional interfaces and subnets can be generated.
- Use the commands below to generate a second subnet for eth2. Additional subnets can be generated using the same principle.

```powershell
$ipv6.AddSubnetID()
$ipv6.GetAzSubnet(1)
```

Use the text below to build kea-dhcp6.conf file content.

kea-dhcp6.conf content:

```javascript
{
# DHCPv6 configuration starts on the next line
"Dhcp6": {

# First we set up global values
    # Address expiration.
    # default: 4000; Lab: 600 (10 minutes)
    "valid-lifetime": 600,
    # This is the T1 time.
    # Default: 1000; Lab: 300 (5 minutes)
    "renew-timer": 300,
    # This is the T2 time.
    # Default: 2000; Lab: 480 (8 minutes)
    "rebind-timer": 2000,
    # Determines when the address becomes deprecated.
    # Default: 3000; Lab: 540 (9 minutes)
    "preferred-lifetime": 540,

# Next we set up the interfaces to be used by the server.
    "interfaces-config": {
        "interfaces": [ "eth1" ],
        "service-sockets-require-all": true,
        "service-sockets-max-retries": 5,
        "service-sockets-retry-wait-time": 5000
    },

# And we specify the type of lease database
    "lease-database": {
        "type": "memfile",
        "lfc-interval": 3600
    },

# Finally, we list the subnets from which we will be leasing addresses.
    "subnet6": [
        {
            # update the interface name here
            "interface": "eth1",
            # change the subnet here
            "subnet": "fd::/64",
            "option-data": [
               {
                   "name": "dns-servers",
                   "data": "<eth1 IPv6 address>"
               }
            ],
            "pools": [
                {
                    # update the pool here
                    # if the generate subnet was fda4:6d59:9bbb:fc9d::/64, then the pool would be fda4:6d59:9bbb:fc9d:1::/80
                    "pool": "fd::1::/80"
                }
             ]
        }
    ]
# DHCPv6 configuration ends with the next line
}

}
```

Backup any example kea-dhcp6.conf file.

```sh
mv /etc/kea/kea-dhcp6.conf /etc/kea/kea-dhcp6.conf.original
```

Edit/Create the kea-dhcp6.conf file.

```sh
nano /etc/kea/kea-dhcp6.conf
```

Copy/paste the kea-dhcp6.conf content, then  Ctrl+X -> Y -> Enter to save the file.

Try this command to reload the DHCPv6 server.
- Press Ctrl+D to perform the reload.

```sh
kea-shell --host 127.0.0.1 --port 8000 --auth-user kea-api --auth-password $(cat /etc/kea/kea-api-password) --service dhcp6 config-reload
```

- The command might throw an error and I don't know why.
- If the command outputs something like "unable to forward command to the dhcp6 service: No such file or directory. The server is likely to be offline" then run this command to restart the service.

```sh
systemctl restart kea-dhcp6-server.service
```

- Make sure the service started using this command. Press Q to exit the status output.

```
systemctl status kea-dhcp6-server.service
```


Enable IPv6 forwarding.

```sh
sysctl -w net.ipv6.conf.all.forwarding=1
sysctl -w net.ipv6.conf.default.forwarding=1
sed -i '/net.ipv6.conf.all.forwarding/s/^#//' /etc/sysctl.conf
sed -i '/net.ipv6.conf.default.forwarding/s/^#//' /etc/sysctl.conf
sysctl -p
```

Enable IPv6 MASQUERADE 

```sh
ip6tables -t nat -A POSTROUTING -j MASQUERADE -o eth0
ip6tables -t nat -A POSTROUTING -j MASQUERADE -o eth1
ip6tables-save > /etc/iptables/rules.v6
```

## Use radvd to advertise the IPv6 gateway

Install radvd. This enables router advertisements, needed to send the IPv6 gateway to clients.

```sh
apt install radvd radvdump
```

Edit the radvd.conf file.

```sh
nano /etc/radvd.conf
```

Add this content to the file.
- This does not advertise an IPv6 prefix, only the M-bit (Managed flag) to use DHCPv6.
- Adding other config (DNS servers) is not in this example, but can be added by updating the DHCPv6 config.
- Add one entry per interface.
- Leave the WAN (eth0) interface off.

```sh
interface eth0 {
        AdvSendAdvert off;
        AdvOtherConfigFlag off;
        AdvManagedFlag off;
};

interface eth1 {
        AdvSendAdvert on;
        MinRtrAdvInterval 3;
        MaxRtrAdvInterval 10;
        AdvOtherConfigFlag off;
        AdvManagedFlag on;
};
```

Restart radvd to reload the changes and get the status to make sure it restarted successfully.

```sh
systemctl restart radvd
systemctl status radvd
```

Run this command to make sure the configuration is working. It may take a minute or so for results to appear.

```sh
radvdump
```

Radvd is working if router advertisements are coming from eth1 (and optionally eth2), but not eth0.

Press Ctrl+C to stop testing.

Edit the netplan yaml file where the IPv4 gateway address was added (example: `/etc/netplan/50-cloud-init.yaml`)
- Add `dhcp6: true` on eth0.
- Add a static IPv6 address from the ULA address space to eth1 and other internal adapters.
- Use a well known IPv6 address, such as <ULA>::1/64.

Example:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp6: true
    eth1:
      addresses: [10.1.0.1/24, <ULA subnet>::1/64]
      nameservers:
        addresses: [192.168.1.1, 1.1.1.1, 2606:4700:4700::1111, 2620:fe::fe]
```

Save and close: Ctrl+x, y, Enter

Run:

```sh
netplan try
```

Use `ip addr` to confirm eth1 (and/or other interfaces) have a static ULA address.

Boot up a lab client on the BLUE network.

The client should have IPv4 and IPv6 addresses from the gateway's DHCP servers. And internet connectivity should work.

```sh
ping -4 example.com
ping -6 example.com
```

## Setup BIND9 for DNS and DNS64

This configuration causes bind9 to act like a cache server with no domains of its own. Feel free to add zones if you want.

References:

https://documentation.ubuntu.com/server/how-to/networking/install-dns/#set-up-a-primary-server
https://www.isc.org/blogs/doh-talkdns/

Install bind9 and dnsutils.

```sh
apt install bind9 dnsutils
```

Enable the bind9 named service to auto-start.

```sh
systemctl enable named
```

Edit /etc/bind/named.conf.options

```sh
nano /etc/bind/named.conf.options
```

Uncomment the forwarders section and add some forwarders. This example will use Cloudflare DNS (1.1.1.1, 1.0.0.1, 2606:4700:4700::1111, 2606:4700:4700::1001).

```sh
        forwarders {
             1.1.1.1;
             1.0.0.1;
             2606:4700:4700::1111;
             2606:4700:4700::1001;
        };
```

Add the following lines under `listen-on-v6 { any; };`

```sh
        listen-on { any; };

        recursion yes;
        allow-recursion { any; };
```

The final file should look like this, at a minimum.

```sh
options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
             1.1.1.1;
             1.0.0.1;
             2606:4700:4700::1111;
             2606:4700:4700::1001;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;

        listen-on-v6 { any; };
        listen-on { any; };

        recursion yes;
        allow-recursion { any; };
};
```

Save and close: Ctrl-X, Y, Enter

Validate the configuration. No output is good output.

```sh
named-checkconf /etc/bind/named.conf.options
```

Restart the named (bind9) service and make sure there are no errors.

```sh
systemctl restart named
systemctl status named
```

Test the bind9 DNS server locally and on a VM in the lab.

On the gateway server:

```sh
dig example.com AAAA
dig example.com A
```

On the Windows client:

```powershell
$query = "example.com"
$gtwys = Get-NetIPConfiguration | ForEach-Object {@($_.IPv6DefaultGateway.NextHop, $_.IPv4DefaultGateway.NextHop)}
$gtwys | ForEach-Object {Write-Host -fore Green "Gateway: $_"; Resolve-DnsName $query -Server $_}
```

Continue if everything is working this far.

Get your ULA IPv6 prefix.
- This is the fd::/64 address space generated earlier in the instructions.
- You can use the command `ip addr` to get the IPv6 address details from the console; however, this needs to be in prefix format, not IPv6 address format.
- An address of `fd1a:7148:e9a0:186a::1/64` in the `ip addr` output would be `fd1a:7148:e9a0:186a::/64` in prefix format.

Edit /etc/bind/named.conf.options to enable dns64.

```
nano /etc/bind/named.conf.options
```

Add the following to the config file:

### OPTION 1: Configure for NAT64 (Recommended)

This prepares BIND9 DNS64 to work with jool NAT64.

```
        dns64 64:ff9b::/96 {
            clients { any; }; // Affect all clients
            mapped { any; };  // Map all IPv4 addresses
        };
```

### OPTION 2: Configure for ULA

This option configures DNS64 to use your ULA (Unique Local Address) prefix instead of the well-known NAT64 prefix. Use this if you're not planning to use NAT64, or if you have a specific ULA-based IPv6 addressing scheme.

Template:
```
dns64 <your_ULA_prefix>/<prefix_length> {};
```

Example:
```
dns64 fd1a:7148:e9a0:186a::/64 {};
```


Save and close the file: Ctrl+X, Y, Enter

Verify the file and restart the service.

```
named-checkconf /etc/bind/named.conf.options
systemctl restart named
systemctl status named
```

Now perform a DNS lookup for a website with only an IPv4 address.

```
Resolve-DnsName jammrock.com -Server 10.1.0.1
```

This should return the IPv4 address and a DNS64 translated address using the lab's ULA address space.

```
Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
jammrock.com                                   AAAA   300   Answer     64:ff9b::1763:fa4c
jammrock.com                                   A      3600  Answer     23.99.250.76
```

## Next Steps

Once IPv6 setup is complete, you can proceed with:
- [NAT64 Setup](GEARS-GW-NAT64.md) to enable NAT64 translation
- [IPv6 NCSI Responder](GEARS-GW-NCSI-Responder.md) for internal network connectivity detection
