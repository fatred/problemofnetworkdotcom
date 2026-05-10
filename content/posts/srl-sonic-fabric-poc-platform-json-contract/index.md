---
title: "Platform.json and hwsku.json: the contract docker-sonic-vs forgot to mention"
date: 2026-05-08
description: "If you've ever tried to run docker-sonic-vs with an Accton hwsku and watched it silently fail to bring up your fabric ports, this is the post. A walk through SONiC's port-init contract, why the stock files in the container don't satisfy it, and what you actually need to provide."
slug: srl-sonic-fabric-poc-platform-json-contract
tags:
  - sonic
  - containerlab
  - networking
  - lab
---

This is part 1 of a series on building a multi-vendor BGP fabric in containerlab. See the [introduction post](/posts/srl-sonic-fabric-poc-introduction/) for context.

I wanted to use the real Accton hwskus in my lab — `Accton-AS7326-56X` for the leaves (48×25G + 8×100G) and `Accton-AS7726-32X` for the spines (32×100G) — because their port layouts match what I run in production. The default `Force10-S6000` only has 32 ports, so building a 4-leaf 3-spine fabric using the spines' high-numbered uplinks just doesn't fit.

Setting `HWSKU=Accton-AS7326-56X` in the clab node config seemed like it should be all you need. It isn't. The leaves wouldn't boot — every supervisord service stuck in `STOPPED`, no `Ethernet*` interfaces visible, no errors in any obvious place.

This post is the writeup of what's actually going on, distilled from a couple of days of source-diving.

## The symptom

```bash
docker exec clab-fabric-az01-leaf1 supervisorctl status | head
arp_update                       STOPPED   Not started
bgpd                             STOPPED   Not started
buffermgrd                       STOPPED   Not started
...
```

Nothing's running because nothing got past container init. The trick to find out why is to re-run the init script with stderr captured, but you have to be careful: `bash -x /usr/bin/start.sh` strips the original `set -e`, so the script blows past its first failure and the trace goes nowhere useful. You want `bash -ex`, which keeps `-e` *and* echoes commands:

```bash
docker exec clab-fabric-az01-leaf1 bash -ex /usr/bin/start.sh
```

That trace eventually pointed at `sonic-cfggen` complaining about a missing `hwsku.json` in `/usr/share/sonic/device/x86_64-kvm_x86_64-r0/Accton-AS7326-56X/`. Fine — I'll generate one. So I generated a minimal `hwsku.json` with one port per `Ethernet*` and a `default_brkout_mode` and bind-mounted it in.

That moved me to the next error:

```text
KeyError: 'lanes'
  File "/usr/local/lib/python3.11/dist-packages/sonic_py_common/portconfig.py", line 312
```

After more sniffing around: this isn't a hwsku.json problem. It's a `platform.json` problem. The two files have different jobs and both need to be valid.

## The contract

I had to read SONiC's [`portconfig.py`](https://github.com/sonic-net/sonic-buildimage/blob/master/src/sonic-py-common/sonic_py_common/portconfig.py) to see exactly what gets read at boot. Here's the chain:

1. Cfggen reads `hwsku.json`. For each `Ethernet*` key it pulls `default_brkout_mode` (e.g. `"1x25G"`).
2. Cfggen reads `platform.json`. For each interface it expects a dict like `{ "alias", "index", "lanes", "breakout_modes" }`.
3. For each port, cfggen calls `get_child_ports(intf, brkout_mode, platform_json_file)` which instantiates `BreakoutCfg(intf, brkout_mode, properties)` where `properties` is `platform.json["interfaces"][intf]`.
4. `BreakoutCfg.__init__` immediately reads `properties['lanes'].split(',')` — KeyError if missing.
5. It then walks `properties['breakout_modes']` looking for the mode name. If `default_brkout_mode` from hwsku.json doesn't match a key in the platform.json's `breakout_modes` dict, the breakout fails.

**The two files are a contract.** hwsku.json says "I want to use breakout mode X on port Y." platform.json says "here are the lanes/indices/aliases for port Y, and here are the modes the platform supports — X is one of them, and these are the resulting child port aliases."

The stock `platform.json` shipped in the docker-sonic-vs image has only `default_brkout_mode` and nothing else. The Accton hwskus don't ship a `hwsku.json` at all. Both gaps need to be filled.

## What a minimal valid pair looks like

For a single 25G port on the AS7326, you need entries like this.

In **platform.json**:

```json
{
  "interfaces": {
    "Ethernet0": {
      "alias": "twentyfiveGigE1",
      "index": "1",
      "lanes": "3",
      "breakout_modes": {
        "1x25G": ["twentyfiveGigE1"]
      }
    }
  }
}
```

In **hwsku.json**:

```json
{
  "interfaces": {
    "Ethernet0": {
      "default_brkout_mode": "1x25G"
    }
  }
}
```

The mode string in hwsku.json (`"1x25G"`) appears as a key in platform.json's `breakout_modes`. The list value (`["twentyfiveGigE1"]`) gives the child port alias names. For a 1xN mode that's just the parent's alias, but it generalises to e.g. `"4x10G": ["tenGigE1", "tenGigE2", "tenGigE3", "tenGigE4"]`.

The other thing worth knowing: **breakout mode strings are strict.** I tried `"1x25G[10G]"` (with a bracketed fallback rate) early on — the regex in BreakoutCfg explicitly rejects bracketed fallback for single-lane modes. Plain `"1x25G"` works.

## Where I got the lane mappings from

The hwsku ships in the container at `/usr/share/sonic/device/x86_64-kvm_x86_64-r0/Force10-S6000/`. Inside there are two files describing the platform's port wiring:

- `lanemap.ini.orig` — `<port-index> -> <comma-separated lane numbers>`
- `port_config.ini.orig` — `<eth-name> <lane-string> <alias> <index> <speed>`

The two join on the lane string. AS7326 has non-sequential lane numbering, which trips you up if you assume `eth49 .. lane (49-1)*4 = 192`. It's not. eth49 maps to lane 64 (Ethernet48 in SONiC-speak), eth50 to lane 68, eth51 to lane 72.

I wrote a small script that does the join and produces a unified YAML port table:

```yaml
- name: Ethernet48
  alias: hundredGigE49
  eth: eth49
  index: 49
  lanes: "64,65,66,67"
  speed: 100000
```

From that one yaml file, a builder generates both `platform.json` and `hwsku.json` with matching mode strings. Run it once per hwsku. Result: the per-node clab bind looks like

```yaml
binds:
  - port_maps/Accton-AS7326-56X.platform.json:/usr/share/sonic/device/x86_64-kvm_x86_64-r0/platform.json:ro
  - port_maps/Accton-AS7326-56X.hwsku.json:/usr/share/sonic/device/x86_64-kvm_x86_64-r0/Accton-AS7326-56X/hwsku.json:ro
```

Note that `platform.json` lives at the *platform* level (`x86_64-kvm_x86_64-r0/`), not the hwsku level, while `hwsku.json` lives inside the hwsku-specific directory. They're at different paths because they describe different things — the platform is "what hardware exists", the hwsku is "what configuration the operator chose". One platform can host many hwskus on real gear; in our virtualised case both are bound per-node.

## The readiness probe

Once the platform/hwsku files are right, start.sh runs to completion and the rest of SONiC comes up. But there's one more gotcha when automating around it: **how do you tell from outside the container that SONiC has finished booting?**

A naive approach is to poll for `redis-cli -n 4 hget 'DEVICE_METADATA|localhost' hostname`. Don't. The `hostname` field isn't populated by every hwsku init template — the Accton ones leave it empty. I learned this when my apply scripts timed out waiting for a key that was never going to appear. The reliable signal is `hwsku`:

```bash
got=$(docker exec "$c" redis-cli -n 4 hget 'DEVICE_METADATA|localhost' hwsku)
[ -n "$got" ] && echo "ready, hwsku=$got"
```

That value gets set during `configdb-load.sh`, which is also when the rest of the stack is reliably up. It's a much better readiness signal.

## Takeaway

The `platform.json`/`hwsku.json` separation is genuinely useful for thinking about what a SONiC box is — the platform is the silicon and ports, the hwsku is the operator's chosen config — but the docker-sonic-vs image ships incomplete versions of both for the Accton hwskus. You can't fix it by just creating one or the other; you need both, with matching mode strings, mounted at the right paths.

Once they're right, every other layer in the stack just works. Until then, nothing does.

Full source for the port_maps generator and the rest of the fabric is at [github.com/fatred/srl-sonic-poc](https://github.com/fatred/srl-sonic-poc).

Next post: [why every sonic-vs port has the same MAC, and what that means for BGP unnumbered](/posts/srl-sonic-fabric-poc-mac-collision-unnumbered/).
