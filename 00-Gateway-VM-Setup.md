# Virtaul Machine Setup

These instructions are designed for running Ubuntu as a Hyper-V VM.

Requirements:

- Windows Hyper-V host
  -  Windows 11 and Windows Server 2025 have been tested.
  -  Newer version and Server 2019+ should work.
  -  A computer that is Hyper-V capable (which is all modern computers).
  -  At least 16GB RAM, 32GB+ is recommended.
  -  At least 100GB of free disk space for virtual machines (VMs).
  -  An IPv6 Global Unicast Address (GUA) from your ISP.
- Ubuntu Server ISO

https://ubuntu.com/download/server

Create a generation 2 VM using the ISO.

Open VM settings.

- Security
- Change the Secure Boot template to "Microsoft UEFI Certificate Authority"
- Enable all Integration Services
- [Windows 11] Checkpoints: uncheck "Use automatic checkpoints"
- Adjust resources as needed
  - Dynamic memory can be used
  - A minimum of 2 vCPUs are recommended

Start the VM and go through the Ubuntu server setup. 

- Username: gw
- Password: P@ssw0rd
- Install the OpenSSH server when prompted

SSH to the server.

	ssh gw@<IP>
	
- Yes when asked about the fingerprint.
- Enter the gw password.
- You can get the IP address from Hyper-V Manager, or by logging into the Ubuntu VM. The IP will appear in the logon text.


Elevate to super user, enter password when prompted. The instructions assume you are always su.

	sudo su

Run these commands to update the server. Remember that everything in Linux/Unix is case sensitive.

	apt update && apt upgrade -y
		
Install required components for the lab. You can copy/paste the commands when using the SSH session in PowerShell/Windows Terminal.

	apt install iproute2 python3-flask bridge-utils net-tools iptables-persistent git -y

[Optional] Setup an SSH key to skip entering the password in the future.

https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement

[Optional] Install mDNS. 
- This will allow you to ssh using <hostname>.local rather than hunting for the IP address of the gateway.
- mDNS will not work if the gateway is behind a Default Switch, it will only work if the gateway is attached to an external vmSwitch.

```bash
sudo apt install avahi-daemon
sudo systemctl start avahi-daemon
sudo systemctl enable avahi-daemon
sudo reboot
```

[Optional] Example ssh command with mDNS where the username is `gw` and the hostname is `gateway`.

```
ssh gw@gateway.local
```
