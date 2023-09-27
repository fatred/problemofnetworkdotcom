+++
title = "VPP Adventures Part 3 - The Testbed"
date = "2023-09-27T13:20:25+02:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
So far we have covered what VPP is, and why its interesting to us.

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
