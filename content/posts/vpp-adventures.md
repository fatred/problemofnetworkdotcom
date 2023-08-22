+++
title = "VPP Adventures"
date = "2023-08-22T22:29:17+02:00"
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
Linux Routing is becoming a thing with me. I cant decide if the motivation is the extreme cost of dedicated hardware, or the knowledge that with a little effort you can make a free/cheap thing into a giant killer. David and Goliath is a fun story I guess.

VPP has been on my radar now for a few years. I have tried and failed a few times to get it into production typically on the internet edge of a datacentre in place of something expensive like a Cisco ASR or a Juniper MX.

If you dont know what VPP is yet, then I wonder how you ended up on this page, but should you need that primer, you can find it [https://s3-docs.fd.io/vpp/23.10/](on their docs here). TL:DR - its the opensource side of the project that underpins the ASR routers themselves. Yes. Thats right. VPP is essentially the engine behind the ASR.

WHat it does is utilise Intel's DPDK tooling to have a process acquire full control of the NIC, away from the kernel. This might sound stupid, but there is logic here. Rather than force each and every packet along a waterfall pipeline of packet processing - whether the packet needs to visit that branch of code or not, VPP builds a graph for packet processing and then you get an optimised path that reduces latency across the board. Half of the way this is achieved, is by leveraging CPU design and preloading the operations into faster memory registers so the CPU doesnt have to lean on slowed subsystems like RAM or Swap. Pim covers this in a much more approachable way at SwiNOG in 2021 - [https://www.swinog.ch/wp-content/uploads/2021/12/Pim-van-Pelt-IPng-Networks-Evolution-of-DPDK-Controlplanes.pdf](his slides are here)

As a result, VPP can provably route 100Gbit/s and more, using 10yo server hardware. Again, proven by Pim[https://www.youtube.com/watch?v=D7PF1cOAAUk&themeRefresh=1](at DENOG 2022).

## Why did the previous attempts fail?

Two problems:

1. Janky Config Management
2. Lack of talent on my part

Lets pick them off one by one.

### Janky config management

VPP has long been defined as an API project. They openly admit the CLI (vppctl) is not a production asset, and in fact is there more as a hacky tool to help with debugging and development of the API featured themselves. As a result of that, the CLI is buggy, inconsistent and very occasionally, fully broken.

The expectation is that you take the C++ APIs and then write your tooling against these for config operations. Projects like Netgate's TNSR (they of PFsense fame), have done exactly that; they use the opensource VPP project, and wrap the config with their closed source tooling, so they have something they can sell as a product.