---
title: "Using Vault PKI to Secure Devices"
date: 2024-12-14T08:50:37+01:00
draft: true
---

# abstract

Following on from the Hashicorp Vault "how-to" series. Lets dial things up a notch, and setup a PKI in vault that can issue "real" certificates for your devices. 

This has a couple of real tangible benefits. 

1. No more `verify=false` and/or urllib hacks to connect to TLS secured endpoints
2. No need to fight `openssl` to wrangle self signed (or for the really brave, a manual CA)
3. Full automation support to enable estate wide renewal in minutes, not _half a lifetime_.

So lets get into it.

---

# pre-requisites

First, we need a vault. If you are really going to do this in production, I recommend you get a "real" vault. up and running. The ![hashi docs](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-deploy) are great. If you just want to faff about a bit to learn, you can use my docker setup from ![here](https://www.problemofnetwork.com/posts/bootstrapping-hashi-vault/) if you like. 

It will be really important to build out an authentication system, but its too much to cover here. I'm cheating and using a root token for the terraform bit, but we will create some scoped tokens for the _actual_ signing requests, within the terraform repo itself. You should poke around this more to setup a scoped token for the terraform to use that leverages policy to only have permissions to the parts of the vault needed.

Because there was soooo much to do inside of vault, I decided to make this into terraform and put it into git. You should clone the repo from `https://github.com/fatred/exploring-vault.git` into a place on your system. 

Before running anything you will need the ![vault client](https://developer.hashicorp.com/vault/docs/install) on your system, and the ![terraform client](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) too.

