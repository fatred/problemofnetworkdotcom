---
title: "Building a SONiC + SR Linux fabric in containerlab: an introduction"
date: 2026-05-08
description: "A four-part series unpacking what it actually takes to stand up a multi-vendor BGP unnumbered fabric — FRR on SONiC leaves and spines, SR Linux cores, all glued together with containerlab. This first post sets the scene; the rest are deep dives into the things that needed fixing to get to MVP"
slug: srl-sonic-fabric-poc-introduction
tags:
  - sonic
  - sr-linux
  - containerlab
  - bgp
  - frr
  - networking
  - lab
---

As growth at my employer remains up and to the right, we have seen three doublings of our footprint in 5 years, going from ca. 20 cabs, to ca. 50 cabs and now well over 100. Moving from Juniper to Nokia was a big part of the success story, where we increased our reliability, made automation more or less the only change approach, and saved over 40% on our capital spend in the process. Its been a huge success for us. 

Looking forward, if we assume another three doublings (ambitious but not unreasonable on current form), this means we should expect to see 200/400/800 cabs before the end of 2030, and so we need to start thinking now what our next unit of scale is. I know that [Nokia SR Linux](https://www.nokia.com/networks/ip-networks/service-router-linux-NOS/) can support us in this mission, but when it comes to this cloudscale, it would be negligent not to spend some time with [SONiC](https://sonicfoundation.dev/). Enough ink has been spilled on what SONiC is and how it works, so I will not spend too much time on this. This mini series is about evaluating the product and the tooling for use beyond the 50 rack scale. We are limiting ourselves to the Community Edition here. If I am paying for it, I may as well stick to Nokia which is a much more refined product for sure.

One thing that I have observed and been advised on is that MLAG/ESI is kinda trash in SONiC land. Given the grounding in the Cloudscalers where they don't choose that kind of bonding often anyways, that should not be a surprise either. As a sort of hint towards the scale we know we have to handle, and as a way to avoid the limitations of scaling L2 bridge domains in use, we have decided to deprecate L2 VLANs for endpoints as a major breaking change in this iteration of our design. 

So to validate our assumptions, here we are in this blog series. Our aim is to build a small enough lab topology that we can run it locally on a laptop, and with enough equipment to be representatve of the network blocks we need to verify with. Specifically: a small leaf-spine-core topology where the leaves and spines run SONiC, the cores run SRLinux, all wired up in [containerlab](https://containerlab.dev/), and the BGP underlay uses unnumbered eBGP everywhere and the overlays are also using IPv6 link-local to reach the connected systems. The RFC 8950 dance then lets you skip bonding without also requiring per-link IP allocation. Endpoints share their v4 and v6 routable assignments over BGP to route reflectors and so all reachability is L3 based.

That sentence took me two 5 hour journeys on the TGV to make true.

This is the index post for a four-part series unpacking the build. The lab is **MVP-quality**: the leaf/spine plane is rock solid, the core gets BGP from the spines, but the core/core mesh isn't fully working yet (more on that in part four). I'm publishing anyway because each individual problem I hit along the way is worth its own post — those problems are the real artefact of the project, not the eventual converged fabric.

The full code is at [github.com/fatred/srl-sonic-poc](https://github.com/fatred/srl-sonic-poc) if you want to follow along or skip straight to a working starting point.

## Topology

Four leaves, three spines, three cores. ASN scheme is a 10-digit `420<SS><AA><T><II>` that makes tracing over the underlay GRT derivatibe: site, AZ, tier, instance is all there to see.

```text
      cr1 ──── cr2 ──── cr3
       │\      │\      /│
       │ \     │ \    / │       core mesh + 3×3 core↔spine
       │  \    │  \  /  │
      sp1     sp2     sp3
       │┼┼┼   │┼┼┼   │┼┼┼       leaves dual/triple-attach
      leaf1  leaf2  leaf3  leaf4
```

Leaves are SONiC running on the Accton AS7326-56X hwsku (48×25G + 8×100G). Spines are SONiC on AS7726-32X (32×100G). Cores are SR Linux 25.10.2 running as IXR 7220 D5 (32x400G). All running as containers under containerlab on a single Linux host.

The whole fabric is generated from a single `topology.clab.yaml`, an inventory of defaults, and one render pass per side (SONiC and SRL) that produces per-host config artefacts. Apply scripts push them into the running lab. Verify scripts read state back and tell you what's broken.

## Why unnumbered?

For a lab this small you could pick any addressing model and it'd work. I chose unnumbered because:

- **It's how real hyperscale fabrics are built**, mostly because the operational simplicity wins out at scale. Worth getting comfortable with.
- **No subnet bookkeeping per fabric link.** With 19 fabric links just for this test topology, manual subnet allocation is the kind of operational tax I want to design away.
- **Avoids multiprotocol routing.** I am a firm believer in the value of IS IS in underlays for low effort, high stability loopback reachability. Surveying a subset of SONiC operators, none of them use it for underlays, because there are concerns about the extra complexity it imposes on FRR. eBGP is the preference of the community, so here today I follow their lead. I would like to do more testing in this in the future however.

## What broke

Roughly in the order I hit them:

1. **The SONiC `platform.json` / `hwsku.json` contract.** docker-sonic-vs ships a generic platform.json with only `default_brkout_mode` and no `lanes`/`index`/`breakout_modes`. The Accton hwskus don't ship a hwsku.json at all. Both files are required by `sonic-cfggen`. Generating valid platform.json + hwsku.json for each hwsku turned out to require reading SONiC's `portconfig.py` source to understand the exact contract. **Part 1.**

2. **All ports got the same MAC.** sonic-vs assigns the container's eth0 MAC to every Ethernet port. With unnumbered BGP relying on each side learning the peer's IPv6 link-local from RAs, identical MACs mean identical link-locals mean BGP can't tell them apart and most sessions never come up. The fix is per-port locally-administered MACs stamped during dataplane bring-up. **Part 2.**

3. **SR Linux gNMI configuration via `gnmic`**, including the `Routing` non-default network-instance gotcha (you can't use `system0` outside the default NI), the JSON shape that gnmic returns and how to parse it without going insane, and the `extended-next-hop-encoding` knob that has to be on for v4-over-v6-LL exchange to actually work. **Part 3.**

4. **My verification-during-apply was racy.** I'd built per-host verify into apply.sh and watched leaf1 fail because sp1 hadn't been configured yet. Restructuring apply (push only) and verify (separate pass after convergence) made the whole thing legible. Plus the final state of the SRL core mesh — what's working, what isn't, and why the docs disagree with reality on dynamic-neighbour active/passive behaviour. **Part 4.**

## Reading order

You can read these in any order; they cross-reference but don't depend on each other.

- [Part 1 — Platform.json and hwsku.json: the contract docker-sonic-vs forgot to mention](/posts/srl-sonic-fabric-poc-platform-json-contract/)
- [Part 2 — Why every sonic-vs port has the same MAC, and what that means for BGP unnumbered](/posts/srl-sonic-fabric-poc-mac-collision-unnumbered)
- [Part 3 — Pushing config into SR Linux with gnmic from a YAML inventory](/posts/srl-sonic-fabric-poc-srl-gnmic-config/)
- [Part 4 — Verify is not apply: structuring fabric automation to avoid lying to yourself](/posts/srl-sonic-fabric-poc-verify-not-apply/)

## A note on the "MVP" label

This works as a learning artefact. The leaves and spines all converge clean — every fabric link comes up, every BGP session establishes, every loopback shows up in the right number of RIBs with correct ECMP. The cores accept the spine sessions and exchange routes correctly. **The core↔core mesh sessions don't establish.** I have a strong hypothesis (RAs not actually being sent on the inter-core subifs despite config saying they should be) but haven't run it down yet. Part 4 covers what I know and what I'd check next.

If you want a fabric to deploy in production, this isn't it. If you want to see how each layer of the stack actually behaves when prodded, read on.
