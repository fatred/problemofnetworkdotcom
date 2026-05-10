---
title: "Why every sonic-vs port has the same MAC, and what that means for BGP unnumbered"
date: 2026-05-08
description: "If you run docker-sonic-vs and try to set up BGP unnumbered between containers, you'll watch most sessions sit in Idle forever. The cause is buried in how the container assigns MAC addresses to its Ethernet ports. Here's the failure mode and the fix."
slug: srl-sonic-fabric-poc-mac-collision-unnumbered
tags:
  - sonic
  - frr
  - bgp
  - ipv6
  - networking
  - lab
---

This is part 2 of a series on building a multi-vendor BGP fabric in containerlab. See the [introduction post](/posts/srl-sonic-fabric-poc-introduction/) for context.

After [getting the platform.json/hwsku.json contract right](/posts/srl-sonic-fabric-poc-platform-json-contract/) and bringing up SONiC cleanly with the Accton hwskus, my next problem was that BGP unnumbered just didn't work. The interfaces were UP, FRR was running, the config looked right, but most sessions were stuck in `Idle` forever.

This is the writeup of what I learned about how `docker-sonic-vs` handles MAC addresses on its Ethernet ports, why that's a problem for BGP unnumbered specifically, and what to do about it.

## The setup

Standard FRR unnumbered: each fabric port gets `interface EthernetX` / `no ipv6 nd suppress-ra` / `ipv6 nd ra-interval 5`, and the BGP config has `neighbor PEER_GROUP peer-group` with `neighbor EthernetX interface peer-group PEER_GROUP`. FRR sends ICMPv6 router advertisements; the peer learns the local link-local address; both sides initiate sessions to each other's link-local. No IP per link, no subnet allocation. Beautiful when it works.

What I saw instead, on every sonic-vs node:

```text
Ethernet48       DOWN           e2:d8:ff:8e:15:43 <BROADCAST,MULTICAST>
Ethernet52       DOWN           e2:d8:ff:8e:15:43 <BROADCAST,MULTICAST>
Ethernet56       DOWN           e2:d8:ff:8e:15:43 <BROADCAST,MULTICAST>
```

Three fabric ports, three identical MACs. Every Ethernet port on the box gets the container's `eth0` MAC. The `<LOWER_UP>` flag is missing — they aren't even oper-up yet, but the MAC issue is already visible.

## Why same-MAC kills unnumbered

IPv6 link-local addresses, by default, are EUI-64 derived from the interface's MAC. Same MAC means same link-local. So once we bring the ports up:

```text
Ethernet48  fe80::e0d8:ffff:fe8e:1543
Ethernet52  fe80::e0d8:ffff:fe8e:1543
Ethernet56  fe80::e0d8:ffff:fe8e:1543
```

The kernel will happily assign duplicate link-locals on different interfaces (they're scoped to the interface index, not the address itself). But BGP unnumbered uses the link-local of the *peer's* interface as the neighbor address. When a router advertisement arrives saying "I'm `fe80::e0d8:ffff:fe8e:1543`", FRR's session lookup keys on the address — and there's already a session with that address bound to a different interface. Most sessions either never come up, or one comes up and the others get stuck in `Active`/`OpenSent`.

This isn't a SONiC bug per se. It's an emergent property of running SONiC inside a container that started life with a single `eth0` MAC, combined with an init flow that doesn't bother stamping unique MACs on the dataplane ports because real hardware doesn't have this problem.

## The fix

Stamp a unique, locally-administered MAC on each fabric port during dataplane bring-up. Locally-administered MACs have the second-least-significant bit of the first octet set (`0x02`), which avoids collisions with vendor-assigned MACs. The rest of the bytes can encode whatever you find useful — I use the device's ASN modulo 256 and the per-port index so the MAC tells me what device and which port it belongs to:

```bash
# inside the container, during dataplane bring-up:
new_mac="$(printf '02:00:%02x:%02x:%02x:%02x' \
    $((ASN % 256)) \
    $((INSTANCE % 256)) \
    $((PORT_IDX / 256)) \
    $((PORT_IDX % 256)))"

ip link set dev "$port" down
ip link set dev "$port" address "$new_mac"
sysctl -q -w net.ipv6.conf."$port".disable_ipv6=0
ip link set dev "$port" addrgenmode eui64
ip link set dev "$port" up
```

The `addrgenmode eui64` line is important — if the kernel had already generated a link-local under the old MAC, just changing the MAC won't refresh it. Setting `addrgenmode` flushes and regenerates. The down/up bookend makes sure both the address change and the regeneration apply cleanly.

After this, the picture looks right:

```text
Ethernet48  fe80::b1ff:fe01:0
Ethernet52  fe80::b1ff:fe01:1
Ethernet56  fe80::b1ff:fe01:2
```

Three different link-locals, BGP can tell them apart, all sessions establish.

## A couple of small SONiC-specific gotchas

While I was in the dataplane bring-up code, two other things tripped me up that are worth mentioning.

**`sudo` doesn't exist in the container.** docker-sonic-vs runs as root inside, and there's no sudo binary. If you've got dataplane bring-up scripts that work outside the container with `sudo config interface startup EthernetX`, drop the sudo. Similarly, you can mostly skip `config` CLI commands for what we're doing here — `ip link set ... up` does the same thing, faster, with no Python startup tax. The one place where `config` is the right call is creating Loopback0 and giving it the device /32, because that needs to land in CONFIG_DB so it survives a reload. For anything ephemeral or that we'll re-apply on next render, the kernel is fine.

**`docker_routing_config_mode` defaults to `unified`.** SONiC has two modes for how it manages FRR: `unified` (bgpcfgd reads CONFIG_DB and writes frr.conf, overwriting yours) and `split` (FRR is managed independently). For lab work where you're pushing your own frr.conf, you want `split`. Set it once in CONFIG_DB:

```bash
redis-cli -n 4 hset 'DEVICE_METADATA|localhost' docker_routing_config_mode split
```

You also need to flip `bgpd=yes` in `/etc/frr/daemons` — sonic-vs ships with `bgpd=no` by default. `supervisorctl restart bgpd` after.

**Loopback0's address shows up where you don't expect.** When you assign `10.0.12.1/32` to Loopback0 with `ip addr add`, the kernel installs a `local` route in table 255, not a regular route in the main table. So if your verify script does `ip route show 10.0.12.1/32` looking for the loopback, it'll come back empty even though everything's working. Use `ip addr show dev Loopback0` to check the address is attached, or `ip route show table local` if you specifically want the synthetic local route. I burned an hour on this one writing a "loopback missing in kernel" check that was actually checking the wrong thing.

## Why you don't see this on real SONiC

Real SONiC running on real hardware has each port backed by an ASIC port whose MAC comes from the device's MAC pool — typically derived from a base MAC plus a per-port offset. The Accton platform's `platform.json` would have a `mac` field per interface, and orchagent/syncd would program the ASIC accordingly.

In sonic-vs, the dataplane is veth pairs with kernel-assigned MACs, and nobody's bothered to teach syncd to stamp unique ones. It's not wrong — sonic-vs is an integration testing harness, not a production lab tool — but it does mean any feature that depends on per-interface uniqueness needs a workaround.

## Takeaway

If you're building anything in `docker-sonic-vs` that depends on each port having a distinct identity in the L2/L3 plane — not just BGP unnumbered, but anything with link-local addressing, IPv6 ND, ARP-with-multiple-uplinks, etc. — assume the ports all have the same MAC and stamp your own. Five lines of bash in your bring-up script gets you there.

Full source for the dataplane bring-up template (`templates/sonic_dataplane.sh.j2`) is at [github.com/fatred/srl-sonic-poc](https://github.com/fatred/srl-sonic-poc).

Next post: [pushing config into SR Linux with gnmic from a YAML inventory](/posts/srl-sonic-fabric-poc-srl-gnmic-config/).
