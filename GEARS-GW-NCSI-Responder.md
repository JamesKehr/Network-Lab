# Create an internal IPv6 NCSI responder

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

This guide covers setting up an internal IPv6 NCSI (Network Connectivity Status Indicator) responder on the GEARS-GW Ubuntu gateway.

## Prerequisites

Before proceeding, ensure you have completed:
- [Basic Setup](GEARS-GW-Basic-Setup.md)
- [IPv6 Setup](GEARS-GW-IPv6-Setup.md)

Based on: https://ubuntu.com/tutorials/install-and-configure-nginx#1-overview

## Install nginx

Install nginx.

```bash
apt install nginx
```

Create the connecttest.txt file in /var/www/msftconnecttest

```bash
mkdir /var/www/msftconnecttest
echo "Microsoft Connect Test" > /var/www/msftconnecttest/connecttest.txt
```

## Setup the nginx virtual host

```bash
cd /etc/nginx/sites-enabled
nano msftconnecttest
```

Add the following contents to the file.
- Replace the servername with the appropriate URLs that will serve the connection test file.

```sh
server {
       listen 80;
       listen [::]:80;

       server_name ipv4.contoso.com;

       root /var/www/msftconnecttest;
       index connecttest.txt;

       location / {
               try_files $uri $uri/ =404;
       }
}

server {
       listen 80;
       listen [::]:80;

       server_name ipv6.contoso.com;

       root /var/www/msftconnecttest;
       index connecttest.txt;

       location / {
               try_files $uri $uri/ =404;
       }
}
```

Close and save: Ctrl+x, y, Enter

Restart nginx.

```bash
service nginx restart
```

## Add DNS records

Add A and AAAA records to the DNS server for the domains used above.
- These steps will cover adding ipv4.contoso.com and ipv6.contoso.com to BIND9 on the gateway.
- There are BIND9 front ends you can install and service with nginx if you want to try one of those instead.
- Or if you have a Windows DNS server for an AD environment you can use that.
- It doesn't matter where the DNS server is so long as the internal network clients can resolve the records from the local IPv6 ULA network.

Edit named.conf.local.

```bash
nano /etc/bind/named.conf.local
```

This is an basic example of contoso.com in named.conf.local.

```sh
zone "contoso.com" {
        type master;
        file "/var/lib/bind/db.contoso.com";
};
```

Close and save: Ctrl+x, y, Enter

Create the zone file, which is the file from the zone definition in named.conf.local.
- Replace the file if different.

```bash
nano /var/lib/bind/db.contoso.com
```

Create the zone file based on the output below, the RFC standards, and the BIND9 documentation.

https://bind9.readthedocs.io/en/v9.18.14/chapter3.html#soa-rr
https://wiki.debian.org/Bind9#File_.2Fetc.2Fbind.2Fnamed.conf.local

Example file (replace IP addresses and records as apprpriate):

```
; base zone file for example.com
$TTL 2d    ; default TTL for zone
$ORIGIN contoso.com. ; base domain-name
; Start of Authority RR defining the key characteristics of the zone (domain)
@         IN      SOA   ns1.contoso.com. hostmaster.contoso.com. (
                                2003080800 ; serial number
                                12h        ; refresh
                                15m        ; update retry
                                3w         ; expiry
                                2h         ; minimum
                                )
; name server RR for the domain
           IN      NS      ns1.contoso.com.
; domain hosts includes NS and MX records defined above
; plus any others required
; for instance a user query for the A RR of joe.example.com will
; return the IPv4 address 192.168.254.6 from this zone file
ns1        IN      A       10.1.0.1
ns1        IN      AAAA    fd29:d3fc:a205:9511::1
ipv4       IN      A       10.1.0.1
ipv6       IN      AAAA    fd29:d3fc:a205:9511::1
```

Reload the named to reload BIND9 configuration.

```bash
systemctl restart named
systemctl status named
```

## Verify DNS and Web Server

Make sure the internal URLs are being served by DNS.

From the gateway:

```bash
dig ipv4.contoso.com A
dig ipv6.contoso.com AAAA
```

From Windows client:

```powershell
Resolve-DnsName ipv4.contoso.com -Type A
Resolve-DnsName ipv6.contoso.com -Type AAAA
```

Use curl to confirm the file is being served.
- The output of each command must be: `Microsoft Connect Test`


From gateway:

```bash
curl http://ipv4.contoso.com/connecttest.txt
curl http://ipv6.contoso.com/connecttest.txt
```

From internal Windows client:

```powershell
curl.exe http://ipv4.contoso.com/connecttest.txt
curl.exe http://ipv6.contoso.com/connecttest.txt
```

## Configure Windows Client

Setup and test the internal probe on a test system.
- Open regedit.
- Go to: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\NlaSvc\Parameters\Internet`
- Change `ActiveWebProbeHost` to the internal IPv4 probe URL (`ipv4.contoso.com` in this example).
- Change `ActiveWebProbeHostV6` to the internal IPv6 probe URL (`ipv6.contoso.com` in this example).
- Close regedit.
- Restart the computer.

## Next Steps

Once the NCSI responder is configured, you can proceed with:
- [IPv6-Only Configuration](GEARS-GW-IPv6-Only.md) to switch to IPv6-only mode
