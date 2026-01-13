# Setup certbot for DNS over HTTPS (DoH)

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

This guide covers setting up certbot to generate certificates needed for DNS over HTTPS (DoH) on the GEARS-GW Ubuntu gateway.

## Prerequisites

Before proceeding, ensure you have completed:
- [Basic Setup](GEARS-GW-Basic-Setup.md)
- [IPv6 Setup](GEARS-GW-IPv6-Setup.md)

## Important Note

This step requires public facing DNS or HTTP[S] site for Let's Encrypt to challenge or it will not hand out a certificate.

## Status

**FUTURE** - This section is currently under development and instructions will be added in a future update.

## References

- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Certbot Documentation](https://certbot.eff.org/)
- [BIND9 DNS over HTTPS](https://www.isc.org/blogs/doh-talkdns/)
