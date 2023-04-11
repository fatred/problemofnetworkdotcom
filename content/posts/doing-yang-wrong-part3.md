+++
title = "Doing YANG Wrong: Part 3 - Using the python bindings"
date = "2020-06-27T00:44:00+02:00"
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
aliases = ['2020/06/doing-yang-wrong-part-3-using-python.html']
+++
## Part 3: Using the python bindings to push a config

Given we generated that python file locally in a machine, we assume here that you are still in that subdirectory.

The below code was stolen fully from the YANG Book. Much love and all credit to them for their work.

```python
from interface_setup import openconfig_interfaces
from pyangbind.lib.serialise import pybindIETFXMLEncoder
from ncclient import manager

# device settings
username = 'yangconf'
password = 'my_good_password.'
device_ip = '192.168.70.21'

# config settings 
inside_interface = 'GigabitEthernet4'
inside_ip_addr = '109.109.109.2'
inside_ip_prefix = 24


def send_to_device(**kwargs):
    rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>'
    with manager.connect_ssh(host-kwargs['dev'], port=830, username=kwargs['user'], password=kwargs['password'], hostkey_verify=False) as m:
        try:
            m.edit_config(target='running', config=rpc_body)
            print('Successfully configured IP on {}'.format(kwargs['dev']))
        except Exception as e:
            print('Failed to configure interface: {}'.format(e))


if __name__ == '__main__':
    # instanciate the openconfig model
    ocintmodel = openconfig_interfaces()

    # create an instance of the interfaces
    ocinterfaces = ocintmodel.interfaces

    # create a new interface instance in that parent object
    inside_if = ocinterfaces.interface.add(inside_interface)

    # even a routed interface required a subinterface, its just at index 0
    inside_if.subinterfaces.subinterface.add(0)

    # create an instance of that subinterface object to edit
    inside_sub_if = inside_if.subinterfaces.subinterface[0]

    # apply an IP to that object
    inside_sub_if.ipv4.addresses.address.add(inside_ip_addr)
    
    # read that ip object into an ip object
    ip = inside_sub_if.ipv4.addresses.address[inside_ip_addr]

    # set the IP and the subnet mask properly

    ip.config.ip = inside_ip_addr
    ip.config.prefix_length = inside_ip_prefix
    send_to_device(dev=device_ip, user=username, password=password, py_obj=ocinterfaces)
```

When I run this, it fails.

```shell
<pyangbind.lib.yangtypes.YANGBaseClass object at 0x7f3e5fac6170>
<pyangbind.lib.yangtypes.YANGBaseClass object at 0x7f3e5f9a69e0>
Traceback (most recent call last):
  File "<stdin>", line 19, in <module>
  File "<stdin>", line 2, in send_to_device
NameError: global name 'pybindIETFXMLEncoder' is not defined
```

Turns out the serialiser that the book code uses, relies on a library function that isn't in the pip version 0.8.1 of the pyangbind code. Bit of googling says build from the github repo here. Visiting that repo and alarm bells are ringing - the last commits are 2 years ago, and somehow the pip version is still out of date? Why? Anyways.

```shell
    pip install --upgrade git+https://github.com/robshakir/pyangbind.git
```

produces...

```shell
    python ./push_inside_if.py 
    Traceback (most recent call last): 
      File "./push_inside_if.py", line 43, in <module>
        send_to_device(dev=device_ip, user=username, password=password, py_obj=ocinterfaces)
      File "./push_inside_if.py", line 16, in send_to_device 
        rpc_body = '<config>' + pybindIETFXMLEncoder.serialise(kwargs['py_obj']) + '</config>' 
      File "/home/gns3/.local/lib/python2.7/site-packages/pyangbind/lib/serialise.py", line 380, in serialise
        doc = cls.encode(obj, filter=filter)
      File "/home/gns3/.local/lib/python2.7/site-packages/pyangbind/lib/serialise.py", line 375, in encode 
        return cls.generate_xml_tree(obj._yang_name, obj._yang_namespace, preprocessed) 
    AttributeError: 'YANGBaseClass' object has no attribute '_yang_namespace' 
```

What fresh hell is this?

So a github issue now tells us that we generated the binding against the old version of pyangbind, so we have to redo our export for the ENV var and then rebuild the python module....

```shell
    pyang --plugindir $PYBINDPLUGIN -f pybind -o interface_setup.py *.yang                          
    openconfig-if-ethernet@2018-01-05.yang:346: warning: node "openconfig-interfaces::state" is config false and is not part of the accessible tree                          
    openconfig-vlan-types@2016-05-26.yang:84: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning  
    openconfig-vlan-types@2016-05-26.yang:100: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:102: warning: the escape sequence "\*" is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:121: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:123: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:125: warning: the escape sequence "\*" is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:130: warning: the escape sequence "\*" is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:131: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    openconfig-vlan-types@2016-05-26.yang:133: warning: the escape sequence "\." is unsafe in double quoted strings - pass the flag --lax-quote-checks to avoid this warning 
    INFO: encountered (<pyang.error.Position object at 0x7fdbf28fb820>, 'XPATH_REF_CONFIG_FALSE', (u'openconfig-interfaces', u'state'))                                      
    FATAL: pyangbind cannot build module that pyang has found errors with.
```

Oh my word - so much rage.

I tried the --lax-quote-checks and that didn't work, so I edited each of those lines in the openconfig-vlan yang file to swap double quotes in the regexes to single quotes. These warnings went away.

```shell
    pyang --plugindir $PYBINDPLUGIN -f pybind -o interface_setup.py *.yang
    openconfig-if-ethernet@2018-01-05.yang:346: warning: node "openconfig-interfaces::state" is config false and is not part of the accessible tree
    INFO: encountered (<pyang.error.Position object at 0x7fe9b551f2d0>, 'XPATH_REF_CONFIG_FALSE', (u'openconfig-interfaces', u'state'))
    FATAL: pyangbind cannot build module that pyang has found errors with.
```

This one had be stumped. Google had nothing.  I was going around in circles until I broke the cycle by working on my laptop instead of my workstation. During the first time setup of the tools I found myself looking at all the repos again in github, and so I thought I would take a look at the blame on the affected file here. The error stood out like a saw thumb.

In my downloaded model, it referred to oc-if:state and in the repo model it referred to oc-if:config. The error now stands to reason since the state model is more for telemetry - its a read only view of the interface state, not the config. I edited the field and we now have a compiled module again.

Back to running the script...

```shell
    python push_inside_if.py
    Failed to configure interface: expected tag: name, got tag: subinterfaces
```

WAT? Lets dump out what we generated prior to send...

we add `print(pybindIETFXMLEncoder.serialise(ocinterfaces))` just above the `send_to_device` call, and then run again.

```shell
    python push_inside_if.py

    <interfaces xmlns="http://openconfig.net/yang/interfaces">
      <interface>
        <subinterfaces>
          <subinterface>
            <ipv4 xmlns="http://openconfig.net/yang/interfaces/ip">
              <addresses>
                <address>
                  <config>
                    <prefix-length>24</prefix-length>
                  </config>
                  <ip>109.109.109.2</ip>
                </address>
              </addresses>
            </ipv4>
            <index>0</index>
            <config>
              <description>Inside IP Address</description>
            </config>
          </subinterface>
        </subinterfaces>
        <config>
          <enabled>true</enabled>
          <description>Inside Interface</description>
        </config>
        <name>GigabitEthernet4</name>
      </interface>
    </interfaces>
```

Well it looks correct, but maybe it doesn't like the fact the name tag is the bottom? Seems like a dumb complaint to have - its a machine readable structure and the positioning in that structure is technically accurate (it's in the correct layer of the XML?)

Only way to prove this is to make a manual copy of this as a string var and then push it directly instead of rendering it with this tool.

First I comment the existing rendering of the rpc_body in the function to just use the kwargs['py_obj'] verbatim (I provide valid XML in my string) and then I make a multiline string in the main function with a human ordered XML envelope.

```shell
python push_inside_if.py 

original
<interfaces xmlns="http://openconfig.net/yang/interfaces">
  <interface>
    <subinterfaces>
      <subinterface>
        <ipv4 xmlns="http://openconfig.net/yang/interfaces/ip">
          <addresses>
            <address>
              <config>
                <prefix-length>24</prefix-length>
              </config>
              <ip>109.109.109.2</ip>
            </address>
          </addresses>
        </ipv4>
        <index>0</index>
        <config>
          <description>Inside IP Address</description>
        </config>
      </subinterface>
    </subinterfaces>
    <config>
      <enabled>true</enabled>
      <description>Inside Interface</description>
    </config>
    <name>GigabitEthernet4</name>
  </interface>
</interfaces>


ordered
<interfaces xmlns="http://openconfig.net/yang/interfaces">
  <interface>
    <name>GigabitEthernet4</name>
    <config>
      <enabled>true</enabled>
      <description>Inside Interface</description>
    </config>
    <subinterfaces>
      <subinterface>
        <index>0</index>
        <config>
          <description>Inside IP Address</description>
        </config>
        <ipv4 xmlns="http://openconfig.net/yang/interfaces/ip">
          <addresses>
            <address>
              <ip>109.109.109.2</ip>
              <config>
                <prefix-length>24</prefix-length>
              </config>
            </address>
          </addresses>
        </ipv4>
      </subinterface>
    </subinterfaces>
  </interface>
</interfaces>
  
Successfully configured IP on 192.168.70.21
```

Ugh. That's so lame. Clearly the problem here is the XML Serialiser is not rendering the objects in an order that the netconf agent on the CSR likes. Kill me now.

But wait. It gets better.

Having returned to my workstation, I decide that sending commands straight from VSCode to the GNS3 simulated CSR via the GNS3 simulated Ubuntu box with a simple NAT on the Ubuntu VM.

```shell
    sudo sysctl -w net.ipv4.ip_forward=1
    sudo iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
```

I then fire up code, pull in the changes from the laptop via my git repo, and fire off the request as seen to see whats what.

```shell
    python3 ./models/interface/push_inside_if.py
    Successfully configured IP on 192.168.70.21
```

Eh? This is not the same ordered code. This is the standard generated XML blob. Only difference, is Python3.8 is default on my workstation.

If nothing else, what this has taught me is that when it comes to YANG modelling in Python - environment matters - a lot. I get the feeling this is also why the developer of pyangbind let it die on the vine a bit, since moving over to Golang in his day job probably translates better to this use case as well. Golang for the initiated, generates C-like (in speed and performance) binary files that are all inclusive - no dependencies, no libraries. Build an app in Go, and its ready to rock and roll anywhere

At this point, i have been able to build a model saying what I want, and push it to the box, and it "made it so". What happens if I make a change out of band and then push something back to the box?

```shell
in1rt001#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
in1rt001(config)#int gi4
in1rt001(config-if)#ip address 109.109.109.3 255.255.254.0
in1rt001(config-if)#^Z

in1rt001#sh run int gi 4 
Building configuration...

Current configuration : 203 bytes
!
interface GigabitEthernet4
 description Inside IP Address
 ip address 109.109.111.3 255.255.254.0 secondary
 ip address 109.109.109.3 255.255.254.0
 negotiation auto
 no mop enabled
 no mop sysid
end
```

So I hacked up the subnet mask, oh and btw there is a secondary IP there too...

```shell
    python3 ./models/interface/push_inside_if.py
    Failed to configure interface: /native/interface/GigabitEthernet[name='4']/ip/address/secondary[address='109.109.109.2']/secondary is not configured
```

woooommmmpp whomp...

maybe I need to make sure that secondary isnt confusing things?

```shell
in1rt001#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
in1rt001(config)#int gi 4
in1rt001(config-if)#no  ip address 109.109.110.3 255.255.254.0 secondary
in1rt001(config-if)#^Z
in1rt001#sh run int gi 4
Building configuration...

Current configuration : 153 bytes
!
interface GigabitEthernet4
 description Inside IP Address
 ip address 109.109.109.3 255.255.254.0
 negotiation auto
 no mop enabled
 no mop sysid
end
```

Try again then...

```shell
    python3 ./models/interface/push_inside_if.py
    Failed to configure interface: /native/interface/GigabitEthernet[name='4']/ip/address/secondary[address='109.109.109.2']/secondary is not configured
```

big fat nope.

At this point, I think we need to consider the use of config candidates and the push many, apply once concept. Time for a new post...
