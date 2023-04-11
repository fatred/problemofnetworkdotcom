+++
title = "Doing YANG Wrong: Part 2 - Python Bindings"
date = "2020-06-27T00:28:00+02:00"
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
aliases = ['/2020/06/doing-yang-wrong-part-2-python-bindings.html']
+++
## Part 2: Python bindings for models

Having a model is one thing.  Using it requires you to ingest that model somewhere, apply values to its elements (leaves) as necessary/appropriate and then submit that completed model to the appliance for application to the config.

I'm a python guy, you might like Go, or ruby or whatever. thats up to you, but I use python right now, which means I use pyangbind and pyang to create pythonic modules I can import into a script, and then interact with the model attributes like I would any other object in python. We can then push that out to a device from that script.

I will assume you have pyangbind working, if not, use the first few steps from ![here](http://yang.ciscolive.com/pod0/labs/lab9/lab9-m6).

So, using the collection of model files we extracted for openconfig interfaces on our csr1000v, lets try to make a python module for interacting with this modelset.

```shell
    pyang --plugindir $PYBINDPLUGIN -f pybind -o interface_setup.py *.yang

    openconfig-if-aggregate@2018-01-05.yang:13: warning: imported module iana-if-type not used
    openconfig-if-aggregate@2018-01-05.yang:186: warning: prefix "ianaift" is not defined
    openconfig-if-ethernet@2018-01-05.yang:12: warning: imported module iana-if-type not used
    openconfig-if-ethernet@2018-01-05.yang:346: warning: prefix "ianaift" is not defined
    openconfig-vlan@2016-05-26.yang:15: warning: imported module iana-if-type not used
    openconfig-vlan@2016-05-26.yang:370: warning: prefix "ianaift" is not defined
    INFO: encountered (<pyang.error.Position object at 0x7f04005cc9d0>, 'UNUSED_IMPORT', u'iana-if-type')
    INFO: encountered (<pyang.error.Position object at 0x7f0400519a90>, 'WPREFIX_NOT_DEFINED', u'ianaift')
    FATAL: pyangbind cannot build module that pyang has found errors with.
```

Ugh. So we have an import somewhere in the model that isnt actually needed, and another one that isnt actually defined. End result: bad dog - no biscuit.

This one had me up against a wall for ages. I tried a pyang flag called `--yang-remove-unused-imports` but that didnt work either.

I then decided to look at these warnings properly. Turns out the iana-if-type module is imported with a prefix "ift", but later on these are refered to as "ianaift". Someone changed one part of the module, but not the other. In other words, the module on the CSR is technically broken.

![broken](/img/doing-ang-wrong/oc-yang-if-ethernet-model.png)

I changed the line in the three affected openconfig modules to

```yang
    import iana-if-type { prefix ianaift; }
```

and boom We have a python module.

Now that we have that basic module in place, we can build a script to deliver our first two requirements; an Interface and an IP.
