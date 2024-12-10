---
date: 2024-03-15
categories:
 - ubuntu
 - server
 - linux
 - kubernetes
 - docker
 - minecraft
 - java
 - bedrock
title: Getbukkit Expired
---

## An Unexpected Minecraft "Update"

One day while looking at the
[monitoring](../../../../2020-03-21-detailed-system-and-process-monitoring.html)
in [lexicon](../../../../2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
I noticed there was something *big* missing: the
[minecraft server](../../../../2023-08-10-running-minecraft-java-server-for-bedrock-clients-on-kubernetes.html)
that normally takes over 4GB of RAM was not running:

<!-- more --> 

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

Applying this changes the replicaset, brings RAM usage
down from 5.14 GB to 2.69 GB, but also shows the
Geyser plugin fails to load:

```
$ kubectl apply -f minecraft-server.yaml
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server configured

$ kubectl get all -n minecraft-server
NAME                                    READY   STATUS        RESTARTS   AGE
pod/minecraft-server-545c4b56f5-sz5j8   1/1     Running       0          9s
pod/minecraft-server-7f847b6b7-tv6tw    1/1     Terminating   0          4h35m

NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
service/minecraft-server   NodePort   10.110.215.139   <none>        25565:32565/TCP,19132:32132/UDP   299d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minecraft-server   1/1     1            1           4h47m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/minecraft-server-545c4b56f5   1         1         1       9s
replicaset.apps/minecraft-server-7f847b6b7    0         0         0       4h35m

$ kubectl get all -n minecraft-server
NAME                                    READY   STATUS    RESTARTS   AGE
pod/minecraft-server-545c4b56f5-sz5j8   1/1     Running   0          106s

NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
service/minecraft-server   NodePort   10.110.215.139   <none>        25565:32565/TCP,19132:32132/UDP   299d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minecraft-server   1/1     1            1           4h49m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/minecraft-server-545c4b56f5   1         1         1       106s
replicaset.apps/minecraft-server-7f847b6b7    0         0         0       4h37m

$ kubectl -n minecraft-server logs minecraft-server-545c4b56f5-sz5j8
[init] Running as uid=1003 gid=1003 with /data as 'drwxrwxr-x 1 1003 1003 750 Mar 15 18:33 /data'
[init] Resolving type given PAPER
[mc-image-helper] 14:43:03.576 INFO  : Resolved paper to version 1.20.4 build 459
[mc-image-helper] 14:43:04.869 INFO  : Downloaded /data/paper-1.20.4-459.jar
[mc-image-helper] 14:43:07.295 INFO  : Created/updated 1 property in /data/server.properties
[init] Setting initial memory to 4G and max to 4G
[init] Starting the Minecraft server...
Downloading mojang_1.20.4.jar
Applying patches
Starting org.bukkit.craftbukkit.Main
System Info: Java 17 (OpenJDK 64-Bit Server VM 17.0.10+7) Host: Linux 5.15.0-101-generic (amd64)
Loading libraries, please wait...
2024-03-23 14:43:20,628 ServerMain WARN Advanced terminal features are not available in this environment
[14:43:25 INFO]: Environment: Environment[sessionHost=https://sessionserver.mojang.com, servicesHost=https://api.minecraftservices.com, name=PROD]
[14:43:26 INFO]: Loaded 1174 recipes
[14:43:27 INFO]: Loaded 1271 advancements
[14:43:27 INFO]: Starting minecraft server version 1.20.4
[14:43:27 INFO]: Loading properties
[14:43:27 INFO]: This server is running Paper version git-Paper-459 (MC: 1.20.4) (Implementing API version 1.20.4-R0.1-SNAPSHOT) (Git: 88419b2)
[14:43:28 INFO]: Server Ping Player Sample Count: 12
[14:43:28 INFO]: Using 4 threads for Netty based IO
[14:43:28 INFO]: [ChunkTaskScheduler] Chunk system is using 1 I/O threads, 1 worker threads, and gen parallelism of 1 threads
[14:43:28 WARN]: [!] The timings profiler has been enabled but has been scheduled for removal from Paper in the future.
    We recommend installing the spark profiler as a replacement: https://spark.lucko.me/
    For more information please visit: https://github.com/PaperMC/Paper/issues/8948
[14:43:28 INFO]: Default game type: CREATIVE
[14:43:28 INFO]: Generating keypair
[14:43:28 INFO]: Starting Minecraft server on *:25565
[14:43:28 INFO]: Using epoll channel type
[14:43:28 INFO]: Paper: Using libdeflate (Linux x86_64) compression from Velocity.
[14:43:28 INFO]: Paper: Using OpenSSL 1.1.x (Linux x86_64) cipher from Velocity.
[14:43:29 INFO]: [Geyser-Spigot] Loading server plugin Geyser-Spigot v2.1.0-SNAPSHOT
[14:43:29 ERROR]: [Geyser-Spigot] Error initializing plugin 'Geyser-Spigot.jar' in folder 'plugins' (Is it up to date?)
java.lang.NoSuchMethodError: 'void org.yaml.snakeyaml.parser.ParserImpl.<init>(org.yaml.snakeyaml.reader.StreamReader)'
        at org.geysermc.geyser.platform.spigot.shaded.com.fasterxml.jackson.dataformat.yaml.YAMLParser.<init>(YAMLParser.java:191) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.platform.spigot.shaded.com.fasterxml.jackson.dataformat.yaml.YAMLFactory._createParser(YAMLFactory.java:504) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.platform.spigot.shaded.com.fasterxml.jackson.dataformat.yaml.YAMLFactory.createParser(YAMLFactory.java:392) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.platform.spigot.shaded.com.fasterxml.jackson.dataformat.yaml.YAMLFactory.createParser(YAMLFactory.java:15) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.platform.spigot.shaded.com.fasterxml.jackson.databind.ObjectMapper.readValue(ObjectMapper.java:3542) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.util.FileUtils.loadConfig(FileUtils.java:57) ~[Geyser-Spigot.jar:?]
        at org.geysermc.geyser.platform.spigot.GeyserSpigotPlugin.onLoad(GeyserSpigotPlugin.java:144) ~[Geyser-Spigot.jar:?]
        at io.papermc.paper.plugin.storage.ServerPluginProviderStorage.processProvided(ServerPluginProviderStorage.java:59) ~[paper-1.20.4.jar:git-Paper-459]
        at io.papermc.paper.plugin.storage.ServerPluginProviderStorage.processProvided(ServerPluginProviderStorage.java:18) ~[paper-1.20.4.jar:git-Paper-459]
        at io.papermc.paper.plugin.storage.SimpleProviderStorage.enter(SimpleProviderStorage.java:39) ~[paper-1.20.4.jar:git-Paper-459]
        at io.papermc.paper.plugin.entrypoint.LaunchEntryPointHandler.enter(LaunchEntryPointHandler.java:36) ~[paper-1.20.4.jar:git-Paper-459]
        at org.bukkit.craftbukkit.v1_20_R3.CraftServer.loadPlugins(CraftServer.java:507) ~[paper-1.20.4.jar:git-Paper-459]
        at net.minecraft.server.dedicated.DedicatedServer.initServer(DedicatedServer.java:274) ~[paper-1.20.4.jar:git-Paper-459]
        at net.minecraft.server.MinecraftServer.runServer(MinecraftServer.java:1131) ~[paper-1.20.4.jar:git-Paper-459]
        at net.minecraft.server.MinecraftServer.lambda$spin$0(MinecraftServer.java:319) ~[paper-1.20.4.jar:git-Paper-459]
        at java.lang.Thread.run(Unknown Source) ~[?:?]
[14:43:29 INFO]: Server permissions file permissions.yml is empty, ignoring it
[14:43:29 INFO]: Preparing level "world"
[14:43:29 INFO]: -------- World Settings For [world] --------
[14:43:29 INFO]: Item Merge Radius: 2.5
[14:43:29 INFO]: Experience Merge Radius: 3.0
[14:43:29 INFO]: Mob Spawn Range: 6
[14:43:29 INFO]: Item Despawn Rate: 6000
[14:43:29 INFO]: Arrow Despawn Rate: 1200 Trident Respawn Rate:1200
[14:43:29 INFO]: Zombie Aggressive Towards Villager: true
[14:43:29 INFO]: Nerfing mobs spawned from spawners: false
[14:43:29 INFO]: Allow Zombie Pigmen to spawn from portal blocks: true
[14:43:29 INFO]: Entity Activation Range: An 32 / Mo 32 / Ra 48 / Mi 16 / Tiv true / Isa false
[14:43:29 INFO]: Entity Tracking Range: Pl 48 / An 48 / Mo 48 / Mi 32 / Di 128 / Other 64
[14:43:29 INFO]: Hopper Transfer: 8 Hopper Check: 1 Hopper Amount: 1 Hopper Can Load Chunks: false
[14:43:29 INFO]: View Distance: 10
[14:43:29 INFO]: Simulation Distance: 10
[14:43:29 INFO]: Custom Map Seeds:  Village: 10387312 Desert: 14357617 Igloo: 14357618 Jungle: 14357619 Swamp: 14357620 Monument: 10387313 Ocean: 14357621 Shipwreck: 165745295 End City: 10387313 Slime: 987234911 Nether: 30084232 Mansion: 10387319 Fossil: 14357921 Portal: 34222645
[14:43:29 INFO]: Max TNT Explosions: 100
[14:43:29 INFO]: Tile Max Tick Time: 50ms Entity max Tick Time: 50ms
[14:43:29 INFO]: Cactus Growth Modifier: 100%
[14:43:29 INFO]: Cane Growth Modifier: 100%
[14:43:29 INFO]: Melon Growth Modifier: 100%
[14:43:29 INFO]: Mushroom Growth Modifier: 100%
[14:43:29 INFO]: Pumpkin Growth Modifier: 100%
[14:43:29 INFO]: Sapling Growth Modifier: 100%
[14:43:29 INFO]: Beetroot Growth Modifier: 100%
[14:43:29 INFO]: Carrot Growth Modifier: 100%
[14:43:29 INFO]: Potato Growth Modifier: 100%
[14:43:29 INFO]: TorchFlower Growth Modifier: 100%
[14:43:29 INFO]: Wheat Growth Modifier: 100%
[14:43:29 INFO]: NetherWart Growth Modifier: 100%
[14:43:29 INFO]: Vine Growth Modifier: 100%
[14:43:29 INFO]: Cocoa Growth Modifier: 100%
[14:43:29 INFO]: Bamboo Growth Modifier: 100%
[14:43:29 INFO]: SweetBerry Growth Modifier: 100%
[14:43:29 INFO]: Kelp Growth Modifier: 100%
[14:43:29 INFO]: TwistingVines Growth Modifier: 100%
[14:43:29 INFO]: WeepingVines Growth Modifier: 100%
[14:43:29 INFO]: CaveVines Growth Modifier: 100%
[14:43:29 INFO]: GlowBerry Growth Modifier: 100%
[14:43:29 INFO]: PitcherPlant Growth Modifier: 100%
[14:43:30 INFO]: -------- World Settings For [world_nether] --------
[14:43:30 INFO]: Item Merge Radius: 2.5
[14:43:30 INFO]: Experience Merge Radius: 3.0
[14:43:30 INFO]: Mob Spawn Range: 6
[14:43:30 INFO]: Item Despawn Rate: 6000
[14:43:30 INFO]: Arrow Despawn Rate: 1200 Trident Respawn Rate:1200
[14:43:30 INFO]: Zombie Aggressive Towards Villager: true
[14:43:30 INFO]: Nerfing mobs spawned from spawners: false
[14:43:30 INFO]: Allow Zombie Pigmen to spawn from portal blocks: true
[14:43:30 INFO]: Entity Activation Range: An 32 / Mo 32 / Ra 48 / Mi 16 / Tiv true / Isa false
[14:43:30 INFO]: Entity Tracking Range: Pl 48 / An 48 / Mo 48 / Mi 32 / Di 128 / Other 64
[14:43:30 INFO]: Hopper Transfer: 8 Hopper Check: 1 Hopper Amount: 1 Hopper Can Load Chunks: false
[14:43:30 INFO]: View Distance: 10
[14:43:30 INFO]: Simulation Distance: 10
[14:43:30 INFO]: Custom Map Seeds:  Village: 10387312 Desert: 14357617 Igloo: 14357618 Jungle: 14357619 Swamp: 14357620 Monument: 10387313 Ocean: 14357621 Shipwreck: 165745295 End City: 10387313 Slime: 987234911 Nether: 30084232 Mansion: 10387319 Fossil: 14357921 Portal: 34222645
[14:43:30 INFO]: Max TNT Explosions: 100
[14:43:30 INFO]: Tile Max Tick Time: 50ms Entity max Tick Time: 50ms
[14:43:30 INFO]: Cactus Growth Modifier: 100%
[14:43:30 INFO]: Cane Growth Modifier: 100%
[14:43:30 INFO]: Melon Growth Modifier: 100%
[14:43:30 INFO]: Mushroom Growth Modifier: 100%
[14:43:30 INFO]: Pumpkin Growth Modifier: 100%
[14:43:30 INFO]: Sapling Growth Modifier: 100%
[14:43:30 INFO]: Beetroot Growth Modifier: 100%
[14:43:30 INFO]: Carrot Growth Modifier: 100%
[14:43:30 INFO]: Potato Growth Modifier: 100%
[14:43:30 INFO]: TorchFlower Growth Modifier: 100%
[14:43:30 INFO]: Wheat Growth Modifier: 100%
[14:43:30 INFO]: NetherWart Growth Modifier: 100%
[14:43:30 INFO]: Vine Growth Modifier: 100%
[14:43:30 INFO]: Cocoa Growth Modifier: 100%
[14:43:30 INFO]: Bamboo Growth Modifier: 100%
[14:43:30 INFO]: SweetBerry Growth Modifier: 100%
[14:43:30 INFO]: Kelp Growth Modifier: 100%
[14:43:30 INFO]: TwistingVines Growth Modifier: 100%
[14:43:30 INFO]: WeepingVines Growth Modifier: 100%
[14:43:30 INFO]: CaveVines Growth Modifier: 100%
[14:43:30 INFO]: GlowBerry Growth Modifier: 100%
[14:43:30 INFO]: PitcherPlant Growth Modifier: 100%
[14:43:30 INFO]: -------- World Settings For [world_the_end] --------
[14:43:30 INFO]: Item Merge Radius: 2.5
[14:43:30 INFO]: Experience Merge Radius: 3.0
[14:43:30 INFO]: Mob Spawn Range: 6
[14:43:30 INFO]: Item Despawn Rate: 6000
[14:43:30 INFO]: Arrow Despawn Rate: 1200 Trident Respawn Rate:1200
[14:43:30 INFO]: Zombie Aggressive Towards Villager: true
[14:43:30 INFO]: Nerfing mobs spawned from spawners: false
[14:43:30 INFO]: Allow Zombie Pigmen to spawn from portal blocks: true
[14:43:30 INFO]: Entity Activation Range: An 32 / Mo 32 / Ra 48 / Mi 16 / Tiv true / Isa false
[14:43:30 INFO]: Entity Tracking Range: Pl 48 / An 48 / Mo 48 / Mi 32 / Di 128 / Other 64
[14:43:30 INFO]: Hopper Transfer: 8 Hopper Check: 1 Hopper Amount: 1 Hopper Can Load Chunks: false
[14:43:30 INFO]: View Distance: 10
[14:43:30 INFO]: Simulation Distance: 10
[14:43:30 INFO]: Custom Map Seeds:  Village: 10387312 Desert: 14357617 Igloo: 14357618 Jungle: 14357619 Swamp: 14357620 Monument: 10387313 Ocean: 14357621 Shipwreck: 165745295 End City: 10387313 Slime: 987234911 Nether: 30084232 Mansion: 10387319 Fossil: 14357921 Portal: 34222645
[14:43:30 INFO]: Max TNT Explosions: 100
[14:43:30 INFO]: Tile Max Tick Time: 50ms Entity max Tick Time: 50ms
[14:43:30 INFO]: Cactus Growth Modifier: 100%
[14:43:30 INFO]: Cane Growth Modifier: 100%
[14:43:30 INFO]: Melon Growth Modifier: 100%
[14:43:30 INFO]: Mushroom Growth Modifier: 100%
[14:43:30 INFO]: Pumpkin Growth Modifier: 100%
[14:43:30 INFO]: Sapling Growth Modifier: 100%
[14:43:30 INFO]: Beetroot Growth Modifier: 100%
[14:43:30 INFO]: Carrot Growth Modifier: 100%
[14:43:30 INFO]: Potato Growth Modifier: 100%
[14:43:30 INFO]: TorchFlower Growth Modifier: 100%
[14:43:30 INFO]: Wheat Growth Modifier: 100%
[14:43:30 INFO]: NetherWart Growth Modifier: 100%
[14:43:30 INFO]: Vine Growth Modifier: 100%
[14:43:30 INFO]: Cocoa Growth Modifier: 100%
[14:43:30 INFO]: Bamboo Growth Modifier: 100%
[14:43:30 INFO]: SweetBerry Growth Modifier: 100%
[14:43:30 INFO]: Kelp Growth Modifier: 100%
[14:43:30 INFO]: TwistingVines Growth Modifier: 100%
[14:43:30 INFO]: WeepingVines Growth Modifier: 100%
[14:43:30 INFO]: CaveVines Growth Modifier: 100%
[14:43:30 INFO]: GlowBerry Growth Modifier: 100%
[14:43:30 INFO]: PitcherPlant Growth Modifier: 100%
[14:43:30 INFO]: Preparing start region for dimension minecraft:overworld
[14:43:30 INFO]: Time elapsed: 266 ms
[14:43:30 INFO]: Preparing start region for dimension minecraft:the_nether
[14:43:30 INFO]: Time elapsed: 64 ms
[14:43:30 INFO]: Preparing start region for dimension minecraft:the_end
[14:43:30 INFO]: Time elapsed: 56 ms
[14:43:30 INFO]: [Geyser-Spigot] Enabling Geyser-Spigot v2.1.0-SNAPSHOT
[14:43:30 INFO]: [Geyser-Spigot] Disabling Geyser-Spigot v2.1.0-SNAPSHOT
[14:43:31 INFO]: Starting remote control listener
[14:43:31 INFO]: Thread RCON Listener started
[14:43:31 INFO]: RCON running on 0.0.0.0:25575
[14:43:31 INFO]: Running delayed init tasks
[14:43:31 INFO]: Done (3.866s)! For help, type "help"
[14:43:31 INFO]: *************************************************************************************
[14:43:31 INFO]: This is the first time you're starting this server.
[14:43:31 INFO]: It's recommended you read our 'Getting Started' documentation for guidance.
[14:43:31 INFO]: View this and more helpful information here: https://docs.papermc.io/paper/next-steps
[14:43:31 INFO]: *************************************************************************************
[14:43:31 INFO]: Timings Reset
```

It apperas the Geyser-Spigot.jar from May 27, 2023 is too outdated:

```
[14:43:29 ERROR]: [Geyser-Spigot] Error initializing plugin 'Geyser-Spigot.jar' in folder 'plugins' (Is it up to date?)
```

To [Update Plugins](https://docs.papermc.io/paper/updating#step-2-update-plugins)
in a PaperMC server, they need only be downloaded into the
server's `plugins/update` folder, and the next time the
server is restarted it will automatically update them.

1. Download the latest **Paper** version from https://hangar.papermc.io/GeyserMC/Geyser
2. Copy it into `/home/k8s/minecraft-server/plugins/update`
3. Restart the server with the 
   [start/stop scripts](../../../../2023-08-10-running-minecraft-java-server-for-bedrock-clients-on-kubernetes.htmlhtml#minecraft-start-k8s)

This time, the Geyser plugin starts successfully:

```
$ minecraft-stop-k8s && sleep 10 && minecraft-start-k8s
deployment.apps "minecraft-server" deleted
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (1/1), done.
remote: Total 3 (delta 2), reused 3 (delta 2), pack-reused 0
Unpacking objects: 100% (3/3), 297 bytes | 297.00 KiB/s, done.
From github.com:stibbons1990/lexicon-deployments
   b584f18..f0308df  main       -> origin/main
Updating b584f18..f0308df
Fast-forward
 minecraft-server.yaml | 4 +---
 1 file changed, 1 insertion(+), 3 deletions(-)
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server created

$ kubectl -n minecraft-server logs -f minecraft-server-545c4b56f5-6xmw5
...
[15:30:57 INFO]: [Geyser-Spigot] Enabling Geyser-Spigot v2.2.2-SNAPSHOT
[15:30:57 INFO]: [Geyser-Spigot] ******************************************
[15:30:57 INFO]: [Geyser-Spigot] 
[15:30:57 INFO]: [Geyser-Spigot] Loading Geyser version 2.2.2-SNAPSHOT (git-master-c64e8af)
[15:30:57 INFO]: [Geyser-Spigot] 
[15:30:57 INFO]: [Geyser-Spigot] ******************************************
[15:31:03 INFO]: [Geyser-Spigot] Started Geyser on 0.0.0.0:19132
[15:31:03 INFO]: [Geyser-Spigot] Done (5.952s)! Run /geyser help for help!
[15:31:03 INFO]: Starting remote control listener
[15:31:03 INFO]: Thread RCON Listener started
[15:31:03 INFO]: RCON running on 0.0.0.0:25575
[15:31:03 INFO]: Running delayed init tasks
[15:31:03 INFO]: Done (11.204s)! For help, type "help"
[15:31:03 INFO]: Timings Reset
[15:31:04 INFO]: [Geyser-Spigot] Downloading Minecraft JAR to extract required files, please wait... (this may take some time depending on the speed of your internet connection)
[15:31:05 INFO]: [Geyser-Spigot] Minecraft JAR has been successfully downloaded and loaded!
```

Reminder: the server is listening on port 25565 in the
container, but the service is mapping (`NodePort`) port
**32565** to it, so that is the port that must be used when
connecting to it from **Java** clients. For **Bedrock**
clients, the correct (UDP) port is **32132**.

### Fix The Scripts

With the Spigot server, to send commands to the server the scripts were sending
those via a pipeline. With the Paper server, this no longer works:

```
kubectl -n minecraft-server exec deploy/minecraft-server -- mc-send-to-console "say hi"
ERROR: console pipe needs to be enabled by setting CREATE_CONSOLE_IN_PIPE to true
ERROR: named pipe /tmp/minecraft-console-in is missing
command terminated with exit code 1
```

There seems to be nothing about `CREATE_CONSOLE_IN_PIPE` but there is
[Issue #2485](https://github.com/itzg/docker-minecraft-server/issues/2485):
*ERROR: named pipe /tmp/minecraft-console-in is missing* where
[the recommendation](https://github.com/itzg/docker-minecraft-server/issues/2485#issuecomment-1807339110)
is to use `rcon-cli` instead:

```
$ kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli "say hi"
```

These show in the logs as

```
[17:38:28 INFO]: Thread RCON Client /0:0:0:0:0:0:0:1 started
[17:38:28 INFO]: Thread RCON Client /0:0:0:0:0:0:0:1 shutting down
[17:38:28 INFO]: [Not Secure] [Rcon] hi
```

### Further Improvements

#### Tweak Flags

[Recommended JVM Startup Flags](https://docs.papermc.io/paper/aikars-flags)
include allocating **10 GB** of RAM to the server.
This *should* be fine since the node has a total of 32 GB.

*We recommend using at least 6-10GB, no matter how few players! If you can't afford 10GB of memory, give as much as you can, but ensure you leave the operating system some memory too. G1GC operates better with more memory.*

```sh
java -Xms10G -Xmx10G -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 
-XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch 
-XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40 -XX:G1HeapRegionSize=8M 
-XX:G1ReservePercent=20 -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 
-XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 
-XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem 
-XX:MaxTenuringThreshold=1 -Dusing.aikars.flags=https://mcflags.emc.gs 
-Daikars.new.flags=true -jar paper.jar --nogui
```

The RAM allocation flags `-Xms10G -Xmx10G` are already
set from the `MEMORY` variable in the deployment:

```yaml
          env:
            ...
            - name: MEMORY
              value: 10G
```

Other `XX` flags must be set via the `env` variable
`JVM_XX_OPTS` in the deployment, as explained in
[Variables > General options](https://docker-minecraft-server.readthedocs.io/en/latest/variables/#general-options):

```yaml
          env:
            - name: MEMORY
              value: 10G
            - name: JVM_XX_OPTS
              value: >
                  -XX:+UseG1GC
                  -XX:+ParallelRefProcEnabled
                  -XX:MaxGCPauseMillis=200
                  -XX:+UnlockExperimentalVMOptions
                  -XX:+DisableExplicitGC
                  -XX:+AlwaysPreTouch
                  -XX:G1NewSizePercent=30
                  -XX:G1MaxNewSizePercent=40
                  -XX:G1HeapRegionSize=8M
                  -XX:G1ReservePercent=20
                  -XX:G1HeapWastePercent=5
                  -XX:G1MixedGCCountTarget=4
                  -XX:InitiatingHeapOccupancyPercent=15
                  -XX:G1MixedGCLiveThresholdPercent=90
                  -XX:G1RSetUpdatingPauseTimePercent=5
                  -XX:SurvivorRatio=32
                  -XX:+PerfDisableSharedMem
                  -XX:MaxTenuringThreshold=1
                  -Dusing.aikars.flags=https://mcflags.emc.gs
                  -Daikars.new.flags=true
```

**Note:** the `--nogui` flag does not seem to be necessary.

#### Floodgate

The companion to Geyser,
[Floodgate](https://hangar.papermc.io/GeyserMC/Floodgate)
is a hybrid mode plugin which allows *Minecraft: Bedrock
Edition* connections to join online mode *Minecraft: Java
Edition* servers without the need for a *Java Edition*
**account**. Bedrock clients are still required to be
logged in through Microsoft.

[Adding Plugins](https://docs.papermc.io/paper/adding-plugins)
should be a simple as downloading the latest version,
dropping it under
`/home/k8s/minecraft-server/plugins/` (not `update`),
and then restarting the server:

```
$ minecraft-stop-k8s && sleep 10 && \
  minecraft-start-k8s && sleep 20 && \
  minecraft-logs
deployment.apps "minecraft-server" deleted
Already up to date.
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server created
...
[16:02:43 INFO]: [floodgate] Loading server plugin floodgate v2.2.2-SNAPSHOT (b96-7f38765)
[16:02:44 INFO]: [floodgate] Took 1,508ms to boot Floodgate
...
[15:58:19 INFO]: [floodgate] Enabling floodgate v2.2.2-SNAPSHOT (b96-7f38765)
[15:58:20 INFO]: [Geyser-Spigot] Enabling Geyser-Spigot v2.2.2-SNAPSHOT
[15:58:20 INFO]: [Geyser-Spigot] ******************************************
[15:58:20 INFO]: [Geyser-Spigot] 
[15:58:20 INFO]: [Geyser-Spigot] Loading Geyser version 2.2.2-SNAPSHOT (git-master-c64e8af)
[15:58:20 INFO]: [Geyser-Spigot] 
[15:58:20 INFO]: [Geyser-Spigot] ******************************************
```

**Note:** it apperas that a `.` is prepended to the username of
users logging via Floodgate; this is important to use with server commands:

```
[17:14:49 INFO]: [Geyser-Spigot] /10.244.0.1:61211 tried to connect!
[17:14:50 INFO]: [Geyser-Spigot] Player connected with username Isa________45
[17:14:50 INFO]: [Geyser-Spigot] Isa________45 (logged in as: Isa________45) has connected to the Java server
[17:14:51 INFO]: [floodgate] Floodgate player logged in as .Isa________45 joined (UUID: 00000000-0000-0000-0009-0__________8)
```

To send a command for this user, the `.` must be in front of the username:

```
$ kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli "gamemode survival Isa________45"
No player was found

$ kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli "gamemode survival .Isa________45"
Set .Isa________45's game mode to Survival Mode
```

#### Perms

[LuckPerms](https://luckperms.net/) is
*a permissions plugin for Minecraft servers*.

Once again, download the latest version,
drop it under `/home/k8s/minecraft-server/plugins/` 
and restart the server:

```
$ minecraft-stop-k8s && sleep 10 && \
  minecraft-start-k8s && sleep 20 && \
  minecraft-logs
deployment.apps "minecraft-server" deleted
Already up to date.
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server created
...
[16:02:47 INFO]: Server permissions file permissions.yml is empty, ignoring it
[16:02:47 INFO]: [LuckPerms] Enabling LuckPerms v5.4.121
[16:02:47 INFO]:         __    
[16:02:47 INFO]:   |    |__)   LuckPerms v5.4.121
[16:02:47 INFO]:   |___ |      Running on Bukkit - Paper
[16:02:47 INFO]: 
[16:02:47 INFO]: [LuckPerms] Loading configuration...
[16:02:48 INFO]: [LuckPerms] Loading storage provider... [H2]
[16:02:48 INFO]: [LuckPerms] Loading internal permission managers...
[16:02:48 INFO]: [LuckPerms] Performing initial data load...
[16:02:48 INFO]: [LuckPerms] Successfully enabled. (took 1319ms)
...
```

Next, decide what permissions to set on each user/s and
continue from 
[Step 4: Configuring LuckPerms](https://luckperms.net/wiki/Installation).

### Install Plugins From Deployment

Plugins can also be defined installed by simply adding
their download URLs in the `PLUGINS` variable from, as in
[/examples/geyser/docker-compose.yml](https://github.com/itzg/docker-minecraft-server/blob/master/examples/geyser/docker-compose.yml):

```yaml
          env:
            - name: EULA
              value: "TRUE"
            - name: TYPE
              value: PAPER
            - name: PLUGINS
              value: |
                https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot
                https://download.geysermc.org/v2/projects/floodgate/versions/latest/builds/latest/downloads/spigot
                https://download.luckperms.net/1534/bukkit/loader/LuckPerms-Bukkit-5.4.121.jar
            - name: MEMORY
              value: 10G
```

#### Replace Timings Profiler with Spark

Follow the recommendation from the logs:

```
[15:30:53 WARN]: [!] The timings profiler has been enabled but has been scheduled for removal from Paper in the future.
    We recommend installing the spark profiler as a replacement: https://spark.lucko.me/
    For more information please visit: https://github.com/PaperMC/Paper/issues/8948
```

Having read the start and end of
[PaperMC issue #8948](https://github.com/PaperMC/Paper/issues/8948): *Replace Timings with Spark*
as the discussion stands now, this will be a place to
revisit in the future once their plans have solidified.
