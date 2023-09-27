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

### So its close... but how close?

Part of the story with any new service/implementation always centres around testing. How do you prove, definitively, that something does what it says on the tin. [RFC2544](https://www.ietf.org/rfc/rfc2544.txt) outlines a series of testing strategies and for the purpose of this work we try to keep it simple.

I have deployed a [TRex traffic generator](https://trex-tgn.cisco.com/) on Debian 11 (OFED 5.7-1 doesn't build on deb12) with Mellanox 2x100G ConnectX5 CDAT cards with Trex v3.04. There are really nice instructions [here](https://trex-tgn.cisco.com/trex/doc/trex_appendix_mellanox.html) for that. Happily ignore the CentOS7 religious texts - Debian is fine. Death to DeadRat.

Before we could test a router, we have to test the tester of course. Again, I think its important to give thanks and credit to Pim and Michal for their previous work on making the [trex-loadtest script](https://ipng.ch/s/articles/2021/02/27/coloclue-loadtest.html) and the [ruby rendering tool](https://github.com/wejn/trex-loadtest-viz/).

According to the instructions, we should expect line rate everywhere except 64b (and since imix uses 64b in places, it should impact that too):

For this first test then, we will run a unidirectional flow to see what is the absolute max performance for a single flow of traffic over this Mellanox NIC.

![mlx5-throughput-stats](/img/vpp-adventures/trex-mlx5-max-stats.png)

So here, we have my self test, with a back to back 100g DAC:

![trex-selftest-1514b](/img/vpp-adventures/trex-selftest-1514.png)

Line rate all day long. Official stats were 8.172Mpps with no loss.

![trex-selftest-imix](/img/vpp-adventures/trex-selftest-imix.png)

Interesting. Also line rate? Official stats were 32.697Mpps with no loss.

![trex-selftest-64b](/img/vpp-adventures/trex-selftest-64.png)

So close! Peak number was 133.145Mpps (89.47% of line rate), but we started to get >0.1% loss after 132.526Mpps. This aligns with the numberss in the original appendix for the CX5 card, and so we can call that "a success".

Therefore, we can now run any test on a downstream "Device under test (DUT)" and know that any drop in recieved traffic is down to that device and not the tester itself.

### Who _actually_ uses MACSEC these days?

My first interest for a real world test was straight BGP. Kinda obvious no? For long and complicated reasons, it actually wasn't classic DFZ routing tho - it was something more esoteric.

For a long time it was drummed into me that any ISP service could be snooped on. In the modern age of TLS all the things, I don't care if someone taps my transit/dia for example. I would prefer they don't clearly, but doing so is unlikely to expose anything other than metadata that can be hoovered up at any other midpoint on the network too. Private circuits are a little more sensitive however. Network virtualisation like MPLS has long been a challenge for privacy minded places. You are much more likely to find some unencrypted flows on private connections, and no less likely to find a silent/undocumented tap at the service provider. Historically for many a year the solution here was either a VPN or MACSEC, which would encrypt the data flowing over your private link between your facilities. Having known and used both in production before, I can speak from experience when I say neither is a silver bullet. The former is usually complicated to setup (to some degree at least) and rarely able to perform at high speeds. The latter is often significantly faster since it utilises high powered crypto engines on specialised hardware, taking the speeds much closer to line rates. This speed comes at a cost tho - invariably a _very high_ cost. Its common to find the MACSEC features and blessed hardware to add 50k per device to your budget.

Side effect - large amounts of people use VPNs (point to point, or perhaps SD-WAN), complain that theyre very slow, or the hardware to run fast is too expensive. Those who have more money than sense (or a gun pointed at them), will likely have MACSEC running already.

We had a new build coming and the MACSEC cost was extortion. Somehow i stumbled on an Intel Whitepaper describing VPP with AVX512 offloads for high speed Wireguard - the final spec offered 80gbits/s encrypted flow.

So the challenge was set - replace MACSEC with wireguard, and keep as close to 100G line rate as possible.

### Requirements

MACSEC is an L2 encryption tech where the frame is encrypted, and wrapped in a new frame with the MACSEC encryptors src/dst mac and the crypto header. In practical terms that means we have an L2 path and "something" in the middle just magically encrypts/decrypts as things pass on by.

Wireguard is a L3 protocol that encrypts traffic according to a src/dst IP profile (in a similar concept to IPSEC tunnel mode).

This created an immediate conflict. In order to replicate existing functionality, I need to add a point to point VXLAN (headend replication style), bound to the wiregueard point to point inside addresses, which are connecting over an outside point to point IP pair on the underlying medium.

Playing through the layers:

* DUT1 connects to DUT2 with a 100G DAC in port 1
  * DUT1 applies underlay IP 100.101.0.1/30
  * DUT2 applies underlay IP 100.101.0.2/30
  * DUT1 and DUT2 can ping each other and have full, unencrypted reachability to one another.
* DUT1 uses the underlay to peer with DUT2 and negotiate a wg0 tunnel point to point interface
  * DUT1 applies cryptolay IP 100.101.10.1/32
  * DUT2 applies cryptolay IP 100.101.10.2/32
  * DUT1 and DUT2 can ping each other at the crypto layer and have full encrypted reachability to one another
* DUT1 uses the cryptolay to form a vxlan_tunnel1 interface with DUT2, and bridges its second local 100G port 2 to the VXLAN tunnel
  * Traffic reaching DUT1 port 2 is bridged directly to vxlan_tunnel1, encapsulated into a VXLAN packet, forwarded to wireguard and encrypted and then passed via the underlay to DUT2
  * DUT 2 decrypts the traffic, hands the vxlan frame to the tunnel interface, which then forwards the original L2 frame to DUT2 port 2.

At first this sounds complicated, and there are a few areas this can go wrong, so lets break it down into each section, and test them all one by one, layering up the features as we go.

### Test 1: VXLAN over a cleartext p2p interface

VXLAN is quite cheap, and the L2XC used to bridge the physical port to the vxlan tunnel endpoint is even cheaper. Lets start with a simple VXLAN tunnel from DUT1 to DUT2, using the underlay p2p as the medium.
