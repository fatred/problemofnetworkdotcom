+++
title = "The SDN Conundrum"
date = "2015-05-22T02:48:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Policy","Design","Datacentre","Architecture","ACI","SDN","Devops"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2015/05/the-sdn-conundrum.html']
+++
Oh how the world has changed since I started out in the wonderful trade.

We used to have VLANs and subnets; switches, routers and firewalls.  People would moan things didn't work and we did a traceroute to figure out why.  We would bash out a fix, and if it broke, we would bash out another.  It was the wild west, and that was fun.  Cowboy hats were standard issue.

Then along came the bad guys, and with them, the policy doctors.  Changes became more structured and requirements became more complex.  Environments spiraled out into wider geographical areas and management became less about break fix and more about tightly structured architecture.  The industry responded with protocols and toolchains, each with their own use case, and bit by bit, the sector split up into the key areas of WAN, DC and Campus.

Product portfolios splintered and expanded, use cases became more and more tailored and expansive, until now, most enterprise network admins have hundreds of devices in geographically diverse locations, each with their own subtle differences.  They tend to adhere to policies, and usually templates, but the nature of the beast today is that no two unique devices are configured truly the same.

End result is that managing enterprise networks, which almost always encompass all three key areas, is no trivial task.  To counter this, management have to invest time and money into training and drills to ensure that engineering changes are tightly controlled, and defining standards to as far as possible enforce uniformity into the environment.  The net result tends to be simplified troubleshooting (win), at the cost of slower mean time to delivery and diminished capability for rapid reaction to events (lose).

This is surely not a one size fits all summary, but i'd be surprised if some if not all of that rings true to a large amount of my peers today.

Again, the industry has responded.  In this, the era of devops and scripting, many vendors have looked to enable automation into their equipment, however, automation isn't new.  SPs have been taking CSV files and regexing templates for TFTP delivery since the 90s.  The first batch of automation we have seen recently has been an extension of that - essentially allowing machine control of the CLI, without that interim step of regexing a config template.  Big whoop. The configs still diverge from the template over time and all we have done is decrease rollout times.  Management and Maintenance is in no better shape.

The true leap forward has been with a phrase that fills most old school engineers (me included) with fear and trepidation - Software Defined Networking (SDN).

Back in those good old days, software in the networking context, was the bit that broke. IOS/JunOS/PIXOS you name it, was the part that engineers trusted as far as they could throw it, and moaned about endlessly.  When you found a flash image that worked, you used it religiously across your infrastructure, and you only upgraded it when you hit a bug - even then you would think twice. Its what was drilled into us;
data plane (hardware) = good
control plane (h/w accelerated s/w) = scary
management plane (pure s/w) = flakey

So, the first time someone described a Software Defined Network to me, I laughed very loudly and moved onto other things in my head.

My next interaction with SDN was a demo of VMware NSX.  I was stunned at the seeming maturity of it.  The ability to say: This VM in this host, let it talk to this VM in this host. Nothing else. Oh and don't burn a whole VLAN in the process.  I think I was excited about it for at least a week until I tried to research the setup and maintenance, at which point I became wholly disillusioned with it.  I had been reminded of the perils of swallowing the marketing spaff.  When I reached out to our partner for more info, I was almost immediately drowned in calls and offers of workshops and engineering time.  No pricing was available either.  Cue my moment of Zen and the realization that this was box fresh and they needed guinea pigs. No thanks.

I resolved to ignore all the SDN noise and focus on automation.  I learned python, spent some time with Kirk Byers' blog and the paramiko libraries and before too long I had a little collection of read/regex/deploy scripts that could do simple day to day tasks such as spread a VLAN throughout my core, or add/remove/update aaa setups.  Life was good.  Except I was still auditing changes manually, and was white-knuckle scared every time I ran the scripts.  It was actually slowing me down in the long run.  

Meanwhile business pressures changed.  People started comparing our operations to "the cloud", and it became apparent that all the structure, all the audit, compliance and governance, all the delays were staring to piss off our users.  It was taking too long to deliver what the business needed.  What they need is a stable, secure, responsive and highly available service; at a moment's notice, yesterday.

So, now we care less about automation, and more about orchestration.  Different word, different processes - same eventual outcome.  We need to join the dots of our various changes together, and abstract them into human consumable requirements.  

Developers want a standard environment to chuck some code into that is as prod like as possible.  To me that's a Public IP, a NAT, a load balanced VIP, pool members and delivery rules, firewall configs; public, DMZ, and across multiple zones in the core to list but a few.  Each time I deliver such an environment, the only real differences are the names and IPs - to such a point that a simple spreadsheet covers the unique elements, and the rest is "use object group a" and "configure balance profile b", etc.  We're really good at knowing what settings we need, and just apply the names and IPs accordingly.  It still has to go out manually tho, thus takes ages, and due to the human input part, we still need that structure, and change control, to ensure no impact to other running services when mistakes are made.  

All that means sod all to the requester - to the dev its a black box. A bubble of service. A consumable object.  Same as the one they had last week, only a new set of them for doing something else.

What I need to do is join those bits together, to simplify the change, and abstract the nuts and bolts away till all I am filling in, is those blanks that have changed.  Cloud can do it - why cant I?

Again - the industry has responded.  The concept of SDN splits the physical network (underlay) from the parts of the network where services are consumed (overlay).  This split satisfies the need for a strong, stable hardware driven infrastructure, whilst delivering a platform that can be centrally controlled, and interlaced with all the various services dynamically and on demand (orchestration).

Separating the Underlay from the Overlay means that finally enterprise can do away with wasteful overcommits in the network layout, and start to use CLOS fabrics and Equal Cost Multi Path (ECMP) designs.  Couple this with 10/40G bearer speeds and before you know it, we have some serious clout in the network core, and finally a way to utilise it fully.  Great - but wait...

Unfortunately in the Overlay things are still quite fluid.  To date there are a number of different overlay options available.  
OpenStack is the most commonly touted and most widely understood way to manage OpenFlow, depending on the scope you want to consider. It essentially manages the OpenFlow enabled switch estate and programs flow entries around the network given its global view of the infrastructure.
This was later splintered to OpenDaylight, creating a product suite that a lot of people talk about preferring but to this day I still don't understand the real difference  It seems to me that ODL is more about managing more than just OpenFlow entries on switches, but like I say - its not entirely clear on the goal.  
Various vendors have then taken the open-source tools and applied their own spit and polish, such as HP's Helion extensions OpenStack and Brocade's Vyatta extension of OpenDayLight.
VMware have had the jump on most with NSX being around in the marketing landscape the longest, but its use case remains quite niche and takeup still doesnt sound that great.  
Cisco have then looked to corner the enterprise space with ACI, which is still very new, but at least in my use present use case, the one that covers all my required bases.

I will take separate posts to discuss the various solutions as I see them, but allow me to summarise my position as follows:

Openstack/OpenDaylight: Webscale/Cloud hosting centric. Mature in the context of present SDN, but still short of something you would bet your career on.  Seemingly the de-facto starting point for third party integrations.
VMware NSX: Designed for VMware IaaS shops i.e. Service Delivery teams in a larger business delivering Private Cloud services to internal customers. Not a lot of third party work going on here.
Cisco ACI: Enterprise centric.  Good at providing the simple stuff like 3 tier applications in a single DC. Totally at sea when things get more complicated than that.  Seems to have a good roadmap, but 3rd party buy-in limited to the time they get after OS/ODL plugin dev.

So, where are we now?  

The cloud revolution has forced all kinds of businesses to re-evaluate their infrastructure strategies.

* Webscale folks have had to change tack to ensure they can deliver IaaS/PaaS services at the click of a button, whilst squeezing every last drop of tin they have for RoI
* Internal IaaS folks have had to deal with secure multi-tenancy without the pain of configuring PVLAN or VRF each and every time they want to sneeze, also squeezing tin for RoI
* Enterprise folks: have to compete with the above, whilst retaining control, audit, visibility and stability.

For most enterprise businesses the options are becoming clear; to cloud or not to cloud.  Depending on your Capex/Opex ratios the answer is obvious really.  If you like Opex - cloud.  If you like Capex, stick to on-prem.  For me however, I think that real SDN opens up a third and more sensible approach of Hybrid deployments.  Use Cloud where appropriate, but keep the crown jewels in house.  The only way for that to be truly viable, is to take an SDN approach to your deployment.

It took me a while to get my head around the point and the use case for SDN.  The length of this rant kind of demonstrates that.  Hopefully it also provides a thought process to others as well - your answers may not be the same as mine obviously.  I haven't spoken to the nuance of each option out there, nor covered the genuine concerns that any reasonable engineer would have regards the stability and future viability of each offering either.  This will come in time, however, I make one bold and contentious statement:

It is my belief that any business not considering and reviewing an SDN strategy within the next 18 months, will be in real competitive trouble in its market within 3 years.

That could be as simple as migrating to Cloud since they handle the SDN component for you.  it is still an SDN strategy.  To those who remain on prem and don't have true, application level orchestration within the next 3 years, you will be left behind by your peers, who will innovate and deliver new product at an exponential rate.  You have been warned!*  

_* by a complete nobody who has no idea what he is talking about._

Before too many people get up in my chops as well, my point with this post was to challenge the default NIMBY response of most engineers, and to spark internal debate within those same engineering minds.  SDN is not a fad, nor is it a panacea. It is a genuine swiss army knife that when used appropriately can allow you, the beardy engineer, more time to innovate, rather than just administer. Never say never...
