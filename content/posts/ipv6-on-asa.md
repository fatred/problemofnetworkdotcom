+++
title = "IPv6 on Cisco ASA"
date = "2015-05-22T00:25:00+00:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["IPv6", "SLAAC", "ASA"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
This is the first in what will no doubt be a many part series on deploying IPv6.  This is a problem I have had to overcome recently, and still to this day I battle with the implementation subtleties.  If you find yourself in my shoes, I hope you find this helpful.

So, firstly, you need to consider your architecture. Mine, I am reliably told is totally wrong.

I use ASAs as routers in the campus.  Why? Cos shut up - that's why!  I like to maintain user group segmentation, and I also like to maintain security policy quite tightly.  Therefore I put all users in VLANs managed by 802.1x, and all those VLANs gateway on an ASA trunk subinterface.  I then provide an appropriately sized routed L3 subnets from a site specific IPv4 /21 and have ACLs as access-groups.  Voila - policy is maintained at the gateway border.

To those of you who do it in transparent mode, and/or use Layer 3 Switching with ACLs, I salute you.  This works for me tho ;)

The quickest way to get you from nothing to something in IPv6 world is to take on board the following:

* You should totally forget subnet sizing for any campus deployment. Your networks will get a /64 allocated from a site specific /48.
* You should use Stateless Automatic Address Configuration (SLAAC), in conjunction with static addressing.
* You should make sure your DNS infrastructure is configured for auto registration and crucially from trusted devices only.
* You should have strongly considered your firewalling policy.

The last point there is the key one.  IPv6 is not designed for NAT.  Therefore, you cannot rely on NAT/PAT as a way of masquerading your hosts from prying eyes.  Remember, you are only ever a few milliseconds away from every bad guy on the internet these days!

So, back to the ASA.  For me, the most logical thing was to put /64 prefixes onto the existing VLAN sub-interfaces alongside my IPv4 /26s.  This effectively Dual Stacks my ASA, and allows the end device to deterministically select the most appropriate path.

_Note that unless otherwise instructed, most application stacks will attempt IPv6 DNS lookups, and therefore prefer IPv6 communication when available. Java is the only notable exception I can think of._

Assigning IPv6 to an interface is dead easy.  Assume we have 2aaa:2bbb:2ccc::/48 as our site prefix.

```cisco
interface Port-channel2.100
  vlan 100
  nameif MGMT
  security-level 100
  ip address 10.0.0.1 255.255.255.192 standby 10.0.0.2
  ipv6 address 2aaa:2bbb:2ccc:100::1/64
  ipv6 address fe80::fac2:88ff:feac:d2e5 link-local standby fe80::fac2:88ff:feac:d4e5
  ipv6 enable
  ipv6 nd prefix 2aaa:2bbb:2ccc:100::/64 86400 86400
```

Lets look at that for a moment.  I will ignore the standard interface stuff - port channels are optional as is the failover config

* First we assign our Global IPv6 address to the interface.  This is the publicly routable address.
* Next, we need our unique link-local addresses.  This will be explained shortly.
* This third line turns IPv6 on for the interface
* Finally, we set up SLAAC neighbour discovery on this L2 VLAN, letting other endpoints know that we are a router for this prefix.  Think of it as a one liner for essential DHCP style network address distribution.

Next, we need a route.
