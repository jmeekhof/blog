---
title: Custom DNS with Tailscale
date: 2026-08-12
categories: [tailscale, dns, caddy, route53, podman]
tags: tailscale dns caddy route53 letsencrypt dnsscale
author: jmeekhof
pin: true
---

## Introduction

I'm still on my Tailscale kick, and my server is still running everything I
care about: music, photos, recipes, game servers, and a password manager.
Each service runs in a rootless Podman container with a Tailscale sidecar,
managed by systemd and [quadlets][quadlets].

That setup grew a problem I didn't notice until it started biting me: every
service had two names.
This is the story of how I restructured my DNS so that every service has
exactly one name, on my own domain, while Tailscale keeps doing all the
heavy lifting underneath.

## Two names for everything

The Tailscale sidecar pattern gives each service its own tailnet node, so
each service gets a [MagicDNS][magicdns] name like
`navidrome.<TAILNET-NAME>.ts.net`, along with an automatically provisioned
[TLS certificate][ts-https] for it.
That part is great.

But I also own a domain, and at some point I started hand-creating DNS
records like `music.twotheleft.com` in Route53, pointing at my server's
tailnet address, with a Caddy instance on the host proxying to the right
service.
Some services were reachable both ways.
Some declared one name in their config while traffic arrived on the other.
My recipe app had `BASE_URL=https://mealie.twotheleft.com` while half its
visits came in as `mealie.<TAILNET-NAME>.ts.net`.
That mismatch is where broken redirects, mis-scoped cookies, and OIDC
`redirect_uri` errors come from.

Two ingress planes, drifting apart, each maintained by hand.
Time to pick one.

## The constraint that decides everything

I wanted the `twotheleft.com` names to win.
My domain, my names, and they'll outlive whatever mesh network I happen to
be running.

Here's the constraint that shapes the whole design: Tailscale will only
issue certificates for `*.ts.net` names.
If I want `music.twotheleft.com` to serve TLS, I need a real certificate
from Let's Encrypt, and since nothing on my network is reachable from the
internet, that means the [DNS-01 challenge][dns-01].
The only real question is where that certificate lives.

Before I tell you what I did, two dead ends worth recording so nobody
(including future me) retries them:

- **CNAME the custom name to the ts.net name.**
  It cannot work.
  The browser connects with SNI `music.twotheleft.com`, and the sidecar's
  [tailscale serve][ts-serve] only holds a certificate for the `ts.net`
  name.
  The TLS handshake fails before any routing decision gets made.
  This is the obvious-looking answer and it is a trap.
- **Tailscale split DNS with a custom domain.**
  Tailscale can happily resolve a custom domain for your tailnet, but it
  still won't issue certificates for it, so the actual problem is untouched.

## The design

What I landed on:

- One wildcard DNS record, `*.twotheleft.com`, pointing at my server's
  tailnet address.
- One wildcard certificate on the system Caddy, issued via DNS-01 with the
  [caddy-dns/route53][caddy-route53] plugin.
- Every HTTP service is one Caddy stanza, proxying to the container over
  loopback.
- The `ts.net` names get demoted to plumbing: transport,
  service-to-service calls, and node identity.
  Humans type `twotheleft.com`; machines keep talking `ts.net`.

Adding a service used to mean a DNS record, a certificate, and an ACME
round trip.
Now it's a stanza like this in the Caddyfile:

{% highlight text %}
@wallabag host wallabag.twotheleft.com
handle @wallabag {
	reverse_proxy 127.0.0.1:8060
}
{% endhighlight %}

A wildcard has a second benefit I didn't appreciate at first: it
enumerates nothing.
Per-service records in a public zone are a published inventory of
everything I run.
One wildcard says nothing about what's behind it.

The exceptions prove the rule.
Game servers speak raw UDP and can't be reverse-proxied, so they get real
per-node records pointing at their own sidecars.
And my identity provider stays on its `ts.net` name permanently, because
its OIDC issuer is derived from its tailnet hostname and its notion of
"who are you" comes from the Tailscale connection itself — put a proxy in
front and everyone becomes the proxy.

## dnsscale: the custom DNS half

Hand-maintaining Route53 records was the original sin here, so the other
half of this project was automating the zone.
I'd been running [dnsscale][dnsscale-upstream], a small Go daemon that
watches the Tailscale API and writes a DNS record for each tagged node.
I ended up maintaining [a fork][dnsscale-fork] and sending some of it back
upstream, because production found the sharp edges:

- **Removed nodes leaked records forever.**
  The record-listing call filtered to A/AAAA records, but the delete path
  matched on the TXT ownership marker it could never see.
  Nothing was ever deleted.
- **Cleanup needed to be safe before it could be on.**
  Pruning only touches A, AAAA, and TXT records carrying dnsscale's own
  ownership marker, so an MX or CAA record sharing a name survives.
  Mail records are exactly the kind of thing you don't notice are gone
  until delivery fails.
- **Only tagged nodes get published.**
  The zone is public, so an empty filter would put every personal laptop
  and phone on the tailnet into public DNS.
  Only nodes carrying `tag:dns` are managed.
- **Aliases and the wildcard.**
  The wildcard record is declared as an alias owned by the host node, so
  it tracks the host's address instead of pinning a literal IP.

The sneakiest fix was authentication.
dnsscale talked to the Tailscale API with an API key, and [Tailscale API
keys expire after 90 days][ts-api].
When mine expired, dnsscale didn't crash.
It kept polling, every request returned 401, and reconciliation silently
stopped — for months, as it turned out.
The fix was switching to an [OAuth client][ts-oauth], which doesn't
expire and gets exactly one scope: read devices.

While I was in there I stopped running it as a daemon at all.
It polled every 30 seconds — on a quiet day that was 2,879 API calls
producing zero DNS writes, for records with a 300-second TTL that can't
propagate observably faster anyway.
Now it runs as a oneshot from a [systemd timer][systemd-timer], hourly.

dnsscale.timer

{% highlight systemd %}
[Unit]
Description=Hourly dnsscale reconciliation pass

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
{% endhighlight %}

The real win wasn't the quiet log.
The whole-zone sweep — drift detection, orphan cleanup, the static
records — only ran at process startup, never on a poll tick.
As a daemon it effectively ran once per deploy.
On a timer it runs every hour.

## Three things that bit me

### My LAN lies about DNS

The wildcard certificate refused to issue for two days, and every
diagnostic said the DNS-01 challenge record was never created.
It was created.
Something on my network transparently intercepts all port-53 traffic — UDP
and TCP, even queries aimed directly at the zone's authoritative
nameservers — and answers from its own cache, which
[negative-caches][rfc2308] NXDOMAIN for about fifteen minutes.
I blamed my ISP first, but the call was coming from inside the house.
A nameserver that is a 9 ms round trip away was answering me in 1 ms,
flagged authoritative *and* offering recursion — a combination no real
authoritative server sends.
The only device that close is my UniFi Dream Machine, quietly answering on
every nameserver's behalf.

That produced a genuine observer effect: Caddy's ACME library polls the
challenge record to confirm it propagated before telling Let's Encrypt to
validate, and that poll *seeded* the negative cache that then convinced it
the record didn't exist.
Every `dig` I ran to debug it made it worse.

The fix is to disable the local propagation self-check and just wait:

{% highlight text %}
tls {
	dns route53
	propagation_delay 60s
	propagation_timeout -1
}
{% endhighlight %}

Let's Encrypt validates from outside my network against the real
authoritative servers, so it was never confused — only I was.
Issuance completed in 95 seconds.
The lesson: on this LAN, ground truth for DNS state is the Route53 API,
never a local query.

### The phantom writes

For a while, dnsscale logged successful record upserts for five
game-server names on every single pass, and none of them ever appeared in
Route53.
No errors, ever.
I went in expecting a swallowed API error and found something more
embarrassing: the log line announced nodes *before* the tag filter
discarded them.
The writes weren't failing — they were never attempted.
A reconciler that reports intent instead of outcome is a reconciler you
can't trust, and this class of bug is exactly how the 401 outage stayed
invisible too.

### The host that couldn't reach itself

The elegant version of this design has Caddy proxy to each service's
`ts.net` name, so a service could move to another machine with zero
config changes.
It didn't work, in the strangest way: from the server itself, connections
to its own sidecars timed out — while every other machine on the tailnet
reached them in 60 milliseconds, and [`tailscale ping`][disco-ping]
pinged them just fine.

Working ping plus dead connections looks exactly like a NAT problem, and
I burned real time on that theory.
It was the [ACL][ts-acl].
No grant in my tailnet policy allowed the *host's* tags to reach the
sidecars' tag — and ACLs are enforced by the receiving node's packet
filter, which sits *above* the disco layer that answers pings.
So the pings pong and the packets drop, silently.
When a connection times out but `tailscale ping` works, check the
receiving node's ACLs before you touch a single NAT theory.

I added the grant, verified it works, and then kept loopback upstreams at
the edge anyway — same box, why pay for two extra encryption hops per
request.

## What a wildcard changes about how you think

A few mental-model updates I'm still internalizing:

- **Deleting a record no longer makes a name stop resolving.**
  Under the wildcard, a deleted name resolves to the host and hits Caddy's
  fallback.
  "This name must not be served" is now a decision expressed in the
  Caddyfile, not in DNS.
- **Typos get valid TLS.**
  `mealei.twotheleft.com` resolves, completes a handshake with the
  wildcard cert, and 404s, instead of failing at NXDOMAIN.
- **Some resolvers hate CGNAT answers.**
  Tailnet addresses live in `100.64.0.0/10` ([RFC 6598][rfc6598] shared
  address space), and some routers' DNS-rebinding protection drops public
  DNS answers in that range.
  A device that resolves `ts.net` names fine but fails only on your custom
  domain has hit this — MagicDNS resolves locally and sidesteps it, which
  is the one durable advantage the `ts.net` plane keeps.
- **Renaming a service is a migration, not a config edit.**
  The scary one was my password manager: its configured domain is the
  [WebAuthn Relying Party ID][webauthn-rpid], so changing the name
  invalidates every passkey registered against the old one.
  That move got done last, deliberately, with re-enrollment planned.

## Conclusion

Every HTTP service now has exactly one user-facing name, on my own domain,
with one certificate and one DNS record for all of them.
Tailscale still carries every packet; it just stopped being the brand name
on the front door.
And the zone that was hand-maintained and silently stale for months is now
reconciled hourly by a oneshot that fails loudly.

One name per service.
It took a fork, an ADR, and three good debugging stories to get there, but
the config finally agrees with reality.

[quadlets]: https://podman-desktop.io/blog/podman-quadlet
[magicdns]: https://tailscale.com/kb/1081/magicdns
[ts-https]: https://tailscale.com/kb/1153/enabling-https
[ts-serve]: https://tailscale.com/kb/1312/serve
[ts-api]: https://tailscale.com/kb/1101/api
[ts-oauth]: https://tailscale.com/kb/1215/oauth-clients
[ts-acl]: https://tailscale.com/kb/1018/acls
[disco-ping]: https://tailscale.com/docs/reference/ping-types#disco
[dns-01]: https://letsencrypt.org/docs/challenge-types/#dns-01-challenge
[caddy-route53]: https://github.com/caddy-dns/route53
[dnsscale-upstream]: https://github.com/jaxxstorm/dnsscale
[dnsscale-fork]: https://github.com/jmeekhof/dnsscale
[systemd-timer]: https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html
[rfc2308]: https://datatracker.ietf.org/doc/html/rfc2308
[rfc6598]: https://datatracker.ietf.org/doc/html/rfc6598
[webauthn-rpid]: https://www.w3.org/TR/webauthn-2/#rp-id
