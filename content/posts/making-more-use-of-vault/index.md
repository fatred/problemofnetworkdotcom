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

## Vault in Python

Here we cover the use case of once-off scripting and something more structured like Temporal or Stackstorm.

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

