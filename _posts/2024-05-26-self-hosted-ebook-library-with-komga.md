---
title:  "Self-hosted eBook library with Komga"
date:   2024-05-26 16:05:24 +0200
categories: linux kubernetes docker server ebook komga
---

After weeks of using
[Audiobookshelf]({{ site.baseurl }}/2024/02/28/audiobookshelf-on-kubernetes.html)
to listen to audiobooks daily, it dawned on me that the PDF reader
was probably not the best I could be using.

{% assign media = site.baseurl | append: "/assets/media/" | append: page.path | replace: ".md","" | replace: "_posts/",""  %}

Then is also dawned on me that Audible is not my only source of
eBooks; I have a few from HumbleBundle deals and a few indipendent
authors who sell PDF files directly, as well as a small collection
of appliance manuals and electronics datasheets. All these files
have been scattered all over the place, never having a common home
where they could all be conveniently navigated and read.

Until now. Enter... [Komga](https://komga.org/).

![Contents of the Art library]({{ media }}/komga-art.png)

Komga is described as *a media server for your comics, mangas,
BDs, magazines and eBooks*. Most importantly, for me, is that it
handles individual books, file names and metadata better than a
few others.

## Installation

To deploy Komga in Kubernetes, the setup is essentially a fork of
[Audiobookshelf]({{ site.baseurl }}/2024/02/28/audiobookshelf-on-kubernetes.html)
deployment.

A single phisical volume to store the application's databases is
needed, while the books are read from `/home/depot/books/`:

```
# mkdir /home/k8s/komga/config
# chown -R audiobookshelf.audiobookshelf /home/k8s/komga/
```

One the directories and files a ready, we run Komga as the same
non-privileged user as Audiobookshelf, based on
[Konga's `docker-compose`](https://komga.org/docs/installation/docker/#docker-compose):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: komga
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: komga-pv-config
  namespace: komga
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/k8s/komga/config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: komga-pv-books
  namespace: komga
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/depot/books
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: komga-pvc-config
  namespace: komga
spec:
  storageClassName: manual
  volumeName: komga-pv-config
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: komga-pvc-books
  namespace: komga
spec:
  storageClassName: manual
  volumeName: komga-pv-books
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: komga
  name: komga
  namespace: komga
spec:
  replicas: 1
  revisionHistoryLimit: 0
  selector:
    matchLabels:
      app: komga
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: komga
    spec:
      containers:
        - image: gotson/komga
          imagePullPolicy: Always
          name: komga
          args: ["user", "1006:1006"]
          env:
          - name: TZ
            value: "Europe/Madrid"
          ports:
          - containerPort: 25600
          resources: {}
          stdin: true
          tty: true
          volumeMounts:
          - mountPath: /config
            name: komga-config
          - mountPath: /data
            name: komga-books
          securityContext:
            allowPrivilegeEscalation: false
            runAsUser: 1006
            runAsGroup: 1006
      restartPolicy: Always
      volumes:
      - name: komga-config
        persistentVolumeClaim:
          claimName: komga-pvc-config
      - name: komga-books
        persistentVolumeClaim:
          claimName: komga-pvc-books
---
kind: Service
apiVersion: v1
metadata:
  name: komga-svc
  namespace: komga
spec:
  type: NodePort
  ports:
  - port: 25600
    nodePort: 30600
    targetPort: 25600
  selector:
    app: komga
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: komga-ingress
  namespace: komga
  annotations:
    acme.cert-manager.io/http01-edit-in-place: "true"
    cert-manager.io/issue-temporary-certificate: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/websocket-services: komga-svc
spec:
  ingressClassName: nginx
  rules:
    - host: komga.ssl.uu.am
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: komga-svc
                port:
                  number: 25600
  tls:
    - secretName: tls-secret
      hosts:
        - komga.ssl.uu.am
```

```
$ kubectl apply -f komga.yaml 
namespace/komga created
persistentvolume/komga-pv-config created
persistentvolume/komga-pv-books created
persistentvolumeclaim/komga-pvc-config created
persistentvolumeclaim/komga-pvc-books created
deployment.apps/komga created
service/komga-svc created
ingress.networking.k8s.io/komga-ingress created

$ kubectl -n komga get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/komga-5c865d6587-tvlt2   1/1     Running   0          28s

NAME                TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
service/komga-svc   NodePort   10.103.226.126   <none>        25600:30600/TCP   8s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/komga   1/1     1            1           29s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/komga-5c865d6587   1         1         1       28s
```

## Configuration

Once the application is running for the first time, register an
account (which will be the administrator) and start creating
libraries based on the files available under `/data`.

When creating libraries in Komga, an important step if there are
individual books, is to put then under one (or more) directories
that will be recognized as
[One-Shots](https://komga.org/docs/guides/oneshots).

My libray is rather small and exclusively made ouf of *one-shots*,
although I found it useful to store a few files as a *kind of*
series:

- *Art* and *LEGO*: eBooks from HumbleBundle deals.
- *Audible*: *visual aids* (PDF files) from Audible.
- *Cosplay*: books from 
  [Kamui Cosplay](https://www.kamuicosplay.com/product-category/tutorialbooks/) and
  [Punished Props](https://www.punishedprops.com/product-category/ref-mats/)
  - Including a couple of sets of templates, which make sense to
    store as *kind of* small series.
- *Terry Pratchett*: a small collection of
  [Discworld Fanfics](https://www.reddit.com/r/discworld/s/UcM6AvRDXK),
  mirrored at home just for convenience.

These are organized under the `/data` volume as follows:

```
/data/Art/books/
/data/Audible/books/
/data/Cosplay/books/
/data/Cosplay/templates/Jade Rabbit/
/data/Cosplay/templates/Ripper Axe/
/data/LEGO/books/
/data/Terry.Pratchett/books/
```

All individual books are stored under a `books` subdirectory under
each library directory, and then this name is set as the
**One-Shots directory** for every library:

![Setup for the Art library]({{ media }}/komga-art-setup.png)

## Alternatives

Komga is not the only Free Software application available for this
purpose, so it is worth mentioning why it was chosen over the
alternatives.

Priot to setting up Komga, I spent some time trying the same with
[kavitareader.com](https://www.kavitareader.com/#downloads-v1-docker).
The Kubernetes deployment was essentially the same, based on
[linuxserver.io/images/docker-kavita](https://docs.linuxserver.io/images/docker-kavita/). The main drawback that kept me from using
this one long-term was that it really is built for *series* and
*really not* for individual books.

Admitedly, that was the only one I tried among other alternatives
[mentioned here](https://www.reddit.com/r/selfhosted/comments/1b4fg8l/book_server/).
Others include:
- **Calibre Web**, available as [janeczku/calibre-web](https://github.com/janeczku/calibre-web)
  or
  [linuxserver/calibre-web](https://docs.linuxserver.io/images/docker-calibre-web/).
  It only supports a single library and requires using the
  Calibre desktop application. While this wouldn't be a problem
  for myself, it would prevent kids from reading the books because
  they only have school laptops where Calibre cannot be installed.
- [Librum](https://github.com/Librum-Reader/Librum?tab=readme-ov-file#self-hosting)
  could be a good option, were it not for a similar requirement
  to install and run a client application. This is not a web app.