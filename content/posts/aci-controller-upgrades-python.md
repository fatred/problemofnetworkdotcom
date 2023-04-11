+++
title = "ACI: Controller Upgrades with Python"
date = "2016-01-28T01:42:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Policy","Automation","ACI","Tools","API","Devops","Python",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
So I bought my ACI bundles so long ago that they're still running 1.0(3f). Right now mainline is 1.2(1k), so i'm a bit behind.

Using the official Cisco doc I did the first staged upgrade from 1.0 to 1.1 using the Web GUI. I wanted to see what happened in a visual sense.

Basically you setup a connection between the APIC and a host that has staged the firmware files, then you setup a policy defining what versions the fabric should be on, and when that should be made active. For me it was 1.1(4f) and now basically.

This caused the APIC controller to go an pull the image off the staging server (I used a windows box with IIS, but you could easily use a Linux box with SSH enabled too.  Once it had pulled the image down, it started to apply that to APIC1, then 3, then 2 (it does say its random).  Each cluster node updated, rebooted, rejoined the cluster, and soaked in, before the next controller started the process. It was about 15 mins per controller.

So, whilst I was waiting for that to happen, I started to look at the python equivalents for these GUI commands.  When I come to the other DC, and indeed upgrades later down the line, I would like to avoid all this clicking.  The spec is simple:

Have a python script that you run with 3 main arguments

1. path to images on local host
2. apic address
3. upgrade --now or --later

The workflow is less simple:

* get username for apic login
* get password for user
* test them to obtain current apic version
* test location for .iso and .bin files (maybe regex them to present version num?)
* if not --now, then prompt for when (next maint slot or specific time option)
* present planned execution of from version to version @ time to user (now or later option)
* pull a backup for the fabric config to the local directory (timestamped)
* setup local http server in directory with SimpleHTTPServer module
* submit temp firmware update source to apic
* wait for transfer to complete (maybe monitor the file access from the server side?)
* check the firmware repo on the apic for our new version num
* submit a controller policy for the updated version with time as per user req
* present the user the url to monitor the upgrade on the GUI.
* remove temp firmware source
* end

To do this we will be using the cobra framework, and a nice tool from the ACI Devnet guys called arya.  The concept here is to perform the task once in the GUI, then using the APIC inspector, pull out the get/post calls and get arya to parse them and create equivalent python code.  I did start out building the script raw using requests and json objects, mimicking the calls I tested using postman.  Its important to realise that worked well, but was too time consuming considering there was a whole API suite already there.  I will leave those scripts in Git under a dormant branch for future reference I think.

Once the script has worked at least once, I will post it.
