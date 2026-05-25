---
layout: post
title: "Manually upgrade Plex on FreeBSD"
date: 2026-05-25
tags: FreeBSD Plex Jail
---

# Manually upgrade Plex Media Server on FreeBSD

## Context

Sometimes I want or have to upgrade my Plex Media Server Jail, but the latest release isn't yet available via pkg.
In such cases, it is entirely possible to manually upgrade Plex via the .tar.bz2 archive.

---

## Steps

First, visit the Plex Media Server Download page: https://www.plex.tv/media-server-downloads/?cat=computer&plat=freebsd
Next, copy the URL from the download button.

Now you can login to your FreeBSD host or jail via SSH and download the latest .bz2 archive.
Replace the example URL with the one you copied.
```shell
fetch https://downloads.plex.tv/plex-media-server-new/1.43.0.10492-121068a07/freebsd/PlexMediaServer-1.43.0.10492-121068a07-FreeBSD-amd64.tar.bz2
```

Stop the plexmediaserver service:
```shell
doas service plexmediaserver_plexpass stop
```

Extract the .bz2 archive:
```shell
doas bzip2 -d PlexMediaServer-1.43.0.10492-121068a07-FreeBSD-amd64.tar.bz2
```

Extract the .tar archive over the existing files in /usr/local/share/:  
```shell
doas tar --strip-components=1 -xvf PlexMediaServer-1.43.0.10492-121068a07-FreeBSD-amd64.tar --directory /usr/local/share/plexmediaserver-plexpass/
```

After all the files have been extracted, start your Plex server again:
```shell
doas service plexmediaserver_plexpass start
```

Now you can remove the .tar archive.
```shell
rm PlexMediaServer-1.43.2.10687-563d026ea-FreeBSD-amd64.tar
```
---

## Conclusion

If you don't have the Plex Pass version, just remove `-plexpass` from the directory and `_plexpass` from the service name.
When a new version is available via pkg, you can use that instead, since it basically does the same thing.
In rate cases, Plex might get stuck when starting it, in which case just restart the service once to make it work again.

---

## Credits

The steps in this post have been inspired by this Plex forum thread:
https://forums.plex.tv/t/upgrade-plex-on-freebsd-using-tar-bz2-official-download-packages/880488
