---
date: 2025-09-25
categories:
 - kubernetes
 - gaming
 - streaming
 - steam
title: ddns-updater on Kubernetes
---

After years of enjoying a static IPv4 address *for free*, migrating to a new ISP required
either paying a monthly fee for such a priviledge... or simply running a Dynamic DNS
service to keep the relevant domains pointing to the correct IPv4 address as it updated.

<!-- more -->

## DDNS Updater

The service of choice was the
[Lightweight universal DDNS Updater program](https://github.com/qdm12/ddns-updater?tab=readme-ov-file#lightweight-universal-ddns-updater-program), available for Docker at
[qmcgaw/ddns-updater](https://hub.docker.com/r/qmcgaw/ddns-updater/) and fairly simple to
run in [Kubernetes](https://github.com/qdm12/ddns-updater/tree/master/k8s#kubernetes).

Create a `ddns-updater` directory and download the base manifest files:

``` console
$ curl -O https://raw.githubusercontent.com/qdm12/ddns-updater/master/k8s/base/deployment.yaml
$ curl -O https://raw.githubusercontent.com/qdm12/ddns-updater/master/k8s/base/secret-config.yaml
$ curl -O https://raw.githubusercontent.com/qdm12/ddns-updater/master/k8s/base/service.yaml
$ curl -O https://raw.githubusercontent.com/qdm12/ddns-updater/master/k8s/base/kustomization.yaml
```

Then edit `secret-config.yaml` as explained in the DDNS Updater's `README`
[Configuration](https://github.com/qdm12/ddns-updater/blob/master/README.md#configuration), following the example provided for the relevant DNS provider, e.g.
[Porkbun](https://github.com/qdm12/ddns-updater/blob/master/docs/porkbun.md#porkbun):

``` json linenums="1" hl_lines="4-7"
{
  "settings": [
    {
      "provider": "porkbun",
      "domain": "very-very-dark-gray.top",
      "api_key": "pk1_c16d...62c4",
      "secret_api_key": "sk1_64f2...9589",
      "ip_version": "ipv4",
      "ipv6_suffix": ""
    }
  ]
}
```

The configuration can be added to the `CONFIG` variable in `secret-config.yaml`

``` yaml linenums="1" hl_lines="7" title="ddns-updater/secret-config.yaml"
apiVersion: v1
kind: Secret
metadata:
  name: ddns-updater-config
type: Opaque
stringData:
  CONFIG: '{"settings":[{"provider":"porkbun","domain":"*.very-very-dark-gray.top","api_key":"pk1_c16d...62c4","secret_api_key":"sk1_64f2...9589","ip_version":"ipv4","ipv6_suffix": ""}]}'
```

To make the service's web UI externally accessible, securely and resilient
to DNS records being out of date after the IPv4 address changes, add the
following `ingress.yaml` (based on
[`k8s/overlay/with-ingress-tls-cert-manager/ingress.yaml`](https://github.com/qdm12/ddns-updater/blob/master/k8s/overlay/with-ingress-tls-cert-manager/ingress.yaml)) to enable access through a
[Cloudflare Tunnel](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#cloudflare-tunnel)
(like the one previously setup to access Home Assistant):

``` yaml linenums="1" hl_lines="12 24" title="ddns-updater/ingress.yaml"
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ddns-updater-nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: ddns-updater.very-very-dark-gray.top
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: ddns-updater-svc
              port:
                name: "http"
  tls:
    - hosts:
        - ddns-updater.very-very-dark-gray.top
      secretName: tls-secret
```

With all the above, the service can be started by applying the deployment:

``` console
$ kubectl apply -k .
namespace/ddns-updater created
secret/ddns-updater-config created
service/ddns-updater-svc created
deployment.apps/ddns-updater created
ingress.networking.k8s.io/ddns-updater-nginx created
```

Once the services have been running for a few minutes, the web UI will be
accessible at <https://ddns-updater.very-very-dark-gray.top/> for those users
authorized to access it by the Cloudflare Tunnel policies.

!!! note the above does not specify a `namespace` so everything is in the 
    `default` namespace.

``` console
$ kubectl get all 
NAME                                READY   STATUS      RESTARTS   AGE
pod/cm-acme-http-solver-zr4lm       1/1     Running     0          49m
pod/ddns-updater-5cbcfc865d-8jrqj   1/1     Running     0          49m

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/cm-acme-http-solver-drlkv   NodePort    10.110.215.98    <none>        8089:32080/TCP   49m
service/ddns-updater-svc            ClusterIP   10.104.110.181   <none>        80/TCP           49m
service/kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP          152d

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ddns-updater   1/1     1            1           49m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/ddns-updater-5cbcfc865d   1         1         1       49m

$ kubectl get ingress
NAME                        CLASS    HOSTS                                  ADDRESS         PORTS     AGE
cm-acme-http-solver-fqqw4   <none>   ddns-updater.very-very-dark-gray.top   192.168.0.171   80        49m
ddns-updater-nginx          nginx    ddns-updater.very-very-dark-gray.top   192.168.0.171   80, 443   49m
```

## (Why not) DDCLIENT

[DDCLIENT](https://github.com/ddclient/ddclient?tab=readme-ov-file#ddclient) seemd at
first like the best option, but would not have worked in this case due to
[Issue #569:](https://github.com/ddclient/ddclient/issues/569)
*porkbun: Support Wildcard DNS entries*.