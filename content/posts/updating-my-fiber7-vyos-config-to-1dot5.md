+++
title = "My Updated Fiber7-X VyOS 1.5 Config"
date = "2024-06-12T13:00:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["vyos", "fiber7", "design", "linux-routing"]
keywords = []
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2024/06/updated-my-fiber7-vyos-config-to-1dot5.html']
+++

A while ago I wrote about my VyOS config for Init7's Fiber7-X product. Since then there has been a number of breaking changes, and a few additions that I would like to cover. 

I will copy/paste a lot of the narrative from that post, and avoid a bit of the abstract conversation that went with it, so that this stands on its own. 

If you have questions or comments, hit me up.

---

### The Initial Config

Trying to type in loads of things to the Console without copy/paste and all that stuff is sort of annoying, so I always go with a basic config that allows me to SSH in and complete the config in a terminal from my laptop. Firstly, I double checked the MAC addresses on the interfaces so that I knew which one was which. Thankfully the MGMT interface came up as eth0, and the CX4 ports were eth1 and eth2. Easy peasy.

Here are the basic commands I ran to sort out remote access:

```vyos
set interfaces ethernet eth0 address '10.31.74.1/27'
set interfaces ethernet eth0 description 'MGMT'
set system domain-name 'mydomain.co.uk'
set system host-name 'vyos01'
set service ssh listen-address 10.31.74.1
set service ssh port 22
```

Commit and Save and then swap to a terminal attached to the MGMT LAN

### The Network Design

My home network is quite simple really (for a network specialist). There is a MGMT interface (eth0), a WAN interface (eth1) and an inside interface (eth2). WAN is a routed port and MGMT is an untagged member of vlan10. eth2 is a dot1q trunk and carries two VLANs. All VLANs are represented on the switch, and most host ports are untagged members of VLAN9 for Home use. VLAN8 is there for the Lab to be able to use the 10G directly. This isn't possible outside of the Hypervisor yet, but i'm looking into alternatives to that CSS610 with at least 4 SFP+ ports to help with this.

Note: I like to use asciiflow for markdown friendly diagrams. Blogger seems to have a fixed width here, and this broke the ASCIIArt, so here you have a screenshot.

![VyOS Topology](/img/fiber7-x-vyos-config/vyos-topology.png)

Physically, there is quite the sprawl of stuff. I laid out all the assets that use a cabled connection only. The Wifi attached stuff (notably the tons of IoT things for HomeAssistant) would just clutter it all up even further.

![Full Network Topology](/img/fiber7-x-vyos-config/full-network-topology.png)

When you see it on a picture, it's kinda obvious that whilst logically simple, my network is far from simple in practical terms. A lot of it is accumulated crap I should probably destroy, but for now, this is what I am working with.

### The Core Config

Here we look at the minimum viable product configs. This is what you need to make VyOS work as a firewall/router in the home setting.

#### WAN Interface

```vyos
set interfaces ethernet eth1 address 'dhcp'
set interfaces ethernet eth1 description 'Init7'
set protocols static route 0.0.0.0/0 dhcp-interface 'eth1'
set system name-server 'eth1'
```

That is all you need to have the router itself talk to the internet. Now we need to setup the client side.

#### LAN Interface

My config has the inside configured as a trunk port and then a subinterface (vif) is configured for the VLAN:

```vyos
set interfaces ethernet eth2 vif 9 address '192.168.99.1/24'
```

Comically that is pretty much all you need to make the most simple router of all time. We don't want that tho. We want something a bit more useful than this!

#### Router Services

In my network, I delegate the internal DHCP Server role to a PiHole that runs on a Pi3 in the closet. I do use DHCP in the Management Zone tho, and that lives here on the VyOS box.

```vyos
set service dhcp-server listen-interface 'eth0'
set service dhcp-server shared-network-name mgmt authoritative
set service dhcp-server shared-network-name mgmt description 'MGMT'
set service dhcp-server shared-network-name mgmt option name-server '192.168.99.4'
set service dhcp-server shared-network-name mgmt option name-server '192.168.99.2'
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 subnet-id '2'
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 option default-router '10.31.74.1'
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 range scope1 start '10.31.74.2'
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 range scope1 stop '10.31.74.30'
```

Lets assume you want some specific IPs on things in that management zone so you can have polling in the future. Here is the couple of lines you need for a reservation in that segment

```vyos
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 static-mapping 02-core-sw ip-address '10.31.74.2'
set service dhcp-server shared-network-name mgmt subnet 10.31.74.0/27 static-mapping 02-core-sw mac '2c:c8:1b:6a:c8:8d'
```

Logging is important, so lets ensure its all enabled

```vyos
set system syslog global facility all level 'info'
set system syslog global facility local7 level 'debug'
```

Next up, to ensure our logs are helpful, we setup NTP to ensure the clock is synced at all times

```vyos
set service ntp server 0.ch.pool.ntp.org pool
set service ntp server 1.ch.pool.ntp.org pool
```

And to ensure that LLDP works both ways, we set this up on our internal interfaces:

```vyos
set service lldp interface eth0
set service lldp interface eth2.9
set service lldp management-address '10.31.74.1'
```

I, for reasons of lazyness would also like SSH to work over the Home interface

```vyos
set service ssh listen-address '192.168.99.1'
```

Finally, we want the OS to be tuned for performance, cos we are going to use a fast feed (may require reboot). A description of what this does is in the docs.

```vyos
set system option performance 'throughput'
```

### NAT Setup

There is not a lot of point having an IPv4 firewall/router if it doesnt do NAT. Here we do all the fun things and stuff to make IPv4 source NAT work. Here I use 77 as a rule ID 'prefix' and then have dedicated rules for each source subnet. There are other ways to do this, but this is more surgical and verbose is my preference when it comes to things like NAT and Firewalls.

```vyos
set nat source rule 771 outbound-interface name 'eth1'
set nat source rule 771 source address '192.168.99.0/24'
set nat source rule 771 translation address 'masquerade'
set nat source rule 772 outbound-interface name 'eth1'
set nat source rule 772 source address '10.31.74.0/24'
set nat source rule 772 translation address 'masquerade'
```

Now assuming we want to allow some inbound services from the internet, we need some destination NAT as well... I picked the rule number for a reason, but I guess its obvious this idea doesnt scale well. I kinda don't care that much tho ;)

```vyos
set nat destination rule 443 description 'HTTPS to Ingress'
set nat destination rule 443 destination port '443'
set nat destination rule 443 inbound-interface name 'eth1'
set nat destination rule 443 protocol 'tcp_udp'
set nat destination rule 443 translation address '192.168.99.252'
set nat destination rule 443 translation port '443'
```

### Zone Based Firewall

This is going to be a bit more chunky. ZBF is universally agreed to be more scalable, but it is also universally agreed to suck balls to configure. The first build is annoyingly slow, but once it is in, adding interfaces into zones is trivial and policy can be inherited/deduplicated, which I think is a big win. I spent a little bit of time on automating this in Nornir, which I will publish soon. I think I could probably write an entire essay on ZBF concepts, so I will save you my ramblings and refer you to the Docs instead: VyOS Zone-Based Firewall Guide

#### ZBF - Basics

A lot of this is probably default config, but since VyOS configdb is idempotent, if you paste a command that already exists, it will just skip on past.

```vyos
set firewall global-options all-ping 'enable'
set firewall global-options broadcast-ping 'disable'
set firewall global-options ip-src-route 'disable'
set firewall global-options log-martians 'enable'
set firewall global-options receive-redirects 'disable'
set firewall global-options send-redirects 'enable'
set firewall global-options source-validation 'disable'
set firewall global-options syn-cookies 'enable'
set firewall global-options twa-hazards-protection 'disable'
set system conntrack modules ftp
set system conntrack modules h323
set system conntrack modules nfs
set system conntrack modules pptp
set system conntrack modules sip
set system conntrack modules sqlnet
set system conntrack modules tftp
```

I also defined a group for inside networks that can be used later on. Feel free to define as many network and service groups as you like to clean up your config.

```vyos
set firewall group network-group inside-nets network '192.168.99.0/24'
set firewall group network-group inside-nets network '10.31.74.0/27'
```

#### ZBF - Policies

So we need policies before we can apply those to zones. The Zone assignments refer to the policy names, so lets make them first.

In each policy I use a default-action drop, which is a sort of best practice. In more trusted settings, you might want a default permit (outbound from LAN to WAN for example), and typically people will use a default-action of accept. I prefer to put a drop policy in and then a single rule to permit all at rule 1, since if something horrible happens and I need to block something outbound (malware C2 or somesuch), I can insert a rule above to drop something specific. I can also change that rule 1 to only permit certain ports or protocols and drop all others. Basically, better set a sane default and override it, than have to change the whole policy in an emergency, which might have undesired consequences, thus compounding an already shitty situation.

I like to put a description in but lets face it, its pretty self describing.

I also enable the default log statement so all firewall rule hits are logged. This is again a good best practice. You wont care until you need this and then future you will thank you for this.

#### ZBF - Policies - Local

Local policies are those that originate or terminate on the VyOS instance directly. We need traffic in both directions (inbound and outbound) from and to the router, so we have four policies here.

```vyos
set firewall ipv4 name lan-local-v4 default-action 'drop'
set firewall ipv4 name lan-local-v4 default-log
set firewall ipv4 name lan-local-v4 description 'LAN to This Router IPv4'
set firewall ipv4 name lan-local-v4 rule 1 action 'accept'
set firewall ipv4 name lan-local-v4 rule 1 description 'explicit allow inbound ssh always (anti-lockout)'
set firewall ipv4 name lan-local-v4 rule 1 destination port '22'
set firewall ipv4 name lan-local-v4 rule 1 protocol 'tcp'
set firewall ipv4 name lan-local-v4 rule 1 source group network-group 'inside-nets'
set firewall ipv4 name lan-local-v4 rule 2 action 'accept'
set firewall ipv4 name lan-local-v4 rule 2 description 'explicit allow dhcp'
set firewall ipv4 name lan-local-v4 rule 2 destination port '67-68'
set firewall ipv4 name lan-local-v4 rule 2 protocol 'udp'
set firewall ipv4 name lan-local-v4 rule 2 source port '67-68'
set firewall ipv4 name lan-local-v4 rule 3 action 'accept'
set firewall ipv4 name lan-local-v4 rule 3 description 'default allow from known nets to router'
set firewall ipv4 name lan-local-v4 rule 3 destination address-mask '0.0.0.0'
set firewall ipv4 name lan-local-v4 rule 3 source group network-group 'inside-nets'

set firewall ipv4 name local-lan-v4 default-action 'drop'
set firewall ipv4 name local-lan-v4 default-log
set firewall ipv4 name local-lan-v4 description 'This Router to LAN IPv4'
set firewall ipv4 name local-lan-v4 rule 2 action 'accept'
set firewall ipv4 name local-lan-v4 rule 2 description 'allow dhcp'
set firewall ipv4 name local-lan-v4 rule 2 destination port '67-68'
set firewall ipv4 name local-lan-v4 rule 2 protocol 'udp'
set firewall ipv4 name local-lan-v4 rule 2 source port '67-68'
set firewall ipv4 name local-lan-v4 rule 3 action 'accept'
set firewall ipv4 name local-lan-v4 rule 3 description 'default allow from known nets to router'
set firewall ipv4 name local-lan-v4 rule 3 destination address-mask '0.0.0.0'
set firewall ipv4 name local-lan-v4 rule 3 source group network-group 'inside-nets'

set firewall ipv4 name wan-local-v4 default-action 'drop'
set firewall ipv4 name wan-local-v4 default-log
set firewall ipv4 name wan-local-v4 description 'WAN to This Router IPv4'
set firewall ipv4 name wan-local-v4 rule 1 action 'accept'
set firewall ipv4 name wan-local-v4 rule 1 state 'established'
set firewall ipv4 name wan-local-v4 rule 1 state 'related'
set firewall ipv4 name wan-local-v4 rule 2 action 'drop'
set firewall ipv4 name wan-local-v4 rule 2 state 'invalid'
set firewall ipv4 name wan-local-v4 rule 3 action 'accept'
set firewall ipv4 name wan-local-v4 rule 3 description 'DHCPv4 replies'
set firewall ipv4 name wan-local-v4 rule 3 destination port '67,68'
set firewall ipv4 name wan-local-v4 rule 3 protocol 'udp'
set firewall ipv4 name wan-local-v4 rule 3 source port '67,68'

set firewall ipv4 name local-wan-v4 default-action 'drop'
set firewall ipv4 name local-wan-v4 default-log
set firewall ipv4 name local-wan-v4 description 'This Router to WAN IPv4'
set firewall ipv4 name local-wan-v4 rule 1 action 'accept'
```

Inbound policies (lan-local and wan-local) are all about things talking to the router directly. LAN side I allow anything and WAN side we block everything not related to an existing "inflight" outbound conn, but I also had to enable the inbound DHCP flows since these are stateless.

Outbound I again just permit all the things.

#### ZBF - Policies - Transit

Transit policies are ones where the flow is designed to transit through the router. In iptables world these are FORWARD rules.

```vyos
set firewall ipv4 name lan-wan-v4 default-action 'drop'
set firewall ipv4 name lan-wan-v4 default-log
set firewall ipv4 name lan-wan-v4 description 'LAN to WAN IPv6'
set firewall ipv4 name lan-wan-v4 rule 1 action 'accept'

set firewall ipv4 name wan-lan-v4 default-action 'drop'
set firewall ipv4 name wan-lan-v4 default-log
set firewall ipv4 name wan-lan-v4 description 'WAN to LAN IPv6'
set firewall ipv4 name wan-lan-v4 rule 1 action 'accept'
set firewall ipv4 name wan-lan-v4 rule 1 state 'established'
set firewall ipv4 name wan-lan-v4 rule 1 state 'related'
set firewall ipv4 name wan-lan-v4 rule 2 action 'drop'
set firewall ipv4 name wan-lan-v4 rule 2 state 'invalid'
set firewall ipv4 name wan-lan-v4 rule 443 action 'accept'
set firewall ipv4 name wan-lan-v4 rule 443 description 'internet to ingress'
set firewall ipv4 name wan-lan-v4 rule 443 destination address '192.168.99.252'
set firewall ipv4 name wan-lan-v4 rule 443 destination port '443'
set firewall ipv4 name wan-lan-v4 rule 443 protocol 'tcp_udp'
```

As well as the default log and description, I have a rule that matches our destination NAT rule we defined previously. Notice how we use the "real" IP of the host on the inside. This can catch some people out who are used to firewall rules referring to the mapped IP on the outside of the firewall. I chose the same rule ID as the NAT rule ID, but that is my personal choice, there is no requirement to syncronise these rule IDs.

#### ZBF - Zones

Now that we have policies, we can assign these policies to zones. Zones and interfaces are a one-to-many mapping, i.e. one Zone can contain many interfaces, but for the avoidance of doubt, an interface can only exist in one Zone ;)

```vyos
set firewall zone wan default-action 'drop'
set firewall zone wan from lan firewall name 'lan-wan-v4'
set firewall zone wan from local firewall name 'local-wan-v4'
set firewall zone wan interface 'eth1'

set firewall zone lan default-action 'drop'
set firewall zone lan from local firewall name 'local-lan-v4'
set firewall zone lan from wan firewall name 'wan-lan-v4'
set firewall zone lan interface 'eth2.9'
set firewall zone lan interface 'eth0'

set firewall zone local default-action 'drop'
set firewall zone local from lan firewall name 'lan-local-v4'
set firewall zone local from wan firewall name 'wan-local-v4'
set firewall zone local local-zone
```

Here we created our three zones called wan, lan and local. We then assigned the firewall policies "inbound", and finally assigned interfaces to the zones. Note the special "local-zone" flag for that local zone.

A lot of people then go on to ask, what happens to traffic between interfaces within one zone, in our example. eth0 and eth2.9. These flows are unrestricted - think of them as a plain routed interface.

_Note: MGMT and User traffic? Routed? No FW? WHATTTT? - This is my house, I don't care. Don't @ me._

At this point you should have a fully operational internet connection on eth0 and eth2.9, providing you legacy internet access. Whats legacy internet access? IPv4 only of course!

### Enabling IPv6

First, I should point out that my approach here is rather opinionated. Some will argue it is literally stupid. I also do not care about this ;)

The point of IPv6 was to deliver enough internet addressing that we could never run out like we have with IPv4. A side benefit of this was that NAT was essentially deprecated, since we dont need to share one public IP with many clients inside a LAN zone. For many, the obvious benefit is that all devices in your LAN get a public routable IPv6 address, and you control the flows on the border with a firewall, just like normal. In my workplace we have dual internet feeds, and we get IPv6 addressing from both. Which IP address pool should we use to assign IPv6 addresses to clients? If both, which should a client machine use to route outbound?

In some settings, the SLAAC addressing system makes a lot of sense, but in an enterprise, I still need concrete services, which means predictable, reliable DNS/IPs. Ultimately, I will find myself using some form of static addressing. Now, what happens when I change ISP? That /48 I got from the first one is now not mine, and I have to re-address my entire LAN. This is unsustainable.

Thankfully, someone thought of all of these use cases and came up with Unique Local Addressing, and Prefix Translation (sometimes known as NAT66) as a solution. With ULA, I can generate (and optionally register) a ULA prefix that should be globally unique for my site, and use that for all internal addressing purposes. I then configure NAT66 to swap the first 48 or 64 bits of my 128 bit IPv6 address to one from our IPv6 Prefix delegation we recieved from upstream. If the ISP assigns us something new, or we change ISP (via failover or physically replacing a supplier), the prefix substitution just does the work for me. So inside my LAN, I have a private address, from a single site specific prefix, and then as traffic heads outside my LAN I have an otherwise identical address, but with an internet routable prefix.

e.g. Inside my LAN my PC looks like this:

```shell
% ifconfig
   en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=50b<RXCSUM,TXCSUM,VLAN_HWTAGGING,AV,CHANNEL_IO>
        ether 00:3e:e1:c9:28:10
        inet6 fe80::ca2:34c4:8b00:daa8%en0 prefixlen 64 secured scopeid 0x4
        inet6 fda4:7911:df45:9:4cc:2a5a:8ee5:a917 prefixlen 64 autoconf secured
        inet 192.168.99.111 netmask 0xffffff00 broadcast 192.168.99.255
        nd6 options=201<PERFORMNUD,DAD>
        media: autoselect (1000baseT <full-duplex>)
        status: active
```

but on the internet, I am seen as:

```shell
% curl -6 https://ifconfig.co
2a02:168:4047:9:4cc:2a5a:8ee5:a917
```

All that happened is we swapped that first 48 bits.

Now some gamers might be screaming at me right now, I have no idea if this is good or bad for you guys, but given it is a 1:1 mapping, it is probably ok tbh. We will see in the comments I guess.

#### IPv6 - Interfaces

Init7 offer a /48 and in my case, whilst it is assigned via DHCPv6, this assignment is reserved for me in their system. It's logically static. Your mileage may vary, and I know of at least two other init7 customers who had their /48 change, but randomly. This in essence is why I follow this NAT process btw. So first we need to enable dhcpv6 on our internet interface and request an IPv6 prefix delegation that we will anchor to an inside interface for operational reasons. If there were a better way to do this, i would, since we dont actually use this IP on this interface in daily use, but we need to anchor it somewhere. If you are concerned, you could bind this to a non existent VLAN maybe.

```vyos
set interfaces ethernet eth1 address 'dhcpv6'
set interfaces ethernet eth1 dhcpv6-options pd 0 interface eth2.9 address '1'
set interfaces ethernet eth1 dhcpv6-options pd 0 length '48'
set interfaces ethernet eth1 ipv6 address autoconf
```

So our WAN interface uses SLAAC for the interface IP, getting an address in a /64 owned and operated by Init7. We then use DHCPv6 to request an IPv6 /48 that we can assign to the inside interfaces of our router.

Here I am cheating and statically assigning the :9::1/64 from our site /48 to the vlan 9 under the eth2 interface. I follow this scheme for all VLANs since I am unimaginative.

I assume you have gone and got your own ULA prefix and registered it. Dont use mine!

```vyos
set interfaces ethernet eth2 vif 9 address 'fda4:7911:df45:9::1/64'
```

When you commit this, you can query your eth2.9 interface to see the PD you got. Here you see I got 2a02:168:4047::9/64. I am hopeful there is a better way to do this. Right now I cant see it tho.

```shell
jhow@rtr-iojh-vyos01:~$ show interfaces ethernet eth2.9
eth2.9@eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 50:6b:4b:1c:09:fb brd ff:ff:ff:ff:ff:ff
    inet 192.168.99.1/24 brd 192.168.99.255 scope global eth2.9
       valid_lft forever preferred_lft forever
    inet6 2a02:168:4047::9/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fda4:7911:df45:9::1/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::526b:4bff:fe1c:9fb/64 scope link
       valid_lft forever preferred_lft forever
```

#### IPv6 - NAT66

Here we do the magic prefix switcheroo. I have a rule for each vlan ID so I can swap the /64 in a granular way. Realistically, I could compress this to a /48 on a single rule, but again, I like to be verbose so people know exactly what is happening at each stage.  

```vyos
set nat66 destination rule 9 destination address '2a02:168:4047:9::/64'
set nat66 destination rule 9 inbound-interface name 'eth1'
set nat66 destination rule 9 translation address 'fda4:7911:df45:9::/64'
set nat66 source rule 9 outbound-interface name 'eth1'
set nat66 source rule 9 source prefix 'fda4:7911:df45:9::/64'
set nat66 source rule 9 translation address '2a02:168:4047:9::/64'
```

#### IPv6 - Router Advertisements

Setup Router Advertisements on your inside interface, ensuring internal clients get a random SLAAC address from my ULA pool, and DNS Servers.

```vyos
set service router-advert interface eth2.9 name-server '2606:4700:4700::1111'
set service router-advert interface eth2.9 name-server '2606:4700:4700::1001'
set service router-advert interface eth2.9 prefix fda4:7911:df45:9::/64 no-onlink-flag
# new addition since 1.5
set service ndp-proxy interface eth1 prefix fda4:7911:df45:9::/64 mode 'static'

```

#### IPv6 - Firewalling

Here it is essentially a copy paste of the v4 rules from earlier but edited to a v6 approach.

```vyos
set firewall global-options ipv6-receive-redirects 'disable'
set firewall global-options ipv6-src-route 'disable'

set firewall ipv6 name local-lan-v6 default-action 'drop'
set firewall ipv6 name local-lan-v6 default-log
set firewall ipv6 name local-lan-v6 description 'This Router to LAN IPv6'
set firewall ipv6 name local-lan-v6 rule 1 action 'accept'
set firewall ipv6 name local-lan-v6 rule 1 destination address-mask '::'

set firewall ipv6 name lan-local-v6 default-action 'drop'
set firewall ipv6 name lan-local-v6 default-log
set firewall ipv6 name lan-local-v6 description 'LAN to This Router IPv6'
set firewall ipv6 name lan-local-v6 rule 1 action 'accept'
set firewall ipv6 name lan-local-v6 rule 1 destination address-mask '::'

set firewall ipv6 name local-wan-v6 default-action 'drop'
set firewall ipv6 name local-wan-v6 default-log
set firewall ipv6 name local-wan-v6 description 'This Router to WAN IPv6'
set firewall ipv6 name local-wan-v6 rule 1 action 'accept'

set firewall ipv6 name wan-local-v6 default-action 'drop'
set firewall ipv6 name wan-local-v6 default-log
set firewall ipv6 name wan-local-v6 description 'WAN to Thie Router IPv6'
set firewall ipv6 name wan-local-v6 rule 1 action 'accept'
set firewall ipv6 name wan-local-v6 rule 1 state 'established'
set firewall ipv6 name wan-local-v6 rule 1 state 'related'
set firewall ipv6 name wan-local-v6 rule 2 action 'accept'
set firewall ipv6 name wan-local-v6 rule 2 protocol 'ipv6-icmp'
set firewall ipv6 name wan-local-v6 rule 3 action 'accept'
set firewall ipv6 name wan-local-v6 rule 3 description 'DHCPv6 replies'
set firewall ipv6 name wan-local-v6 rule 3 destination port '546'
set firewall ipv6 name wan-local-v6 rule 3 protocol 'udp'
set firewall ipv6 name wan-local-v6 rule 3 source port '547'


set firewall ipv6 name lan-wan-v6 default-action 'drop'
set firewall ipv6 name lan-wan-v6 default-log
set firewall ipv6 name lan-wan-v6 description 'LAN to WAN IPv6'
set firewall ipv6 name lan-wan-v6 rule 1 action 'accept'

set firewall ipv6 name wan-lan-v6 default-action 'drop'
set firewall ipv6 name wan-lan-v6 default-log
set firewall ipv6 name wan-lan-v6 description 'WAN to LAN IPv6'
set firewall ipv6 name wan-lan-v6 rule 1 action 'accept'
set firewall ipv6 name wan-lan-v6 rule 1 state 'established'
set firewall ipv6 name wan-lan-v6 rule 1 state 'related'
set firewall ipv6 name wan-lan-v6 rule 2 action 'accept'
set firewall ipv6 name wan-lan-v6 rule 2 protocol 'ipv6-icmp'
set firewall ipv6 name wan-lan-v6 rule 443 action 'accept'
set firewall ipv6 name wan-lan-v6 rule 443 description 'internet to ingress'
set firewall ipv6 name wan-lan-v6 rule 443 destination port '443'
set firewall ipv6 name wan-lan-v6 rule 443 destination address 'fda4:7911:df45:9::252'
set firewall ipv6 name wan-lan-v6 rule 443 protocol 'tcp_udp'
```

I think most of this is self explanatory with the one small exception being we had to swap our DHCPv4 rule for a DHCPv6 rule in the wan-local-6 policy. Same problem, different execution.

#### IPv6 - Zones

```vyos
set firewall zone lan from local firewall ipv6-name 'local-lan-v6'
set firewall zone wan from local firewall ipv6-name 'local-wan-v6'
set firewall zone local from lan firewall ipv6-name 'lan-local-v6'
set firewall zone local from wan firewall ipv6-name 'wan-local-v6'
set firewall zone lan from wan firewall ipv6-name 'wan-lan-v6'
set firewall zone wan from lan firewall ipv6-name 'lan-wan-v6'
```

As before, we assign our policies to our zones.

At this point you should now have a fully operational dual-stack router.

### tv7 - Multicast

Last on my list here is the multicast work to allow TV7 to work on wired clients in vlan9. You can add other vlans if you like.

First we need an IGMP Proxy. I translated these settings from the edgerouter configs shared by Init7 themselves. The EdgeRouter is a fork of vyatta, just like VyOS, so it should be identical. I found I didnt need the alt-subnet on the downstream side tho.

```vyos
set protocols igmp-proxy interface eth1 alt-subnet '0.0.0.0/0'
set protocols igmp-proxy interface eth1 role 'upstream'
set protocols igmp-proxy interface eth2.9 role 'downstream'
```

Secondly we need firewall rules. The rules from init7 are pretty broad, so I went more specific.

Something a little interesting is that in an early release of 1.4, the IGMP proxy was a literal man in the middle, and so the firewall policy had to be `wan-local`. Somewhere along the line this changed and I had to reproduce the rules in the `wan-lan` zone. I keep both because it works, but its highly probably the `wan-lan` is all you need here. 

```vyos
set firewall ipv4 name wan-local-v4 rule 771 action 'accept'
set firewall ipv4 name wan-local-v4 rule 771 description 'tv7'
set firewall ipv4 name wan-local-v4 rule 771 destination address '239.77.0.0/16'
set firewall ipv4 name wan-local-v4 rule 771 destination port '5000'
set firewall ipv4 name wan-local-v4 rule 771 protocol 'udp'
set firewall ipv4 name wan-local-v4 rule 772 action 'accept'
set firewall ipv4 name wan-local-v4 rule 772 description 'tv7'
set firewall ipv4 name wan-local-v4 rule 772 protocol 'igmp'

set firewall ipv4 name wan-lan-v4 rule 771 action 'accept'
set firewall ipv4 name wan-lan-v4 rule 771 description 'tv7'
set firewall ipv4 name wan-lan-v4 rule 771 destination address '239.77.0.0/16'
set firewall ipv4 name wan-lan-v4 rule 771 destination port '5000'
set firewall ipv4 name wan-lan-v4 rule 771 protocol 'udp'
set firewall ipv4 name wan-lan-v4 rule 772 action 'accept'
set firewall ipv4 name wan-lan-v4 rule 772 description 'tv7'
set firewall ipv4 name wan-lan-v4 rule 772 protocol 'igmp'
```

And that is it. A fully operational VyOS config for Init7. 

Please let me know your comments and thoughts. I am always willing to learn.
