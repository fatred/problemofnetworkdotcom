+++
title = "ACI: The Setup"
date = "2016-01-12T12:30:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Design","Datacentre","Architecture","SDN","ACI",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2016/01/aci-setup.html']
+++
On Friday last week we rolled out our ACI solution into one of our DCs. The setup is simple, comprising of;

    2x Nexus 9336pq "Baby" Spines
    4x Nexus 9396px Leaf Switches
    3x APIC Controllers
    2x ASA 5585x Firewalls

The compute behind it is UCS based and we have F5 LTMs in the ADC role.

Over the weekend I provisioned it.  That did not go well.  Today I had to go back and revisit the cabling, and then the Fabric initial setup, and then redid the entire thing from scratch again. Oops.

Lets hope the rest of the journey is a bit less fraught eh?
