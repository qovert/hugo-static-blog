---
title: "Deploying Nextcloud Part 1"
date: 2021-04-12T12:08:45-07:00
draft: false
taxonomies:
  categories: ["blog post"]
  tag: ['home network', 'containers', 'podman']
---

This post will focus on a recent project of mine to containerize my personal Nextcloud instance. I've been running Nextcloud for years as my personal cloud file storage. I find the ability to automatically sync my phone's pictures and videos back home particularly handy! My previous Nextcloud instance was running on a FreeBSD VM for quite some time, so why the change? 

## Goals and Planning

A lot of projects began moving to containers as a way to standardize development environments and practices, as well as a bunch of other reasons. From a systems administration perspective, containerizing is a great way to stretch your hardware resources, reduce bottlenecks in your deployment flow, and really improve flexibility and portability. 

For my purposes I settled on using [podman](https://podman.io/) for several reasons. First, I'm already somewhat familiar with it since I'm a [fedora](https://getfedora.org/) linux user. Secondly, the Podman project is intended to be an improvement over docker while retaining the same concepts and often literal commands. Unlike Docker, however, it's daemonless which is fantastic, and **allows for non-privileged user control!** 

### Layout

My home network runs on [proxmox](https://www.proxmox.com/) backed by **OpenZFS** storage. This lets me easily snapshot and backup my VMs and containers using [sanoid/syncoid](https://github.com/jimsalterjrs/sanoid) as well as many other benefits that we won't go into here. 

I'm going to start by provisioning a single Fedora Server 34 VM with the following resources:

| Hostname            | Description      | vCPU Count | RAM (MB) | HDD (GB)             |
| :-------            | :----------      | ---------: | -------: | -------:             |
| pod1.lan.hrkrdr.com | Fedora 34 Server | 4          | 2048     | 12GB boot/500GB data |

Eventually I'll add additional VMs to build a Kubernetes cluster to study for the [CKA](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) but for now, it will be a stand alone server. 

Reading the Nextcloud docs, there's several components that need to run together (in this case, in the same **pod**):
- **Database:** MariaDB
- **Memory Caching:** Redis
- **Application:** Nextcloud
- **Cron:** Nextcloud
- **Web:** Nginx with LetsEncrypt

I currently have an [HAProxy](http://www.haproxy.org) host on my network that takes care of all my LetsEncrypt certificates, so I won't be going over certificate provisioning here. If you need LetsEncrypt, the Nextcloud documentation uses [this](https://github.com/nginx-proxy/docker-letsencrypt-nginx-proxy-companion) and it's very well documented. 

Alright, now that I know what I need, we can get to building! I'd like to bring up that Podman fully supports docker-compose files with [podman-compose](https://github.com/containers/podman-compose) (they're really trying not to make this hard!). Nextcloud provides great [example compose files](https://hub.docker.com/_/nextcloud#running-this-image-with-docker-compose) to get you started. If you're familiar with docker-compose, it's a drop in replacement and should work great. I'm going to go 'the long way' here for purposes of learning and getting into the guts of podman a bit. 

## Implementation

Before we spin up any containers, let's make sure we can communicate with them. For this particular project, we're going to have the Nextcloud app listen on http://your_container_server:8080/. Let's poke a hole in the firewall: 

```bash
# Firewalld is what we're changing here, since this is an RHEL based system. Debian/Ubuntu users will need to use UFW

~$ sudo firewall-cmd --zone=public --permanent --add-port=8080/tcp

# Check to verify

~$ sudo firewall-cmd --zone=public --permanent --list-ports

```

Next, we'll create the folder structure our containers will be mapped to in the /data/ directory: 

```bash
~$ mkdir -p /data/containers/nextcloud/{db,nginx,html}
```

Now let's define a pod to contain our containers: 

```bash
# Note the '$'. I'm running these commands as a normal user, not root or sudo. 

~$ export PODNAME=nextcloud

~$ podman pod create --hostname ${PODNAME} --name ${PODNAME} -p 8080:80

```

Now we have a shared space for our containers. The concept of **pods** was introduced in Kubernetes, and podman pods are similar to that. When you create a pod, a new container is pulled that basically just goes to sleep. This container holds all of the global attributes for the pod members, such as port bindings, kernel namespaces, etc. 

You can see this in action like so: 

```bash
~$ podman pod ps

CONTAINER ID  IMAGE                 COMMAND  CREATED         STATUS             PORTS                 NAMES
4c3e0c9e1862  k8s.gcr.io/pause:3.2           4 minutes ago  Up 4 minutes ago  0.0.0.0:8080->80/tcp  63f269df4b2a-infra
[...]

```

### Containers!
On the VM we need a place to store Nextcloud data and the database. I've mounted the data disk of my VM to /data/ and created a directory called 'containers' under that, which my non-root user has read/write access to. Let's grab an official Nextcloud Nginx config and place it in the appropriate path:

```bash
~$ curl -o /data/containers/nextcloud/nginx/nginx.conf https://raw.githubusercontent.com/nextcloud/docker/master/.examples/docker-compose/with-nginx-proxy/mariadb-cron-redis/fpm/web/nginx.conf
``` 

We've got our VM, a place to store data, and we've got our pod, let's make some containers: 

```bash
# Keep an eye on the :z and :Z flags. 
# These tell podman to label the volumes appropriately for Selinux!


# MariaDB. Upper case :Z tells podman to 
# label the content as private/unshared. 

podman run \
  -d --restart=always --pod=${PODNAME} \
  -e MYSQL_ROOT_PASSWORD="SoSecurePassword" \
  -e MYSQL_DATABASE="nextcloud" \
  -e MYSQL_USER="nextcloud" \
  -e MYSQL_PASSWORD="SecureButDifferentThanTheOtherOne" \
  -v /data/containers/nextcloud/db:/var/lib/mysql:Z \
  --name=${PODNAME}-db docker.io/library/mariadb:latest \
  --transaction-isolation=READ-COMMITTED --binlog-format=ROW
	
# Now for Redis

podman run \
  -d --restart=always --pod=${PODNAME} \
  --name=${PODNAME}-redis docker.io/library/redis:alpine \
  redis-server --requirepass RedisSecurePasswordHere
	
# Nextcloud. Lower case :z tells podman that multiple
# containers share the content, so label it shared.

podman run \
  -d --restart=always --pod=${PODNAME} \
  -e REDIS_HOST="localhost" \
  -e REDIS_HOST_PASSWORD="RedisSecurePasswordHere" \
  -e MYSQL_HOST="localhost" \
  -e MYSQL_USER="nextcloud" \
  -e MYSQL_PASSWORD="SecureButDifferentThanTheOtherOne" \
  -e MYSQL_DATABASE="nextcloud" \
  -v /data/containers/nextcloud/html:/var/www/html:z \
  --name=${PODNAME}-app docker.io/library/nextcloud:fpm-alpine
	
# Nextcloud cron

podman run \
  -d --restart=always --pod=${PODNAME} \
  -v /data/containers/nextcloud/html:/var/www/html:z \
  --entrypoint=/cron.sh \
  --name=${PODNAME}-cron docker.io/library/nextcloud:fpm-alpine
	
# Nginx web server
podman run \
  -d --restart=always --pod=${PODNAME} \
  -v /data/containers/nextcloud/html:/var/www/html:ro,z \
  -v /data/containers/nextcloud/nginx/nginx.conf:/etc/nginx/nginx.conf:ro,Z \
  --name=${PODNAME}-nginx docker.io/library/nginx:alpine
	
```

At this point you should have a running Nextcloud instance on your host, listening on port 8080! To finish the configuration of Nextcloud, we're going to execute the installation script within the container itself: 

```bash
~$ podman exec -t -u www-data nextcloud app \
	php occ maintenance:install \
	--database "mysql" \
	--database-host "127.0.0.1" \  
	--database-name "nextcloud" \
	--database-user "nextcloud" \
	--database-pass "SecureButDifferentThanTheOtherOne" \
	--admin-pass "AdminNextcloudPassword" \
	--data-dir "/var/www/html"
```

Now if you browse to http://your_container_host:8080, you should see a Nextcloud login page. Log in using the **admin** user and the password you specified in the last command, and you're ready to go. 

Next up, I'll go over running these containers via systemd as a non-root user. 
