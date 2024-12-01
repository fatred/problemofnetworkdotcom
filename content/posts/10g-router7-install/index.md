+++
title = "10G Router7 Install"
date = "2023-06-02T13:16:30+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["homelab", "fiber-7x", "go", "vyos", "opnsense", "linux-routing"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
Last year I wrote extensively about my experience with deploying VyOS to support my new uber fast internet connection.

I learned a lot in the process, and for this past year it has mostly worked fine. I am one of those people that can't leave things alone however, and I was always tinkering with the setup. The VyOS box itself was happily communicating at 10G, but I would find the internal LAN would get choked up a lot and rarely hit 1G even with extreme threading (say 50-60 conns). I got grumpy about this for no reason and started to tinker. TL:DR - This was a mistake...

What I found was that VyOS itself is fine for saturating this link, but for whatever reason (NAT/FW blah), I was hitting the well known wall of about 6gbits/s of peak throughput _over_ the linux instance. I decided this had to be the fault of VyOS and went off to find alternatives.

## OPNsense

This one irritates me. I am a big fan of the BSD Firewall appliances, and I have used them in Production before to provide me an IPSEC based emergency fix when we lost access to wavelengths between DCs. They're bulletproof stable, and according to some online communities, there are people running fiber7 at 10G through OPNsense. **I cannot possibly see how this is true.**

I took the latest release and flashed it onto my Dell Optiplex SFF machine which I now use as my gateway. It's a 7th Gen Intel i5 and had been doing a decent job with VyOS 1.4. Installation was easy, and I was able to configure pretty much everything I had on VyOS within 2 sittings. I was up and going online within maybe 10 mins, and fully configured in about 1h elapsed time. Great!

If only it were true. Speed tests were crap. I think we hit 2Gbit on the Init7 speedtest server, _direct from the router_. Online I was able to find a ton of materials about tuning and each one was essentially a wall of text with sysctl settings. This was really opaque. Some of these were obvious - tweaking queues and buffers. Others were advanced, but logical - playing with Selective ACKs and some more esoteric features of TCP. Lastly there were some extremely cryptic features that even after 20 years in networking I had to go and look up in detail to work out their purpose. Even now, I don't think I could explain these coherently. That could just be me being stupid however.

Ultimately, this was a pretty poor experience in terms of speed.

What I can say though, is if you are not a speed junkie like me, this installation was able to handle a few 1GBit flows with the IDS fully kitted out with a snort subscription (mostly ET based), and all the analytics enabled as well (netflow etc). If you are motivated by features and security, and line rate isn't your jam, then OPNsense is very worth a look.

## router7

Router7 was a project spun up by another init7 customer [https://stapelberg.ch](Michael Stableberg), with the aim to deliver a router appliance written purely in golang.

The background can be found on [https://router7.org](router7.org) but in abstract, Michael hit a bug in dhcpv6 handling on his openwrt install, and went rogue. The original build was designed for 1Gbit using the very impressive, but sadly now deprecated PCengines [https://pcengines.ch](APU) boards. He was then bitten by the same bug as I was to try and chase the 25GBit rate. As it happens, his own testing shows it handles that well, and Pim also did some trex tests of that installation which showed it was plenty capable of handling 25G all day long. This makes it a prime candidate for me here...

Sadly, I havent been able to make this work yet. The rtr7 project utilises an applicance toolchain called gokrazy, which packages up appliances for installation. The common platform here is RPIs and the like, so deploying to a full size PC is less well documented. Michael covers the process in his blog post, but I fear there are a few things either missing in the instructions, or assume more advanced knowledge of gokrazy/golong itself. He is fair and reasonable in his git README when he said, this is a tech showcase and its not expected that someone consumes this. I feel that bending his ear for help is exactly what he didnt want, so it remains on me to figure this out over time.

### Back to VyOS

Since the last two options didn't work, and OpenWRT is known to have issues (as previously noted above), I took a moment with myself and realised life wasnt so back with VyOS really.

I redeployed the latest version and updated a few things in the config to match my latest layout here at home.

What I can say is that if I connect my hypervisor at 10G into the same switch as the VyOS router, and tweak the sysctl settings on the server side, I can occasionally get up to 10G line rate on the bigger downloads. In reality, this is probably as good as it will get for the time being, and I am fine with that for now.

So ther ewe have it. A complete waste of everyones time.  

Sorry about that :)
