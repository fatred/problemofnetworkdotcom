+++
title = "Network Automation: from spreadsheets to YANG and everything between"
date = "2020-06-22T17:52:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = []
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2020/06/network-automation-from-spreadsheets-to.html']
+++
Over the last few years I have spent a significant amount of time in YAML and Ansible. I'm not an expert by any means, but I can probably get anything I need done in maybe a few days worth of tinker time. I'm what you would call a fair weather scripter.

One thing I learned from building about 30,000 lines of YAML whilst orchestrating ACI, was that Ansible is a square peg, and everything networking is a round hole. Sometimes it fits, sometimes it doesn't, but it is never perfect. Never.

This has led some in the trade to poopoo Ansible as a networking automation tool. That in my opinion is vastly over-dramatic and in many cases counter productive.

The saying goes, "don't let perfect be the enemy of good." Ansible is good. Its not perfect, but who is?

When someone asks me what should they learn as a networking person looking to grow, my first answer is 100% Ansible. It works in most of the major platforms and it can help you deliver changes in a structured way, with minimum inputs and usually a reliable output. That is then closely followed by some sort of CI Pipelining tool. Gitlab makes sense cos you can deploy it onprem easily enough and not have to worry about "data protection" by accidental repo exposure. You can just as easily use a hosted version or github even now they changed their billing model, but ultimately you probably don't want to be installing Jenkins or Concourse as a relative beginner.

So, as someone who isn't a relative beginner, why start a post about Ansible, with a subject of YANG? Because its about progression - both mine and the industry.

When I started on the DC automation drive about 3 years ago, our active platform of the time (ACI) was in v2, and the REST API was pretty well documented. There was also a lot of information available on the internal Object Model, and some Python Modules that allowed you to interact with that directly. Ansible was nowhere in networking really, and most of the Cisco SEs we using csv files and Postman to build fabrics. I spoke to a couple about what I was thinking to do, and they all said the CSV/Postman thing is a bootstrap tool - it stands up a fabric to be managed independently thereafter.

I decided I knew best (thanks ![dunning-kruger](https://en.wikipedia.org/wiki/Dunning%E2%80%93Kruger_effect)) and spent about 3 months taking the csv file and postman approach, to initial great success, before begrudgingly realising that it was absurd to try and make small additive changes in this way. I then started clicking the "generate code" button in postman to dump out python scripts that would complete each task, adapting the computed output into something more generic, first using command line args and then later pulling in the variables from a sqlitedb. Very devops-y. I felt the beard grow. I then tried to put a flask GUI over that DB, which is when the wheels came off. That was another 4 months down the pan.

I then started to mangle the parts of those scripts into more modular scripts that followed a workflow component role (think add new collection of things rather than add this, then that and then something else individually), so that we could feed these from a well known orchestrator like vRO. With every day my python chops grew, but comments and git commits or not, I quietly and internally started to worry that only I knew how this worked. I think I was about half way through this when the ACI modules for Ansible got a serious upgrade by the Network to Code folks. I think on that day I must have looked like I had a stroke, because I was half happy and half sad. Visibly pained for all the lost time, but happy since I would be able to get something more maintainable into production. These python scripts would be intrinsically linked to me forever, whereas these Ansible playbooks were vastly more accessible.

At this point I had been telling everyone that we could, would and must have a reliable orchestration workflow in place before the first major deployment of our new platform. However, by now we were in fact one semi automated deployment down, the second of six was looming, and we were still surrounded by lots of components with "no glue". Ansible was that glue. In maybe 2 months I had not only reproduced all my work with the generic modules, but I had also finished off all the missing parts. We deployed our second site into Prod within I think 3 months, and I back-filled the first site with the critical automation a few months later (more due to time rather than any technical problem).

Having lived with that over the last few years the first cut of that Ansible playbook work is nowhere to be seen. Somewhere like a month or so after the first site was redeployed we hired a new engineer with some thirst for automation and over the following months he took over the lead role from me on the Ansible front. That was a godsend as I was also "managing" a team of 12 engineers around the world and the quotes there speak for themselves.

His first drive was to use AWX (the newly open sourced version of Ansible Tower) as a driving force rather than executing playbooks off a jumphost. In the learning of that, he designed a new structure to the git repo that broke the components up into pieces that were more manageable from a change and risk perspective, and mated that with Templates to execute specific playbooks with more comfort. Having had that go well, we bought proper tower and network device support for Ansible to backstop me and that guy. Enterprise support from RedHat keeps the CIO happy obviously. On completion of the first refactor, we then did some more work on repo variable structure, since there was still no CMDB in sight and the state in the repo was getting quite cumbersome. Today, all our sites are managed off this one git repo and maybe 8 or 9 Ansible Tower templates (playbooks). We still have a lot of state in the repo itself, and a drive to pull much of that repo state into Netbox instead of flat YAML. What we have works, but it won't last forever.

Above all else, the biggest issue we have come across with Ansible as a tool for ACI specifically, is not so much rolling things into the config, but pulling things out. Today, we have no reliable delete tooling. We can add, we can change, but we only ever delete by hand. Thankfully for us, we rarely delete anything, which I guess is a compounding factor for why we don't have much of a solution here either - necessity is the mother of all invention, and there is no major necessity for us to delete things. You would think that state: absent was all you needed, but you soon learn that for reliable rollback, you cant live on idempotency alone. Often you need to put logic into the hierarchical removal of things; i.e. you remove what you added in reverse order. Thus the safest thing to do is for each playbook rolling in, you do an equal and opposite directional one to roll back out. Again, all fine, providing your platform doesn't spit endless errors for things being removed that don't yet exist. ignore_errors: True is rarely a good idea, and tags might work instead, but we found it all very very cumbersome, and it never got out of the dev branch.

Technically, the right answer is the target platform needs to be more stable and consistent in its application of Idempotent actions, or live closer to true promise theory for its restful APIs, but I should technically be able to eat what I want and lose weight. In the real world things don't often work the way we want them too.

So having lived with a very fine and functional automation framework, delivering hand crafted configs via generically accessible tooling for a few years, I have nothing major to complain about in terms of the final product. I can do in a matter of hours what would take a week at least to do in the past. I know that when I ask the system to deliver that config from the normalised YAML values, it will either work or fail cleanly. I also know that I can adapt or change small things within a running config and not risk major disasters either.

Whilst I have little to complain about, being British, I do not find it hard to try.

What changed for me is that I was seconded into a new group in the business focusing on Public Cloud Transformation. Historically the public cloud was off limits to us due to commercial and financial reasons. The pay as you go setup didn't marry with the way we ran our business financially, which preferred capital purchases of assets. A change of leadership and some keen interest from a cloud friendly major shareholder, meant the shackles were off. I was instantly thrust into AWS and Azure and specifically I was knee deep in Terraform. I finally got to see real promise theory and started to envy the ability to force the running state to exactly match the described config. I immediately revisited the Networking modules and poked about the ACI ones, and was significantly underwhelmed. They were all community owned and seemed to be written to scratch a few itches rather than as a serious attempt at providing full support for the tool. I was transported 2 years back to my horrible hacky python. I was sad.

Over the course of the last 9 months or so I have come to really like the hashicorp tooling like terraform, packer and vagrant as an approach to immutable infrastructure, but I have also realised these are ultimately provisioning tools, and they have obvious limitations in day 2 operations. I can provision an asset today, I can bootstrap its config as well, but if something changes it outside of terraform, and then you rerun the tf apply, you can cause some significant drama. This wouldn't be so bad if the tf modules covered all the operations you need, but they don't.

I then see a lot of people chaining TF and Ansible together in CI/CD pipelines, as a sort of treatment to this limitation. I'm one of them too now. It works. It's using the right tool for the job rather than a leatherman for everything. It is just such a shame to look at the terraform workflow being so tight and conformist, then having the point and shoot (and pray) approach in Ansible following behind. It feels wrong.

So finally, we arrive at the juiciest meat on the barbeque today - YANG.

Funnily enough, YANG has nothing to do with networking itself. Read any YANG book (and I strongly suggest you read ![this one](https://lesen.amazon.de/kp/embed?asin=B07RMK59YC&preview=newtab&linkCode=kpe&ref_=cm_sw_r_kb_dp_sdm8Eb8H206Q6)), and all the examples for how YANG structures work together don't use anything networking related at all. YANG itself is a modelling language sort of like UML in programming. It allows you to create strongly typed structures that hold information about a thing, and its inter-relationships within a collection, known as a module.

What the IETF and OpenConfig (to name a few) have then done is use that modelling language to describe all the pieces and parts of networking entities, like a Physical Interface, IPv4 Routing and BGPv4 Unicast or BFD sessions. They went away and built up all the components of each protocol or service or component, and exposed that in a collection of YANG files in a ![big git repo](https://github.com/YangModels/yang).

Now of course we all know that whilst OSPF is OSPF, OSPF on Brocade doesn't always play well with OSPF on Cisco straight out of the box. Just because an RFC exists, doesn't mean that all vendors follow these to the letter. So to avoid the pain of having so many of these models out there with vendor specific augmentation, YANG allows you to load up a generic model, and then apply "deviations" over the top which overlay changes or appends additional items to the generated model. All you have to do is load that model up, insert your data where your implementation or the model itself requires values, and then this finished model is ready to be ingested by a platform, and applied. Each vendor has created a small engine as part of its netflow support that will take this model, convert it to the configuration parlance of that vendor and then apply that vendor specific config to the platform for you. In other words, you make a model saying you want an OSPF process with id 100 and redistribute-connected enabled, and when you send that to a cisco box, it will generate:

```cisco
router ospf 100
 redistribute connected subnets
```

You could then send that to a juniper or an extreme box and it will generate the equivalent parlance and apply it for you. This is why YANG modelling is so powerful. You say what you want in the abstract based on a domain specific knowledge and leave it to the receiving platform to apply the correct config.

At this point things get optional. YANG is just a blob of structured data in memory and that input needs to go to a device to be parsed and applied. For many, this takes the form of NETCONF, but could also be RESTCONF or gRPC for example.

RESTCONF is usually a point and shoot approach to a single device config or a single model deployment. This is due to its nature of hitting endpoints relating to the leaf of the model one call at a time. Think Ansible with multiple modules being hit in series to deliver a total output. Typically people use json as a request encapsulation scheme, often converted from the originally generated model XML. Due to this endpoint driven design, and the clunky movement between formats, it's actually not a common use case for most people; further reading is left to the reader as appropriate.

The Google approach is gRPC and this is apparently quite popular in Brocade/Extreme environments, using Protobuf as a request encapsulation (serialisation) scheme. gRPC itself is super lightweight and is very effective for use in the other side of YANG which is streaming telemetry, but I wont go there today. Protobufs themselves have a sort of handshake scheme menaing all participants share the schema of the data in transit first, and then the rest is binary data meeting that schema. As it is sort of unicorn snowflakey, again further reading is left to the reader.

What I will cover more here is YANG models rendered into python objects, serialised on the wire as XML and sent to the device using NETCONF (over ssh tcp/830) as the transport.

So as discussed we start out with a problem to solve. We want to configure a BGP peering session on a device we own with a device we don't. To do this, we know we need a few key bits of information:

1. Remote AS
2. Remote IP
3. Local IP
4. Local Interface for Local L3
5. Networks expected to receive
6. Networks to announce
7. Any BGP session settings (timers/graceful restart/bfd etc)
8. Any local BGP policies we need to apply (filters/route-maps etc)

The first 5 items are directly significant to the BGP peering session itself.
The 6th item is likely router specific, but could be session specific
The 7th is going to be locally standard with perhaps some session specific overrides
The 8th is almost certainly going to be locally standard with some specific overrides (always filter bogons, but maybe additional filters on an IXP peer vs transit, or application of peer significant internal community on learn for example).

Ultimately, you could easily build a YANG model once for all your BGP peering sessions, based on the IETF BGP model, supplemented by the router model and running version deviations, and use that as the base of all BGP sessions. You then create your common values for timers and BFD, use the route-map model to build your standard route maps, and then the access-list and prefix-list models to create your filters and associate them all together with the session itself. At the end, you have a YANG model with all the settings for the BGP session. If you haven't used anything in the vendor specific space, your model is also vendor independent. You can send this config to a Juniper MX, or a Cisco ASR and you get the same result - a BGP session.

Let that sink in a second.

You built a model in code, with all the settings you need, and you can send it to any NETCONF enabled device and it will do the needful for you. ZOMG YAY!
What happens under the hood is that the netconf agent in the device receives the model from your script, renders the model as its native configuration, and places it into a config candidate. You receive a confirmation back from the agent that the config candidate has been accepted and is ready for commit. You then have the choice to drop or apply the config to the running config. ZOMGWAT? Yes, you can get Juniper style commit and rollback features, on devices that don't necessarily expose that functionality to you in the same way.

Brilliant. Everything I always wanted in life. Give it to me! There is a catch however. YANG models are super, super janky to setup.

I have been chasing this dragon now (in personal time) for about 2 months. It took the above mentioned book to loosen the lid.

Big things for me that I would recommend to others is definitely, buy this book. Read it fully. I have not put an affiliate link in there, just buy it. Secondly start small. Get a vEOS or a CSR1000v up in GNS3 and point a MGMT VRF NIC at the NAT bubble and another one at a BGP container. Start with python on your box (or an Ubuntu box as a jump if you prefer), and get pyang and netconf-console installed. The latter is a major gamechanger for figuring this stuff out IMHO. In the book they cover a series of commands that have you pulling YANG models the device supports off the unit directly, saving them to your disk, then generating config in XML to push back as a change. This then opens the door to using blog content like that from ![NetworkOp](https://networkop.co.uk/). Prior to reading this book, and even with some podcast content and other blogs, I never really got what they were on about in the NetworkOp blogs, and all this compiling things and command line that went over my head. The book fixed this for me.

Once you figure out how to make and utilise those models as python bindings, or maybe ruby or go modules if that's your thing, the benefits start to open up really really fast. Being able to go from a few fields of data to a validated model, and then onto a vendor specific running config without touching those middle things, really is powerful.

Unfortunately, as I saw in my early adventures a few years back in Python, and Ansible, I guess we are up at the leading edge here, and so experience is mostly word of mouth and beard to beard. Over time, this barrier of entry will come down.

So in summary, my journey from nothing to something in the last 3 or so years has been interesting, but for those pushing into YANG now, I leave this high level plan with you:

* let the book teach you the difference between YANG and NETCONF. Even if you know that, do it anyways. its not a big book
* setup your environment to talk to something in a sandbox, and get used to pulling supported models out of the capabilities (hello packet), and then extracting and rendering them with pyang on the screen.
* get used to rendering python bindings from the extracted models and script up a basic config of something like a port.
* build out a full script to build a from scratch BGP session with lots of models, using minimal input data.
* put said data into a CMDB (like netbox) and then call the netconf using ansible (e.g. the napalm-netconf module).
* break your elements up to atomic units that match netbox CIs and use the netbox workflows to create, read, update and delete (CRUD) the atomic objects.
* celebrate.
