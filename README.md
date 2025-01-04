# VPN Gateway for Your LAN

## Introduction

This is a slightly refined version of the original steps outlined in Scott’s guide [here](https://discussion.scottibyte.com/t/vpn-gateway-server-for-your-lan/49) and in his [video tutorial](https://www.youtube.com/watch?v=HRmOPXf4q5Q). A big thank you to Scott for his contributions, which laid the foundation for this guide.

Just a heads-up, this tutorial is provided 'as is.' While it works for many setups, you should make sure it fits your specific network needs. I've just included some common issues I encountered during the process to help you avoid them.

## Overview

This tutorial demonstrates how to create a VPN Gateway for your LAN, routing traffic through a VPN service like NordVPN. By changing the gateway address of your systems to the VPN Gateway server, you can route all network traffic through a public VPN server. This solution allows multiple systems on your LAN to use the VPN service, even if they do not have VPN client software installed. It is particularly useful for devices like Roku that cannot run a VPN client.

In this guide, we will use NordVPN as the VPN provider. However, any service that supports UDP connections (not TCP) and uses port 1194 should work.

## Prerequisites

We are using a Virtual Machine (VM) instance of Ubuntu Server 18.04, with the following resources:

- 2 vCPUs
- 2GB of RAM
- 25GB of storage

You’ll also need a NordVPN account and access to your NordVPN configuration files.

## Install Dependencies

First, update your system and install the required dependencies:

```bash
sudo apt update
sudo apt install openvpn openssh-server unzip
```

## Set Up VPN Configuration

### Download NordVPN Configuration Files

Download the NordVPN configuration files for OpenVPN:

```bash
cd /etc/openvpn
sudo wget https://downloads.nordcdn.com/configs/archives/servers/ovpn.zip
sudo unzip ovpn.zip
```

### Create the Connection Script

Create a script to connect to the VPN:

```bash
sudo nano /etc/openvpn/connect.sh
```

Paste the following content:

```bash
#!/bin/bash
sudo openvpn --config "/etc/openvpn/ovpn_udp/us9144.nordvpn.com.udp.ovpn" --auth-user-pass /etc/openvpn/auth.txt
```

Save and exit the editor.

### Create the Service Credentials File

Now, create the `auth.txt` file, which will contain your NordVPN service credentials (not your regular account login credentials):

```bash
sudo nano /etc/openvpn/auth.txt
```

Add your **NordVPN service username** and **password** in the following format:

```
username@email.com
password
```

This is **not** your account login credentials but the service credentials you obtain from the [manual configuration section](https://support.nordvpn.com/Setup-Help/) in your NordVPN account.

Save and exit the editor.

### Configure iptables for Routing and Security

Next, create the iptables script to set up routing, security rules, and the VPN kill switch:

```bash
sudo nano /etc/openvpn/iptables.sh
```

Paste the following content:

```bash
#!/bin/bash

# Flush existing rules
iptables -t nat -F
iptables -t mangle -F
iptables -F
iptables -X

# Block All traffic
iptables -P OUTPUT DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP

# Allow localhost traffic
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow DHCP communication
iptables -A OUTPUT -d 255.255.255.255 -j ACCEPT
iptables -A INPUT -s 255.255.255.255 -j ACCEPT

# Allow communication within your own network (adjust to your network range)
iptables -A INPUT -s 192.168.1.0/24 -d 192.168.1.0/24 -j ACCEPT
iptables -A OUTPUT -s 192.168.1.0/24 -d 192.168.1.0/24 -j ACCEPT

# Allow established sessions
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow TUN interface (VPN traffic)
iptables -A INPUT -i tun+ -j ACCEPT
iptables -A FORWARD -i tun+ -j ACCEPT
iptables -A FORWARD -o tun+ -j ACCEPT
iptables -t nat -A POSTROUTING -o tun+ -j MASQUERADE
iptables -A OUTPUT -o tun+ -j ACCEPT

# Allow VPN connection (UDP port 1194)
iptables -I OUTPUT 1 -p udp --destination-port 1194 -m comment --comment "Allow VPN connection" -j ACCEPT

# Drop any other traffic
iptables -A OUTPUT -j DROP
iptables -A INPUT -j DROP
iptables -A FORWARD -j DROP

# Log dropped packets (for debugging)
iptables -N logging
iptables -A INPUT -j logging
iptables -A OUTPUT -j logging
iptables -A logging -m limit --limit 2/min -j LOG --log-prefix "IPTables general: " --log-level 7
iptables -A logging -j DROP

echo "saving"
iptables-save > /etc/iptables.rules
echo "done"
```

### Enable IP Forwarding

Enable IP forwarding to allow traffic to pass through the VPN gateway:

```bash
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```

### Test the Setup

Run the following commands to test the VPN connection and firewall rules:

```bash
sudo bash /etc/openvpn/iptables.sh
sudo bash /etc/openvpn/connect.sh
```

Check if the VPN is working by using the following command on another device:

```bash
curl ifconfig.me
```

This should return the IP address of the NordVPN server.

### Set Up Auto-Start on Boot

To ensure the VPN connection starts automatically when the VM boots, create a systemd service.

1. **Create the rc.local service:**

```bash
sudo nano /etc/systemd/system/rc-local.service
```

Add the following content:

```ini
[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```

2. **Create the rc.local script:**

```bash
sudo nano /etc/rc.local
```

Add the following content:

```bash
#!/bin/sh -e
sudo bash /etc/openvpn/iptables.sh &
sleep 10
sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
sudo bash /etc/openvpn/connect.sh &
exit 0
```

3. **Enable and start the service:**

```bash
sudo systemctl enable rc-local
sudo systemctl start rc-local
```

4. **Reboot the VM:**

```bash
sudo reboot
```

After reboot, any device on your LAN that changes its default gateway to the VPN Gateway server address will route its traffic through the VPN.

---

## Troubleshooting: VPN Configuration Warnings

When you run the connection script for the first time, you may encounter warnings related to the VPN configuration file. These warnings are typically related to the security or compatibility settings in the `.ovpn` file. To address this, here are a few recommendations:

1. **Disable `comp-lzo` compression:** 
   This is often deprecated due to security vulnerabilities. Open the `.ovpn` file and comment out any lines that contain `comp-lzo`.

2. **Force the use of UDP:** 
   Ensure that your configuration file uses UDP (not TCP) for the VPN connection by checking the `proto` line. It should look like:

   ```
   proto udp
   ```

3. **Add extra security configurations:**
   Add the following lines to the `.ovpn` file to increase security:

   ```bash
   remote-cert-tls server
   cipher AES-256-CBC
   auth SHA256
   ```

4. **Adjust MTU settings for compatibility:** 
   If you encounter connectivity issues, try adjusting the MTU (Maximum Transmission Unit) to ensure packet size compatibility. Add this line to your `.ovpn` file:

   ```bash
   tun-mtu 1500
   ```
