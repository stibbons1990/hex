---
date: 2024-12-28
draft: true
categories:
 - monitoring
 - python
 - tapo
---

# Continuous Monitoring for TP-Link Tapo devices

[Continuous Monitoring](../../conmon.md) describes the current, complete setup with the OSS versions of InfluxDB and Grafana.

Explain how this script in a cronjob works:

*  [`conmon-tapo.py`](../../conmon.md#conmon-tapopy) reports temperature, humidity,
*  [`tapo.yaml`](../../conmon.md#tapoyaml) is a minimal configuration file for

Create new database `home` with retention policy of **5 years**, because
[`monitoring` with only 30 days](2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md#conmon-migration)
is not enough to capture seasonal and yearly trends.

``` console
$ influx -host localhost -port 30086
Connected to http://localhost:30086 version 1.8.10
InfluxDB shell version: 1.6.7~rc0

> auth
username: admin
password: 

> CREATE DATABASE home
> USE home
Using database home

> CREATE RETENTION POLICY "5_years" ON "monitoring" DURATION 2000d REPLICATION 1
> ALTER RETENTION POLICY "5_years" on "monitoring" DURATION 2000d REPLICATION 1 DEFAULT
```

TODO:

- Update `conmon-tapo.py` to support daily periods on `always_on`.
- Find a way to hot-update list of `always_on` devices.
- Create script to import CSV data from Tapo app.
- Create script to copy data over from `monitoring` to `home`.
- Create script to export minimal data over to Pi Pico or ESP32.
