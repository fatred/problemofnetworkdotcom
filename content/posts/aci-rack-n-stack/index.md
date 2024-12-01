+++
title = "ACI: Rack & Stack"
date = "2016-01-15T12:31:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Design","Datacentre","Architecture","ACI","SDN","Troubleshooting",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2016/01/aci-rack-with-falling-at-first-hurdle.html']
+++
Plumbing ACI is something that YouTube has you covered on.  I wont reinvent that wheel.  For the initial standup, I am doing the bare minimum connectivity; each leaf has one 40G uplink to each spine, meaning, 80G of North/South Bandwidth.  This will double up when we are preparing for Production service, matching my UCS/FI Bandwidth between each Chassis (4x10G links to each side of my 2208XPs).  My 3 APICs are configured as follows:

APIC1 e2-1 -> Leaf 1 e1/48
APIC1 e2-2 -> Leaf 2 e1/48
APIC2 e2-1 -> Leaf 2 e1/47
APIC2 e2-2 -> Leaf 3 e1/47
APIC3 e2-1 -> Leaf 2 e1/48
APIC3 e2-2 -> Leaf 3 e1/48

You start the ACI journey on the APIC1 CLI (I used rear console, but you can use the VGA if you like).

There is a short wizard that you run, which initializes the Fabric.  It requests the following:

1. Fabric Name
2. Controller count
3. This controller number
4. This controller name
5. This controller IP address/mask/gw
6. TEP address pool
7. Multicast discovery pool
8. Strong password support *
9. Admin password *

_* only requested on APIC1._

It then checks if you want to edit any of that, and if not, it applies it to that APIC.  If you are using the CLI via serial, you lose that access now.  No further feedback is given (that lost me 20 mins of head scratching!).  If you are on the VGA or CIMC, then you should get dropped to a normal CentOS login with all your settings applied.

You now go to each of the APICs and set this up on each other CLI.  Once completed, you should be able to access the APIC web interface via the IP of the APIC1 on https.

I later found this is where I went wrong.

Fabric discovery is well covered in the user guide, so I won't rehash that, but my problem was one of time and misplaced ingenuity.

I did the first config on the CLI from home via console server. At this time, I had only patched the fabric uplinks and the copper connection to the rear MGMT port. I had not patched the other two ports labelled eth1-1 and eth1-2.  It so happens that these are where the OOBMGMT interface is homed, and where the APIC GUI is hosted.  So once I had completed the CLI setup, I was stuck until I could return to the DC and patch those last cables.  

The next week I was there, patched those ports, and reviewed the documentation.  I decided that the default Fabric name of "ACI Fabric 1" was probably a stupid idea, so I used the CLI command "eraseconfig setup" on APIC 1 to factory reset the thing.  I then reran the setup on APIC1, with APIC2/3 powered down.

I then booted up APIC2/3 and started fabric discovery. I imported all the Leaves and Spines as per the documentation, gave them all nice names and got right to the end. At which point I noticed, that APIC2/3 were not on the fabric. There were nowhere. Checking the alerts I could see them, and they were in err-disable effectively, since the fabric name was invalid. DOH! When I tried to login to the APIC2/3 hosts with the admin user it didn't work either, so I couldn't erasecommand them either.  There was a helpful guide from a TAC engineer, that shows you how to use a USB sitck with a special file, and a grub command edit to boot the APIC into password recovery mode (which it did, and I updated admin's user account), but I still couldnt login to the CLI on APIC2/3. FAIL. Later I discovered that there is a "rescue-user" account that only works when the APIC is uninitialised.  That could have saved me 3 hours, but anyways...

So I followed the guide again, and did a full wipe, this time including the n9k commands to blank them as well.  I reset the Fabric name on APIC1 to the default and was able to rebuild the APIC cluster.  Unfortunately, the Spines and Leaves were now idle, and not auto joining the fabric.  I had to go to the fabric config, and manually define them based on the Serial Number.  Annoying, but not impossible.  After 5 hours I had a fully converged and ready to use ACI Fabric - only with a default name.  My OCD was on fire.  So I did the whole thing again from Scratch.

Lessons learned:

* Over document the process and be super anal about what you enter where.
* Google prospective pitfalls before you start, so you know where to look when you have issues.
* When working remote, floodwire your kit: shut/no shut is easy. Driving 100 miles to patch a wire is not worth it.

Outputs achieved:

`Full ACI fabric deployed and ready for abuse`
