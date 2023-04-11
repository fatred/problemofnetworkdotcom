+++
title = "Why I Bought an Airconsole"
date = "2015-06-29T14:56:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Tools","Datacentre","Opinion","Troubleshooting",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

Today I was reminded what a great git of Kit the ![AirConsole](https://youtu.be/1hmi03APwnc) really is.  Its essentially a box that gives you Serial Access to a device via an RJ45 (Cisco pin-out) using WiFi, Bluetooth or wired, using a web GUI, or a bonkers driver setup on your machine.

For me, I use the AirConsole at work in a jack of all trades way.

* I cable the Serial Dongle to the Router
* I have a WiFi client profile configured that will auto join my (pervasively configured) corporate dirty network.
* I have a WiFi AP setup in the AirConsole that securely presents a new network that I can join to access the Serial Port
* I have NAT configured on the AP->Client WiFi so that I can still access the internet from that client Laptop
* I have the Ethernet port configured to bridge with the AP interface, so I can get a wired device to connect to the Serial setup, and the internet.

So, most of the time what I find myself doing is plugging in the AirConsole, then going to a nearby desk and connecting to the AirConsole Web interface via HTTPS over the dedicated WiFi.  I can then configure my box, and still access the internet.  

If you are a sadist you can use these units via Bluetooth paired with your iPad or Nexus.  I tried that, and whilst it worked well, I didn't consider it to be a major benefit to me - the interface via touch screen is a bit dangerous for production configs!  

Where this gets more interesting, is when you look at the Private Server setup.  I bought my AirConsole in the "Pro" variant.  Its the same device, but includes a code for a single seat of Private Server licensing.  You need a seat per AirConsole you connect.  I have then deployed a Private Server OVA into my DC, and configured it appropriately.  So, when my AirConsole is able to access the internet, it calls home, and registers the session with the Private Server.  I can then centrally access all of my running units, and interact with them over HTTPS. You can then extend this to build expect scripts, which is a nice way to provision kit, or to automate diagnostics, particularly if you are using the tablet setups.

This presents a few interesting use cases. In my world, I'm acquiring these for each of my Engineers, and roaming units for each location where Opengear boxes are not economical.  It ensures that my guys have a common consistent way to using serial access, and avoids the recent issues with dodgy Chinese PL2303 USB/Serial clones not working in Windows.

The GCPS Tunneling can be a bit slow sometimes, and you end up with laggy sessions, but when you consider that it allows me to share that session with another technician, it can be really useful for training, or debugging an isolated device without relying on other screen sharing tools.  Using the Web Interface terminal is lag free and very slick.

They recently upgraded the product line and as a result, you can acquire for just $150, a 4 port, hard wired unit (the Mini + 4 port FTDI adapter), to build a fully featured, internet accessible terminal server setup for a lab.  This too is something I am looking at.

In summary, I like the fact that I only have to carry a little white box and a short wire to access all my terminal stuff - it takes up less room in my bag.  I also like that I can use it for straight up Wifi - Wired bridging, and for internet sharing in hotels - ignoring the Serial element. Its very multi purpose and something that saves me time.
