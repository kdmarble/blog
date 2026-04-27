---
title: "Self-Hosting in 2026: What's Actually Working"
date: 2026-04-26
draft: false
tags: ["self-hosting", "homelab", "mesh-networking", "security", "public-voice"]
---

I've been deep in this self-hosting thing for years, and the gap between "hobby project" and "daily driver" has finally collapsed for most of what regular people need. The tools are mature; the real shift was mesh networking. Once that landed, the parts that used to require a port-forwarding crash course just stopped requiring one.

Here's the lay of the land in 2026, starting with the part that gates everything else: access.

---

## Access

Every self-hosted setup hits the same wall first: how do you reach your services from outside your house? There's no single answer, but there's a clear spectrum, with convenience on one end and control on the other.

**Cloudflare Tunnels** are pretty nice if you only need to expose a few simple, clean services. The catch is that Cloudflare sits in the path of every byte, which means anything you push through their tunnel has to fit their terms of service. That's a hard no-go on streaming your less scrupulously acquired media. Performance is decent for blogs, static content, and anything that doesn't push against ToS edges.

**Tailscale** is the same convenience play with a little less corporate exposure. You're still relying on Tailscale's servers for management, but most people don't mind, and the product is genuinely polished. It was my first foray into mesh networking and it does its job cleanly. With a little forethought on network planning, you can onboard friends and family without installing a client on every single device. If the idea of someone else managing your network bothers you, you can run your own Headscale instance and use Tailscale clients while keeping the control plane in-house. For most people, Tailscale is a fine place to stop.

**Netbird** is where I landed. It's my current management plane and the one I've enjoyed the most. Pangolin and other reverse-proxy projects are gaining ground fast, but Netbird feels like the standard-bearer for open-source mesh networking right now. Entirely self-hosted, you control everything, and you still get the polish of a product built by an experienced dev team. If you're inclined to host your own stuff and go full bore into privacy-focused hosting, check it out.

The fair counter to all of this is: aren't you worried these companies will become future gatekeepers? Sure, partly. It's a sliding scale of convenience, control, and complexity, and the same axis carries risk. Audit yourself and why you're doing this. Just trying to save money and get away from big tech? Go for the more convenient, less complex side. You'll still land at significantly higher privacy standards than you started with, and you'll naturally pick up some skills along the way. Want to dig in and learn the stack? Go for the control side.

---

## Login with what?

The second problem after access is auth. People don't want a new login for every new service, and they especially don't want one to watch a movie at your house. Netbird's built-in reverse proxy and auth management is another reason I like it: the whole thing is a bit like a Lego set, with lots of ways to put it together and still end up at a correct answer. Social OAuth is a perfectly valid way to provision users in your own stack.

In practice, I only share one service with friends and family: Jellyfin. I run a split-horizon DNS setup, so my LAN can resolve all subdomains from my Caddy reverse proxy. AdGuard Home rewrites entries for my domain to my local Caddy IP. If you're off my LAN, you still hit everything when you hop on the Netbird VPN. Feels like you're at home. If you don't have the Netbird client on your phone and you're completely off-LAN, I still punch Jellyfin through the Netbird reverse proxy and set a password just to resolve through it. Just a shared phrase, like slipping the secret word past the tavern entryway. Past that, you still get shook down by ~~the bouncers~~ Jellyfin for the actual login we set up the one time you visited.

---

## "Easy mode" is the genuinely hard part

There's a real question buried in all of this: should there be a sane, opinionated starting point for self-hosting? There have been attempts — Unraid, HexOS, others — but opinionated all-in-ones still lock you into a vendor, especially if you're not technical enough to peek under the hood when something breaks (no shame). With some community effort, I think we could put together a great "step 1" Docker setup that gets people 80% of the way there without making them pick a platform up front. The hard part is striking the balance.

---

## The usual counterarguments

> People should just pay for services.

Nah, I'm sick of getting milked for every single dollar. Companies have made more than enough profit for far too long. They can handle living without my extra $60 a year.

> Normal users can't handle security.

I think we can get to a shared, easy, open-sourced tier-1 of self-hosted security. Mesh networking already collapsed the worst of it.

> Self-hosting media is legally messy.

Buy your stuff. Rip your stuff. Self-host your stuff. Re-sell the physical media if you really need the space back.

---

## What's actually running

Two Proxmox hosts run most of the LXCs and VMs: Gitea, Home Assistant, Vaultwarden, the arr stack, the usual suspects. A Docker host handles Jellyfin, SearXNG, Caddy as the reverse proxy with Cloudflare DNS-01 certs auto-renewing in the background, and AdGuard Home for DNS, ad blocking, and the split-horizon rewrites. A separate box for local LLM tinkering — qwen3.5-122B on a Strix Halo for chat, qwen3.6-35B on a 7900 XTX for the cron tier, all feeding my personal AI agent. Beszel watches the fleet, OPNsense sits at the edge, and Netbird's control plane lives on a small VPS so even that piece is mine.

The whole thing has been humming along for years without a single port forwarded to the public internet.

Firewalling is the part most tutorials skip right past. My setup: OPNsense at the edge, Gluetun on the Docker host pushing outbound through ProtonVPN, Netbird handling all inbound. No port forwards. If you *do* need to expose something, the right move is a ForwardAuth pattern at the proxy: pre-auth before any app sees a request, so a misbehaving app can't be the thing that gets your network owned. Netbird's built-in reverse proxy gives me that for free, with auth in one place instead of scattered across every app.

The nice bonus: I never touch firewall rules to add a new service. New container goes up, the mesh picks it up, done. Low-maintenance and boring is exactly what you want from infrastructure.

---

## Why I do this

I'm all-in on privacy-focused, local-first computing. Indie computing, if we need a name for it. I'm not selling anyone on it as ideology — the path isn't a tightrope, just a pretty easy stroll. Meet people where they are, and start with the friction that already bothers them.

I always start with myself as the primary user. This is my hobby, and I won't force it on the people I live with. I won't make my partner stop watching Netflix because a show is only there — I'll try to grab it and get it on Jellyfin first.

That's the test: the self-hosted version has to be as frictionless as the cloud one, or genuinely more useful. If it is, people switch on their own. If it isn't, you're just adding chores to their day in the name of your principles.

What are you self-hosting in 2026?
