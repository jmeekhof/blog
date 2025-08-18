---
title: Welcome to my blog
---

# Introduction

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
I'm using systemd and [quadlets](https://podman-desktop.io/blog/podman-quadlet) to keep all of them running.
Using quadlets makes it easy to slide a Tailscale container into the mix for each of the services, and makes it easy to [share with Tailscale](https://tailscale.com/kb/1084/sharing).

## Music

Let's talk a bit about music.
I've taken my collection of digital music and stored it on my server.
It's mostly flac files, and I use [beets](https://beets.io) to keep the metadata maintained.
Not too long ago I discovered [Navidrome](https://www.navidrome.org/), and using it serve my music to my devices.

As I mentioned earlier, I've been on a Tailscale kick.
My Navidrome use case has been served securely to my phone with the help of Tailscale.

Recently I decide my home stereo was due for an upgrade.
After a bit of research I decided to purchase an [Eversolo DMP-A6 Gen 2](https://www.eversolo.com/Product/index/model/DMP-A6+Gen+2/target/GmOiCZnWE6feq7k9e%5Bld%5D3ulg%3D%3D.html)
I found it easy to connect it to my network and grant it access to my files via SMB.
What I really wanted was to be able to browse my music on my phone using [Symphonium](https://symfonium.app/) and play it on my new device.
My "just try it" attempts didn't work.
It was clear that DMP-A6 was trying to play something, but it wasn't working.

It's hardly a secret the Eversolo is using a heavily modified version of Android as their operating system.
It's also hardly a secret that you can [sideload applications](https://www.eversolo.com/Support/support_guide/guide_target/J7gTZpYWB63eq7k9e%5Bld%5D3ulg%3D%3D.html) onto these devices.

With this knowledge I installed the Android Tailscale client onto my DMP-A6, joined it to my tailnet, and tagged it.
Added a grant that allowed the `music-player` to communicate with the `music-server`, and it worked.
