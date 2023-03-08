---
layout: post
title:  "Home Assistant installation"
date:   2023-03-08 10:00:00 -0600
img: home-assistant-banner-logo.png
thumb: home-assistant-logo.png
tags: [HomeAssistant, Proxmox] # add tag
---

## Why Home Assistant
When choosing what hub to use for my home automation there were a couple of key features I was looking for: privacy, local control, and wide compatibility/vendor agnostic. The big players, Google Home, Amazon Alexa, and Apple HomeKit, obviously are going to come up short in that comparison. After searching around, the open source project [Home Assistant](https://www.home-assistant.io/) seemed to check all the boxes.  There are other projects that seem to fit my requirements as well, such as [openHAB](https://www.openhab.org/) which also seemed highly recommended with a good user community. Honestly, I do not remember what made me choose Home Assistant over OpenHab or similar projects, but hopefully this answers why I choose it over a proprietary platform like HomeKit.

## Hardware/VM selection
Initially I installed Home Assistant on a Raspberry Pi running directly on the MicroSD card. When selecting the Raspberry Pi from their installation documentation (the first option on the list) I followed their recommended hardware list.
 
> **DO NOT DO THIS**

If you really want to use the Raspberry Pi, get an external SSD drive attached and install to it. There are far too many writes to the disk and the MicroSD card will fail eventually.

After rebuilding Home Assistant on my Pi several times, I finally decided to move it to other hardware. Ultimately, I decided on setting up a ProxmoxVE host and running Home Assistant as a VM on it. This also gave me a platform for other VMs and containers which I immediately found useful and wished I had done this much sooner.  See my post [Proxmox VE]({% post_url 2023-03-06-proxmoxve %}) for details of how I setup ProxmoxVE. In addition to the stability and not dealing with failed SD cards, I noticed a significant jump in speed, primarily when working on new integrations that required restarts.

## Creating the VM
Log in to the proxmox console, and press the "Create VM" button.
Here are the settings I entered or changed from the defaults:
- General
	- Name `homeassistant`
	- Start at boot
- OS
	- Do not use any media
- System
	- Bios UEFI
	- Storage - the only option: local-lvm
- Hardisk
	- defaults
- CPU
	- 4 cores
- Memory
	- Max 6144
	- Min 2048
- Network
	- VLAN Tag `2`

Note that I am using tagged VLANs and you need to ensure that VLAN is setup in your Proxmox networking configuration. If you aren't using VLANs leave that field blank.

When downloading Home Assistant, there is a section for ["Alternative" installations](https://www.home-assistant.io/installation/alternative).  From that page, select the "KVM/Proxmox (.qcow2)" download.

This download is compressed as a `.xz` file. I used [7Zip](https://www.7-zip.org/) on windows to extract the contents.

SFTP the extracted image to proxmox. (I used [FileZilla](https://filezilla-project.org/))

Then you need to import that disk image to your vm.  Log into the proxmox shell and execute:

`qm importdisk 100 /root/haos_ova-9.5.qcow2 local-lvm --format qcow2`

Change `100` in that command to the id of your Home Assistant VM.

You can delete the uploaded image file from proxmox.

From the proxmox ui, select the Home Assistant VM, Hard Disk (current one) and click detach

Select the detached hard disk and click remove

Click on the imported HA disk and click **Edit** to add the disk

From the Menu select options and boot order, select the new drive and drag to the top

Click on the Console, then Start the VM

In the console, hit `Esc` to enter the VM's BIOS and disable secure boot.  Save/restart and Home Assistant should boot normally.

## ZWave / Zigbee hub
I grabbed a [Nortek GoControl USBZB-1 zwave/zigbee](https://www.amazon.com/dp/B01GJ826F8) hub off amazon to be able to connect both ZWave and Zigbee devices. 

Plug the hub into the Proxmox server. In the proxmox console, select the Home Assistant VM > Hardware > Add > USB Device

Select `Use USB Vendor/Device ID` and select the Zigbee/Zwave usb adapter.  Click Add.
Reboot Home Assistant

## Setup
The console/shell of the VM should print the IP address.  The default port is `8123`.

You might be able to access Home Assistant at [http://homeassistant.local:8123/](http://homeassistant.local:8123/)

I initially accessed it via the IP address: [http://192.168.3.166:8123/](http://192.168.3.166:8123/), although I later updated my gateway to assign a pretty domain name... `ha.theharps.cloud`

You will be greeted with an onboarding workflow to create an initial admin user and review a few settings, so just walk through that and you'll now have Home Assistant up and running.

In a follow up post, I will walk through setting up ssh access, ssl certificates, and automating backups (highly recommended).


