+++
title = "10G Router7 Install"
date = "2023-06-02T13:16:30+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["homelab", "fiber-7x", "go"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
Last year I wrote extensively about my experience with deploying VyOS to support my new uber fast internet connection.

I learned a lot in the process, and for this past year it has mostly worked fine. I am one of those people that can't leave things alone however, and I was always tinkering with the setup. The VyOS box itself was happily communicating at 10G, but I would find the internal LAN would get choked up a lot and rarely hit 1G even with extreme threading (say 50-60 conns). I got grumpy about this for no reason and started to tinker. This was a mistake...

What I found was that VyOS itself is fine for saturating this link, but for whatever reason (NAT/FW blah), i was hitting the well known wall of about 6gbits/s of peak throughput _over_ the linux instance. I went off to find alternatives.

## OPNsense

This one irritates me. I am a big fan of the BSD Firewall appliances, and I have used them in Production before to provide me an IPSEC based emergency fix when we lost access to wavelengths between DCs. Theyre bulletproof stable, and according to some online communities, there are people running 10G through OPNsense. **I cannot possibly see how this is true.**

I took the latest release and flashed it onto my Dell Optiplex SFF machine which I now use as my gateway. It's a 6th Gen Intel i5 and had been doing a grand job with VyOS 1.4. Installation was easy, and I was able to configure pretty much everything I had on VyOS within 2 sittings. I was up and going online within maybe 10 mins, and fully confiugred in about 1h elabsed time. Great. 

Speed tests were crap. I think we hit 2Gbit on the init7 speedtest server. 



# router7

i needed to install the commands to build a kernel and then build it. 

i had to install flex and bison to complete that kernel build.

i had to run as root because reasons

