---
title: "Nextcloud"
date: 2022-12-07T19:16:00+05:30
draft: false
---

## Yet another nextcloud installation guide

I recently started using nextcloud as my main cloud storage and so far it seems like a good choice. Some of the features that I think are note worthy:

- **You can make multiple accounts for your friends and family**
- **Limit the storage**
- **Integrated office**
- **For organizing meetings, there's something called Talk**

This won't be a step-by-step guide as there are already a good amount of tutorials on the internet which does exactly the same. This guide will cover some of the "pitfalls" that you might encouter while setting up your own instance.

Also, keep in mind that nextcloud is not a light-weight application. I would suggest to run it either on a dedicated server or on a VPS with around 8 gigs of ram.

## Not the actual stuff

I will be using docker for this guide. I am sure you already have it. If you don't, [duck it!](https://duckduckgo.com)

## Actual stuff

We will use the following docker-compose. Save the file as docker-compose.yml

```YAML
version: "3.8"

services:
  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - MYSQL_DATABASE=nexcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=securepassword
      - MYSQL_HOST=nextcloud-db
      - NEXTCLOUD_ADMIN_USER=root
      - NEXTCLOUD_ADMIN_PASSWORD=securepassword
    volumes:
      - /your/path/nextcloud/appdata:/config
      - /your/path/nextcloud/data:/data
    ports:
      - 127.0.0.1:8888:80
    depends_on:
      - nextcloud-db
    restart: unless-stopped

  nextcloud-db:
    image: linuxserver/mariadb
    container_name: nextcloud-db
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=securepassword
      - MYSQL_PASSWORD=securepassword
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
    volumes:
      - /your/path/mariadb/config:/config
    restart: unless-stopped

```

We will be using mariadb for our nextcloud instance. You could use postgresql but it is not recommended for production level use.

Let's look at some of the enviroment variables:

- `NEXTCLOUD_ADMIN_USER` and `NEXTCLOUD_ADMIN_PASSWORD` are the admin credentials.
- `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` set this to your desired values and make sure to use a strong password.

Change the volume paths accordingly. After that, you are good to go.

## Pitfalls

Now, you have a running nextcloud instance. I encountered a lot of issues before I could actually use it. Two of which are **Trusted Domains** and **Over Write Protocol**.

The environment variables for the same didn't work for me so I had to manually sort it out. Let's talk a little bit about the first issue and it's solution.

### Trusted domains

Basically, trusted domains are the domains on which you could run your instance. If I am running it on say **cloud.example.com** then that domain has to be added to trusted domains. It's that simple.

If you are interested in knowing more about it, here's the [official documentation](https://docs.nextcloud.com/server/23/admin_manual/installation/installation_wizard.html#trusted-domains) which goes a little bit deeper.

Nextcloud has a fine utility called `occ`. It is used to make changes to the `config.php` file. We will be using that to solve this issue.

The docker image that we've used sets up a user `abc`. We will use that user to make some changes.

#### One of many solutions

Let's find `occ` first. We can use `find` to look for it.

```shell
docker exec -it nextcloud find / -iname *occ*
```

![occ](/occ.png)

For some reason, you cannot use `/usr/bin/occ` which is infact added to the path. For instance, there is a module called **security:bruteforce:reset** which is used to unblock your IP incase it got blocked by the bruteforcing mechanism. If I try to use it, I would get something like this

![example](/example.png)

We would use the full path which is `/config/www/nextcloud/occ`. We can set the trusted domain by running the following

```shell
docker exec --user abc nextcloud php /config/www/nextcloud/occ config:system:set trusted_domains 2 --value=yourdomain
```

Replace `yourdomain` with the domain or subdomain of your choice.
