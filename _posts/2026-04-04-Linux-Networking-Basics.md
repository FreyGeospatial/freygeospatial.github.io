---
layout: post
title:  "Networking Basics: NAT, SSH, and Port Forwarding"
categories: [Linux, Networking, virtual machines]
---

I've been working through a Linux networking course lately, and wanted to document some of the initial concepts that I found valuable. This post covers NAT, SSH, network interface inspection via the command line, and port forwarding, with a practical focus on getting SSH working inside a virtual machine (VM) environment.

### Why This Matters for Data Engineers

Data engineering work is rarely confined to a single machine. Pipelines pull from remote databases, push to cloud storage, and run on servers accessible only through a terminal. Networking knowledge is therefore empowering, helping you debug issues with these varied services more independently.

### NAT: Network Address Translation

NAT, or Network Address Translation, is the mechanism that allows devices on a private network to communicate with external networks — and with each other.

In VirtualBox (the open-source hypervisor software allowing me to run VMs), a VM is given a NAT adapter by default. This means the VM can reach the internet through the host machine, and the host can reach the VM through a NAT. What it does *not* allow by default is VM-to-VM communication. For that, you need to configure a **NAT Network** — a shared virtual network that multiple VMs can join. This can be done through the VirtualBox settings UI.

### Network Interfaces: ip a and ifconfig

A network interface is a point of connection between a device and a network. This can be wifi, ethernet, loopback/localhost, etc.

Before setting up SSH or port forwarding, it helps to understand what your network interfaces look like. Two commands are commonly used for this:

- `ip a` — the modern standard on Linux
- `ifconfig` — the older Unix/macOS equivalent, part of the `net-tools` package

Both display information about your active network interfaces, including interface names (e.g., `eth0`, `lo`), IPv4 and IPv6 addresses, MAC addresses, and interface status. They usually show the private ip address of devices on network.

*Note: if you want to use ifconfig on a Linux machine where it isn't preinstalled, run `sudo apt install net-tools` and then add it to the path. To add it to the path, edit your `~/.bashrc file` and append `export PATH="$PATH:/sbin:/usr/sbin"` to it.*

#### What addresses will you see?

The addresses you see fall into a few categories:

**Private (RFC 1918) addresses** — routable only within a local network:
- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

**Other common addresses:**
- `127.0.0.1` / `::1` — loopback, also known as localhost. Traffic sent here is intercepted by the OS and looped back to the local machine — it never touches a NIC or a router.
- `169.254.x.x` — link-local, OR APIPA. **APIPA** is typically assigned when DHCP fails. DHCP is the Dynamic Host Configuration Protocol. It's the system that automatically assigns IP addresses to devices when they join a network, so you don't have to set one. APIPA stands for Automatic Private IP Addressing. It's a fallback mechanism some OS's use when a device can't reach a DHCP server. In practice, seeing a 169.254.x.x address is usually a sign that something is wrong — your machine couldn't find the DHCP server, meaning the router is down, the network cable is unplugged, or there's some other connectivity issue. Conversely, **link-local** refers to addresses that are only valid on a single network segment — they can't be routed beyond the directly connected link (cable, Wi-Fi, etc.). *Link-local does not necessarily mean that there is a connectivity issue.*
- `fd00::/8` or `fe80::/10` — IPv6 private and link-local ranges

On a typical home or office machine, `ip a` will show only private IPs. On a cloud VM (AWS, GCP, etc.), you will often see a private IP on the interface even if the machine has a public IP — the public IP is handled by the cloud provider's NAT layer and won't appear in `ip a` at all.

To find your **public IP**, query an external service:

```bash
curl ifconfig.me      # returns whichever IP your request arrived from (may be either IPv4 or IPv6)
curl -4 ifconfig.me   # show IPv4
curl -6 ifconfig.me   # show IPv6
```

### SSH: Secure Shell

SSH (Secure Shell) lets you control a remote system from the command line over an encrypted connection. If `openssh-server` is not already installed on your target machine, you can add it with:

```bash
sudo apt install openssh-server
sudo systemctl enable ssh
```

The default port for SSH (or other secured connections) on any machine is **22**.

To connect:

```bash
ssh <username>@<ip-address>
```

By default, there is no direct route from the host machine to the VM's private IP in VirtualBox. Port forwarding is the solution.

### Port Forwarding

Port forwarding redirects traffic arriving at one address and port to a different address and port. When data arrives at a machine on a specific port, instead of that machine handling it directly, the traffic is forwarded elsewhere.

#### The Home Router Example

Your router has a single public IP. Behind it are many devices with private IPs, none of which are reachable directly from the internet. Port forwarding solves this:

1. Someone connects to your public IP on port `8080`
1. Your router sees the incoming traffic and forwards it to a private IP on port `80`
1. The server on that machine responds, and the router sends the reply back

The outside world only ever sees the public IP — the internal machine stays hidden.

Other common uses include:
- Hosting a game server at home
- SSH-ing into a home machine from outside your network
- Exposing a local web server to the internet

#### Port Forwarding with VirtualBox

To SSH from your host machine into a VirtualBox VM, configure port forwarding in the VM's network settings:

| Setting | Value |
|--|--|
| Host IP | `127.0.0.1` |
| Host Port | Any unused port, e.g. `22022` |
| Guest IP | The VM's IP address (from `ip a` inside the VM) |
| Guest Port | `22` (default SSH port) |

<br>

<div style="display: flex;">
    <div style="flex: 100%; text-align: center;">
        <img src="/images/linux-networking/Screenshot 2026-04-04 at 10.15.58 AM.png" alt="VirtualBox NAT Network configuration">
        <span style="font-size: 10px">Figure 1</span>
    </div>
</div>

Then SSH into the VM from your host:

```bash
ssh -p 22022 <username>@127.0.0.1
```

The `-p` flag specifies the host port you configured. The traffic hits your localhost on port `22022`, gets forwarded by VirtualBox to the VM on port `22`, and you have a shell.

### Conclusion

Networking is one of those topics that feels irrelevant until you find yourself unable to explain why a pipeline can't reach a database you know is running. The concepts here — NAT, private vs. public IPs, SSH, and port forwarding — come up often enough in data engineering work to make learning these foundations worthwhile, even when they aren't the focus of what you're building. I hope this serves as a useful starting point. And as always, happy coding!
