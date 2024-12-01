+++
title = "ACI: Mini Rant to INSBU"
date = "2016-01-15T14:34:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Design","Datacentre","Architecture","Opinion","ACI","SDN",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2016/01/aci-mini-rant-at-insbu.html']
+++
Before I get too wound up I should probably say that all of this was directed to my friends there first, and whilst I won't say much about their thoughts, I don't think this is particularly new to them, or out of place.

I have a fondness for ACI.  I think its innovative, and once you break through the naming conventions and the terminology, it's exactly what I think Enterprise should be doing in terms of Next Generation Networking.  That said, INSBU are not helping themselves penetrate the market, and as such, are putting themselves at risk of falling behind to Openstack.

I have already had plenty to say about SDN in the enterprise (see The SDN Conundrum), so I won't lift that rock yet again, but one thing to me is quite clear; the sales channel as it exists today is ineffective.  Engineers and Architects (like me), don't really care about your noddy environments that you put up on the slide in my meeting room, with your simple 3 tier application.  Mine dont look like that, and knowing what I know about my environment, I fear the complexity of applying the SDN transform to that workload.  Greg Ferro disagrees with me, vehemently. He says that enterprise architects are blinded by an incorrect assumption that their world is a unique snowflake and that if they just suck it up and got on the bus, we would all be fine.  Whilst I have seen that, and I agree with much of it, as Ethan Banks, his own co-host at PacketPushers says, there are definitely shades of grey.

For me, Cisco saw we were a valid opportunity, and they put me on the ACI test drive.  Test Drives are a great idea, and give you the chance to get hands on and implement a solution for yourself.  The key is "a solution", not "my solution".  It got the cogs moving in my brain, and I was able to start joining the dots between my world and what the ACI world looks like.

Between that day and the day I powered up my APICs, there were zero opportunities to take that further.  Yes, there are people at our reseller, and people in the Cisco Sales Engineering teams who can help, and apply context.  But there are no opportunities for me to model or test my workloads in an ACI fabric until I buy.

Compare that to Openstack.  I can go onto github and clone the OpenStack Devstack repo, or install the OVA for OpenStack Cookbook into a VM - all for free. I can then setup the various elements with the cookbook manual (available for about £35).  Within a week I can have a workload of mine running in an contrived environment, and within that week I have been able to properly evaluate if that product will work for me. Also, in that time I am able to decide if Greg was right, or if he was being obnoxious (as is his forte).  As it happens, Ethan was spot on.  My environment is not unique, nor is it cookie cutter same old same old.  It consists of a fair few 3 tier applications that have interdependencies.  I can model my environment in Openstack, and in my case, decide that its not for me (not today anyways).  I cannot do that with ACI.

Since going to Cisco Live last year, I have fallen in love with DevNet.  I love the labs, I love the way I can go to dcloud or devnet labs, and spin up systems that someone else manages, and try out my mad idea.  I love the way I can do that, and sit on the phone or in a room with my CSE and work out what I did wrong, or what i'm doing right, and have that real example of things that are relevant to me, and relevant to the expert, and come to an understanding at the end of it.

So, the question sits with me - Where is the ACI lab?

If I had a lab environment, I could do that same modelling I did with Openstack.  I could see for myself that (as I now know) I am on the right track, and most importantly, I can start to put real tangible numbers into my business case for where and how this will benefit the business.  I can justify what it is I am doing.

When I was in San Diego, I was lucky enough to sit with Prem Jain at dinner, and I think I must have had this same argument with him, for about an hour, drunk (me not him). He was very gracious and didn't tell me to shut up, and took the feedback.  About a week later the APIC labs were announced as coming soon in DevNet (I was so happy until I found out it had been on the cards for months!). Today, they are still "coming soon".  Thats like 6 months later.  And looking at the Cisco GPL, there is a SKU, APIC-SIM-S which is valued at $16000.  That is exactly what it is I am calling out for.  Its a 1U C220 server that emulates a 2 spine 2 leaf fabric, and allows you to perform pretty much everything you need to trial your environment.

Dear Cisco.  WHY ARE YOU MAKING ME PAY FOR THAT!!!

Make that free, make that only available behind a login portal if need be, and let people like me, model their own environments in ACI.  I guarantee you will be more competitive with Openstack, and you will make much much more money.

In the meantime, I am having to further my modelling and environment design against one of my two ACI kits, effectively treating my DR site as my lab.  That is fine, but its definitely inefficient.  I am pursuing access to one of these Sims with my account team in the meantime, since I think that will help me no end in finalising my design and push on with the build into production.
