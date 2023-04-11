+++
title = "node_exporter in VyOS 1.4"
date = "2022-04-04T23:23:00+02:00"
author = "John Howard"
authorTwitter = "fatred" #do not include @
cover = ""
tags = ["grafana","monitoring","prometheus","telemetry","vyos"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
aliases = ['/2022/04/nodeexporter-in-vyos-14.html']
+++
So it turns out that if you want metrics from VyOS, your two options are SNMP or Telegraf (towards InfluxDB).

SNMP is one of those things that has been around so long, you think its good, but really, its trash. Its a 1990s technology that is mostly singlethreaded and provides you very very fuzzy numbers. 5 min averages are not that useful in situations like today where clients plausibly have access to gigabit+ grades of connectivity. If you cant observe let alone react to a microburst, you're pretty much blindfolded in a modern DC network setting.

Telegraf and InfluxDB is at least telemetry, but I find it a little disapointing that I have to maintain two separate TSDBs. Sure, I can use one Grafana to speak to both, but its harder to mix and match the metrics without some sort of compromise.

What I wanted was node_exporter. I guess I was then pretty surprised to learn that this wasn't built in.

## Containers to the rescue

VyOS 1.4 supports containers via podman, which is the pure opensource fork of docker. This means we can configure node_exporter in a container, connect it to the host network and then poll the device on its usual :9100.

```vyos
set container name node-exporter allow-host-networks
set container name node-exporter description 'Prometheus Node Exporter'
set container name node-exporter image 'quay.io/prometheus/node-exporter:latest'
set container name node-exporter port node-exporter destination '9100'
set container name node-exporter port node-exporter protocol 'tcp'
set container name node-exporter port node-exporter source '9100'
set container name node-exporter volume hostroot destination '/host'
set container name node-exporter volume hostroot source '/'
set container registry 'quay.io'
```

With this configured, we end up with a container running with node exporter, and we can add this to our existing prometheus node_exporter scrape job as a target.

I then added the very impressive dashboard ![11074](https://grafana.com/grafana/dashboards/11074) to my grafana setup and gave it a spin.

I get all the basic host level metrics, cpu/mem/disk monitoring and some pretty decent networking stats too.

Upgrading is as easy as pulling the container image again and restarting it.

If you already have prom and want to get stats from VyOS - give it a go. Its quick and easy and very actionable information.
