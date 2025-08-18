---
title: Welcome to my blog
---

Anyone who knows me will tell you that I've been on a Tailscale kick lately.
I've been using it to connect all my devices and generally manage my network safely and securely.

Not too long ago I acquired a server.
I have a handful of things that I do with it.

- File serving
- Music hosting
- Game servers
- Photo backup and hosting
- Recipe storage

All (except for the file serving) of these services I run in containers using Podman.
I'm using systemd and (quadlets)[https://podman-desktop.io/blog/podman-quadlet] to keep all of them running.
Using quadlets makes it easy to slide a Tailscale container into the mix for each of the services, and makes it easy to share.
