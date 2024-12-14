+++
title = "VPP Adventures Part 1 - uwotm8?"
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

If you dont know what VPP is yet, then I wonder how you ended up on this page, but should you need that primer, you can find it [on their docs here](https://s3-docs.fd.io/vpp/23.10/). TL:DR - its the opensource side of the project that underpins the ASR routers themselves. Yes. Thats right. VPP is essentially the engine behind the ASR.

What it does is utilise Intel's DPDK tooling to have a process acquire full control of the NIC, away from the kernel. This might sound stupid, but there is logic here. Rather than force each and every packet along a waterfall pipeline of packet processing - whether the packet needs to visit that branch of code or not, VPP builds a graph for packet processing and then you get an optimised path that reduces latency across the board. Half of the way this is achieved, is by leveraging CPU design and preloading the operations into faster memory registers so the CPU doesnt have to lean on slower subsystems like RAM or Swap. Pim covers this in a much more approachable way at SwiNOG in 2021 - [his slides are here](https://www.swinog.ch/wp-content/uploads/2021/12/Pim-van-Pelt-IPng-Networks-Evolution-of-DPDK-Controlplanes.pdf)

As a result, VPP can provably route 100Gbit/s and more, using 10yo server hardware. Again, proven by Pim [at DENOG 2022](https://www.youtube.com/watch?v=D7PF1cOAAUk&themeRefresh=1).

## Why did the previous attempts fail?

Two problems:

1. Janky Config Management
2. Lack of talent on my part

Lets pick them off one by one.

### Janky config management

VPP has long been defined as an API project. They openly admit the CLI (vppctl) is not a production asset, and in fact is there more as a hacky tool to help with debugging and development of the API featured themselves. As a result of that, the CLI is buggy, inconsistent and very occasionally, fully broken.

The expectation is that you take the C++ APIs and then write your tooling against these for config operations. Projects like Netgate's TNSR (they of PFsense fame), have done exactly that; they use the opensource VPP project, and wrap the config with their closed source tooling, so they have something they can sell as a product.

These third party controlplanes utilise the VPP API to send the low level commands required to program that data plane. If you observe the [CSIT test plans](https://docs.fd.io/csit/rls2302/report/test_configuration/vpp_performance_configuration_2n_icx/l2_xxv710.html#n1l-25ge2p1xxv710-eth-l2xcbase-ndrpdr), you will see the API calls and parameters used as a very handy hint.

The CLI exposes these API calls in a very basic CLI, which having been written in C++ along with everything else, has some severe usability problems. As far as i can tell, C++ doesnt have a handy argparse module like python. Therefore, writing all the logic to correctly offer, parse and then utilise, every possible option in a command is challenging. If you type in an option that doesnt exist, or put things in the wrong order, you almost always get a bland generic error that give no context or feedback. Typically you end up with a "error (-54)" or similar. To move further, you then have to find the function behind the cli command, then backtrace until you find the point at which that error was raised, where you can finally lookup the enum to _maybe_ get half an idea as to what is wrong.

Finally, for all you _just google it_ engineers out there, let it be known that VPP has moved through a lot of major versions in teh past 3-4 years, and much of the indexed content, within and beyond the fd.io site, is already wildly out of date thanks to breaking changes.

I spent maybe 2 days chasing a valid IPSEC (p2p tun type) config, only to find later that I was basing my config off an example written for v17, and referring to documentation that was in the 23 tree but actually only valid for >=22.10.

I recently stumbled upon [govpp](https://github.com/FDio/govpp), and I am interested to see if I can take that API interface, and then wrap my configs into a pared down CLI just for my use case. I think this would be much more stable for my team and more reliable for ongoing work, which for us is still quite constrained.

Ultimately, you have to accept that the value of VPP is in its dataplane performance, and, sadly, the operator experience is a little janky today.

### Lack of talent

I am a Network Engineer. I am an amateur programmer with plenty of Python knowledge, and a growing Golang capability. I am _rubbish_ at low level languages and C/C++ are very hard for me to engage without outside of basic troubleshooting.

This, coupled with the janky CLI, meant that my initial attempts to _just make VPP work_ were completely fruitless.

Before you get buried into the VPP config, you first have the DPDK fun as well. It's pretty badly documented that DPDK has some strict requirements for the driver/firmware/hw rev collections. Each release of VPP is compiled with a specific version of DPDK under the hood, and so its important to lookup which DPDK your VPP instance uses, and then trawl the support matric to find the versions of the required dependencies. For my use case we have the Intel E810 NICs, which use the `ice` driver. The Recomended versions list is [here](https://doc.dpdk.org/guides/nics/ice.html#recommended-matching-list). I will write a short technical post on bare metal bootstrap later, but know that it's non-trivial to ensure all your required devices have the correct driver and fw package. If you find ourself unable to get VPP core working, always double check these basics before you rip all your hair out.

A lot of my more recent successes with VPP came as a result of spending time with Pim from [IPng Networks](https://ipng.ch). Pim is a VPP veteran who hsa been using it in Production on his hobby network that runs around Europe in a number of hosting centres. I was able to crross his palm with a very very modest amount of silver in exchange for some of his advice. After 2 hours I was up and running with instances and load testing hardware. After 4 hours I had production grade configs. He is still consulting with me to help me understand the performance aspects and the monitoring situation.

If you are unsure of how to proceed, but really want to do so, reach out to him and see how he can accellerate your journey like he did mine.

Since this is getting kinda long, lets split this up a bit. In the next installment, we will look at why we even care about VPP.
