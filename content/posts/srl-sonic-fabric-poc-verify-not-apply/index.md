---
title: "Verify is not apply: structuring fabric automation to avoid lying to yourself"
date: 2026-05-08
description: "Why running verification inside your apply loop produces noise instead of signal, why a separate verify pass is the right shape, and a few examples of bash gotchas that kept biting me along the way. Plus: an honest writeup of the one thing in this fabric that still doesn't work."
slug: srl-sonic-fabric-poc-verify-not-apply
tags:
  - automation
  - bash
  - networking
  - sr-linux
  - sonic
  - lab
---

This is the final part of a series on building a multi-vendor BGP fabric in containerlab. See the [introduction post](/posts/srl-sonic-fabric-poc-introduction/) for context, and the previous posts on the [SONiC platform.json contract](/posts/srl-sonic-fabric-poc-platform-json-contract/), [sonic-vs MAC collision](/posts/srl-sonic-fabric-poc-mac-collision-unnumbered/), and [SR Linux config push via gnmic](/posts/srl-sonic-fabric-poc-srl-gnmic-config/).

As part of my task to deploy config I had baked in tests that verify what we delivered. I was pretty tired by this point so I was not thinking so clearly when I bolted the test on the bottom of the config, ready to loop over each device in order. 

Every run produced this:

```text
================================================================
  az01-leaf1
================================================================
-- bgp sessions --
  0 / 3 peers Established
[FAIL] az01-leaf1
```

Followed by the same on leaf2, leaf3, leaf4. Then the spines started running and looked partially OK. By the time the cores ran, everything was actually converged, but my verify report said the leaves had failed.

The fabric was fine. The verify was lying.

## The structural problem

`apply.sh` was iterating per host: configure leaf1, verify leaf1, configure leaf2, verify leaf2, etc. The verify runs the moment leaf1's config is in place — but at that moment, **none of the spines have been configured yet**. So leaf1's BGP sessions can't establish, because there's nothing on the other end. The "0/3 peers Established" message is technically correct in that instant. It's also useless, because we know it'll be wrong after the spines come up.

The right structure is to separate the two concerns entirely:

- **`apply.sh`** pushes config to every host and exits. It doesn't verify anything beyond "the config landed" — i.e. the docker cp succeeded, supervisorctl restart succeeded, vtysh reload succeeded.
- **`verify_all.sh`** is a separate script you run after apply has finished. It checks fabric-wide state, ideally after BGP has had a few seconds to converge. It's idempotent — you can run it as many times as you want, watching state stabilise.

That's it. Once the two are separate, each becomes simple:

```bash
python3 render.py                            # produces rendered/<host>/{frr.conf, sonic_dataplane.sh, verify.sh}
./apply.sh clab-fabric                       # configure everything, exit
sleep 5                                      # let BGP do its thing
./verify_all.sh clab-fabric                  # report fabric-wide state
```

Apply also stages each host's verify.sh into `/tmp/verify.sh` inside the container, so you can also do a per-host check on demand:

```bash
docker exec clab-fabric-az01-sp1 bash /tmp/verify.sh
```

But the system-of-record for "is the fabric working" is `verify_all.sh`, not whatever apply happened to print on its way through.

## The verify itself

Per-host SONiC verify is a shell script that runs inside the container and checks:

1. zebra and bgpd processes running, `/etc/frr/daemons` has `bgpd=yes`
2. Loopback0 has the device's /32 attached (using `ip addr show`, not `ip route show` — see [part 2](/posts/srl-sonic-fabric-poc-mac-collision-unnumbered/) for why)
3. Each fabric interface is UP
4. Each fabric interface has a unique IPv6 link-local
5. BGP session count matches expected (== number of fabric links)
6. BGP RIB has at least N+1 prefixes (peers + own loopback)

For SRL, the verify runs externally via gnmic — same checks adapted to gNMI paths. It pulls the BGP state subtree once and counts established neighbors, total received routes per AFI, and so on.

`verify_all.sh` walks the rendered tree, dispatches each host to the appropriate path (in-container for SONiC, gnmic-from-outside for SRL), tallies pass/fail, and prints a summary at the end.

### `bash -x` strips `set -e`

I was debugging start.sh inside docker-sonic-vs (see [part 1](/posts/srl-sonic-fabric-poc-platform-json-contract/)) and ran `bash -x /usr/bin/start.sh` to trace where it failed. `bash -x` enables xtrace but doesn't preserve any flags the script set internally. So a script with `set -e` at the top, when run via `bash -x`, runs *without* `-e`, and a failure mid-script doesn't stop execution. The trace runs all the way to the end and you can't tell where the original failure was.

The fix: `bash -ex /usr/bin/start.sh` enables `-e` and `-x` from the outset. The script halts on its first non-zero exit, the trace shows you the last command that ran. Gold.

### `sudo: command not found`

Trivially obvious in retrospect: docker-sonic-vs runs as root inside, and the container image doesn't include `sudo`. If you've copy-pasted dataplane bring-up commands from real-SONiC tutorials, every `sudo config interface ...` will print "sudo: command not found" and fail silently. Drop the sudo. Container init runs as root, and so do your scripts when you `docker exec` into it.

## The thing that still doesn't work

I promised an honest writeup. Here it is.

The lab converges almost completely. Leaf/spine sessions all establish (12 sessions). Spine/core sessions all establish (9 sessions). Each leaf sees 10 prefixes in its BGP RIB (1 own + 3 spines + 6 cores via spines). Each spine sees a similar count.

What doesn't work: the **core-to-core mesh**. Each core has two physical links to the other two cores, and those links are configured into `dynamic-neighbors interface` on both ends, with the appropriate `allowed-peer-as`. Verify says each core has 3/5 peers established — the 3 spine sessions are up, the 2 core-mesh sessions aren't.

I'm publishing the lab as MVP and the writeup as-is because the rest is genuinely working and useful, but I want to be honest about the still-open question.

The Nokia docs on [dynamic-neighbors behaviour](https://documentation.nokia.com/srlinux/) say:

> transport passive-mode is always false. BGP always initiates a connection when informed by ICMPv6, unless it already has a connection.

So both ends of a core/core link should be sending RAs, both should be receiving each other's RAs, both should be initiating sessions. But they're not — there's no neighbour state at all on those interfaces in the gnmi response.

My current best hypothesis is that the ICMPv6 RAs aren't actually being sent on the inter-core subinterfaces, despite the config saying they should be. The config we render for spine-facing and core-facing links is identical apart from the interface number, so it's not a config rendering bug. The next debugging step would be `tcpdump -i ethernet-1/31 icmp6` on one core and seeing whether RAs are leaving / arriving.

Maybe SRL's RA logic has a "don't advertise on a link where I'm also listening for RAs" interlock. Maybe there's an admin-state default that needs poking. I don't know yet — and I'm publishing the writeup before I find out, because the work to get *here* is already worth sharing, even if I can't tell you the answer to the last question.

If you've hit the same thing, [let me know](https://github.com/fatred/srl-sonic-poc/issues).

## Takeaway

Two structural lessons from this part of the project:

**Apply does one thing, verify does another.** The temptation to interleave them comes from wanting fast feedback per host, but in any system where state on host A depends on state on host B, you'll get wrong answers from per-host verify-during-apply. Push everything, then check.

**Be honest about partial wins.** This lab works for what I built it to demonstrate (the SONiC + FRR + SR Linux interop story, the unnumbered eBGP plumbing, the gnmi config workflow). The fact that one specific session type isn't establishing doesn't invalidate the rest of it. Calling it done would be lying; calling it useless would be wrong; calling it MVP and writing up what's known and unknown is the version that respects both my time and yours.

Full source for everything covered in this series is at [github.com/fatred/srl-sonic-poc](https://github.com/fatred/srl-sonic-poc).

Thanks for reading.
