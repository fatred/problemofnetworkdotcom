---
title: "Making Use of Vault: Ansible"
date: 2024-11-19T18:00:00+01:00
author: John Howard
tags: ["vault", "netdevops", "ansible", "python"]
showFullContent: false
draft: true
---
# WIP!

As we come towards the end of this mini series, we talked about how to [bootstrap](/posts/bootstraping-hashi-vault/) a hashicorp vault for non-prod use, what [primitives](/posts/hashi-vault-primitives/) vault uses for secrets management, and how to talk to vault from [python](making-use-of-vault-python). 

Here we will dig into how you can access vault content within an Ansible workflow, ensuring you never more have the pain of managing secrets with `ansible-vault`, or worse, storing them plain text in a repo somewhere.


## Vault in Ansible

Main Options: 
* HTTPS API via uri (but...don't do this to yourself)
* Community "Hashi Vault" collection: [link](https://github.com/ansible-collections/community.hashi_vault)

### HTTPS API use

> I will hit this extremely briefly, because I think its a terrible idea. There is ONE case where this makes sense - you have to get one small little thing out and stick it in a variable - thats it. Anythind more complicated you should do it with the collection "properly".

### The hashi-vault collection

