+++
title = "Doing YANG Wrong: Part 5 - Manufacturer/Model deviations"
date = "2020-07-02T15:26:00+02:00"
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
aliases = ['/2020/07/doing-yang-wrong-part-5.html']
+++
## Part 5: Deviations

So we can talk to the device, and we can use candidate configs to stage and then apply configs in aggregate, but we still can't make a CSR1000v take a simple openconfig IP address. At the beginning I deliberately called out I wanted to use generic models, and avoid the deviations. This is because the python model binding i then generate only works on that vendor's box now. This isn't terrible, but it's not really what we want. Lets see if it works tho.

Lets look at the hello statement for the ip address model again, and then fetch in the deviation it describes.

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --hello | grep openconfig | grep ip

    <nc:capability>http://openconfig.net/yang/interfaces/ip?module=openconfig-if-ip&amp;revision=2018-01-05&amp;deviations=cisco-xe-openconfig-if-ip-deviation,cisco-xe-openconfig-interfaces-deviation</nc:capability>
```

So there are actually two:

* cisco-xe-openconfig-if-ip-deviation
* cisco-xe-openconfig-interfaces-deviation

Lets pull these off the box like the others.

```shell
    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --get-schema cisco-xe-openconfig-if-ip-deviation | xml_grep 'data' --text_only > cisco-xe-openconfig-if-ip-deviation.yang

    netconf-console --host 192.168.70.21 --port 830 -u yangconf -p my_good_password. --get-schema cisco-xe-openconfig-interfaces-deviation | xml_grep 'data' --text_only > cisco-xe-openconfig-interfaces-deviation.yang 
```

Lets rebuild our python module with these deviations in the bundle:

```shell
    clear && pyang --plugindir $PYBINDPLUGIN -f pybind -o interface_setup.py *.yang --deviation cisco-xe-openconfig-interfaces-deviation.yang cisco-xe-openconfig-if-ip-deviation.yang

    cisco-xe-openconfig-interfaces-deviation.yang:11: warning: imported module "openconfig-if-ethernet" not used
    cisco-xe-openconfig-interfaces-deviation.yang:23: warning: imported module "ietf-yang-types" not used
    INFO: encountered (<pyang.error.Position object at 0x7fb3adcc8370>, 'UNUSED_IMPORT', u'openconfig-if-ethernet')
    INFO: encountered (<pyang.error.Position object at 0x7fb3adcc6cd0>, 'UNUSED_IMPORT', u'ietf-yang-types') 
```

Hmm. So its saying that in one deviation file (the interfaces one) there are two models that are defined, but not used. This is annoying, and goes to show how flaky some of these modules can be. I have to find the module definitions in the interfaces yang file at lines 11 and 23, and then comment them out. We can then rerun to get a compiled python module.

We then re-run our script:

```shell
    python3 push_inside_if.py
    Successfully configured IP on 192.168.70.21
```

Oooooh. Sensecheck please....

```shell
    in1rt001#sh run int gi 4
    Building configuration...

    Current configuration : 122 bytes
    !
    interface GigabitEthernet4
     description Inside Interface
     ip address 109.109.109.2 255.255.255.0
     negotiation auto
    end
```

Yay. Lets see if promise theory can be used with our models too. We will change the IP and the description.

```shell
    python3 ./push_inside_if.py

    Failed to configure interface: illegal reference /oc-if:interfaces/interface[name='GigabitEthernet4']/subinterfaces/subinterface[index='0']/oc-ip:ipv4/addresses/address[ip='50.60.70.1']/ip
```

Womp Womp :o(

Ok lets try everything except the IP itself (desc and mask):

```shell
    python3 push_inside_if.py
    Successfully configured IP on 192.168.70.21

    in1rt001#sh run int gi 4
    Building configuration...

    Current configuration : 122 bytes
    !
    interface GigabitEthernet4
     description People are stupid
     ip address 109.109.109.2 255.255.254.0
     negotiation auto
    end
```

What is interesting therefore, is that I can change everything except the IP... Subnet mask - fine, Description - fine. So, looking at that earlier error, it could well be that the existing interface model states subinterface[0] with IP address 50.60.70.1 doesn't match the existing IP address 109.109.109.2 when we push the model to the box. Thus, we cant reference the IP address we want in the interface model since it doesn't exist in the device's copy of the model.

So to round this out, lets get the IP off the interface instead.

we edit line 66 and make that add(inside_ip_addr) into delete()

```python
    inside_sub_if.ipv4.addresses.address.delete(inside_ip_addr)
```

fail:

```shell
    python3 ./models/interface/push_inside_if.py
    Traceback (most recent call last):
      File "/home/jhow/.local/lib/python3.8/site-packages/pyangbind/lib/yangtypes.py", line 848, in delete
        del self._members[k]
    KeyError: '109.109.109.2'

    During handling of the above exception, another exception occurred:

    Traceback (most recent call last):
      File "./models/interface/push_inside_if.py", line 66, in <module>
        inside_sub_if.ipv4.addresses.address.delete(inside_ip_addr)
      File "/home/jhow/.local/lib/python3.8/site-packages/pyangbind/lib/yangtypes.py", line 852, in delete
        raise KeyError("key %s was not in list (%s)" % (k, m))
    KeyError: "key 109.109.109.2 was not in list ('109.109.109.2')"
```

This makes sense because the model in the python memory is independent of the config of the device. You cant delete something that doesnt exist in memory yet, and adding it then removing it leaves you with an empty model to push, which has no effect on the device. We would have to pull the config and parse it into a model before we could know what we have to delete, or know that what we have injected into our python model matches the state exactly on the device. The latter is fine if you have a good source of truth.

It is at this point I spot that we have to add a netconf operation "delete" to the modeled interface to delete it.

```xml
    nc:operation="delete"
```

...needs to be put into the xml wrapper around the ipv4 address of the subinterface.

I can't see how this pyangbind xml serialiser supports providing netconf operations. To validate this theory, I first try to export the serialised XML for a create operation, and manually add the operation="delete" to the ipv4 tag.

```python
    manual = '''


<interfaces xmlns="http://openconfig.net/yang/interfaces">
  <interface>
    <name>GigabitEthernet4</name>
    <config>
      <enabled>true</enabled>
    </config>
    <subinterfaces>
      <subinterface>
        <index>0</index>
        <config>
          <description>Deleted</description>
        </config>
        <ipv4 xmlns="http://openconfig.net/yang/interfaces/ip">
          <addresses>
            <address operation="delete">
              <ip>109.232.176.2</ip>
              <config>
                <ip>109.232.176.2</ip>
                <prefix-length>23</prefix-length>
              </config>
            </address>
          </addresses>
        </ipv4>
      </subinterface>
    </subinterfaces>
  </interface>
</interfaces>
'''
```

We then alter the send_to_device function exactly as we did before, and send this formatted XML in the `<config>` brackets, instead of our modeled object.

I save this as pull_inside_if.py to keep the files separate, then push the interface in, before pulling it back out.

```shell
    python3 ./models/interface/push_inside_if.py
    Successfully configured IP on 192.168.70.21
    python3 ./models/interface/pull_inside_if.py
    Successfully configured IP on 192.168.70.21
```

Checking the running config on the box and it did the trick.

```shell
    in1rt001#sh run int gi 4
    Building configuration...

    Current configuration : 122 bytes
    !
    interface GigabitEthernet4
     description Deleted
     no ip address
     negotiation auto
    end 
```

So now we have to ask ourselves. Do we care about idempotency? When it comes to pushing a change out, there are 3 states we will observe:

1. the IP on the interface doesn't exist and should exist. <- works
2. the IP on the interface exists and we need to change something that isn't the IP itself <- works
3. the IP on the interface exists and we want to remove it <- works.

What we cannot do in one step is an in situ replacement of the IP, since that is a key value in the model itself. Considering this sort of operation would be bound to the operational hooks of a CMDB, the events we would expect are add a new thing, change something on that existing thing that isnt the thing itself, or delete the thing. On that basis, it shouldn't be unreasonable to codify that you should decommission an interface, and then commission your new one as a new config. Thus you cannot "update" keyed values in situ, but must instead remove/commit/add/commit in order.

Given the likely use case for a CMDB hook response, I guess we can live with that.
