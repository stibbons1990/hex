---
title:  "Getbukkit Expired"
date:   2024-03-15 15:03:05 +0200
categories: ubuntu server linux kubernetes docker minecraft java bedrock
---

## An Unexpected Minecraft "Update"

One day while looking at the
[monitoring]({{ site.baseurl }}/2020-03-21-detailed-system-and-process-monitoring.html)
in [lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
I noticed there was something *big* missing: the
[minecraft server]({{ site.baseurl }}/2023-08-10-running-minecraft-java-server-for-bedrock-clients-on-kubernetes.html)
that normally takes over 4GB of RAM was not running:

```
$ kubectl get all -n minecraft-server
NAME                                   READY   STATUS             RESTARTS        AGE
pod/minecraft-server-88f84b5fc-5kjr2   0/1     CrashLoopBackOff   152 (30s ago)   12h

NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
service/minecraft-server   NodePort   10.110.215.139   <none>        25565:32565/TCP,19132:32132/UDP   291d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minecraft-server   0/1     1            0           12h

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/minecraft-server-88f84b5fc   1         1         0       12h

$ kubectl -n minecraft-server logs minecraft-server-88f84b5fc-5kjr2
[init] Running as uid=1003 gid=1003 with /data as 'drwxrwxr-x 1 1003 1003 722 Dec 17 05:10 /data'
[init] Resolving type given SPIGOT
2024/03/15 17:44:59 Unable to find an element with attribute matcher property=og:title
[init] ERROR: failed to retrieve latest version from https://getbukkit.org/download/spigot -- site might be down

$ curl  https://getbukkit.org/download/spigot
<!doctype html>
<html data-adblockkey="MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBANDrp2lz7AOmADaN8tA50LsWcjLFyQFcb/P2Txc58oYOeILb3vBw7J6f4pamkAQVSQuqYsKx3YzdUHCvbVZvFUsCAwEAAQ==_UL89QGTogxdwKHwZzilx913GmK75KOL2kLgPnkgb9dD1Tc/wjgiP2tuKwPeUMm3vEXLjUWOarjD7XgGHgmalBg==" lang="en" style="background: #2B2B2B;">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="icon" href="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAIAAACQd1PeAAAADElEQVQI12P4//8/AAX+Av7czFnnAAAAAElFTkSuQmCC">
    <link rel="preconnect" href="https://www.google.com" crossorigin>
</head>
<body>
<div id="target" style="opacity: 0"></div>
<script>window.park = "eyJ1dWlkIjoiNDk5MDE1NDQtMTJlZi00YWQzLWI3YmQtMjA5Y2YwYzlmZjFmIiwicGFnZV90aW1lIjoxNzEwNTI2ODY5LCJwYWdlX3VybCI6Imh0dHBzOi8vZ2V0YnVra2l0Lm9yZy9kb3dubG9hZC9zcGlnb3QiLCJwYWdlX21ldGhvZCI6IkdFVCIsInBhZ2VfcmVxdWVzdCI6e30sInBhZ2VfaGVhZGVycyI6e30sImhvc3QiOiJnZXRidWtraXQub3JnIiwiaXAiOiIyMTcuMTYyLjU3LjY0In0K";</script>
<script src="/bNjGNXnzR.js"></script>
</body>
</html>
```

### Short-term Workaround

The server fails to start because it bails out when it
fails to download the latest version, but a few versions
are already downloaded and can be used by switching from
`TYPE=SPIGOT` to `TYPE=CUSTOM` and specifying
`CUSTOM_SERVER=/data/spigot_server-1.20.4.jar`
as explained in
[Issue #2521](https://github.com/itzg/docker-minecraft-server/issues/2521):
*Updated Bukkit download URL https://getbukkit.org/get/*

```yaml
          env:
            - name: EULA
              value: "TRUE"
            - name: TYPE
              value: CUSTOM
            - name: CUSTOM_SERVER
              value: /data/spigot_server-1.20.4.jar
            - name: MEMORY
              value: 4G
```

Reapplying the deployment brings the server back up.

```
$ kubectl apply -f minecraft-server.yaml
```

### Long-term Workaround

A day or two after the server was back up, it was noted in
[Issue #2721](https://github.com/itzg/docker-minecraft-server/issues/2721):
*Fallback to existing server file when getbukkit retrieval fails*
that the whole getbukkit.com site
*seems to be completely shut down*.
A few days later the site was up again.

Nevertheless, [itzg](https://github.com/itzg) seems to
*consistently* recommend switching to
[Paper](https://papermc.io/), stating that
*Spigot/Bukkit should no longer be used and Paper used instead.*

This should be a simple as changing the `TYPE` value,
since Paper also supports [GeyserMC](https://geysermc.org/)
(to allows Bedrock clients to join Java servers).

```yaml
          env:
            - name: EULA
              value: "TRUE"
            - name: TYPE
              value: PAPER
            - name: MEMORY
              value: 4G
```

However, applying this results in a non-ready deployment:

```
$ kubectl apply -f minecraft-server.yaml 
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server created

$ kubectl get all -n minecraft-server
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
service/minecraft-server   NodePort   10.110.215.139   <none>        25565:32565/TCP,19132:32132/UDP   298d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minecraft-server   0/1     0            0           2m
```
