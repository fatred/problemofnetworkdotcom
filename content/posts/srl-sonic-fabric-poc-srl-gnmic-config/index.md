---
title: "Pushing config into SR Linux with gnmic from a YAML inventory"
date: 2026-05-08
description: "Building per-host SR Linux configs from a shared topology and inventory, packaging them as a single root-level JSON blob, and pushing them with one atomic gnmic set call. Also: why you can't put your loopback on system0 if you're not using the default network-instance, and how to parse the variable-keyed JSON that gnmic returns."
slug: srl-sonic-fabric-poc-srl-gnmic-config
tags: 
  - sr-linux
  - gnmic
  - gnmi
  - bgp
  - automation
  - networking
  - lab
---

This is part 3 of a series on building a multi-vendor BGP fabric in containerlab. See the [introduction post](/posts/srl-sonic-fabric-poc-introduction/) for context.

The SONiC side of my fabric was working — leaves and spines all converged with [the platform.json fix](/posts/srl-sonic-fabric-poc-platform-json-contract/) and [the per-port MAC stamping](/posts/srl-sonic-fabric-poc-mac-collision-unnumbered/). Now I needed the Nokia SR Linux cores to peer with the spines, also using BGP unnumbered.

For the SRL side I wanted the same property the SONiC side had: a single source of truth for the topology, a render step that produces per-host config artefacts, an apply step that pushes them. The mechanism is different — SRL doesn't run FRR, it has its own native YANG-driven config — but the abstraction shape can be identical.

This post is about how I built that pipeline using `gnmic`.

## The shape of the artefact

For the SONiC side I render Jinja templates (`frr.conf.j2`, `sonic_dataplane.sh.j2`) into per-host text files. For SRL I considered the same approach — render a CLI script — and discarded it. When you have a "proper" data model and reliable tooling, that is a retrograde step.

`gnmic set --update-path / --update-file config.json` is much better. It reads a JSON file, sends one atomic gNMI Set RPC rooted at `/`, and SRL applies the whole thing as a single candidate-commit. Either it all succeeds or it all rolls back. Re-running is idempotent.

So the artefact per host is one JSON file. The template is a Python function that builds a dict and dumps it — no Jinja involved. Conditionals (mesh on/off, AFI scope) are easier to express as Python `if` than Jinja templating with whitespace control.

The render walks the same `topology.clab.yaml` the SONiC render uses. Anything with `kind: nokia_srlinux` gets a config produced; everything else is skipped:

```python
managed_kinds = set(defaults.get("managed_kinds", ["nokia_srlinux"]))
for hostname, node in sorted(topo["nodes"].items()):
    if node.get("kind") not in managed_kinds:
        continue
    facts = derive_facts(hostname, node, topo["links"], topo["nodes"], defaults)
    config = build_root_config(facts, defaults)
    (out_dir / "config.json").write_text(json.dumps(config, indent=2))
```

`derive_facts` resolves the device's ASN and loopback from its hostname (the same `420<SS><AA><T><II>` scheme the SONiC side uses), pulls the device's fabric links from the topology, and resolves each peer's ASN to build the `allowed-peer-as` list.

## What lives in config.json

The per-host config has three top-level sections:

```json
{
  "interface": [...],
  "network-instance": [...],
  "routing-policy": {...}
}
```

Interfaces include the loopback and one entry per fabric link. Network-instance has the BGP config. Routing-policy contains the named import/export policies that BGP references.

The fabric interface entries look like this:

```json
{
  "name": "ethernet-1/1",
  "description": "fabric :: az01-sp1 :: eth29",
  "admin-state": "enable",
  "subinterface": [{
    "index": 0,
    "admin-state": "enable",
    "ipv4": {"admin-state": "disable"},
    "ipv6": {
      "admin-state": "enable",
      "router-advertisement": {
        "router-role": {
          "admin-state": "enable",
          "min-advertisement-interval": 5,
          "max-advertisement-interval": 10,
          "router-lifetime": 0
        }
      }
    }
  }]
}
```

Note the `router-lifetime: 0`. This isn't an arbitrary tuning knob — it tells receivers "I'm not a default router for you," which is exactly true for a fabric link. Without it, FRR on the other end might install a default route via the SRL link-local. Tiny detail, big consequences.

The RA interval matches what FRR sends on its side (5 seconds). Default is 600s on SRL, which is too slow for a lab — you'd wait minutes for sessions to come up after a config push.

## The Routing network-instance gotcha

SR Linux has a special "default" network-instance that behaves like the global routing table on a traditional router. There's also a magic interface called `system0` that's bound to default and carries the device's loopback. If you do everything in the default NI, life is easy.

I didn't want to. I wanted the fabric in its own non-default network-instance — call it `Routing` — so the default NI is free for other things. This is how I'd build it on real hardware, and the lab should mirror that.

The non-obvious thing for those not already acquainted with SRL: **`system0` only binds to the default NI.** Try to put it in any other NI and SRL rejects the commit. From the SRL docs:

> The system0 loopback interface is reserved for the default network-instance.

So for a non-default NI you have to create a regular loopback (e.g. `lo0`) with a normal `subinterface 0`, attach an IPv4 address, and bind that to your network-instance. The schema looks like this:

```json
{
  "name": "lo0",
  "admin-state": "enable",
  "subinterface": [{
    "index": 0,
    "admin-state": "enable",
    "ipv4": {
      "admin-state": "enable",
      "address": [{"ip-prefix": "10.0.0.1/32"}]
    }
  }]
}
```

Plus the network-instance has to be `type: ip-vrf` rather than `type: default`:

```json
{
  "name": "Routing",
  "type": "ip-vrf",
  "admin-state": "enable",
  "router-id": "10.0.0.1",
  "interface": [
    {"name": "lo0.0"},
    {"name": "ethernet-1/1.0"},
    {"name": "ethernet-1/2.0"}
  ],
  "protocols": {"bgp": {...}}
}
```

To make this swappable, my defaults file has `network_instance_name: Routing` and `loopback_interface: lo0`, and the renderer infers the NI type: anything other than `"default"` becomes `ip-vrf`, with `system0` swapped for the configured loopback automatically. Putting it back in default later is a one-line change.

## BGP unnumbered with dynamic-neighbors

SR Linux's mechanism for BGP unnumbered is `dynamic-neighbors interface`. You don't list peers by address; you list interfaces and an allowed-peer-AS set. Whenever an ICMPv6 RA arrives on one of those interfaces, SRL discovers the peer's link-local and tries to establish a session — accepting it only if the peer's reported AS is in your allowed list.

```json
"dynamic-neighbors": {
  "interface": [
    {
      "interface-name": "ethernet-1/1.0",
      "peer-group": "FABRIC",
      "allowed-peer-as": [4200000002, 4200000003, 4200001101, 4200001102, 4200001103]
    },
    ...
  ]
}
```

Each fabric link gets an entry. The `allowed-peer-as` list is computed at render time from the topology — every peer device's hostname is resolved to its ASN and added.

The peer-group itself carries the policies and AFI configuration:

```json
"group": [{
  "group-name": "FABRIC",
  "admin-state": "enable",
  "import-policy": ["FABRIC-IN"],
  "export-policy": ["FABRIC-OUT"],
  "afi-safi": [{
    "afi-safi-name": "ipv4-unicast",
    "admin-state": "enable",
    "ipv4-unicast": {
      "advertise-ipv6-next-hops": true,
      "receive-ipv6-next-hops": true
    }
  }]
}]
```

The two `*-ipv6-next-hops` knobs are critical. Without them, the IPv4 NLRI gets a v6 link-local next-hop (because that's what the unnumbered session uses) and SRL silently rejects the routes. This is RFC 8950's extended-next-hop encoding capability — both sides have to negotiate it. FRR turns it on by default for unnumbered peers; SRL needs the explicit toggle.

The `routing-policy` block is mandatory even if you want allow-all behaviour. SRL won't let an eBGP peer-group reference an undefined policy. Allow-all looks like:

```json
"routing-policy": {
  "policy": [
    {"name": "FABRIC-IN",  "default-action": {"policy-result": "accept"}},
    {"name": "FABRIC-OUT", "default-action": {"policy-result": "accept"}}
  ]
}
```

Naming them up front lets me replace the bodies later with real prefix-list-driven policies, without touching the BGP config.

## The push

`gnmic` does the heavy lifting:

```bash
gnmic --address "${addr}:57400" \
      --username admin --password 'NokiaSrl1!' \
      --skip-verify --encoding json_ietf \
      --timeout 30s \
      set --update-path / --update-file "$hostdir/config.json"
```

The mgmt IP comes from a `docker inspect` lookup (clab puts SRL boxes on the same docker network as everything else). Port 57400 is SRL's gNMI default. `--skip-verify` because the lab certificates are self-signed; `--encoding json_ietf` because that's what our JSON is shaped as.

`--update-path /` with `--update-file` says "merge this whole tree at the root." Anything we don't include — system services, mgmt config, AAA — is left untouched. If we wanted to wipe-and-replace, that'd be `--replace-path /` instead, but for incremental config evolution `update` is the right call.

## Reading state back: gnmic JSON parsing

For verification, I needed to read state from the boxes and check it matched expectations. `gnmic get --path /network-instance[name=Routing]/protocols/bgp -t state` does the read, but the response shape is awkward to parse with jq one-liners.

Here's what gnmic returns for a state subtree query:

```json
[
  {
    "source": "172.20.20.3:57400",
    "timestamp": 1778090805545473484,
    "updates": [
      {
        "Path": "srl_nokia-network-instance:network-instance[name=Routing]/protocols/srl_nokia-bgp:bgp",
        "values": {
          "srl_nokia-network-instance:network-instance/protocols/srl_nokia-bgp:bgp": {
            "afi-safi": [...],
            "neighbor": [...],
            "group": [...]
          }
        }
      }
    ]
  }
]
```

The wrapping is consistent — `[0].updates[0].values` always gets you to the values dict. But the *key inside* `values` is variable: it includes the YANG module prefix (`srl_nokia-network-instance:`, `srl_nokia-interfaces:`, etc.) which depends on which path you queried. Trying to write `jq '.[0].updates[0].values."srl_nokia-network-instance:..."`' for every possible query path is a nightmare.

The trick is that there's always exactly one key in `values`. So:

```python
def first_value(updates):
    vals = updates[0].get("values", {})
    return next(iter(vals.values()))
```

That gets you to the actual data dict, regardless of what the prefixed key is. From there it's a normal Python dict walk. I switched my verify script from a jq-based bash mess to a python heredoc and the parsing got dramatically simpler.

## What the apply pipeline looks like end-to-end

```bash
cd fabric/srl
python3 render.py                               # produces rendered/<host>/config.json
./apply.sh clab-nokiacr-sonicl3fab              # gnmic set --update-path / per host

cd ..
./verify_all.sh clab-nokiacr-sonicl3fab         # SONiC nodes via docker exec, SRL via gnmic
```

The render is fast (under a second for three cores). Apply is a few seconds per host, dominated by the gnmic round-trip and SRL's commit time. Re-running with no config changes is a no-op.

## Takeaway

Pushing config into SR Linux from a YAML inventory is genuinely pleasant once you accept that the artefact format is JSON and the API is gnmi. The render is a Python dict-builder, not a string template; the push is an atomic transaction; the schema is YANG so you can't typo a field name without the box rejecting the commit.

Full source for the SRL renderer, applier, and verifier is at [github.com/fatred/srl-sonic-poc](https://github.com/fatred/srl-sonic-poc) under `srl/`.

Next post: [verify is not apply — structuring fabric automation to avoid lying to yourself](/posts/srl-sonic-fabric-poc-verify-not-apply/).
