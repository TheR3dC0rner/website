---
title: "Reverse Proxy Setup"
summary:  "Creating our internl reverse proxy"
date: 2025-07-14
tags: ["deployment", "revers", "proxy"]

---
We are going to use a single proxy server to host up our tls certificates in our network. Caddy can support dns-01 acme challenge from Let's Encrypt and renew the cert automatically so it seems like a good candidate to this.  This will provide an trusted SSL certificate for any machine behind our reverse proxy without us requesting one for each machine.

The first thing we are going to do is install docker on this machine using the playbook we previously made.

```css
ansible-playbook -i 192.168.200.226 install_docker -u admin1
```

 
After we have docker installed on that machine we can begin building our proxy server.  

First ,we are going to create a Docker network
```css
docker network create proxy
```

Next we are going to create a caddy direrectory
```css
mkdir -p caddy
mkdir -p caddy/caddy-data
mkdir -p caddy/caddy-config
```

A lot of this came from Jim’s Garage, so we’re modifying his files to build our setup.  To see his github https://github.com/JamesTurland/JimsGarage.

As previously discussed I had registered Th3redC0rner.com as my domain and we are going to build our proxy to server our internal urls.

The internal domain we will use is dev.th3redc0rner.com 

Now we will make our docker-compose.yaml

It will look something like this

```css
services:
  caddy:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: caddy
    restart: unless-stopped
    env_file: 
      - .env
    environment:
      - CLOUDFLARE_EMAIL=${CF_EMAIL}
      - CLOUDFLARE_API_TOKEN=${CF_API_TOKEN}
      - ACME_AGREE=true
    ports:
      - 2019:2019 # remove if you do not want admin API
      - 80:80
      - 443:443
    volumes:
      - caddy-config:/config
      - caddy-data:/data
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./index.html:/usr/share/caddy/index.html
    networks:
      - proxy # add other containers onto this network to use dns name

volumes:
  caddy-config:
  caddy-data:

# create this first before running the docker-compose - docker network create caddy
networks:
  caddy:
    external: true
```

Let's create a Dockerfile for building the docker image.  The reason we need a Dockerfile instead of pulling just an image is that we need to include the Cloudflare module with caddy to modify our dns entries to add the txt record to get our wild card certificate

```css
# For prod you'd want to pin the version: e.g., 2.9.1-builder
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare
FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

now are are going to create our Caddyfile.  This will hold our "routing" information
```css
{
        admin 0.0.0.0:2019
}

*.dev.th3redc0rner.com {
        tls {
                dns cloudflare {env.CF_API_TOKEN}
                propagation_delay 2m
                resolvers 1.1.1.1
        }


        @identity host identity.dev.th3redc0rner.com
        handle @identity {
                reverse_proxy 192.168.201.75:1421
        }

}
```
 

We will then create a .env file that will contain our cloudflare key and email for registering getting our wildcard cert

```css
CF_API_TOKEN=1ufPvdNumd2MJd9jBQmPSPLRweLu_VrNgcW1shxy
CF_EMAIL=your@email.com

```
This is not my real api key its just an example

We then want to ssh into our internal dns.  In the /etc/dnsmasq.d/dns/redcorner.conf we want to add the entries similar to these entries. 

```
192.168.200.226 identity.dev.th3redc0rner.com
192.168.200.226 gitea.dev.th3redc0rner.com

```
Don't forget to restart dnsmasq
```css
systemctl restart dnsmasq
```
We would then ssh back to our proxy machine and be able to do a docker build.  Then a docker compose up

```css
docker compose build
docker compose up
```

If configured correctly you should see in the logs that Caddy will grab a wildcard cert from letsencrypt.  There is nothing hosted yet on the backend though.  That is our next blog.


