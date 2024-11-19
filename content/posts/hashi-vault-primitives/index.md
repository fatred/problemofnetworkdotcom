---
title: "Hashi Vault Primitives"
date: 2024-11-18T20:00:00+01:00
author: John Howard
tags: ["vault", "netdevops", "ansible", "python"]
showFullContent: false
---

## Some Vault Primitives

Pretty much everywhere you go in vault you will find you need a few building blocks to make _anything_ work.

### Env Vars

Regardless of how you choose to talk to vault (CLI/WebAPI/SDK), you will find that the most common way to "encode" the vault settings is in an Environment variable. This is a nod towards its "cloud native" upbringing, where config files are the devil or something.

> Note: there are wayts to have a config file like setup, and in the API you can code in the settings with vars instead of envs, if you RTFM, youll find what you need.

The most useful vars are: 

| name | example | purpose | 
| --- | --- | --- |
| VAULT_ADDR | http://localhost:8200 | tells your client (CLI/SDK) where to find the vault API service |
| VAULT_TOKEN | hvs.something | credential your client (CLI/SDK) will use for accessing secrets |
| VAULT_SKIP_VERIFY | true/false | if you have TLS from self signed certs, youll need this to be false |

### mount points, paths and data

Inside of vault your secrets live somewhere. Before we spoke about engines, which are types of secret backends. When you configure an instance of a backend (e.g. the "key/value v2") engine, you have to name that instance, which will become the vault _mount-point_. In our testbed we have a kv-v2 instance which has a mount-point called `secret`. 

> Note: I don't like that naming, but I cant change the dev instance, and I want this to be generic. Play about with names and referencing the content using the examples with different names - you'll get the hang of it.

A path is describing the top level name for this secret, _within_ the mount-point. Assume for a moment you were looking at the Vault UI, and you clicked on the "secret" _mount-point_. The list of objects you see in this view are all the path options you have to work with. You can have any number of paths within a mount-point, containing one or more secrets underneath in the k/v store.

The data is the actual key value pairs we find under this path, which could be one or more entries, in flat k/v, or if you like, a dictionary encoded with json.

for example: 

```
mount-point: secret
path: myflatsecret
data: {"token_for_thing": "293u4iehjkfdvhqink"}

--- 

mount-point: secret
path: mysimplesecret
data: {"username": "dave, "password": "Very-secure-information!"}

---

mount-point: secret
path: mycomplicatedsecret
data: {"wireguard": { "keys": {"privkey": "privkey1", "pubkey": "pubkey2"}, "ip": {"outer": "100.64.0.0/30", "inner": "100.64.255.0/31"}}}
```

You really can choose any scheme you like. Keep an eye on these specific examples - we will use them a few times in the remaining activities.

Be mindful that there is a high RoI for security by centrally storing your secrets in a place where the access is not only authenticated, but logged and audited. That comes with a cost to operations because all those extra network calls and crypto, and potentially, the vault being locked and unusable, means your data is slow, or posibilty entirely unreachable at runtime. 

TLDR: Vault isn't for general purpose k/v storage. Protect what needs protecting, not everything and the kitchen sink.

So now we know what our building blocks are, lets dig into some examples. 
