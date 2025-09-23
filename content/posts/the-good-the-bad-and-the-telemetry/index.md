---
title: "The Good, the Bad, and the Telemetry: The business case"
date: 2025-08-02T19:12:52+02:00
---
When it comes to observability in the DC Networking space we have been stuck in the 90s for too long. The state of the art is SNMP and syslog, and frankly this is absurd. The Service Providers have had all the fancy tools to themselves for years now, and I think its time we fixed this.

---

In the lead up to Autocon3 in May 2025, I was getting pretty irate at work. We have been running LibreNMS since before I started there, and were using it for SNMP _and_ syslog monitoring. Thats _ok_ I guess, but I very quickly missed my Graylog stack, and since we deployed Nokia SRL, we found that SNMP coverage was actually pretty poor.

To that end, we had been running Streaming Telemetry via gNMIc in parallel for nearly a year and for perhaps 80% of the cases it was as good if not better. The problem was that 20%, and this is what I will cover in the next few posts.

## The business case

Generally speaking SNMP is so popular _because_ its so easy nowadays. You can grab a docker-compose for librenms, `docker compose up -d` and within about 10 mins you will have 175 metrics per device, in a fancy UI, with all the bells and whistles of visualisation, alerting, even billing if you like. It's almost foolproof, and that is a testament to that community for the effort they put into that tooling.

That said, scaling beyong a few thousand ports is non trivial, and in many cases it becomes a full time job. Also, 5 minute averages are pretty poor when you get into customer impacting service outages, the lack of correlation opportunities is pretty frustrating, and the aging of the data can be a problem over time. LibreNMS offers a way to federate your data out of the MRTG storage and into Prometheus, but that's still putting lipstick on a pig.

Streaming Telemetry is generally a 15s sample window, stored into a general purpose timeseries database, which allows for all the same reporting, billing, and correlation tooling, and there are a number of good visualisation tools, chief of which is Grafana.

So the business case starts with getting better data, more frequently, with the ability to retain all these features, PLUS the new option to cross reference data and enrich it.

## The strategy

Most people who pass the business case stage, end up in a bit of a muddle when it comes to implementation. The best examples of streaming telemetry in the market are paid solutions and thus full of secret sauce. We took the mindset of building a heatmap of librenms features, views, and content that were interesting to us, then systematically tried to reproduce that. The low hanging fruit of interfaces and service states is pretty doable tbh and you should be able to get to value quite fast.

The difficulty lies around how some vendors represent their data inside these models, and how much effort it takes to make them consumable. The obvious starting point is openconfig, since its supposed to be multivendor and the hyperscalers have "matured" it over the past 10 years. That should mean the amount of time you end up in vendor specific models is a diminishing amount, and since hyperscalers are pretty advanced, the featuresets should cover at least 80% of what we mere mortals need.

## The openconfig fallacy

My oh my how young and naive I was at the start of this.

The jaded part of me wants to scream from the top of the matterhorn: "openconfig is a lie". That would be a little hyperbolic tho ;).

Lets start here: it should be called openschema, not openconfig. Telemetry is nothing to do with config, its about states, and there are many orgs that dont use open_config_ for CONFIGS, but they use open_config_ for state tracking in telemetry. But thats nitpicking...

Really for me the biggest unexpected drama came from the discovery that some vendors implement the _vendor neutral_ schema, inconsistently.

Here is an example from the openconfig site for [interfaces](https://openconfig.net/projects/models/schemadocs/yangdoc/openconfig-interfaces.html)

```text
/interfaces/interface/state/
admin-status

description:
The desired state of the interface. In RFC 7223 this leaf has the same read semantics as ifAdminStatus. Here, it reflects the administrative state as set by enabling or disabling the interface.

nodetype: leaf (ro)

type: enumeration

    UP
    Ready to pass packets.
    DOWN
    Not ready to pass packets and not in some test mode.
    TESTING
    The interface should be treated as if in admin-down state for control plane protocols. In addition, while in TESTING state the device should remove the interface from aggregate interfaces. An interface transition to the TESTING state based on a qualification workflow, or internal device triggered action - such as the gNOI Link Qualification service
```

Here is the return value of this node of the [SRL interface](https://yang.srlinux.dev/v24.10.4?path=%252Finterface%5Bname%253D*%5D%252Fadmin-state) over the OC "aligned" model:

```yaml
- source: clab-gnp-stack-spine1:57400
  timestamp: 1754602229651662304
  time: "2025-08-07T23:30:29.651662304+02:00"
  updates:
    - Path: openconfig-interfaces:interfaces/interface[name=ethernet-1/1]/state/admin-status
      values:
        openconfig-interfaces:interfaces/interface/state/admin-status: UP
```

So thats OK right? Now lets ask Arista.

```yaml
- source: clab-gnp-stack-edge1:6030
  timestamp: 1754602436949605430
  time: "2025-08-07T23:33:56.94960543+02:00"
  updates:
    - Path: interfaces/interface[name=Ethernet1]/state/admin-status
      values:
        interfaces/interface/state/admin-status: UP
```

Cool, cool. that's perfect! Not always tho... If you look at other vendors, these Enums are not always consistent. I have seen UP, up and 1 in this position so far. Some use a pointer to the enum notation, and some just return a string. Regardless of all of that, a string is useless to Prometheus, who wants a number, so we have to write a string>int or string>float processor in gNMIc anyways.

Now, if we look past the pedantry of formatting, here is another inconsistency complaint. There are three common bits of metadata in a line on the openconfig site:

```text
/interfaces/interface/ethernet/state/
port-speed

description:
When auto-negotiate is TRUE, this optionally sets the port-speed mode that will be advertised to the peer for negotiation. If unspecified, it is expected that the interface will select the highest speed available based on negotiation. When auto-negotiate is set to FALSE, sets the link speed to a fixed value -- supported values are defined by ETHERNET_SPEED identities

nodetype: leaf (ro)

type: identityref

    base: ETHERNET_SPEED

---

/interfaces/interface/ethernet/state/
fec-mode

description:
The FEC mode applied to the physical channels associated with the interface.

nodetype: leaf (ro)

type: identityref

    base: INTERFACE_FEC

---

/interfaces/interface/ethernet/state/
hw-mac-address

description:
Represents the 'burned-in', or system-assigned, MAC address for the Ethernet interface.

nodetype: leaf (ro)

type: oc-yang:mac-address
```

Lets fetch these from Nokia:

what works:

```yaml
- source: clab-gnp-stack-spine1:57400
  timestamp: 1754604177338710396
  time: "2025-08-08T00:02:57.338710396+02:00"
  updates:
    - Path: interfaces/interface[name=ethernet-1/1]/ethernet/state/port-speed
      values:
        interfaces/interface/ethernet/state/port-speed: SPEED_25GB
- source: clab-gnp-stack-spine1:57400
  timestamp: 1754604211117445490
  time: "2025-08-08T00:03:31.11744549+02:00"
  updates:
    - Path: interfaces/interface[name=ethernet-1/1]/ethernet/state/hw-mac-address
      values:
        interfaces/interface/ethernet/state/hw-mac-address: 1A:48:0E:FF:00:01
```

what no worky:

```text
$ gnmic -a clab-gnp-stack-spine1:57400 -u admin -p NokiaSrl1! --skip-verify --encoding PROTO get --path "/interfaces/interface[name=ethernet-1/1]/ethernet/state/fec-mode" | yq -P -oy -

target "clab-gnp-stack-spine1:57400" Get request failed: "clab-gnp-stack-spine1:57400" GetRequest failed: rpc error: code = InvalidArgument desc = Path not valid - unknown element 'fec-mode'. Options are [counters, mac-address, port-speed, hw-mac-address, aggregate-id]
Error: one or more requests failed
```

and Arista...

what works:

```yaml
- source: clab-gnp-stack-edge1:6030
  timestamp: 1754604381446432909
  time: "2025-08-08T00:06:21.446432909+02:00"
  updates:
    - Path: interfaces/interface[name=Ethernet1]/ethernet/state/port-speed
      values:
        interfaces/interface/ethernet/state/port-speed: openconfig-if-ethernet:SPEED_1GB
- source: clab-gnp-stack-edge1:6030
  timestamp: 1754604411072071026
  time: "2025-08-08T00:06:51.072071026+02:00"
  updates:
    - Path: interfaces/interface[name=Ethernet1]/ethernet/state/hw-mac-address
      values:
        interfaces/interface/ethernet/state/hw-mac-address: aa:c1:ab:74:96:aa
```

what doesnt:

```text
$ gnmic -a clab-gnp-stack-edge1:6030 -u admin -p admin --insecure --encoding JSON get --path "/interfaces/interface[name=Ethernet1]/ethernet/state/fec-mode" | yq -P -oy 

target "clab-gnp-stack-edge1:6030" Get request failed: "clab-gnp-stack-edge1:6030" GetRequest failed: rpc error: code = InvalidArgument desc = error getting /interfaces/interface[name=Ethernet1]/ethernet/state/fec-mode: path invalid: failed to access node "fec-mode" in node "state"
Error: one or more requests failed
```

## What the FEC?

Why do I care about FEC tho? Well in the DC we have 25G server ports and these will _always_ require a FEC setting, and that setting depends on a few thngs; Copper vs AOC, length and age. So a coherent DC monitoring stack should expose 25G FEC settings and track error rates for the day an older cable or a more bent cable can be identified for replacement.

So how can I get to the FEC status? Vendor Models?

Kinda.

Nokia has it in their native models under the transceiver:

```text
$ gnmic -a clab-gnp-stack-spine1:57400 -u admin -p NokiaSrl1! --skip-verify --encoding PROTO get --path "native:/interface[name=ethernet-1/1]/transceiver/forward-error-correction"
[
  {
    "source": "clab-gnp-stack-spine1:57400",
    "timestamp": 1754645921119507801,
    "time": "2025-08-08T11:38:41.119507801+02:00",
    "updates": [
      {
        "Path": "native:interface[name=ethernet-1/1]/transceiver/forward-error-correction",
        "values": {
          "interface/transceiver/forward-error-correction": "disabled"
        }
      }
    ]
  }
]
```

Juniper has it in the junos state tree

```text
  "prefix": "state/interfaces/interface[name=et-0/0/1]",
  "updates": [
    {
      "Path": "name",
      "values": {
        "name": "et-0/0/1"
      }
    },
    {
      "Path": "media-type",
      "values": {
        "media-type": "FIBER"
      }
    },
    {
      "Path": "link-level-type",
      "values": {
        "link-level-type": "IFML_ETHER"
      }
    },
...snip...
    {
      "Path": "ethernet/fec-mode",
      "values": {
        "ethernet/fec-mode": "FEC74"
      }
    },
    {
      "Path": "ethernet/fec-codeword-size",
      "values": {
        "ethernet/fec-codeword-size": 0
      }
    },
    {
      "Path": "ethernet/fec-codeword-rate",
      "values": {
        "ethernet/fec-codeword-rate": {
          "precision": 2
        }
      }
    },
...snip...
```

So the good news is the data is there, and the bad news is we have to find it. So far, so SNMP I guess.

## Herding cats

This leads into the inevitable follow up point, which is acknolwedging that the data isnt all in openconfig, and we dont all have one vendor in our midst, so **how should we normalise this data?**

This is a multifaceted question. Assume we solved the problem of knowing what paths we have to target to ingest all the data we want from a device at the frequency we want it (like librenms does out of the box). Next we have to capture this data, structure it in a coherant (normalised) way, and then provide access to it for _things_.

This, after a short while, feels like herding cats into the sea.

Our intial swing at this attempted to review the OC tree for gaps, and fill it with data from vendor world, martialed into the correct OC formats. This is pretty successful in principle, but there are scaling issues here we will cover shortly. For the vendor bits we want but that do not yet exist in OC land, we apply the OC style guide to "produce" an OC-Like metric, that covers this gap sufficiently.

This approach is valid, but certainly fraught with risk, in so far as we don't upstream this (at this time), and the risk is that someone does lobby for that data point, and OC produces it with different names/schemas, or worse, they use our implementation and alter it very very slightly making our version invalid, but correct to the casual observer.

## The solution: consensus

When pondering all of this for weeks on end, it slowly dawned on me that so many other businesses have been trashing at these problems, just as we have for what probably adds up to a monumental amount of wasted effort. When you step back and consider how and why MRTG, Solarwinds, Paessler, Observium and now LibreNMS are so popular, it is the point and shoot nature of the beast. Some group of kind souls spent the time to map the inputs to the right graphs and boom, we can reliably access rich and useful visualisations in minutes.

As I discuss these problems with my peers I find many that are at some level of maturity when it comes to Streaming telemetry and none of them felt like it was a satisfying process. A small group of us have now banded toether to start crowdsourcing a new "LibreNMS for telemetry".

We call it the GNP-Stack (gNMIC, NATS, Prometheus) and the idea is to build a fully opensource packaging of these mature tools, structured to make it as "point and shoot" as possible. I expect it will take much longer than i anticipated, and many challenges will appear as we go.

I do this not because I particularly want to, but more because I believe we need to break this abusive cycle once and for all. So that businesses that want high fidelity metrics and alerting, dont have to commit so many hours of engineering time to repeat the mistakes of the past.

So, if you so feel inclined, join me as we explore this topic over the coming weeks.
