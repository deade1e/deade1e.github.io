---
title: Implementing "networking.tor" in NixOS
date: 2026-04-29
tags:
- nix
- tor
- firewall
---

## Introduction

During the years, I've been struggling to build the perfect onion routing
system. The idea of producing only noise for network observers has always been
fascinating. You know, surfing the web knowing no one knows who you are and
stuff. On top of that, I just [rewrote](https://github.com/deade1e/lor) the
entire Tor protocol back in 2015, so it might just be an obsession.

You could say, "bro, relax, no one is watching you." **Yes... but not
actually.** During my career, I had the opportunity to use some very costly
instrumentation that made me able to **track down threat actors to their home
IP addresses**. Such software is built on a massive, global-scale data
collection process.

**There are hundreds if not thousands of network companies that are selling
sampled packet metadata to other entities.** This makes it possible to know
which public IP contacts which other IP, and I am not speaking of government
intelligence or crazy stuff. These are commercial products, and I am not even
mentioning all the other privacy threats.

**They are selling my data for cheap, and I don't exactly know if I am more
worried or offended!**

## Nix + Nftables

As said above, I always struggled to implement a functional setup that worked
100% of the time. **This was undoubtedly due to my lack of understanding of
`iptables` at the time,** but also due to the intrinsic mutability of the OS.

Programs kept adding and removing iptables rules. It was hard to apply and
remove only your rules. And again, I am sure I wasn't skilled enough, but Nix
and nftables made this a joke instead.

In fact, speaking of nftables, **rules are organized in tables,** which are
basically namespaces, and chains. **You can apply those tables and then remove
them.** To decide which chain gets evaluated first, there is a priority system.
For a chain to be activated, there are "hooks."

I ended up simply being able to maintain a Tor routing
[project](https://github.com/deade1e/alpine-onion) based on Alpine.

All of this, together with Nix's declarability, made for an easy path towards
achieving a functional Tor client and router that lasted more than one week in
my hands (I treat Linux installations very poorly, hence my love for Nix).

## Implementation

Well, the whole thing is just a single file, probably fewer words than this
post's. It's a NixOS module that adds a set of options under the
`networking.tor` key. All of them can be found in the README.

It is divided into two macro areas: the `client` and the `router`. The first
makes sure to forward all of the packets generated on the current machine to
the Tor `TransProxy` functionality. The `router` does the same but only with
packets coming from other machines.

For each macro area there is a netfilter table, `tor` and `tor-router`
respectively, with two chains in it — one for `nat` and one for `filter`
purposes. NAT chains' purpose is to modify certain packet destinations; the
filter's is to drop unwanted ones.

There are a bunch of options to **exclude packets** destined to a certain
interface, subnet, mark, etc. These options made me able to fully run it daily
alongside my WireGuard VPN without any issues. On top of that, there is also an
optional `squid` declaration with `client.clearnet-proxy` that is allowed to
exit to clearnet for ease of use in everyday tasks.

**IMPORTANT:** By default, the implementation does not handle packets destined
to local subnets. The purpose of this project is not to create a 100% stealth
machine for maximum opsec with a V for Vendetta mask. To achieve that level,
you must also block subnets and do a bunch of other things I am not going to
cover here.

## Link

You can find the code here: https://github.com/deade1e/networking-tor.

It should be fairly easy to integrate into your setup and certainly easy to
audit.

## Conclusion

As previously stated, this project's objective is **not to provide a 100% 1337
operative setup** to hide your online activity, but rather to hide like 98% of
it without going through continuous issues when switching back for urgent
activities, like opening the bank website, or anything else where Tor is simply
blocked.

I feel like it's a small step forward for our privacy.

Hope you enjoy it.

## References
- https://github.com/deade1e/networking-tor
- https://github.com/deade1e/lor
- https://nixos.org
- https://www.torproject.org
- https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks
- https://github.com/deade1e/alpine-onion
