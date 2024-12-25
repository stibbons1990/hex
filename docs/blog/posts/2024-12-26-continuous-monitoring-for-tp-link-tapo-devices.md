---
date: 2024-12-26
draft: true
categories:
 - monitoring
 - python
 - tapo
---

# Continuous Monitoring for TP-Link Tapo devices

[Continuous Monitoring](../../conmon.md) describes the current, complete setup with the OSS
versions of InfluxDB and Grafana.

*  [`conmon-tapo.py`](../../conmon.md#conmon-tapopy) reports temperature, humidity,
*  [`tapo.yaml`](../../conmon.md#tapoyaml) is a minimal configuration file for



[`monitoring` 30 days](2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md#conmon-migration)

Create a `monitoring` database (separate from the
`telegraf` database) and set its retencion period to
30 days:

``` console
$ influx -host localhost -port 30086
Connected to http://localhost:30086 version 1.8.10
InfluxDB shell version: 1.6.7~rc0

> auth
username: admin
password: 

> CREATE DATABASE monitoring
> USE monitoring
Using database monitoring

> CREATE RETENTION POLICY "30_days" ON "monitoring" DURATION 30d REPLICATION 1
> ALTER RETENTION POLICY "30_days" on "monitoring" DURATION 30d REPLICATION 1 DEFAULT
```
