+++
title = "Doing YANG Wrong - Part 1: Getting started"
date = "2020-06-25T15:28:00+02:00"
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
aliases = ['/2020/07/doing-yang-wrong-part-1-getting-started.html']
+++
This is more of a discovery post than anything particularly useful. It charts my adventure from a problem statement, through discovery of a solution, to a dead end. I post it here to help people see the logic I followed, and why, against all wisdom, it didn't work.

Note that if you are familiar with YANG, I know why this is NOT the right way to do this, but I only know that having done this as shown here. Please don't waste your time in the comments with nerdsplaining that to me. This is to show people a process and to help people who feel stupid when this stuff doesnt work, to see that they're not alone. I learn by making mistakes.

Firstly, the build environment. This is all done in GNS3.

![GNS3 Topology](/img/doing-yang-wrong/gns3-environment.png)

Up top, we have a management switch (dumb GNS3 default L2 device), all ports in a single VLAN and a management host. This is an Ubuntu 18.04 VMDK with two NICs, one on the MGMT switch, and one attached to a NAT bubble on my host. The NAT NIC is DHCP with default route enabled and the other NIC is in the MGMT L3 as .254 with no routes obvs.

Underneath I have a Cumulus VX node for backend routing on the lower left, an Arista vEOS for switching and a bit of VRF local routing tests in the middle, and the frontend on the right is a CSR1000v. We then pretend to have a BGP session with a supplier on the far right. This I manually configured to just default-originate for now. That internet cloud is just an image... Each box has an interface in the MGMT L2 configured into a MGMT VRF on that box, so we don't taint any default FIB/RIB stuff we do with the BGP processes.

The idea is that the supplier sends a default route to the front end via eBGP, and all frontends (if we had lots) peer with the backends so they get all the paths they need to make a route calculation decision. The workloads live behind these backends. The switch in the middle only does L2 work in the primary use case. The idea of putting an Arista box in was to try out the VRF capabilities and maybe try to use VRFs as a sort of built-in backend tier, eliminating the VMs in that backend. More on that in another post i'm sure.

So the aim is to configure the CSR1000v interface that faces the vendor from nothing to everything. For reasons its not worth going into in detail yet, I will try to use openconfig models for everything as well.

I should need at the least, the following:

1. interface config
2. interface IP address
3. prefix-list outbound
4. as-path filter inbound
5. bgp peer-group
6. bgp peer

## Part 1: The fetching of the models

I started with what I picked up from the Yang book. I can use netconf-console to query the supported models from the hello packet, fetch these down to my machine and then render them on the screen with pyang. This sort of worked. I SSHed from my machine into the management box in the simulation first, then installed all the tools I knew I needed.

```shell
    sudo apt install python-pip python3-pip build-essential libncurses5-dev xml-twig-tools
    pip install lxml netconf-console pyang pyangbind
```

(depending on what base OS you have you might need other things. google can help you).

I setup netconf-yang on the CSR1000v as well.

```cisco
    aaa new-model
    aaa authentication login default local
    aaa authentication enable default none
    aaa authorization console
    aaa authorization exec default local
    aaa authorization network default local

    username yangconf priv 15 sec 0 my_good_password.

    ip vrf mgmt
     description management
     rd 900:1

    interface GigabitEthernet1
     ip vrf forwarding mgmt
     ip address 192.168.70.21 255.255.255.0

    netconf-yang
```

To then validate that the system was working, I used netconf-console to query the capabilities from the CSR.

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --hello 

    <bla bla bla bla> 

         <nc:capability>urn:ietf:params:xml:ns:yang:smiv2:CISCO-RF-MIB?module=CISCO-RF-MIB&amp;revision=2005-09-01</nc:capability>
        <nc:capability>urn:ietf:params:xml:ns:yang:smiv2:CISCO-VLAN-MEMBERSHIP-MIB?module=CISCO-VLAN-MEMBERSHIP-MIB&amp;revision=2007-12-14</nc:capability>
        <nc:capability>urn:ietf:params:xml:ns:yang:smiv2:BRIDGE-MIB?module=BRIDGE-MIB&amp;revision=2005-09-19</nc:capability>
        <nc:capability>urn:ietf:params:xml:ns:yang:smiv2:CISCO-IP-TAP-MIB?module=CISCO-IP-TAP-MIB&amp;revision=2004-03-11</nc:capability>
      </nc:capabilities>
    </nc:hello>
```

Yay. So what I need to do now is find a model that will let me do an interface IP (part 1 of my problem). Easy way to make sure you have a supported model is filter the hello.

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --hello | grep openconfig | grep ip

        <nc:capability>http://openconfig.net/yang/interfaces/ip?module=openconfig-if-ip&amp;revision=2018-01-05&amp;deviations=cisco-xe-openconfig-if-ip-deviation,cisco-xe-openconfig-interfaces-deviation</nc:capability>
        <nc:capability>http://openconfig.net/yang/cisco-xe-openconfig-if-ip-deviation?module=cisco-xe-openconfig-if-ip-deviation&amp;revision=2017-03-04</nc:capability>
        <nc:capability>http://openconfig.net/yang/interfaces/ip-ext?module=openconfig-if-ip-ext&amp;revision=2018-01-05</nc:capability>
```

So we can see there is a module called openconfig-if-ip. Lets pull it down. We learned in the book we can read that model from the box, pipe it to some xml extractor and then redirect that model to a file on our machine like this:

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --get-schema openconfig-if-ip | xml_grep 'data' --text_only > openconfig-if-ip@2018-01-05.yang
```

You see we took the module name and put it into the --get-schema argument, and then also into the prefix of the redirected output, but we also put @2018-01-05 in the redirected name, to denote the revision the device is using. This is a good habit to get into since these models can update a lot and you can track which ones you are using where.

Note also, there is a deviation listed for this module. The deviations could be additional things added by the vendor in question or things replaced by the vendor. I am going to avoid this until I see I really need it.

I can now visualise this model file on my terminal with pyang.

```shell
    pyang -f tree openconfig-if-ip@2018-01-05.yang 


    openconfig-if-ip@2018-01-05.yang:11: error: module "openconfig-inet-types" not found in search path
    openconfig-if-ip@2018-01-05.yang:12: error: module "openconfig-interfaces" not found in search path
    openconfig-if-ip@2018-01-05.yang:13: error: module "openconfig-vlan" not found in search path
    openconfig-if-ip@2018-01-05.yang:13: warning: imported module openconfig-vlan not used
    openconfig-if-ip@2018-01-05.yang:14: error: module "openconfig-yang-types" not found in search path
    openconfig-if-ip@2018-01-05.yang:15: error: module "openconfig-extensions" not found in search path 
```

Ok. No I can't. This module refers to other modules that I havent fetched yet. Off we go to find them in the hello. Repeat the ... grep openconfig | grep thing command where you replace thing with the missing module name, then run the same --get-schema with the correct name and revision id for your device.

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --hello | grep openconfig | grep inet-types
        <nc:capability>http://openconfig.net/yang/types/inet?module=openconfig-inet-types&amp;revision=2017-08-24</nc:capability>

    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --get-schema openconfig-inet-types | xml_grep 'data' --text_only > openconfig-inet-types@2017-08-24.yang
```

Rinse and repeat until you get a full model on screen. Realise that as you pull a new dependent model in, you might grow new dependencies.

Weirdly, I ran the pyang -f tree command after fetching this inet-types module, and it looked like it worked.

```shell
    pyang -f tree *.yang

    module: openconfig-interfaces

       +--rw interfaces
          +--rw interface* [name]
             +--rw name             -> ../config/name
             +--rw config
             |  +--rw name?            string
             |  +--rw type             identityref
             |  +--rw mtu?             uint16
             |  +--rw loopback-mode?   boolean
             |  +--rw description?     string
             |  +--rw enabled?         boolean
    <snip>
```

Notice however, that the first module is now the interfaces. If you scroll down you will see the openconfig-vlan one is still missing and firing an error later on...

```shell
    pyang -f tree * | grep error:
    openconfig-inet-types@2017-08-24.yang:7: error: module "openconfig-extensions" not found in search path
    openconfig-interfaces@2018-01-05.yang:11: error: module "ietf-interfaces" not found in search path
    openconfig-interfaces@2018-01-05.yang:12: error: module "openconfig-yang-types" not found in search path
    openconfig-interfaces@2018-01-05.yang:13: error: module "openconfig-types" not found in search path
    openconfig-vlan@2016-05-26.yang:11: error: module "openconfig-vlan-types" not found in search path
    openconfig-vlan@2016-05-26.yang:13: error: module "openconfig-if-ethernet" not found in search path
    openconfig-vlan@2016-05-26.yang:14: error: module "openconfig-if-aggregate" not found in search path
    openconfig-vlan@2016-05-26.yang:15: error: module "iana-if-type" not found in search path
    openconfig-vlan@2016-05-26.yang:15: warning: imported module iana-if-type not used
    openconfig-vlan@2016-05-26.yang:370: warning: prefix "ianaift" is not defined
```

This is getting mental... Lets follow along tho. After fetching all these openconfig ones, I still have some ietf/iana ones outstanding...

```shell
    pyang -f tree * | grep error:
    openconfig-if-aggregate@2018-01-05.yang:13: warning: imported module iana-if-type not used
    openconfig-if-aggregate@2018-01-05.yang:186: warning: prefix "ianaift" is not defined
    openconfig-if-ethernet@2018-01-05.yang:12: error: module "iana-if-type" not found in search path
    openconfig-if-ethernet@2018-01-05.yang:12: warning: imported module iana-if-type not used
    openconfig-if-ethernet@2018-01-05.yang:346: warning: prefix "ianaift" is not defined
    openconfig-interfaces@2018-01-05.yang:11: error: module "ietf-interfaces" not found in search path
    openconfig-vlan@2016-05-26.yang:15: warning: imported module iana-if-type not used
    openconfig-vlan@2016-05-26.yang:370: warning: prefix "ianaift" is not defined
```

Lets tweak our capabilities filter and get them too:

```shell
    .
    ├── iana-if-type@2014-05-08.yang
    ├── ietf-interfaces@2014-05-08.yang
    ├── ietf-yang-types@2013-07-15.yang
    ├── openconfig-extensions@2017-04-11.yang
    ├── openconfig-if-aggregate@2018-01-05.yang
    ├── openconfig-if-ethernet@2018-01-05.yang
    ├── openconfig-if-ip@2018-01-05.yang
    ├── openconfig-inet-types@2017-08-24.yang
    ├── openconfig-interfaces@2018-01-05.yang
    ├── openconfig-types@2018-01-16.yang
    ├── openconfig-vlan@2016-05-26.yang
    ├── openconfig-vlan-types@2016-05-26.yang
    └── openconfig-yang-types@2017-07-30.yang
```

We should now be able to render the various bits and bobs we need to show us a full model without errors.

![Asciinema screencast](https://asciinema.org/a/G824lfWcGQ1O1qPCLrYfsuaGk)

Yay. This is now a completed model we could use to make a request to this CSR1000V to deploy something.
