---
layout: post
title: "How to fix Homebox 0.26.2 not starting on FreeBSD"
date: 2026-07-04
tags: FreeBSD Homebox Jail
---

# How to fix Homebox 0.26.2 not starting on FreeBSD

## Problem

After upgrading my Homebox jail yesterday from 0.25.2 to 0.26.2, the `homebox` service would not start anymore.
Taking a look into `/var/log/homebox.log`, I got presented with the following error message:
```
panic: auth.api_key_pepper must be set to at least 32 bytes; generate with `openssl rand -base64 48` and provide via HBOX_AUTH_API_KEY_PEPPER. Rotating it invalidates all issued API keys

goroutine 1 [running]:
main.main()
        github.com/sysadminsmedia/homebox/backend/app/api/main.go:106 +0x99
```

## Fix

To make `homebox` work again, it want's a secret key. Let's create it:
```shell
openssl rand -base64 48
```

Now, create a file to store the secret key inside of and restrict it's permissions:
```shell
touch /etc/rc.conf.d/homebox
```
```shell
chmod 600 /etc/rc.conf.d/homebox
```

Next, imput the following into the file, replacing "YOUR_SECRET_KEY" with the output of the `openssl` command.:
```shell
ee /etc/rc.conf.d/homebox
```
```shell
homebox_env="HBOX_AUTH_API_KEY_PEPPER=YOUR_SECRET_KEY"
```

Example:
```shell
homebox_env="HBOX_AUTH_API_KEY_PEPPER=Jsnme5TGMNNso6X3FHEZFvinx7VO5/E4imS35RLyDF5Dx1Ily90WCDFarc2LxkE7"
```
> This is only an example key, do not use it in production!

Now, after starting the `homebox` service, it should run again:
```shell
service homebox start
Starting homebox.
```
```shell
service homebox status
homebox is running as pid 65368.
```

Done. :)
