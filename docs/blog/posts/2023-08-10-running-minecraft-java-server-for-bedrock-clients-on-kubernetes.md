---
date: 2023-08-10
categories:
 - ubuntu
 - server
 - linux
 - kubernetes
 - docker
 - minecraft
 - java
 - bedrock
title: Running Minecraft Java Server for Bedrock clients on Kubernetes
---

Minecraft Java Edition requires that servers match the
version of the clients and updating the server each time
is a bit of a chore, so it is more convenient to run it
on the Kubernetes cluster.

<!-- more --> 

## Basic Setup

Always run Minecraft servers under their own dedicated,
non-privileged user. Create also a dedicated directory
in the partition with plenty of space available:

``` console
# useradd minecraft
# mkdir /home/k8s/minecraft-server
# chown -R minecraft.minecraft /home/k8s/minecraft-server
# ls -ldn /home/k8s/minecraft-server
drwxr-xr-x 1 1003 1003 0 May 29 14:43 /home/k8s/minecraft-server
```

A persistent volume is necessary because otherwise
everything (server settings and the whole world) is lost
when the container is restarted, including when the node
is restarted.

Note that **UID & GID 1003** will be needed later to run
the server as this user.

## Kubernetes Deployment

Create and apply the deployment in
`minecraft-server.yaml` using the
[itzg/minecraft-server](https://hub.docker.com/r/itzg/minecraft-server)
docker image (GitHub:
[itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server)):

??? k8s "Kubernetes deployment: `minecraft-server.yaml`"

    ``` yaml linenums="1" title="minecraft-server.yaml"
    apiVersion: v1
    kind: Namespace
    metadata:
      name: minecraft-server
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: minecraft-server
      namespace: minecraft-server
    spec:
      type: NodePort
      ports:
        - name: java-tcp
          port: 25565
          nodePort: 32565
          targetPort: 25565
          protocol: TCP
        - name: bedrock-udp
          port: 19132
          nodePort: 32132
          targetPort: 19132
          protocol: UDP
      selector:
        app: minecraft-server
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: minecraft-server-pv
      labels:
        type: local
      namespace: minecraft-server
    spec:
      storageClassName: manual
      capacity:
        storage: 100Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /home/k8s/minecraft-server
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: minecraft-server-pv-claim
      namespace: minecraft-server
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: minecraft-server
      name: minecraft-server
      namespace: minecraft-server
    spec:
      selector:
        matchLabels:
          app: minecraft-server
      template:
        metadata:
          labels:
            app: minecraft-server
        spec:
          volumes:
            - name: minecraft-server-storage
              persistentVolumeClaim:
                claimName: minecraft-server-pv-claim
          containers:
            - image: itzg/minecraft-server
              imagePullPolicy: Always
              name: minecraft-server
              ports:
                - containerPort: 25565
              env:
                - name: EULA
                  value: "TRUE"
                - name: TYPE
                  value: SPIGOT
                - name: MEMORY
                  value: 4G
              volumeMounts:
                - mountPath: /data
                  name: minecraft-server-storage
              securityContext:
                allowPrivilegeEscalation: false
                runAsUser: 1003
                runAsGroup: 1003
    ```

!!! note

    To make the server accessible to clients, the above `NodePort`
    is required for each the Java and Bedrock protols separately:

*  The Java server listens on **TCP** port 25565, the
   cluster exposes the server to the local network as
   `192.168.0.6:32565`
*  The Bedrock server listens on **UDP** port 19132, the
   cluster exposes the server to the local network as
   `192.168.0.6:32132`

Both ports can then be exposed externally by adding port
forwarding rules in the local router.

Once the deployment is applied, confirm everything is
running and check the logs:

``` console
$ kubectl apply -f minecraft-server.yaml
namespace/minecraft-server unchanged
service/minecraft-server unchanged
persistentvolume/minecraft-server-pv unchanged
persistentvolumeclaim/minecraft-server-pv-claim unchanged
deployment.apps/minecraft-server configured

$ kubectl -n minecraft-server get all
NAME                                    READY   STATUS    RESTARTS      AGE
pod/minecraft-server-88f84b5fc-ptb5p   1/1     Running   0          2m38s


NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/minecraft-server   ClusterIP   10.110.215.139   <none>        25565/TCP   2m38s
service/minecraft-server   NodePort   10.110.215.139   <none>        25565:32565/TCP,19132:32132/UDP   2m38s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minecraft-server   1/1     1            1           2m38s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/minecraft-server-88f84b5fc    1         1         1       2m38s

$ kubectl -n minecraft-server logs pod/minecraft-server-88f84b5fc-ptb5p
[init] Running as uid=1003 gid=1003 with /data as 'drwxrwxr-x 1 1003 1003 336 May 29 12:57 /data'
[init] Resolved version given LATEST into 1.19.4 and major version 1.19
[init] Resolving type given VANILLA
[init] Setting initial memory to 1G and max to 1G
[init] Starting the Minecraft server...
Starting net.minecraft.server.Main
[12:59:32] [ServerMain/INFO]: Environment: authHost='https://authserver.mojang.com', accountsHost='https://api.mojang.com', sessionHost='https://sessionserver.mojang.com', servicesHost='https://api.minecraftservices.com', name='PROD'
[12:59:33] [ServerMain/INFO]: Loaded 7 recipes
[12:59:33] [ServerMain/INFO]: Loaded 1179 advancements
[12:59:34] [Server thread/INFO]: Starting minecraft server version 1.19.4
[12:59:34] [Server thread/INFO]: Loading properties
[12:59:34] [Server thread/INFO]: Default game type: SURVIVAL
[12:59:34] [Server thread/INFO]: Generating keypair
[12:59:34] [Server thread/INFO]: Starting Minecraft server on *:25565
[12:59:34] [Server thread/INFO]: Using epoll channel type
[12:59:34] [Server thread/INFO]: Preparing level "world"
[12:59:39] [Server thread/INFO]: Preparing start region for dimension minecraft:overworld
[12:59:39] [Server thread/INFO]: Preparing spawn area: 0%
[12:59:39] [Worker-Main-2/INFO]: Preparing spawn area: 0%
[12:59:40] [Worker-Main-2/INFO]: Preparing spawn area: 91%
[12:59:40] [Worker-Main-1/INFO]: Preparing spawn area: 91%
[12:59:40] [Server thread/INFO]: Time elapsed: 1816 ms
[12:59:40] [Server thread/INFO]: Done (6.264s)! For help, type "help"
[12:59:40] [Server thread/INFO]: Starting remote control listener
[12:59:40] [Server thread/INFO]: Thread RCON Listener started
[12:59:40] [Server thread/INFO]: RCON running on 0.0.0.0:25575
```

### Geyser plugin for Bedrock clients

[GeyserMC](https://geysermc.org/)
is a program that allows Bedrock clients to join
Java servers. Installing this requires a *Paper* or
**Spigot** server (source:
[geysermc.org/download#spigot](https://geysermc.org/download#spigot)).
This is why the above deployment specifies the server to
be of type `SPIGOT`.

Install from latest binary release:

``` console
# su minecraft -c "wget -O /home/k8s/minecraft-server/plugins/Geyser-Spigot.jar https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot"
# ls -hal /home/k8s/minecraft-server/plugins/
total 14M
drwxrwxr-x 1 minecraft minecraft  60 May 29 16:08 .
drwxrwxr-x 1 minecraft minecraft 572 May 29 15:48 ..
-rw-rw-r-- 1 minecraft minecraft 14M May 27 15:19 Geyser-Spigot.jar
drwxrwxr-x 1 minecraft minecraft  20 May 29 15:47 PluginMetrics
```

### Server Commands

To enter commands into the running server, see
[Interacting with the server](https://github.com/itzg/docker-minecraft-server/blob/master/README.md#interacting-with-the-server)
and in particular the `rcon-cli` command that
can be run in the container.

No need to get the full name of the current pod, which
changes when deployment restarts:

``` console
$ kubectl -n minecraft-server \
  exec deploy/minecraft-server \
   -- rcon-cli difficulty peaceful
```

Note that using `deploy/minecraft-server` allows running
the command in the pod of this deployment
[without having to get the full name](https://stackoverflow.com/questions/63626842/kubectl-exec-into-a-container-without-using-the-random-guid-at-the-end-of-the-po)
of the pod with `kubectl -n minecraft-server get pods`

With this, one can `reload` the configurations (e.g.
`server.properties`) or even `restart` the whole Java
server without restarting the pod or deployment.

!!! warning

    **Do not** try attaching to the TTY of the container.

``` console
$ kubectl -n minecraft-server get pods
NAME                                READY   STATUS    RESTARTS       AGE
minecraft-server-b89954df9-t9dxg   1/1     Running   2 (112s ago)   2m21s
$ kubectl -n minecraft-server attach -it minecraft-server-b89954df9-t9dxg
If you don't see a command prompt, try pressing enter.
/help
[15:27:18] [Server thread/INFO]: Unknown command. Type "/help" for help.
```

This only seems to show the logs, but commands are not accepted, not even `/help`.
**Even worse:** the only way to detach from the pod is
with `Ctrl+C` and that **kills** the server
**without saving** the worlds! So it seems the `stdin`
and `tty` options are not a good idea. 

### Server Config

To adjust server-wide or default settings, edit the
`server.properties` file and reload it, e.g.

``` console
# su minecraft -c "vi /home/k8s/minecraft-server/server.properties"
gamemode=creative
motd=Be good
pvp=false
difficulty=easy
max-players=5
```

Then send the **reload** [command](#server-commands) to
reload the config without restarting the server.

### Access Control

To restrict access to a few (trusted) users, add them
with their UUID to the `whitelist.json` file:

``` json linenums="1" title="vi/home/k8s/minecraft-server/whitelist.json"
[
  {
    "uuid": "____1e97-____-____-____-f7187fd7____",
    "name": "L________a"
  },
  {
    "uuid": "____41f4-____-____-____-de0061cf____",
    "name": "M________t"
  }
]
```

To find users’ UUID, check the server’s logs:

``` console
# grep -i uuid /home/k8s/minecraft-server/logs/latest.log 
[14:35:10] [User Authenticator #2/INFO]: UUID of player L________a is ____1e97-____-____-____-f7187fd7____
[14:44:06] [User Authenticator #3/INFO]: UUID of player M________t is ____41f4-____-____-____-de0061cf____
```

Need to restart the deployment to make those changes
effective, but first must activate the whitelist by
setting the following values in
`/home/k8s/minecraft-server/server.properties`

``` ini linenums="1" title="/home/k8s/minecraft-server/server.properties"
white-list=true
enforce-whitelist=true
```

Then send the **reload** [command](#server-commands) to
reload the config without restarting the server.

If restarting *the whole server* becomes necessary:

``` console
$ kubectl rollout restart \
  deployment/minecraft-server -n minecraft-server
```

An easier way would be to use the
[EasyWhitelist](https://www.spigotmc.org/resources/easywhitelist-name-based-whitelist.65222/)
tool in SpigotMC, but it looks like it's broken.

### Hourly Backups

Minecraft servers rely *too much* on players behaving,
which of course is a strategy that has proven problematic
many times over.

To recover from any disasters, create hourly backups as
the `root` user in the node. The following scripts
creates one full backup per hour, so that even within a
single day multiple backups are available to restore:

``` bash linenums="1"
#!/bin/bash
minecraft_server_cmd () {
  su coder -c "kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*"
}
minecraft_server_cmd "say Starting full backup."
minecraft_server_cmd "save-off"
minecraft_server_cmd "save-all"
sleep 5
rsync -ahrtp --delete \
  /home/k8s/minecraft-server \
  /home/k8s/minecraft-server-backups/$(date +"%H")
minecraft_server_cmd "save-on"
minecraft_server_cmd "say Backup complete."
```

Put this script in a dedicated directory for the backups
and run it every hour with `crontab`:

``` console
# mkdir /home/k8s/minecraft-server-backups
# vi /home/k8s/minecraft-server-backups/backup.sh
# crontab -e
00  * * * * /home/k8s/minecraft-server-backups/backup.sh
```

### Daily Server Restart

To keep the server up to date, it is easiest to just
restart the whole deployment every day. Besides, there
is no need to keep the server running overnight because
this is a private server, not used from multiple
timezones.

Find the [`minecraft-start-k8s`](#minecraft-start-k8s)
and [`minecraft-stop-k8s`](#minecraft-stop-k8s)
script in the [Appendix](#appendix-more-server-commands) below.

!!! note 

    These commands must be run as the user who has
    [the credentials to run `kubectl`](2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#bootstrap):

``` console
$ crontab -e
10  6 * * *   /home/coder/bin/minecraft-start-k8s
30 22 * * *   /home/coder/bin/minecraft-stop-k8s
```

### World Reset

If at some point we want to start a new world, it is as
simple as renaming `/home/k8s/minecraft-server/world`
to any name when the server is **not running**.
The next time the server starts, a new 
`/home/k8s/minecraft-server/world` folder will 
be created with *a whole new world*.

``` console
# /home/coder/bin/minecraft-stop-k8s
# mv /home/k8s/minecraft-server/world \
  /home/k8s/minecraft-server/old_world
# /home/coder/bin/minecraft-start-k8s
```

## Appendix: more server commands

### `minecraft-server-fortune`

The `minecraft-server-fortune` script ...

{% raw %}
``` bash linenums="1" title="minecraft-server-fortune"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

declare -a fortunes
fortunes[0]="Be good, or be gone."
fortunes[1]="Be nice, or pay the price."
fortunes[2]="Think of the others, don't be a bother."
fortunes[3]="Utilize Bearded Dragon."
fortunes[4]="Agree to disagree."
fortunes[5]="Steams my broccoli."
len=${#fortunes[@]}-1
i=$(shuf -i 0-$((len-1)) -n 1)
fortune=${fortunes[$i]}
minecraft_server_cmd "say $fortune"
```
{% endraw %}

### `minecraft-server-kick`

The `minecraft-server-kick` script will `kick` the user
that is passed as argument:

``` bash linenums="1" title="minecraft-server-kick"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "kick $1"
```

### `minecraft-server-make-creative`

The `minecraft-server-make-creative` script changes the
server into creative mode.

``` bash linenums="1" title="minecraft-server-make-creative"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "gamemode creative M________t"
minecraft_server_cmd "gamemode creative L________a"
minecraft_server_cmd "difficulty peaceful"
```

### `minecraft-server-make-survival`

The `minecraft-server-make-survival` script changes the
server into survival mode.

``` bash linenums="1" title="minecraft-server-make-survival"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "gamemode survival M________t"
minecraft_server_cmd "gamemode survival L________a"
minecraft_server_cmd "difficulty normal"
```

### `minecraft-server-say-shutdown`

The `minecraft-server-say-shutdown` script shuts the
server down (necessary before making a backup).

``` bash linenums="1" title="minecraft-server-say-shutdown"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "say WARNING: server WILL SHUT DOWN in $time"
```

### `minecraft-server-tp-l--------a-m--------t`

The `minecraft-server-tp-l--------a-m--------t` script
*teleports* (`tp`) a certain user wher the other one is.

``` bash linenums="1" title="minecraft-server-tp-l--------a-m--------t"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "tp L________a M________t"
```

### `minecraft-server-tp-l--------a-m--------t`

The `minecraft-server-tp-S-----------e-m--------t` script 
*teleports* (`tp`) a certain user wher the other one is.

``` bash linenums="1" title="minecraft-server-tp-l--------a-m--------t"
#!/bin/bash
minecraft_server_cmd () {
  kubectl -n minecraft-server exec deploy/minecraft-server -- rcon-cli $*
}

minecraft_server_cmd "tp S______________1 M________t"
```

### `minecraft-start-k8s`

The `minecraft-start-k8s` script starts the server by
applying the deployment.

``` bash linenums="1" title="minecraft-start-k8s"
#!/bin/bash

cd /home/coder/head/lexicon-deployments
git pull
kubectl apply -f minecraft-server.yaml
```

### `minecraft-stop-k8s`

The `minecraft-stop-k8s` script stops the server by
removing the deployment.

``` bash linenums="1" title="minecraft-stop-k8s"
#!/bin/bash

kubectl delete -n minecraft-server deployment minecraft-server
```

### `minecraft-logs`

The `minecraft-logs` script shows server logs as they are
produces (with `-f`).

``` bash linenums="1" title="minecraft-logs"
#!/bin/bash

pod=$(kubectl get pods -n minecraft-server | grep '1/1' | awk '{print $1}')
kubectl -n minecraft-server logs -f "${pod}"
```
