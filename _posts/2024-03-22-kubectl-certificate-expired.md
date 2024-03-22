---
title:  "Kubernetes Certificate Expired"
date:   2024-03-22 23:03:05 +0200
categories: ubuntu server linux kubernetes docker
---

## Yet Another Unexpected Hump

The regular user is unable to connect to the Kubernetes API
server because the x509 certificate expired on
2024-03-21T21:37:37Z (nearly 24 hours ago):

```
$ kubectl get all -n minecraft-server
E0322 20:57:59.141510 3545623 memcache.go:265] couldn't get current server API group list: Get "https://10.0.0.6:6443/api?timeout=32s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
E0322 20:57:59.143467 3545623 memcache.go:265] couldn't get current server API group list: Get "https://10.0.0.6:6443/api?timeout=32s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
E0322 20:57:59.145339 3545623 memcache.go:265] couldn't get current server API group list: Get "https://10.0.0.6:6443/api?timeout=32s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
E0322 20:57:59.147141 3545623 memcache.go:265] couldn't get current server API group list: Get "https://10.0.0.6:6443/api?timeout=32s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
E0322 20:57:59.148895 3545623 memcache.go:265] couldn't get current server API group list: Get "https://10.0.0.6:6443/api?timeout=32s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
Unable to connect to the server: tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-22T20:57:59+01:00 is after 2024-03-21T21:37:37Z
```

OTOH the `root` user is trying to connect to the wrong port
(**8443** instead of **6443**) and hitting instead the
UniFi server.

```
# kubectl get all -n minecraft-server
E0322 20:59:13.297860 3571458 memcache.go:265] couldn't get current server API group list: Get "https://localhost:8443/api": tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
E0322 20:59:13.300723 3571458 memcache.go:265] couldn't get current server API group list: Get "https://localhost:8443/api": tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
E0322 20:59:13.304184 3571458 memcache.go:265] couldn't get current server API group list: Get "https://localhost:8443/api": tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
E0322 20:59:13.306950 3571458 memcache.go:265] couldn't get current server API group list: Get "https://localhost:8443/api": tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
E0322 20:59:13.309929 3571458 memcache.go:265] couldn't get current server API group list: Get "https://localhost:8443/api": tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
Unable to connect to the server: tls: failed to verify certificate: x509: certificate is valid for UniFi, not localhost
```

The Kubelet config has not changed in about a year:

```
# ls -l /etc/kubernetes/kubelet.conf 
-rw------- 1 root root 1960 Mar 22  2023 /etc/kubernetes/kubelet.conf

# cat /etc/kubernetes/kubelet.conf 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1ETXlNakl4TXpjek5sb1hEVE16TURNeE9USXhNemN6Tmxvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS29LCkNMNThMQ1l5Ym5lck95UUFlNUtqamZJS3BZSmhEU2x1aUNZSUVBdjgzcjZPWlZoajF4bHhNcSs4N05YTEp1KzIKSHhSSDdNcXd6T3FTMk9SVGRWcmJOTzNOZHdSRUJ6WTFNRkNxMnhzUFdBeFVBemZHMERSMUdBVUhpYStSZjFkLwp3Tm9BeG9qWWFrYzk5N0psbVJvS1NYWkdSSC9yQnNyV0w3bnpZaTkwVGNjSnpZSytOckM2dE53OEcrSm1vRzM2ClFwamhEVHJSQW9JaG5PSkc2dzB0ZmkxZG02Q29yZ2h1M2owNFJEM1dvbnpnTTZqejdRZ25TSGJKNTRkeVJjWkwKZ1NSdlZiTkF2Y0RVRFNjcGJDTmdVUHgySFBJdUV4UFRSNnpiMmlSaXRadTJiQm1WRDhoYzExczN4NGtDWURGawpvQURRSXI2cGpTUmFRLzBKRWJNQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZFK2JxU3NDRlZkT3hmempzeU5sb0dBaHJjaEtNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBR2JaVWRXTVVpbTRBeUFoUmI2bwpNMWVOdmdYVUZVcER5WHFrdCt0MWN5bGtiQlRPMDY4NzZ2MGFpU2NKaXhNS2x1bHBwYzF4WStXbEpnQVV1VW9xCkZtQ2pvcUVaWWdyY2FhRjBlMTBNU3h0cjRaMWRTUFdhL2szdUN3cEEra2VYQ3VhaTlqODh3OW54azZjbnVlTC8KdVBTNzQ2MW8xL29HUEpIbmRMOTc4emJLUWU2NEEvb2xvclFEdzZvVVZaU3RvNWRzNGVha3QzVGc0Y1N1TklSbgoyNE9HMEhNMXBKdjMyRjFEdnpJazRPYzZEU0JmeDdZZGN2K1JZdC9ob0lLVGl0WnFFV3ZjSlJSUlMxNmc4VHRwCkRzMGdqK0lRY29yd0FLM1E4UU1DNDg4T3RwVmh5Y2JaSXJSanZPZnBxeTNsQXIwQng4Ykl6ZkJQTTNycUdnTHMKeFhZPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.0.0.6:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:lexicon
  name: system:node:lexicon@kubernetes
current-context: system:node:lexicon@kubernetes
kind: Config
preferences: {}
users:
- name: system:node:lexicon
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem

# ls -l /var/lib/kubelet/pki/kubelet-client-current.pem /var/lib/kubelet/pki/kubelet-client-current.pem
lrwxrwxrwx 1 root root 59 Jan 15 16:44 /var/lib/kubelet/pki/kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-01-15-16-44-55.pem
lrwxrwxrwx 1 root root 59 Jan 15 16:44 /var/lib/kubelet/pki/kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-01-15-16-44-55.pem

# ls -l /var/lib/kubelet/pki/
total 20
-rw------- 1 root root 2826 Mar 22  2023 kubelet-client-2023-03-22-22-37-39.pem
-rw------- 1 root root 1110 Jan 15 16:44 kubelet-client-2024-01-15-16-44-55.pem
lrwxrwxrwx 1 root root   59 Jan 15 16:44 kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-01-15-16-44-55.pem
-rw-r--r-- 1 root root 2246 Mar 22  2023 kubelet.crt
-rw------- 1 root root 1675 Mar 22  2023 kubelet.key
```

A search for
[*kubectl "couldn't get current server API group list" "tls: failed to verify certificate: x509: certificate has expired or is not yet valid"*]/(https://www.google.com/search?q=kubectl+%22couldn%27t+get+current+server+API+group+list%22+%22tls%3A+failed+to+verify+certificate%3A+x509%3A+certificate+has+expired+or+is+not+yet+valid%22&oq=kubectl+%22couldn%27t+get+current+server+API+group+list%22+%22tls%3A+failed+to+verify+certificate%3A+x509%3A+certificate+has+expired+or+is+not+yet+valid%22)
returns a single result, which doesn't help.

[Unable to connect to the server GKE/GCP x509: certificate has expired or is not yet valid](https://stackoverflow.com/questions/77528050/unable-to-connect-to-the-server-gke-gcp-x509-certificate-has-expired-or-is-not)
*claims* that running `hwclock -s` helped, but it didn't

```
# date -R
Fri, 22 Mar 2024 21:12:39 +0100
# hwclock -s
# date -R
Fri, 22 Mar 2024 21:13:19 +0100
# timedatectl
               Local time: Fri 2024-03-22 21:19:44 CET
           Universal time: Fri 2024-03-22 20:19:44 UTC
                 RTC time: Fri 2024-03-22 20:19:44
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: no
              NTP service: n/a
          RTC in local TZ: no
```

A search for *just*
[*kubectl "tls: failed to verify certificate: x509: certificate has expired or is not yet valid"*]/(https://www.google.com/search?q=kubectl+%22tls%3A+failed+to+verify+certificate%3A+x509%3A+certificate+has+expired+or+is+not+yet+valid%22&oq=kubectl+%22tls%3A+failed+to+verify+certificate%3A+x509%3A+certificate+has+expired+or+is+not+yet+valid%22)
returns more results, including a few that finally offer
hints in the right direction.

**TL;DR:** *By default, the certificate expires after 365 days.* That tracks, because the 
[Kubernetex cluster in lexicon]({{ site.baseurl }}/2023/03/25/single-node-kubernetes-cluster-on-ubuntu-server-lexicon.html)
was created *exactly a little* over one year ago,
on Mar 22, **2023**:

```
# ls -l /etc/kubernetes/kubelet.conf 
-rw------- 1 root root 1960 Mar 22  2023 /etc/kubernetes/kubelet.conf
```

Most other search results seem to be specific to platforms
other than the current set (`kubelet`) but not, knowing that
this is the most likely root cause, a more direct search for
[*renew kubelet certificate*](https://www.google.com/search?q=renew+kubelet+certificate)
produces more relevant results, including
[Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
and a more digestiable answer in Server Fault:
[/var/lib/kubelet/pki/kubelet.crt is expired, How to renew it?](https://serverfault.com/questions/1068751/var-lib-kubelet-pki-kubelet-crt-is-expired-how-to-renew-it).

Indeed that is the non-rotateable certificate that expired:

```
# ls -l /var/lib/kubelet/pki/kubelet.crt 
-rw-r--r-- 1 root root 2246 Mar 22  2023 /var/lib/kubelet/pki/kubelet.crt

# openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text -noout  | grep -A 2 Validity
        Validity
            Not Before: Mar 22 20:37:39 2023 GMT
            Not After : Mar 21 20:37:39 2024 GMT

# ls -l /var/lib/kubelet/pki/kubelet-server-current.pem
ls: cannot access '/var/lib/kubelet/pki/kubelet-server-current.pem': No such file or directory
```

## Manual Certificate Renewal

Kubernetes documentation for
[Manual certificate renewal](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#manual-certificate-renewal)
seem to suggest one can *simply* run
`kubeadm certs renew` to renew specific certificate or,
with the subcommand `all`, renew all of them:

```
# kubeadm certs renew all
```

Clusters built with `kubeadm` often copy the `admin.conf`
certificate into `$HOME/.kube/config`. On such a system,
to update the contents of `$HOME/.kube/config` after
renewing the `admin.conf`, run the following commands:

```
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Before running `kubeadm certs renew all`:

```
# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[check-expiration] Error reading configuration from the Cluster. Falling back to default configuration

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Mar 21, 2024 21:37 UTC   <invalid>       ca                      no      
apiserver                  Mar 21, 2024 21:37 UTC   <invalid>       ca                      no      
apiserver-etcd-client      Mar 21, 2024 21:37 UTC   <invalid>       etcd-ca                 no      
apiserver-kubelet-client   Mar 21, 2024 21:37 UTC   <invalid>       ca                      no      
controller-manager.conf    Mar 21, 2024 21:37 UTC   <invalid>       ca                      no      
etcd-healthcheck-client    Mar 21, 2024 21:37 UTC   <invalid>       etcd-ca                 no      
etcd-peer                  Mar 21, 2024 21:37 UTC   <invalid>       etcd-ca                 no      
etcd-server                Mar 21, 2024 21:37 UTC   <invalid>       etcd-ca                 no      
front-proxy-client         Mar 21, 2024 21:37 UTC   <invalid>       front-proxy-ca          no      
scheduler.conf             Mar 21, 2024 21:37 UTC   <invalid>       ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 19, 2033 21:37 UTC   8y              no      
etcd-ca                 Mar 19, 2033 21:37 UTC   8y              no      
front-proxy-ca          Mar 19, 2033 21:37 UTC   8y              no      
```

After running `kubeadm certs renew all`:

```
# kubeadm certs check-expiration
...
```

## Automatic Certificate Renewal

[Automatic certificate renewal](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#automatic-certificate-renewal)
suggests that the certificates *should* have been renewed
automatically already, but the system show symptoms of
[Kubelet client certificate rotation fails](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#automatic-certificate-renewal).

The most likely reason for this is that there has been no
[control plane upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
for just a bit over a year, and that's precisely the issue
because the automatic certificate renewal
*is designed for addressing the simplest use cases;
if you don't have specific requirements on certificate
renewal and perform Kubernetes version upgrades regularly
(**less than 1 year in between each upgrade**).*

The troubleshoot steps to deal with the case of
[Kubelet client certificate rotation fails](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#automatic-certificate-renewal)
seem to require a multi-node cluster
(*a working control plane node in the cluster*)
which is not available here.

Note that the **client** certificates are correctly
renewed (apparently every 2 months), but the **server**
certificate is still the static one:

```
# ls -l /var/lib/kubelet/pki/kubelet.crt
-rw-r--r-- 1 root root 2246 Mar 22  2023 /var/lib/kubelet/pki/kubelet.crt

# ls -l /var/lib/kubelet/pki/kubelet-server-current.pem
ls: cannot access '/var/lib/kubelet/pki/kubelet-server-current.pem': No such file or directory

# ls -l /var/lib/kubelet/pki/kubelet-client-current.pem
lrwxrwxrwx 1 root root 59 Jan 15 16:44 /var/lib/kubelet/pki/kubelet-client-current.pem -> /var/lib/kubelet/pki/kubelet-client-2024-01-15-16-44-55.pem
```

## Manual Recovery And Certificate Renewal

It appears the only solution available may be the one in
[/var/lib/kubelet/pki/kubelet.crt is expired, How to renew it?](https://serverfault.com/questions/1068751/var-lib-kubelet-pki-kubelet-crt-is-expired-how-to-renew-it).

Edit
`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`
and add the following line to set `KUBELET_EXTRA_ARGS`:

```
Environment="KUBELET_EXTRA_ARGS=--rotate-certificates=true --rotate-server-certificates=true --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
```

Full content of
`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`

```
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
Environment="KUBELET_EXTRA_ARGS=--rotate-certificates=true --rotate-server-certificates=true --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

Reload and restart kubelet with:

```
sudo systemctl daemon-reload
sudo service kubelet restart
```

References:
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/
- https://devopstales.github.io/kubernetes/k8s-cert/
- https://kubernetes.io/docs/tasks/tls/certificate-rotation/
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs

