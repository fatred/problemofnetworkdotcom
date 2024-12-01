+++
title = "A Modern NetDevOps learning plan"
date = "2020-10-20T10:19:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["Golang","Training","Opinion","Development","Planning","Python",]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2020/10/a-modern-netdevops-learning-plan.html']
+++
Now that I have ![a load of free time](https://problemofnetwork.com/posts/end-of-solera-years/) I have set up a sort of a learning plan to plug some gaps in my knowledge that I always wanted to plug, but didn't have the time to do properly.

Mindful that I have an opportunity for learning without external influences on my time, but also that my Daughter only has Kindergarten until lunchtime every day, I really only get the morning before the family time kicks in.

So, every Morning I do my German spoken practice while walking the dog. This clearly amuses various folks, but maybe in a few weeks i'll be able to explain to them in language beyond that of a 5 yo child what it is I am actually doing.

I then put aside 2-3h to do some programming - Python and Go on alternating days. This is a major boost for me since I have had periods in my career when I have been able to get into programming and found it to be immeasurably useful in my day to day work, particularly in speeding up weird and wonderful tasks, but I never really got the time to round out that skill beyond point based problem solving. There was a little period maybe 4 years ago when we re starting the major ACI rollouts that I was writing raw python interacting with the ACI Model Object tree, but it never really got to what I would call Advanced levels.

Typically my python work starts out as a simple script to do one thing, then I refactor it to functions and workflows and eventually I will either modularise it, or hand it off to someone more knowledgeable in the team to improve.

Flask is an area that really interests me since making my random CLI scripts into something presentable in a Web UI or more realistically, exposing it as a REST API.  Presenting my artisinal hand crafted nonsense as an API endpoint would probably increase the takeup of my "little shortcuts" since most of the team understood how to talk to a REST endpoint, but had little interest in hacking up my python abortions.

Go was never really part of my approach until I got into Kubernetes, and found myself needing to at least understand what code did. The more I read, the more I came to realise Go is at the intersection of old school compiled C and modern interpreted Python. Having tried and failed a number of times to learn C to a respectable level, Go seems like a unique opportunity to revisit this.

As a PacktPub subscriber, I started with ![Learning Go Progamming](https://www.packtpub.com/product/learning-go-programming/9781784395438) which was a little dry, but got me moving, and to reinforce that beginner knowledge, I am half way through the ![Go Essential Training](https://www.linkedin.com/learning/go-essential-training) course on LinkedIn, which is working out quite well.

I use a pomodoro technique for the learning time, splitting the sessions into three 45 minute blocks with breaks inbetween to let the brain cool off. I had done this for the original 25/5 mins approach ever since a colleague back in the UK showed me, and whilst it worked, a well known security researcher I follow ![had a thread](https://twitter.com/Fox0x01/status/1317120678617907200) on this topic recently, which resonated with me. Having moved to three sessions of 45 min on, 10 mins off, I find I take more in and retain it.

One very interesting thing I hit this week was a simple challenge in the Go training, which I will write up separately, but from problem statement to working Go code took me about 40 minutes. To then translate that to python3 took me about 4 minutes. That was a major surprise to me. Firstly I thought that was pretty decent for writing in a language I didn't know. Secondly to then take that logic and just apply it into Python3 in such a short amount of time, it really hit me that I have a lot of Python knowledge in there that I just don't credit myself with. I'm absolutely no expert, and the code I write is still a dumpster fire, but at least I can do it quickly.

Once the coding session is done I then have about 1-2h left for Network Lab time. This setup I will post also separately, but needless to say, like all the learning things I do, its sorta over the top stupid.

On a Wednesday I substitute the programming specifically for time in my network telemetry lab. Here I have prometheus setup in my kubernetes cluster as a TSDB, and then I use Cisco's Pipeline project, and things like telegraf to funnel streaming metrics from whatever I can get that runs it into prom so that I can then build out dashboards and the like in Grafana. Once I reach a critical mass on that, I will probably pivot slightly to then build multi metric alerting rules in alert manager. My hope is to find some set of metrics that provide value here to then write a Python/Go app that constantly monitors historical metrics to generate new moving markers for alerting. Idea being, you can use the past to track your baselines and do some basic anomaly detection, without having to go full ML, or buy an expensive product.

At lunchtime it will then be time to pick up my daughter and spend the afternoon entertaining her. When I get my new job I expect that I will be very focussed on that for a little while. It therefore makes sense to use as much of this current free time to top up the good will piggy bank, knowing that I will have to borrow some of that back for a short while as I settle in again.

My first hope at the end of this is to finally get my German up to a level that is more communicative than basics in the shops. My second hope is that I can really embed some of this programming knwoeldge into the front of my brain rather than leaving it busied at the back, covered in cobwebs. The final hope is that I can put all of this together into a compelling package for a company to hire me. Time will tell.
