# Setup jool for NAT64

> [!WARNING]
> These are NOT production-grade instructions. These steps are meant only to be used in a lab-like environment!

This guide covers setting up jool for NAT64 translation on the GEARS-GW Ubuntu gateway.

## Prerequisites

Before proceeding, ensure you have completed:
- [Basic Setup](GEARS-GW-Basic-Setup.md)
- [IPv6 Setup](GEARS-GW-IPv6-Setup.md) with DNS64 configured using Option 1 (the `64:ff9b::/96` well-known IPv6 prefix)

Based on: https://cooperlees.com/2020/12/nat64-using-jool-on-ubuntu-20-04/

Please make sure DNS64 is configured with Option 1, using the `64:ff9b::/96` well-known IPv6 prefix before proceeding.

## Installation

Install jool.

```sh
apt install jool-dkms jool-tools
```

You will be prompted to create a boot password if Secure Boot is enabled. Enter a password or jool will not install.

## Secure Boot ONLY

If Secure Boot is enabled, follow these steps:

Reboot the gateway.

A boot time menu will appear.

Select Enroll MOK.

Follow the prompts until you are prompted for the password.

Enter the password from the jool install.

Reboot.

## Continue with jool setup...

Run:

```bash
sudo modprobe jool
jool instance add --netfilter --pool6 64:ff9b::/96
```

Create a oneshot systemd server by editing/creating a service file with this command.

```bash
nano /etc/systemd/system/jool-oneshot.service
```

Add these contents:

```conf
[Unit]
Description=Add NAT64 netfilter pool6 to jool

[Service]
Type=oneshot
ExecStart=/usr/bin/jool instance add --netfilter --pool6 64:ff9b::/96

[Install]
WantedBy=multi-user.target
```

Close and save: Ctrl+x, y, Enter

Enable the jool-oneshot service.

```bash
systemctl enable jool-oneshot
```

Add jool to the module load list.

```bash
nano /etc/modules-load.d/jool.conf
```

Add the word jool to the file content.

```
jool
```

Close and save: Ctrl+x, y, Enter

Reboot to ensure that jool loads correctly on boot.

Verify jool is loaded:

```bash
lsmod | grep jool
```

## Next Steps

Once NAT64 is configured, you can proceed with:
- [IPv6 NCSI Responder](GEARS-GW-NCSI-Responder.md) for internal network connectivity detection
- [IPv6-Only Configuration](GEARS-GW-IPv6-Only.md) to switch to IPv6-only mode
