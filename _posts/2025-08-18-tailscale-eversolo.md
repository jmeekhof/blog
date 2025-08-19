---
title: Tailscale and Eversolo
date: 2025-08-18
categories: [tailscale, eversolo, android, music, navidrome]
tags: tailscale eversolo android music navidrome
author: jmeekhof
pin: true
---

## Introduction

Anyone who knows me will tell you that I've been on a Tailscale kick lately.
I've been using it to connect all my devices and generally manage my network
safely and securely.

Not too long ago I acquired a server.
I have a handful of things that I do with it.

- File serving
- Music hosting
- Game servers
- Photo backup and hosting
- Recipe storage

All (except for the file serving) of these services I run in containers using
Podman.
I'm using systemd and [quadlets][quadlets] to keep all of them running.
Using quadlets makes it easy to slide a Tailscale container into the mix for
each of the services, and makes it easy to [share with
Tailscale][tailscale-share].

## Music

Let's talk a bit about music.
I've taken my collection of digital music and stored it on my server.
It's mostly flac files, and I use [beets][beets-io] to keep the metadata
maintained.
Not too long ago I discovered [Navidrome][navidrome], and I am using it serve
my music to my devices.

### Navidrome

Navidrome is a really nice piece of software.
I like that it makes my music searchable, and supports live encoding to mp3 or
aac for devices that don't support flac.

When I'm mobile, I use symfonium to browse my music and play it on my phone.
Thanks to Navidrome and Tailscale it's easy to do this securely.

To accomplish this I create a pod for navidrome and run navidrome, and caddy
(with the tailscale plugin) in the pod.

navidrome.pod

{% highlight systemd %}
[Pod]
PodName=navidrome
PublishPort=4533:4533

{% endhighlight %}

navidrome.container

{% highlight systemd %}
[Unit]
Description=Navidrome Music Service
After=network-online.target

[Container]
Image=docker.io/deluan/navidrome:0.56.1
Pod=navidrome.pod
Mount=type=bind,source=/zfs/tank/music,destination=/music,ro=true
SecurityLabelDisable=true
Volume=navidrome.volume:/data
EnvironmentFile=%h/navidrome.env
LogDriver=json-file
LogOpt=path=%h/navidrome.log
LogOpt=max-size=512mb

[Service]
Restart=always

[Install]
WantedBy=default.target
{% endhighlight %}

caddy.container
{% highlight systemd %}
[Unit]
Description=Caddy Navidrome Proxy
After=navidrome.container
Requires=navidrome.container

[Container]
Image=ghcr.io/tailscale/caddy-tailscale:main
Pod=navidrome.pod
Volume=caddy_config.volume:/config
Volume=caddy_data.volume:/data
EnvironmentFile=%h/caddy.env
SecurityLabelDisable=true
Mount=type=bind,source=%h/conf,destination=/etc/caddy
LogDriver=json-file
LogOpt=path=%h/ts-caddy.log
LogOpt=max-size=512mb

[Service]
Restart=always
{% endhighlight %}

With all this in place I can go to `https://navidrome.<TAILNET-NAME>.ts.net`
and listen to my music securely from anywhere.

### Eversolo DMP-A6

Life is easy until you upgrade your stereo.

I had been a user of Sonos for a long time, but my experience with them has
been less than stellar.
I decided to shop around for a new device that would
support higher bit-rates, and a better user experience.

When I received my [Eversolo DMP-A6][dmp-a6], I plugged it into my network and
my stereo.
I connected it to my SMB share and was able to see my music files.
The remote app worked well, and I was able to play music from my server.

What I really wanted was to use Symfonium as my remote app, and play music on
my DMP-A6.
Theoretically this was possible.
In fact, Symfonium gave me the option to select my DMP-A6 as a playback device.
This wasn't going to work though.
My app is connecting to my Navidrome server over Tailscale, but the DMP-A6 was
connecting to the server over my local network.
Any hand-off would fail because the DMP-A6 didn't know how to resolve Tailscale
DNS names.

It's hardly a secret the Eversolo is using a heavily modified version of
Android as their operating system.
It's also hardly a secret that you can [sideload applications][sideload] onto
these devices too.

With this knowledge I installed the Android Tailscale client onto my DMP-A6,
joined it to my tailnet, and tagged it.
Added a grant that allowed the `music-player` to communicate with the
`music-server`, and it worked.

[quadlets]: https://podman-desktop.io/blog/podman-quadlet
[sideload]: https://www.eversolo.com/Support/support_guide/guide_target/J7gTZpYWB63eq7k9e%5Bld%5D3ulg%3D%3D.html
[tailscale-share]: https://tailscale.com/kb/1084/sharing
[dmp-a6]: https://www.eversolo.com/Product/product_detail/J7gTZpYWB63eq7k9e%5Bld%5D3ulg%3D%3D.html
[navidrome]: https://www.navidrome.org/
[beets-io]: https://beets.io/
