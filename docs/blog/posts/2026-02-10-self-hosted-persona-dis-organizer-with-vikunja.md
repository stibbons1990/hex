---
date: 2026-02-10
categories:
 - linux
 - kubernetes
 - self-hosted
 - organizer
title: Self-hosted Personal Dis-Organizer with Vikunja
---

There is always too much to do and too little time, it never gets all done so the real
challenge is to get the right things done at the right time, just to avoid
*Unforseen Consequences*...

In one more attempt at getting better organized than having lots of browser tabs always
open, emails unread in the inbox and, worst of all, uncountable ideas floating in the
brain with nowhere to rest, lets try [Vikunja](https://vikunja.io/), *the fluffy,
open-source, self-hostable to-do app*.

<!-- more -->

## Installation

[Installing](https://vikunja.io/docs/installing/) instructions include a basic example to
run with [docker](https://vikunja.io/docs/installing/#docker) and links to other setups.
[Vikunja Helm Chart](https://github.com/go-vikunja/helm-chart?tab=readme-ov-file#vikunja-helm-chart)
includes are more complete setup, but for simplicity this time round
Vikunja was deployed with this manifest:

??? k8s "`vikunja.yaml`"

    ``` yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: vikunja
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: vikunja-data
      namespace: vikunja
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: longhorn-nvme
      resources:
        requests:
          storage: 5Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: vikunja
      namespace: vikunja
    spec:
      replicas: 1
      revisionHistoryLimit: 0
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app: vikunja
      template:
        metadata:
          labels:
            app: vikunja
        spec:
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            fsGroup: 1000
          containers:
          - name: vikunja
            image: vikunja/vikunja:1.0.0
            ports:
            - containerPort: 3456
            env:
            - name: VIKUNJA_SERVICE_JWTSECRET
              value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
            - name: VIKUNJA_DATABASE_TYPE
              value: "sqlite"
            - name: VIKUNJA_DATABASE_PATH
              value: "/app/vikunja/files/vikunja.db"
            - name: VIKUNJA_SERVICE_PUBLICURL
              value: "https://vikunja.very-very-dark-gray.top"
            volumeMounts:
            - name: data
              mountPath: /app/vikunja/files
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: vikunja-data
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: vikunja
      namespace: vikunja
    spec:
      selector:
        app: vikunja
      ports:
        - protocol: TCP
          port: 80
          targetPort: 3456
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: vikunja-ingress-tailscale
      namespace: vikunja
    spec:
      ingressClassName: tailscale
      defaultBackend:
        service:
          name: vikunja
          port:
            number: 80
      tls:
        - hosts:
            - vikunja
    ```

This deployment includes an `Ingress` using the
[Tailscale kubernetes operator](./2025-03-23-remote-access-options-for-self-hosted-services.md#tailscale-kubernetes-operator)
because the mobile app seems to be unable to have its requests authenticated through
Pomerium, even after singing in using its embeded Chrome browser.
To keep access to the full web UI secured behind
[Pomerium](./2025-12-18-replacing-ingress-nginx-with-pomerium.md),
this ingress is added to
[Pomerium Kustomize setup](./2025-12-18-replacing-ingress-nginx-with-pomerium.md#kustomize-per-service-acls):

??? k8s "`pomerium/pomerium-ingress/vikunja.yaml`"

    ``` yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: vikunja-pomerium-ingress
      namespace: vikunja
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        ingress.pomerium.io/allow_websockets: true
        ingress.pomerium.io/idle_timeout: 0s
        ingress.pomerium.io/pass_identity_headers: true
        ingress.pomerium.io/preserve_host_header: true
        ingress.pomerium.io/timeout: 0s
        ingress.pomerium.io/set_request_headers: |
          X-Forwarded-Proto: "https"
          X-Forwarded-Host: "vikunja.very-very-dark-gray.top"
    spec:
      ingressClassName: pomerium
      rules:
        - host: vikunja.very-very-dark-gray.top
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: vikunja
                    port:
                      number: 80
      tls:
        - secretName: tls-secret
          hosts:
            - vikunja.very-very-dark-gray.top
    ```

Applying this manifest should have the service up and running within the minute:

``` console
$ kubectl apply -f vikunja.yaml 
namespace/vikunja created
persistentvolumeclaim/vikunja-data created
deployment.apps/vikunja created
service/vikunja created

$ kubectl -n vikunja get all
NAME                          READY   STATUS    RESTARTS   AGE
pod/vikunja-75c7776d7-f7njz   1/1     Running   0          33s

NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vikunja   ClusterIP   10.101.73.47   <none>        80/TCP    19m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vikunja   1/1     1            1           19m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/vikunja-75c7776d7   1         1         1       33s
```

Applying the Pomerium Ingress needs a couple of minutes for the DNS challenge to be done:

``` console
$ kubectl apply -k pomerium/pomerium-ingress
...
ingress.networking.k8s.io/vikunja-pomerium-ingress created

$ kubectl get certificate -n vikunja -w
NAME         READY   SECRET       AGE
tls-secret   False   tls-secret   9s
tls-secret   False   tls-secret   97s
```

Once this is ready, the web UI is ready at <https://vikunja.very-very-dark-gray.top>

### Android app

The [Vikunja Android app](https://play.google.com/store/apps/details?id=io.vikunja.app) is
easy to install from the Play Store but not so easy to sing in with, when the web UI is
secured behind SSO. There is not a separate path for the application to talk to the API,
so to enable direct HTTPS access from the app it would be necessary to disable SSO ACLs
on the web UI; not necessarily desirable.

Instead, the app can connect to the server via a
[Tailscale Ingress](./2025-03-23-remote-access-options-for-self-hosted-services.md#tailscale-kubernetes-operator).

Moreover, the latest version of the mobile app is compatible only with version 1.0.0 of
the server, which is why the above manifests deploys that old version instead of the
latest (1.1.0).

On the other hand, the web UI can be accessed with a mobile browser (e.g. Firefox);
time will tell whether the mobile app makes itself worthy all the above workarounds.