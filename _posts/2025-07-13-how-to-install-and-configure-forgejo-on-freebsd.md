---
layout: post
title: "How to install and configure Forgejo on FreeBSD"
date: 2025-07-13
tags: FreeBSD Forgejo Git
---

Today I wanted to try out Bastille 1.0 on a fresh FreeBSD 14.3 machine, so I decided to set up Forgejo in a Jail.
This blog post was heavily inspired by: https://bsdbox.de/en/artikel/gitea/gitea-lokal

# Requirements
You will need:
- a FreeBSD host, VM, or jail with working internet access
- a user with root privileges or sudo/doas

# Getting started

Install the forgejo package:
```shell
doas pkg install -y forgejo
```

# Replace the placeholders secrets

To secure your Forgejo instancen, replace the `CHANGE_ME` secrets with real secrets:

Generate and replace `INTERNAL_TOKEN`:
```shell
doas sed -i '' 's/^INTERNAL_TOKEN.*/INTERNAL_TOKEN = '`forgejo generate secret INTERNAL_TOKEN`'/' /usr/local/etc/forgejo/conf/app.ini
```

Generate and replace `JWT_SECRE`:
```shell
doas sed -i '' 's/^JWT_SECRET.*/JWT_SECRET = '`forgejo generate secret JWT_SECRET`'/' /usr/local/etc/forgejo/conf/app.ini
```

Generate and replace `SECRET_KEY`:
```shell
doas sed -i '' 's/^SECRET_KEY.*/SECRET_KEY     = '`forgejo generate secret SECRET_KEY`'/' /usr/local/etc/forgejo/conf/app.ini
```

## Optional: Replace "localhost" with the Hosts FQDN

Replace `localhost` in `DOMAIN` and `ROOT_URL` with your FreeBSD's hostname:

```shell
doas sed -i '' 's/^DOMAIN.*/DOMAIN        = '`hostname -f`'/' /usr/local/etc/forgejo/conf/app.ini
```

```shell
doas sed -i '' 's/^ROOT_URL.*/ROOT_URL      = http:\/\/'`hostname -f`':3000\//' /usr/local/etc/forgejo/conf/app.ini
```

## Optional: Force HTTP/S for git users

To disable SSH for git:
```shell
doas sed -i '' 's/^DISABLE_SSH.*/DISABLE_SSH   = true/' /usr/local/etc/forgejo/conf/app.ini
```

> You can also manually edit `/usr/local/etc/forgejo/conf/app.ini` and set custom values for `DOMAIN` and `ROOT_URL`.
> It is recommended that `ROOT_URL` points to the IP address or FQDN of Forgejo, especially if it is behind a reverse proxy.
> For example, if put Forgejo behind a reverse proxy and plan to access it via `https://forgejo.internal.examplen.net`, set `ROOT_URL` to `forgejo.internal.examplen.net`.

# Enable and start the Forgejo service

Enable the forgejo service:
```shell
doas service forgejo enable
```

Start the forgejo service:
```shell
doas service forgejo start
```

Check if the forgejo service is running:
```shell
doas service forgejo status
```
> You should get something like `forgejo is running as pid 41964.` as a response.
> If not, check if `app.ini` contains any errors.

# Login to your Forgejo instance

Open the address of your Forgejo instance in a browser, e.g.: `http://10.1.70.102:3000/` or `http://forgejo.internal.example.net:3000/`
> Ensure that you can access port TCP 3000 on your instance and that it is not being blocked by a firewall.
> If you want to make this instance publicly available, it is strongly recommended that you regularly install updates and put it behind a reverse proxy with a valid TLS certificate.: https://forgejo.org/docs/latest/admin/reverse-proxy/
