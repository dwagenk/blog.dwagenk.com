---
layout: post
title: Setting up Nextcloud with Collabora Online Office using Docker
excerpt_separator: <!--more-->
---

## Prelude
I started writing this blog post in a first attempt a couple days ago and it got way too verbose and lengthy, going into detail about the specifics of my setup and the reasoning behind it. So here's a short version.

## Goal
Adding Collabora to an existing Nextcloud instance to enable online collaborative document editing right inside the Nextcloud webinterface.

## Step by Step
### Prerequisites 
I'm assuming you've got Nextcloud installed somewhere already (publically reachable, with it's own domain or subdomain) and also installed the Collabora app (it's listed as one of the *official* apps in the *Office & Text* section of Nextcloud's list of apps). You also need a server for running Collabora, that has docker and docker-compose installed and a domain or subdomain pointing to it. If both, Nextcloud and Collabora, will be running on the same server, take a closer look at the `jwilder/nginx-proxy`, that I'll be using anyhow. It should be no problem having both running behind the proxy on the same server, accessible through different subdomains.

<!--more-->

### The `docker-compose.yml` file
is stitched together from [this](https://help.nextcloud.com/t/collabora-configuration-with-docker-compose/3970) thread:

```yaml
version: '2'

services:
  proxy:
    image: jwilder/nginx-proxy
    container_name: proxy
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./proxy/conf.d:/etc/nginx/conf.d
      - ./proxy/vhost.d:/etc/nginx/vhost.d
      - ./proxy/html:/usr/share/nginx/html
      - ./proxy/certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    networks:
      - proxy-net

  letsencrypt-companion:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: letsencrypt-companion
    volumes_from:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./proxy/certs:/etc/nginx/certs:rw

  collabora:
    image: collabora/code
    container_name: collabora
    expose:
      - 9980
    cap_add:
      - MKNOD
    environment:
      - domain=NEXTCLOUD_DOMAIN
      - VIRTUAL_HOST=COLLABORA_DOMAIN
      - VIRTUAL_NETWORK=proxy-net
      - VIRTUAL_PORT=9980
      - VIRTUAL_PROTO=https
      - LETSENCRYPT_HOST=COLLABORA_DOMAIN
      - LETSENCRYPT_EMAIL=ADMIN_EMAIL
    networks:
      - proxy-net


networks:
  proxy-net:
    external:
      name: proxy-net

```

You'll just need to adapt `NEXTCLOUD_DOMAIN`, `COLLABORA_DOMAIN` (2x) and `ADMIN_EMAIL`. Also the Network, that connects the services (=collabora) with the proxy is declared as external, so you'll have to create it by running

```bash
docker network create --driver bridged proxy-net
``` 

before starting the containers with 
```bash
docker-compose up
```


> Note: if you also want to have access to Collabora's admin interface at https://COLLABORA_DOMAIN/loleaflet/dist/admin/admin.html also provide `username` and `password` in the *environment* section for the Collabora container. 

### Link Nextcloud and Collabora
All that's left to do is enter the domain of your collabora service into the settings of your nextcloud collabora app and start editing documents right inside your nextcloud!
![Nextcloud - Collabora Settings](/assets/collabora-for-nextcloud/settings.png)



[CC-BY-SA-4.0](http://creativecommons.org/licenses/by-sa/4.0/)
