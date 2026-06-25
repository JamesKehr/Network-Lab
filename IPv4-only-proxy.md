These instructions outline setting up tinyproxy for IPv4 only. Proxy is enforced by blocking all HTTP and HTTPS traffic not going through the proxy.

Gateway setup assumptions and recommended state:
- The gatway was setup using [IPv6-Lab-with-IPv4-only-gateway.md](IPv6-Lab-with-IPv4-only-gateway.md)
- The lab is dual stack
- The gateway is single stack, using IPv4


# Proxy

Install tinyproxy.

```
sudo apt update
sudo apt install tinyproxy -y
```

Configure tinyproxy.

```
 nano /etc/tinyproxy/tinyproxy.conf
```

- Change the `Listen` address to the internal (LAN/eth1) IPv4 address.
- Change the `Bind` address to the external (WAN/eth0) Ipv4 address.
- Make sure the `Allow` list contains the internal subnet, or that the Allow line for the internal subnet is uncommented. This should be setup by default, but it's good to check.

Save and close `tinyproxy.conf`.

Restart tinyproxy.

```
sudo systemctl restart tinyproxy
sudo systemctl enable tinyproxy
```

Reset iptables and then block HTTP and HTTPS from internal to external unless the proxy is used.

```
#### RESET/CLEAR ip[6]tables ####

# Set default policies to ACCEPT (avoid locking yourself out)
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Flush all rules
iptables -F
iptables -t nat -F
iptables -t mangle -F
iptables -t raw -F

# Delete all non-default chains
iptables -X
iptables -t nat -X
iptables -t mangle -X
iptables -t raw -X

# Zero counters (optional but common)
iptables -Z
iptables -t nat -Z
iptables -t mangle -Z
iptables -t raw -Z
iptables-save > /etc/iptables/rules.v4
cat /etc/iptables/rules.v4

ip6tables -P INPUT ACCEPT
ip6tables -P FORWARD ACCEPT
ip6tables -P OUTPUT ACCEPT

ip6tables -F
ip6tables -t nat -F
ip6tables -t mangle -F
ip6tables -t raw -F

ip6tables -X
ip6tables -t nat -X
ip6tables -t mangle -X
ip6tables -t raw -X

ip6tables -Z
ip6tables -t nat -Z
ip6tables -t mangle -Z
ip6tables -t raw -Z

ip6tables-save > /etc/iptables/rules.v6
cat /etc/iptables/rules.v6

# Allow return traffic for established/related IPv4 flows
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Drop only NEW forwarded HTTP/HTTPS over IPv6 from internal NIC to external NIC (no v6 subnet match)
sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80,443 \
  -m conntrack --ctstate NEW -j DROP

# Allow other NEW forwarded IPv6 traffic from internal NIC to external NIC
sudo iptables -A FORWARD -i eth1 -o eth0 -m conntrack --ctstate NEW -j ACCEPT

# save the rules
iptables-save > /etc/iptables/rules.v4


# Allow return traffic for established/related IPv6 flows
sudo ip6tables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Drop only NEW forwarded HTTP/HTTPS over IPv6 from internal NIC to external NIC (no v6 subnet match)
sudo ip6tables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80,443 \
  -m conntrack --ctstate NEW -j DROP

# Allow other NEW forwarded IPv6 traffic from internal NIC to external NIC
sudo ip6tables -A FORWARD -i eth1 -o eth0 -m conntrack --ctstate NEW -j ACCEPT

# save the rules
ip6tables-save > /etc/iptables/rules.v6
```

Setup the proxy on the lab VMs to point to http://10.1.0.1:8888

At this point, browsers should be able to reach websites, but going direct using a tool like curl.exe will fail.

Works:
`curl -x http://10.1.0.1:8888 http://example.com`

Fails:
`curl http://example.com`

NOTE: Use `curl.exe` for Windows PowerShell 5.1.

# Proxy File

Install nginx.

```
sudo apt update
sudo apt install nginx -y
```

Create the proxy pack file.

```
mkdir -f /var/www/proxy
touch /var/www/proxy/proxy.pac
echo "function FindProxyForURL(url, host) { return "PROXY proxy.contoso.com:8888"; }" > /var/www/proxy/proxy.pac
```

Create proxy.contoso.com in nginx.

```
touch /etc/nginx/sites-enabled/proxy
```

Edit the file.

```
nano /etc/nginx/sites-enabled/proxy
```

Add this content to the proxy site file.

```
server {
       listen 80;
       listen [::]:80;

       server_name proxy.contoso.com;

       root /var/www/proxy;
       index proxy.pac;

       location / {
               try_files $uri $uri/ =404;
       }
}
```

Restart nginx.

```
sudo systemctl restart nginx
```

Add proxy.contoso.com in DNS. These steps are for when contoso.com is hosted by BIND9 on the gateway. Update accordingly if DNS is handled elsewhere in your lab.

Create db file, if not already made.

```
touch /var/lib/bind/db.contoso.com
```

Edit the BIND9 config file.

```
nano /etc/bind/named.conf.local
```

Add the db file to BIND9 by adding these lines at the end of the file.

```
zone "contoso.com" {
        type master;
        file "/var/lib/bind/db.contoso.com";
};
```

Save and close the file.

Edit the contoso db file

```
nano /var/lib/bind/db.contoso.com
```

Add this content to the db file:
- Edit the IP addresses accordingly.

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
proxy      IN      A       10.1.0.1
```

Restart named (BIND9).

```
systemctl restart named
systemctl status named
```

Go to a lab system and ensure the proxy.pac file is being served.

```
curl http://proxy.contoso.com/proxy.pac
```

Using the proxy.pac file in Windows:
- Setting > Network & internet > Proxy
- Toggle `Off` **Automatically detect settings**
- Select `Edit` under **Use setup script**
- Toggle `On` **Use setup script**
- Enter `http://proxy.contoso.com/proxy.pac` as the **Script address**
- Disable the manual proxy if that was configured during setup or testing.
- **Save**

Verify that website are being served through a web browser.
