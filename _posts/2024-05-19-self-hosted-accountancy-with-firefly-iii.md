---
title:  "Self-hosted accountancy with Firefly III"
date:   2024-05-19 14:05:19 +0200
categories: linux kubernetes docker server firefly-iii
---

Keep track of expenses and stuff is hard, thankless work.

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

Over the years I've done it, with varying degrees of *success*,
using a variety of solutions including my first ever
[LAMP](https://es.wikipedia.org/wiki/LAMP) project, right after
learning PHP and MySQL, and one my bank's own built-in solutions
until they unceremonously took it away with no notice.

After this last disappointment, I decided to go the self-hosted
way taking inspiration from the list of
[Money, Budgeting & Management](https://github.com/awesome-selfhosted/awesome-selfhosted?tab=readme-ov-file#money-budgeting--management)
solutions by
[Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted). Based on comments in several forums, I
decided to first try with
[Firefly III](https://github.com/firefly-iii/firefly-iii).

## Deployment

[Firefly III on Kubernetes.](https://github.com/firefly-iii/kubernetes) includes deployments for each component, of which
I will be using the most basic ones:

*   [`mysql.yaml`](https://github.com/firefly-iii/kubernetes/blob/main/kustomize/mysql.yaml) for the database.
*   [`firefly-iii.yaml`](https://github.com/firefly-iii/kubernetes/blob/main/kustomize/firefly-iii.yaml) for the web application.
*   [`ingress-firefly-iii.yaml`](https://github.com/firefly-iii/kubernetes/blob/main/kustomize/ingress-firefly-iii.yaml) for the ingress.

While these are meant to be used with `kustomize`, I will keep it
simpler by putting it all together in my own `firefly-iii.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: firefly-iii
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: firefly-iii-pv-mysql
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  storageClassName: manual
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/k8s/firefly-iii/mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: firefly-iii-pvc-mysql
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  storageClassName: manual
  volumeName: firefly-iii-pv-mysql
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: firefly-iii-mysql
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  selector:
    matchLabels:
      app: firefly-iii
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: firefly-iii
        tier: mysql
    spec:
      containers:
      - image: yobasystems/alpine-mariadb:latest
        imagePullPolicy: Always
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "**************************"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: firefly-iii-pvc-mysql
---
apiVersion: v1
kind: Service
metadata:
  name: firefly-iii-mysql-svc
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  type: NodePort
  ports:
  - port: 3306
    nodePort: 30306
    targetPort: 3306
  selector:
    app: firefly-iii
    tier: mysql
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: firefly-iii-pv-upload
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /home/k8s/firefly-iii/upload
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: firefly-iii-pvc-upload
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  storageClassName: manual
  volumeName: firefly-iii-pv-upload
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: firefly-iii-svc
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30080
    targetPort: 8080
  selector:
    app: firefly-iii
    tier: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: firefly-iii
  namespace: firefly-iii
  labels:
    app: firefly-iii
spec:
  selector:
    matchLabels:
      app: firefly-iii
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: firefly-iii
        tier: frontend
    spec:
      containers:
      - image: fireflyiii/core
        imagePullPolicy: Always
        name: firefly-iii
        env:
        - name: APP_ENV
          value: "local"
        - name: APP_KEY
          value: "********************************"
        - name: DB_HOST
          value: firefly-iii-mysql-svc
        - name: DB_CONNECTION
          value: mysql
        - name: DB_DATABASE
          value: "fireflyiii"
        - name: DB_USERNAME
          value: "root"
        - name: DB_PASSWORD
          value: "**************************"
        - name: TRUSTED_PROXIES
          value: "**"
        ports:
        - containerPort: 8080
          name: firefly-iii
        volumeMounts:
        - mountPath: "/var/www/html/storage/upload"
          name: firefly-iii-upload 
      volumes:
        - name: firefly-iii-upload
          persistentVolumeClaim:
            claimName: firefly-iii-pvc-upload
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: firefly-iii-ingress
  namespace: firefly-iii
  annotations:
    acme.cert-manager.io/http01-edit-in-place: "true"
    cert-manager.io/issue-temporary-certificate: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/websocket-services: firefly-iii-svc
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
spec:
  ingressClassName: nginx
  rules:
  - host: ffi.ssl.uu.am
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: firefly-iii-svc
            port:
              number: 8080
  tls:
    - secretName: tls-secret
      hosts:
        - ffi.ssl.uu.am
```

**Note**: the `APP_KEY` value **must** have exactly 32 characters,
as noted in 
[#2193: Can't get started - hitting an "encryption key not specified error"](https://github.com/firefly-iii/firefly-iii/issues/2193#issuecomment-529696297),
also better explained in
[monicahq/monica #6449](https://github.com/monicahq/monica/issues/6449#issuecomment-1334845725).

Once the service has started up, initialized the database and
everything else, one can finally visit 
[https://ffi.ssl.uu.am/](https://ffi.ssl.uu.am/)
to create an account and get started:

![Signup form]({{ media }}/firefly-iii-signup.png)

**Note**: the password **must** have at least 16 characters, which
is more than Chrome will use when *suggesting a strong password*.

From this point on, [RTFM](https://docs.firefly-iii.org/)
will be probably the best way to go.
