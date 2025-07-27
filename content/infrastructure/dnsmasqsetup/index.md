---
title: "Internal DNS Setup"
summary:  "Creating our internl dns"
date: 2025-07-26
tags: ["deployment", "dns"]
draft: false
---


First, we need a domain name.  I registered th3redc0rner.com as a domain on Cloudflare.  Cloudflare has an API supported by Caddy, which we will use as a reverse proxy and set up in the next blog.  

I am using OpenWrt as a firewall/router for my vlans and firewall.  One of its features is I can set up a dnsforward entry for another domain.  So in this case I setup a forward for dev.th3redc0rner.com to an internal dns server.  This allows me to setup specific DNS entries each zone.

After deploying the machine on our network we will  Ansible playbook **osupdate** from the last blog to make sure the machine is up to date.  Ansible uses inventory files to specify what machine to perform its configuration on but you can specify machines directly on the command line by ending the list with a ,


```css
ansible-playbook osupdate.yaml -i 192.168.200.96, -u admin1
```

Once we SSH in the machine we are going to setup dnsmasq

```css
sudo apt-get install dnsmasq
```
We have to disable systemd-resolved because this will interfere with dnsmasq

```css
systemctl disable systemd-resolved
systemctl stop systemd-resolved
```

Then we have to delete the current symbolic link for /etc/resolv.conf
```css
sudo rm /etc/resolv.conf
```

Once that is done. create a new file with the nameserver that you use for your lan 

```css
sudo echo nameserver 192.168.2.1 > /etc/resolv.conf
```

We are then going to create a second resolver for dnsmasq itself
```css
sudo echo nameserver 8.8.8.8 > /etc/resolv.dns
```

Now let's configure dnsmasq.  I usually create separate config files in dnsmasq.d based on their functionality.

First we are going to create a resolv.conf to point to that resolv.dns file.  I found that if dnsmasq points to my actual router it can create a weird recursive dns loop.

```css
sudo echo "resolv-file = /etc/resolv.dns" > /etc/dnsmasq.d/resolv.conf
```

Next we we want a to make a listener configuration file.  This allows dnsmasq to listen from other subnets then itself.  This allows the dns forward to work.

```css
sudo echo "listen-address=127.0.0.1,192.168.200.95" > /etc/dnsmasq.d/listen.conf
```
Finally, we can create a file to define custom dns zones
```
sudo echo "addn-hosts=/etc/dnsmasq.d/dns/" > custom_dns.conf 
```

Now we need a directory to store those custom dns files
```css
mkdir /etc/dnsmasq.d/dns/
```

I named my zone file /etc/dnsmasq.d/redcorner.conf

My current zone file looks like this

```css
192.168.200.226 identity.dev.th3redc0rner.com
192.168.200.226 gitea.dev.th3redc0rner.com
```

This points both sites to the reverse proxy, which I will walk through in the next blog article.

The last thing to do is start the DNS server:

```css
systemctl restart dnsmasq
```

