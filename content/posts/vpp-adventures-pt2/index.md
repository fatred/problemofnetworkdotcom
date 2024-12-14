+++
title = "VPP Adventures Part 2 - but why?"
date = "2023-08-23T13:15:32+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["VPP", "linux-routing"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

In the previous post we were talking about what VPP was. Here we explore a little why it matters.

### What's the point anyways?

It's a fair question. Surely its not logical to invest so much time and effort into something that has been described numerous times as "janky". One of my engineers even now says, "I understand why you want to do it, but I don't agree that this is the right solution". She's right too btw.

VPP is really not there yet for a full production 100Gbit/s Edge router. It is not a 1:1 replacement for an MX or an ASR. There. I said it.

But it is _almost_ there.

The Dataplane works flawlessly. This is the thing you are "buying" here, and so thats 100% as described imho.

The control plane has always been delegated off to :airquotes: something else :/airquotes:. In the most common deployments of VPP I see so far (e.g. Netgate TNSR or the Calico CNI for k8s), something proprietary is managing the controlplane and just telling the dataplane what directions packets should go. This is the bit that VPP itself, does not and will never provide.

Until there is a mature, general purpose control plane for VPP it will remain difficult to recommend as a standalone thing.

### Enter LinuxCP-ng

Pim spotted this was a problem a while ago. Linux has a great control plane powered by netlink already, but the challenge was how to handle punting between VPP and Kernel, with minimal impact to forwarding performance. There is an in-tree linux-cp plugin within VPP, but it had significant problems with syncing changes between the two control plane domains and development on that had stagnated after Netgate had written their own tools for TNSR. Pim looked at fixing these himself, but after a while began work on a new out of tree plugin that would implement an alternative approach to syncing between linux's netlink API and VPPs API.

For some time now it has been slated for inclusion in-tree as "lcpng", but for reasons unknown it remains outside [on Pim's github](https://github.com/pimvanpelt/lcpng). There are a number of handy guides on his website that explain how to build and deploy VPP with his LCP plugins.

With this deployed, we now have a high performance router that can utilise standard linux tools like FRR and Bird as routing daemons (control planes), where LCP then copies the actions it observes on the linux side into VPP API calls that instruct the packet forwaring engine itself.

Pim has written extensively about how he has dogfooded this setup for over a year now. It maybe "only" a 10G ring network, but it's got real load and its proven to be stable on the whole.

### ... and vppcfg

Whilst the lcpng plugin is capable of configuring the VPP instance without you typing one command into vppctl, it was clear that there were edge cases where having the ability to start the config sync by writing in the VPP side was preferable. This is what then motivated Pim to build [vppcfg](https://ipng.ch/s/articles/2022/03/27/vppcfg-1.html). This allows a simplified yaml config to be rendered into "verified compliant" cli commands that can be issued to the vppctl CLI, interactively or via the bootstrap method in the config files. By making this declarative he also handles the order of execution properly via a directed graph that is included in the config tooling.

This for me is so very close to the missing piece for "fixing" the janky CLI.

So its close... but how close? In the next chapter we will look at how we ubild a testing setup to prove that VPP is able to handle what we throw at it.
