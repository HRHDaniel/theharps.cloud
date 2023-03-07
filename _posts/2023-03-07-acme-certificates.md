---
layout: post
title:  "ACME SSL Certificates"
date:   2023-03-07 10:00:00 -0600
img: ACME-Horiz.png
thumb: ACME-protocol-icon.png
tags: [ACME, SSL, Proxmox] # add tag
---
The ACME Protocol (Automatic Certificate Management Environment) allows for automating the request and issuance of SSL certificates from supporting certificate authorities. See [RFC8555](https://www.rfc-editor.org/rfc/rfc8555) for all the technical details of the protocol.

Even though the vast majority of the services running on my home network are not externally reachable, I still prefer serving those over SSL. I could do this with self-signed certificates, but then I have to install or trust those certificates throughout my devices (or ignore the self-signed warnings which defeats a large part of the benefits). However, since I own a domain that I'm using for my internal services (theharps.cloud), the ACME protocol allows me to easily generate a certificate for my domain from a trusted CA. I use [Let's Encrypt](https://letsencrypt.org), but there are other CAs that support the protocol (ZeroSSL, Google, Buypass, SSL.com...)

Let's Encrypt is free and requires no registration. However, the max lifetime of an issued certificate is 90 days. This would be very painful if we did not automate requesting new certificates and deploying these to our home lab.

Let's Encrypt recommends [certbot](https://certbot.eff.org/) for this automation. Certbot does not support the DNS registrar I was using, Dynadot. I didn't find any ACME implementations that did support Dynadot, and eventually found out why and I no longer recommend them. This left me needing to write my own integration to Dynadot's apis. I choose to do this with [acme.sh](https://github.com/acmesh-official/acme.sh/) as Certbot is not accepting contributions of new providers. Rather than just writing this for myself I wanted to make it easy for others to benefit as well.

I did get this integration working and submitted a PR to contribute back to the main project. That PR has not yet been reviewed/merged. In the meantime, my code is available on my GitHub [fork](https://github.com/HRHDaniel/acme.sh/tree/dns_dynadot). If you decide to leverage this, please read in detail the comments in the header of file `dnsapi/dns_dynadot.sh` that detail the issues I found with the Dynadot APIs and their implementations. If you wisely choose to go to another registrar, as I eventually did, the rest of this post will still provide the steps needed to automate this process in a Proxmox Container.

## Creating the container
In the proxmox console, expand the proxmox node and select "local (proxmox)" > Storage > CT Templates

![CT Templates](/imgs/ProxmoxCTTemplates.png)

This lists the templates you have available for LXC containers. If you do not have the template you want to run on, clickl the Templates button, select a template, and click Download. I chose the latest version of Alpine linux. The rest of the install commands below may differ if you choose a different distribution.

On the top menu, press the "Create CT" button. Most of the default settings should work. Here's the settings I entered or changed:
```
hostname: acme
password: *****
template: alpine-*.*-...
VLAN Tag: 2  # if your using VLANs and your switch is tagging networks, make sure to select the correct tag)
IPv4: DHCP
```

I updated my Unifi controller and USG to provide a static IP to this container and updated my firewall rules to allow this container to talk with all the systems that need certificates.

## Installing acme.sh
Select the container from the menu, press the start button, then select ">_ Console" from the menu.  You should now have a shell in the running container.

To install the prerequisites and acme.sh, enter the following commands in the container:
```sh
apk update
apk add jq
apk add curl
apk add git
apk add openssl
apk add openssh
git clone https://github.com/acmesh-official/acme.sh.git
cd acme.sh
./acme.sh --install --nocron
cd ..
rm -rf acme.sh  # remove git repo as it is now installed in ~/.acme.sh
```

If you need dynadot support before my PR is accepted, change the clone above to my fork. Note that I provided `--nocron` to the installer. By default acme.sh installs a scheduled cron job to run every morning. I am going to install this cron job differently as I won't leave this container running all the time.

## Creating the initial certificate with acme.sh
For the dynadot integration I wrote, you need to create an API token. Do that from your account on the Dynadot website. If you're using another provider, you'll need to look at the acme.sh documentation for what parameters you need to provide.

```sh
export DYNADOTAPI_Token=APITOKEN
acme.sh --issue -d *.theharps.cloud --dns dns_dynadot --server letsencrypt -k 4096
```

I am creating a wildcard certificate that I can use for all my services. The `--server letsencrypt` argument must be provided as acme.sh defaults to ZeroSSL if not specified.

## Deploying the Certificate to Proxmox
acme.sh has a variety of integrations to automatically deploy the certificates when they are updated. I will cover how I do the deploys to other services as I discuss their installations (HomeAssistant, QNAP, etc.).

In proxmox create an api token. Save the token id and the generated secret someplace safe. 

Create a role `acme_sh` and assign the permission `sys.modify` to that role.

Open the proxmox shell/console or ssh.  Then add the role to the token:
```sh
pveum acl modify / -token 'root@pam!acme_sh' -role "acme_sh"
```

You can then have acme.sh deploy the certificate to proxmox:
```sh
export DEPLOY_PROXMOXVE_SERVER=proxmox.theharps.cloud
export DEPLOY_PROXMOXVE_USER=root
export DEPLOY_PROXMOXVE_USER_REALM=pam
export DEPLOY_PROXMOXVE_API_TOKEN_NAME=acme_sh
export DEPLOY_PROXMOXVE_API_TOKEN_KEY=<token_secret>

acme.sh --deploy -d *.theharps.cloud --deploy-hook proxmoxve
```

## Scheduling the container to run to update certificates
There's no need for this container to run all the time just for a cron job to execute acme.sh, especially since it will only be actually updating the certificates every 20 days. Instead we will automatically start and shutdown our container every morning to run the job.

From the proxmox console or ssh, create the file `/etc/cron.daily/run_acme` with the following contents to start the container every day.
```sh
#!/bin/sh

if [ "$(pct status 101)" = "status: stopped" ]; then
  pct start 101
fi
exit 0
```

Make sure to change `101` to your container ID.

--

Create a startup script that runs acme when this container starts: File: `/etc/init.d/acme`

```sh
#!/sbin/openrc-run

name="acme"
command="/root/runacme.sh"
command_args=""
command_background=true
pidfile="/var/run/acme.pid"

depend() {
        after net
}

```

Set 755 permissions on script
Create file: `/root/runacme.sh`

```sh
#!/usr/bin/env sh

# sleeping 5 minutes - this gives us time if we want to boot up this container and stop this process before
# it runs to do any sort of maintenance tasks or upgrades/updates.
echo "Acme initial sleep..."
sleep 300

echo "Running acme"
/root/.acme.sh/acme.sh --cron --home "/root/.acme.sh" >/root/.acme.sh/boot.log 2>&1

# When we're done, wait 10 minutes and shutdown this instance
echo "Acme complete, waiting before poweroff"
sleep 600
poweroff
```
Run: `rc-update add acme` to add script to start at boot.

We can now shutdown the container. The proxmox Cron job will start the container every morning. The container cron job will run on container boot. That job:
- initially sleeps 5 minutes
- calls acme.sh to update certificates if needed
- sleeps 10 minutes
- shuts down the container

I have added the sleeps to give time for me to kill the acme script if I need to start the container and perform any updates/maintenance and do not want acme.sh running, or to stop it from shutting down the container if I am trying to troubleshoot the script or deployments.
