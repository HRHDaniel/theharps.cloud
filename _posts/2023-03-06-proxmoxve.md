---
layout: post
title:  "Proxmox VE"
date:   2023-03-06 10:00:00 -0600
img: proxmox.png
tags: [Proxmox, HomeAssistant] # add tag
---
[Proxmox VE](https://www.proxmox.com/en/proxmox-ve) is a "complete, open-source server management platform for enterprise virtualization." It hosts my home assistant VM, as well as a variety of VMs and containers for testing and development.

Initially I installed home assistant on a RaspberryPI on just a SD card. DO NOT DO THIS. The drive will fail eventually. After multiple failures and rebuilding home assistant several times, I searched for a better alternative. One option was using an additional hard disk attached to the Pi, but I opted for Proxmox to give myself an environment for more VMs/containers for future projects and experiments. I bought a cheap, refurbished HP EliteDesk 800 G3 as my host.  Apparently, the Intel Nuc is highly recommended for this as well, but at a significantly higher price point.

<img src="/imgs/hp-elite-desk-800.jpg">

## Installation

[Download](https://www.proxmox.com/en/downloads/category/iso-images-pve) the latest Proxmox installation ISO.

I used [Balena Etcher](https://www.balena.io/etcher) to flash the ISO to a USB drive.

Insert the USB into the EliteDesk and power on. Press F10 during boot to get into the bios settings. Ensure **Legacy Boot** is enabled and **Secure Boot** is disabled.  Ensure that **Virtualization Technology (VTx)** and **Virtualization Technology Directed I/O (VTd)** are also enabled.

Save/Exit/Reboot

This time press F9 to select the boot option and select the USB drive.

You should see the Proxmox menu. Select **"Install Proxmox VE"**.

Walk through the installation process.
- Accept the license agreement.
- Select the hard drive for installation.
- Set your country/timezone/keyboard layout.
- Set a password and email. 
- I set my hostname to `proxmox.theharps.cloud`
- I set a static IP address on my "Lab" VLan.  This VLan I use for infrastructure I want separated from most of my network.
- Click Install

After the reboot, you should be able to access the web console at {ip address}:8006

The first thing I did after logging in was to go to updates, **Refresh**, then **Upgrade** to apply all latest updates.

## Network Setup
As mentioned, I placed the ip address for proxmox on my "Lab" VLan. I also have a VLan for Home Automation that the HomeAssistant VM will be assigned to, as well as a Sandbox VLan that has blocked internet connections by default that I'll want to be able to put VMs on, so we need to make Proxmox aware of these VLans.

SSH to Proxmox or log in to the web console and click on `>_ Shell`

Edit `/etc/network/interfaces` file to look like:
```
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
        bridge-vids 2-100

auto vmbr0.3
iface vmbr0.3 inet static
        address 192.168.3.25/24
        gateway 192.168.3.1

auto vmbr0.4
iface vmbr0.4 inet static
        address 192.168.4.25/24

auto vmbr0.5
iface vmbr0.5 inet static
        address 192.168.5.25/24
```
- The `auto` lines just say bring up that interface automatically on boot.
- We added two new parameters to our virtual bridge **vmbr0**
  - `bridge-vlan-aware yes` so that it knows to use VLANs
  - `bridge-vids 2-100` tells it which VLANs it should allow communication to.  3 is my Lab, 4 my Home Automation, and 5 my Sandbox.  I just gave it a large range (2-100) so I can add VLANs and not have to come back into the shell (new interfaces can be added in the web console once this initial step is complete).

After saving that file, reboot proxmox.

Next, I had to go into my Unifi Controller and update the port profile to add the tagged networks for VLANs 3, 4, and 5. The way you do this for your switch may vary. Just search the web for how to add tagged networks for your switch.

I updated my router (Unifi USG) to resolve `proxmox.theharps.cloud` to the IP address I configured for it, `192.168.3.25`.  By placing this on my gateway, this domain name only resolves internally on my network.

## SSL Certificates
I have a wildcard certificate from Let's Encrypt for my domain.  In the proxmox web console, select the proxmox node > System > Certificates.  There's a button there to upload custom certificate.  Upload the private key and full chain.

After the above, I now have a Proxmox VE environment ready to host VMs and LXC containers.  I can connect to it easily from my home network with `https://proxmox.theharps.cloud`, and it's using a "real" certificate, or one from a trusted CA, so it's encrypted without any self signed cert errors.

In future posts, I'll discuss how I created my HomeAssistant VM and automated the updates for my certificate.