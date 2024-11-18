---
title: "Making More Use of Vault"
date: 2024-11-18T20:00:00+01:00
author: John Howard
tags: ["vault", "netdevops", "ansible", "python"]
showFullContent: false
---

It's remarkably easy to get sucked into hardcoding things that probably should live outside your code. 

Recently I found myself trying to clean up a lot of bad behaviour in our repos, not least of all, secrets management. Here we will cover a few basics that show how to use Hashicorp Vault in more places. 

> If you want to get into the examples below, then I recommend you follow the [bootstrap post](/posts/bootstrapping-hashi-vault/) to get a vault setup that matches the below examples. The git repo can also be found [here](github.com/fatred/practical-vault-in-netops.git)

---

## Some Vault Primitives

Pretty much everywhere you go in vault you will find you need a few building blocks to make _anything_ work.

### Env Vars

Regardless of how you choose to talk to vault (CLI/WebAPI/SDK), you will find that the most common way to "encode" the vault settings is in an Environment variable. This is a nod towards its "cloud native" upbringing, where config files are the devil or something.

The most useful vars are: 

| name | example | purpose | 
| --- | --- | --- |
| VAULT_ADDR | http://localhost:8200 | tells your client (CLI/SDK) where to find the vault API service |
| VAULT_TOKEN | hvs.something | credential your client (CLI/SDK) will use for accessing secrets |
| VAULT_SKIP_VERIFY | true/false | if you have TLS from self signed certs, youll need this to be false |

### mount points, paths and data

Inside of vault your secrets live somewhere. Before we spoke about engines, which are types of secret backends. When you configure an instance of a backend (e.g. the "key/value v2") engine, you have to name that instance, which will become the vault _mount-point_. In our testbed we have a kv-v2 instance which has a mount-point called `secret`. 

> Note: I don't like that naming, but I cant change the dev instance, and I want this to be generic. Play about with names and referencing the content using the examples with different names - you'll get the hang of it.

A path is describing the top level name for this secret, _within_ the mount-point. You can have many paths containing one or more secrets underneath in the k/v store.

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

TLDR: Vault isn't for general purpose use. Protect what needs protecting, not everything and the kitchen sink.

So now we know what our building blocks are, lets dig into some examples. 

## Vault in Python

Here we cover the use case of once-shot scripting and something more structured like Temporal or Stackstorm.

Main Options: 
* HTTPS API via requests library (easy, but clunky)
* Python hvac library (easy, pythonic, but seems quite "engineered")

### HTTPS API use

Long story short you want the requests library. 

## Vault in Ansible

Main Options: 
* HTTPS API via uri (but...don't do this to yourself)
* Community "Hashi Vault" collection: [link](https://github.com/ansible-collections/community.hashi_vault)

### HTTPS API use

> I will hit this extremely briefly, because I think its a terrible idea. There is ONE case where this makes sense - you have to get one small little thing out and stick it in a variable - thats it. Anythind more complicated you should do it with the collection "properly".

