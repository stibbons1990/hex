---
date: 2023-05-29
categories: 
 - ubuntu
 - server
 - linux
 - kubernetes
 - docker
 - vscode
 - self-hosted
title: Running Visual Studio Code Server on Kubernetes
---

I would like to be able to *code from anywhere*, or at
least have a development environment that doesn't depend
on my PC.

<!-- more --> 

[Visual Studio Code Server](https://code.visualstudio.com/docs/remote/vscode-server)
seems like one of the best and/or most popular options
out there, so I decided it to try.

Since I've already got my
[Kubernetes cluster](../../../../2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md)
running, the best option seems to be the
[lscr.io/linuxserver/code-server](https://hub.docker.com/r/linuxserver/code-server)
docker image.
I took the deployment from 
[Deploying VSCode on a Kubernetes Cluster](https://www.sobyte.net/post/2021-12/deploy-vscode-on-k8s/)
and
[added an Nginx ingress](../../../../2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#add-ingress-for-the-first-pod).

First, create a dedicated directory in the partition with
plenty of space available:

```
# mkdir /home/k8s/code-server/
# chown -R coder.coder /home/k8s/code-server/
```

A persistent volume is necessary because otherwise
everything (settings and uncommitted changes) is lost
when the container is restarted, including when the node
is restarted. Tried claiming a persistent volume from
[the default storage class local-path](../../../../2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#localpath-pv-provisioner)
but never managed to get the pod to see the claim, until
the claim was changed to the manual storage class to get
a `hostPath` volume, which worked immediately (only,
after granting `coder` all rights on the `hostPath`
directory).

Then create `code-server.yaml` with the deployment and
`apply` it:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: code-server
---
apiVersion: v1
kind: Service
metadata:
 name: code-server
 namespace: code-server
spec:
 ports:
 - port: 80
   targetPort: 8080
 selector:
   app: code-server
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: code-server-pv
  labels:
    type: local
  namespace: code-server
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/k8s/code-server"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: code-server-pv-claim
  namespace: code-server
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: code-server
  name: code-server
  namespace: code-server
spec:
  selector:
    matchLabels:
      app: code-server
  template:
    metadata:
      labels:
        app: code-server
    spec:
      volumes:
        - name: code-server-storage
          persistentVolumeClaim:
            claimName: code-server-pv-claim
      containers:
      - image: codercom/code-server
        imagePullPolicy: IfNotPresent
        name: code-server
        ports:
        - containerPort: 8080
        env:
        - name: PASSWORD
          value: "*************************"
        volumeMounts:
          - mountPath: "/home/coder"
            name: code-server-storage
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: code-server-ingress
  namespace: code-server
  annotations:
    acme.cert-manager.io/http01-edit-in-place: "true"
    cert-manager.io/issue-temporary-certificate: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  rules:
    - host: "code.ssl.uu.am"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: code-server
                port:
                  number: 80
  tls:
    - secretName: tls-secret
      hosts:
        - "code.ssl.uu.am"
```

```
$ kubectl apply -f code-server.yaml
```

Add an **A record** for `code.ssl.uu.am` pointing to the external IP and go to
[https://code.ssl.uu.am](https://code.ssl.uu.am)
to get the certificate creation started,
[forward port 80](../../../../2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#monthly-renewal-of-certificates)
to the `NodePort` of the `cm-acme-http-solver`:

```
$ kubectl -n code-server get svc
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
cm-acme-http-solver-dcwjx   NodePort    10.109.153.157   <none>        8089:31691/TCP   8m53s
```
