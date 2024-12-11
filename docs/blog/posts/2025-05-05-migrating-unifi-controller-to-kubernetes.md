---
date: 2025-05-05
draft: true
categories:
 - linux
 - kubernetes
 - docker
 - server
 - unifi
---

# Migrating UniFi Controller to Kubernetes

<!-- more -->

Install
[linuxserver.io/docker-unifi-network-application/](https://docs.linuxserver.io/images/docker-unifi-network-application/)
losely based on
[Upgrading to linuxserver.io unifi-network-application docker](https://www.reddit.com/r/unRAID/comments/1abxn3i/upgrading_to_linuxserverio/)

```
# useradd uni
# mkdir -p /home/k8s/unifi/config
# mkdir -p /home/k8s/unifi/mongodb
# vi /home/k8s/unifi/init-mongo.sh
# chown -R uni.uni /home/k8s/unifi
# ls -lan /home/k8s/unifi
total 4
drwxr-xr-x 1 119 125  60 Oct 27 13:11 .
drwxr-xr-x 1   0   0 382 Oct 27 13:11 ..
drwxr-xr-x 1 119 125   0 Oct 27 13:11 config
-rw-r--r-- 1 119 125 424 Oct 27 13:11 init-mongo.sh
drwxr-xr-x 1 119 125   0 Oct 27 13:11 mongodb
```

```bash
#!/bin/bash

if which mongosh > /dev/null 2>&1; then
  mongo_init_bin='mongosh'
else
  mongo_init_bin='mongo'
fi
"${mongo_init_bin}" <<EOF
use ${MONGO_AUTHSOURCE}
db.auth("${MONGO_INITDB_ROOT_USERNAME}", "${MONGO_INITDB_ROOT_PASSWORD}")
db.createUser({
  user: "${MONGO_USER}",
  pwd: "${MONGO_PASS}",
  roles: [
    { db: "${MONGO_DBNAME}", role: "dbOwner" },
    { db: "${MONGO_DBNAME}_stat", role: "dbOwner" }
  ]
})
EOF
```

```
$ kubectl apply -f unifi.yaml
namespace/unifi created
persistentvolume/mongo-pv-data created
persistentvolume/mongo-pv-init created
persistentvolumeclaim/mongo-pvc-data created
persistentvolumeclaim/mongo-pvc-init created
deployment.apps/mongo created
service/mongo-svc created
persistentvolume/unifi-pv-config created
persistentvolumeclaim/unifi-pvc-config created
deployment.apps/unifi created
service/unifi-tcp created
service/unifi-udp created
ingress.networking.k8s.io/unifi-ingress created
```

```
$ kubectl get all -n unifi
NAME                            READY   STATUS             RESTARTS      AGE
pod/cm-acme-http-solver-ldk6m   1/1     Running            0             2m58s
pod/mongo-5f4976899-k8mwr       1/1     Running            0             3m2s
pod/unifi-7c7d755bb-d95hl       0/1     CrashLoopBackOff   4 (67s ago)   3m2s

NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                         AGE
service/cm-acme-http-solver-7nd88   NodePort       10.100.201.83   <none>          8089:31931/TCP                                  2m58s
service/mongo-svc                   NodePort       10.104.94.112   <none>          27017:32717/TCP                                 3m2s
service/unifi-tcp                   LoadBalancer   10.102.178.6    192.168.0.123   8443:30359/TCP,8080:30095/TCP,6789:30627/TCP    3m2s
service/unifi-udp                   LoadBalancer   10.104.17.91    192.168.0.123   3478:30876/UDP,10001:32211/UDP,1900:30162/UDP   3m2s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo   1/1     1            1           3m2s
deployment.apps/unifi   0/1     1            0           3m2s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-5f4976899   1         1         1       3m2s
replicaset.apps/unifi-7c7d755bb   1         1         0       3m2s
```

Note: this Unifi image does not support running rootless, thus
attempting to set `securityContext` (as is done for the `mongodb`
image) will result in fatal errors and crash-loop:

```
s6-overlay-suexec: warning: unable to gain root privileges (is the suid bit set?)
s6-mkdir: warning: unable to mkdir /run/s6: Permission denied
s6-mkdir: warning: unable to mkdir /run/service: Permission denied
s6-overlay-suexec: fatal: child failed with exit code 111
```

Having avoided that problem, the Unifi application still fails to
start because it's not correctly receiving the environment
variables to connect to MondoDB:

```
$ kubectl -n unifi logs $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ') -f
[custom-init] No custom files found, skipping...
Exception in thread "launcher" java.lang.IllegalStateException: Tomcat failed to start up
        at com.ubnt.net.Object.ôÔ0000(Unknown Source)
        at com.ubnt.service.ooOO.Òo0000(Unknown Source)
        at com.ubnt.ace.Launcher.Object(Unknown Source)
        at com.ubnt.ace.Launcher.main(Unknown Source)
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'mongoRuntimeService' defined in com.ubnt.service.db.CoreDatabaseSpringContext: The connection string contains an invalid host '~MONGO_HOST~:~MONGO_PORT~'. The port '~MONGO_PORT~' is not a valid, it must be an integer between 0 and 65535
```

The `~MONGO_PORT~` string is coming from one or both of these
`system.properties` files, where
[they should be replaced](https://github.com/linuxserver/docker-unifi-network-application/blob/main/root/etc/s6-overlay/s6-rc.d/init-unifi-network-application-config/run#L48-L52)
with the values of those environment variables set in the
deployment; but they are not:

```
$ kubectl -n unifi exec $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ')  -- cat /defaults/system.properties | grep MONGO
db.mongo.uri=mongodb://~MONGO_USER~:~MONGO_PASS~@~MONGO_HOST~:~MONGO_PORT~/~MONGO_DBNAME~?tls=~MONGO_TLS~~MONGO_AUTHSOURCE~
statdb.mongo.uri=mongodb://~MONGO_USER~:~MONGO_PASS~@~MONGO_HOST~:~MONGO_PORT~/~MONGO_DBNAME~_stat?tls=~MONGO_TLS~~MONGO_AUTHSOURCE~
unifi.db.name=~MONGO_DBNAME~

$ kubectl -n unifi exec $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ')  -- cat /config/data/system.properties | grep MONGO
statdb.mongo.uri=mongodb\://~MONGO_USER~\:~MONGO_PASS~@~MONGO_HOST~\:~MONGO_PORT~/~MONGO_DBNAME~_stat?tls\=~MONGO_TLS~~MONGO_AUTHSOURCE~
unifi.db.name=~MONGO_DBNAME~
db.mongo.uri=mongodb\://~MONGO_USER~\:~MONGO_PASS~@~MONGO_HOST~\:~MONGO_PORT~/~MONGO_DBNAME~?tls\=~MONGO_TLS~~MONGO_AUTHSOURCE~
```

The environment variables are correctly set in the running pod:

```
$ kubectl -n unifi exec $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ')  -- printenv | grep MONGO
MONGO_DBNAME=unifi
MONGO_HOST=mongo-svc
MONGO_PORT=27017
MONGO_PASS=*************************
MONGO_AUTHSOURCE=admin
MONGO_USER=unifi
MONGO_SVC_SERVICE_HOST=10.104.94.112
MONGO_SVC_PORT_27017_TCP=tcp://10.104.94.112:27017
MONGO_SVC_PORT_27017_TCP_PROTO=tcp
MONGO_SVC_PORT_27017_TCP_ADDR=10.104.94.112
MONGO_SVC_PORT_27017_TCP_PORT=27017
MONGO_SVC_SERVICE_PORT=27017
MONGO_SVC_PORT=tcp://10.104.94.112:27017

$ kubectl -n unifi exec $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ')  -- cat /run/s6/container_environment/MONGO_PORT
27017

$ kubectl -n unifi exec $(kubectl get pods -n unifi | grep unifi | cut -f1 -d' ')  -- nc -zv mongo-svc 27017
Connection to mongo-svc (10.104.94.112) 27017 port [tcp/*] succeeded!
```
However,
[this comment](https://github.com/linuxserver/docker-unifi-network-application/issues/80#issuecomment-2017493404)
suggests these are not read from the pod's environment, but
instead *services get access to the whole container environment*.

Setting these as global variables still does not get them picked
up by the `unifi` pod:

```
$ export MONGO_DBNAME=unifi
export MONGO_HOST=mongo-svc
export MONGO_PORT=27017
export MONGO_PASS='*************************'
export MONGO_AUTHSOURCE=admin
export MONGO_USER=unifi
export MONGO_SVC_SERVICE_HOST=10.104.94.112
export MONGO_SVC_PORT_27017_TCP=tcp://10.104.94.112:27017
export MONGO_SVC_PORT_27017_TCP_PROTO=tcp
export MONGO_SVC_PORT_27017_TCP_ADDR=10.104.94.112
export MONGO_SVC_PORT_27017_TCP_PORT=27017
export MONGO_SVC_SERVICE_PORT=27017
export MONGO_SVC_PORT=tcp://10.104.94.112:27017
$ kubectl rollout restart -n unifi deployment unifideployment.apps/unifi restarted
```


**Note:** avoid following
[Install Unifi Controller on Kubernetes](https://anakinfoxe.com/blog/install-unifi-on-k8s/)
which is based on the **deprecated**
[linuxserver/unifi-controller](https://hub.docker.com/r/linuxserver/unifi-controller).
Avoid also
[Setting up the UniFi Network Controller using Docker](https://pimylifeup.com/unifi-docker/)
using the outdated
[jacobalberty/unifi-docker](https://github.com/jacobalberty/unifi-docker).