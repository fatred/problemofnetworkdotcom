+++
title = "Back Once Again"
date = "2013-10-02T00:13:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2013/10/back-once-again-for-renegade-master.html']
+++
...for the renegade master!

Blogging is once again on my radar. Why I will never know, but I guess I have missed having the outlet of shouting into the massive empty room that is the interwebs. Its cathartic.

I am actually sitting here quite happily nodding along to Fatboy Slim now. Those were the days.

So, you know what really grinds my gears (no, not Lindsay Lohan), not having enough compute.  I spend my days right now buried in VMware vCloud Director, trying to Architect a very old square peg into the newest of round holes.

My company (a SaaS house) has been a VMware shop for longer than I have been there, and whilst our DC footprints are quite small (less than 100 hosts probably) in comparison with most Enterprise users, its big enough to keep our department busy.  Couple that with a corporate position that expects growth in sales, innovation and cost management, being in my chair means a lot of looking for ways to optimise the infrastructure and streamline the process of project delivery.

So, vCloud, enter stage left.

vCloud has been a part of our virtualisation strategy for nearly 2 years now, and I have been cursed lucky to be involved from the inception.  I have helped steer this beast from PoC all the way through to our third iteration, and boy oh boy have we learned a lot.

What we have learned:

    vCloud is not like EC2.
    vCloud would be better if it was like EC2.
    PowerCLI is awesome

To those who ask me what I actually get from using vCloud over EC2 I say the following:

    Pain

Why don't I just STFU and use EC2 I hear you shout? (or not. whatever.) Before I respond to that, here is some basic facts that cover Cloud in the enterprise.

    Enterprise Cloud infrastructure is not production grade. Any that are, are not cloud, they're just orchestrated virtualisation. If you disagree, you are not doing "production" right.
    Enterprise Cloud application frameworks are pony.  They're orchestration applications that drive the actual "production" management tools. (vCD is a web app for vSphere, SmartCloud Entry is a web app for PowerVM Systems Director, Azure is a web app for HyperV, etc. etc...)
    Well engineered cloud workloads don't occupy the same footprint as a full fat, Production Grade workload

As a result of this, and many other workload dependant factors, its more than fair enough to say that Cloud = Non-Production.

In our business this has translated to our DEV and TEST environments.  We have engineered a solution that enables us to rapidly create a production baselined environment for our Development teams to work within.  No more static environments with half baked, unfinished deploy code lying about.  No more multi-purpose servers that do not mirror the production setup, just cos it was easier, or quicker, or "more efficient" to streamline services that in production sit on different systems purely due to load factors being lower and a lazyness to deliver another VM.  Consistency without waste.

But, back to the point, why do we run a Private Cloud, over public cloud services like EC2, especially when the management platform is far superior? 

Because sometimes its worth the pain. 

We write 99% of all customer facing code in-house. Its entirely bespoke. We use well known platforms, but all the business logic and much of the tools that are used to interact with it, are designed, built, tested and deployed in house.  Therein sits our squarest of square pegs. That is not to say that Public Cloud services are not a square hole however.  

Our round hole relates to the enterprise triangle of technical doom. 

    Security
        All our software is homebrew, therefore is our Intellectual Property. The old adage says, do not put anything on the internet that you cannot control 
        Access Controls (e.g. firewalling, RAS, console access to systems) can be managed by the content owner, but also managed by the cloud owner.  That inherently introduces risk - if you do not know and/or cannot trust all the people that have access to your systems, it is not secure.
    Support
        I have an issue with my Application Server. my Application Server vendor want to know everything about the host/guest. I only have visibility of an instance. I now have no/limited support. 
        I have an issue with my instance and it needs to be fixed immediately. I am at the behest of someone else. They may be awesome, but invariably, they want me to run through an infuriating tick sheet before they will do what we both know needs to happen anyways. Time is wasted essentially. 
    Availability
        My internet has gone down and I cannot get to my cloud. 
        Global routing is fubar and cant get to my cloud
        I havent paid my bill today and cant get to my cloud

This is where a Private cloud can mitigate much of this.  Its not so much a whittling of the peg, but it allows me to make that peg to hole interaction a bit more achievable... (am i stretching the metaphor enough yet?)

    Security
        IP: The Private Cloud is in my DCs and therefore is as safe as the production systems and/or source code itself
        AC: We own all the platforms, including the security ones. We can control the access fully, and have 100% visibility and control over the physical and logical access.
    Support
        All our implementations can be tailored to meet our needs completely (within the parameters of the Private Cloud feature set).
        We have 24Hr Operations Teams and On-Call.  These guys know the platforms and the DC layout. Problems can be solved in a knowledgable and time sensitive manner.
    Availability
        All our connectivity within the business is fully dual redundant and not internet reliant.  All internet connectivity is N+1
        See previous point
        By being Private Cloud, we are not billed on OpEx via Pay As You Go, we create capacity using CapEx, and increase capacity with further CapEx on demand (add more blades).

So Private Cloud is wonderful. If only it were more manageable. 

Since we have have started with our little adventure into Private Cloud we have gone from a bolt on the side approach within our existing Non-Prod clusters, to a dedicated resource pool within that cluster, and now over to a pseudo-elastic multi-site pair of dedicated clusters.  

Every migration has been sized appropriately, deployed properly, and within days been found to be lacking. You know why? Cos people are greedy. OOOOH COMPUTE! NOM!

We are currently running 96 cores (about 220Ghz), and half a terabyte of RAM in the pool. Today I couldn't boot a 17 machine vApp due to resource constraints. A quick bit of PowerCLI shows that in 12 days we have spun up an additional 100Ghz/200Gb of Compute.  We have spun down 10Ghz/20Gb. 

Someone once said Mo Money, Mo Problems (i'm awfully street you see).  I say, Mo Compute, Mo Sadface.  You make it easy for resource consumers to be able to consume, and you will soon see how easily they will find a reason to consume it.  I'm not doubting their need for it, but I do wonder what on earth they use it for.  Ours it not to reason why...

So, fine, I work in the Solutions Team, and a solution is what I delivered. A colleague just raised a PO for more RAM, as tbh thats where we are falling short on a per blade basis, with a view to maxing each one at 256 (from 64). The PO is mid 5 figures.  That gives is massive growth potential, and requires no additional OpEx (a KPI in our company).  Still seems utterly bonkers tho. Makes you wonder, just how much heat are they packing in EC2, Azure, and Google even. This is why I say, we are not big, but we are plenty big enough...

In reality our Cloud deployment is maturing well, and the uptake from end users within the business is going from "meh" to "I need another one of those things".  Both can be problematic, but at least we are able to serve the end user quickly and provide them with something vastly more valuable than before. It is rapidly becoming less about how to do more with a reduced footprint, and more about how to do more, efficiently.  Sounds like two ways of saying the same thing, but the subtle and salient difference is this: being efficient doesn't necessarily mean you use less of the input.  You could just be utilising it better and delivering results faster.

As time goes by I will expand my ramblings and try to provide more insight into Private Cloud and our experiences, as I think its fair to say not a huge amount of people are willing or bothered to talk about it.  I will never go into nitty gritty details, as that remains proprietary, but I have already got plans to talk about our use of PowerCLI to orchestrate vCloud and how we tie it into other aspects of our application suite so that people might have a better idea of what a real world implementation looks like.  I also reserve the right to spaff out random PowerShell snippets here and there for poops and giggles. :D

Till next time...
