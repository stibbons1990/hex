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

This happens to `root` because it is missing `.kube/config`:

```
# kubectl config view
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null

# ls .kube
cache
# cp /etc/kubernetes/admin.conf .kube/config

# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.0.0.6:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
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

YouTube video
[unable to connect to the Server and having x509 certificate has expired issue](https://youtu.be/SJIhDu4CckI?si=rY5rUUdRbggD91WJ&t=692)
by [Cyber Legum](https://www.youtube.com/@cyberlegum)
(@cyberlegum ‧ 776 suscriptores ‧ 37 vídeos)
show that that `kubeadm certs renew all` does indeed renew
all the certificates, and that it then requires restarting
the 4 components (), which the video doesn't quite show. 

[Manual certificate renewal](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#manual-certificate-renewal)
also does not say exactly *how* one should *restart the control plane Pods*,
there seem to be 2 ways:

```
# kubectl -n kube-system delete pod -l 'component=kube-apiserver'
# kubectl -n kube-system delete pod -l 'component=kube-controller-manager'
# kubectl -n kube-system delete pod -l 'component=kube-scheduler'
# kubectl -n kube-system delete pod -l 'component=etcd'
```

which would require copying again:

`.kube/config`:

```
# cp /etc/kubernetes/admin.conf .kube/config
```

Or possibly just reload and restart `kubelet` with:

```
# systemctl daemon-reload
# service kubelet restart
```

After running `kubeadm certs renew all`:

```
# kubeadm certs renew all
[renew] Reading configuration from the cluster...
[renew] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[renew] Error reading configuration from the Cluster. Falling back to default configuration

certificate embedded in the kubeconfig file for the admin to use and for kubeadm itself renewed
certificate for serving the Kubernetes API renewed
certificate the apiserver uses to access etcd renewed
certificate for the API server to connect to kubelet renewed
certificate embedded in the kubeconfig file for the controller manager to use renewed
certificate for liveness probes to healthcheck etcd renewed
certificate for etcd nodes to communicate with each other renewed
certificate for serving etcd renewed
certificate for the front proxy client renewed
certificate embedded in the kubeconfig file for the scheduler manager to use renewed

Done renewing certificates. You must restart the kube-apiserver, kube-controller-manager, kube-scheduler and etcd, so that they can use the new certificates.

# kubeadm certs check-expiration
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Mar 23, 2025 09:41 UTC   364d            ca                      no      
apiserver                  Mar 23, 2025 09:41 UTC   364d            ca                      no      
apiserver-etcd-client      Mar 23, 2025 09:41 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Mar 23, 2025 09:41 UTC   364d            ca                      no      
controller-manager.conf    Mar 23, 2025 09:41 UTC   364d            ca                      no      
etcd-healthcheck-client    Mar 23, 2025 09:41 UTC   364d            etcd-ca                 no      
etcd-peer                  Mar 23, 2025 09:41 UTC   364d            etcd-ca                 no      
etcd-server                Mar 23, 2025 09:41 UTC   364d            etcd-ca                 no      
front-proxy-client         Mar 23, 2025 09:41 UTC   364d            front-proxy-ca          no      
scheduler.conf             Mar 23, 2025 09:41 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 19, 2033 21:37 UTC   8y              no      
etcd-ca                 Mar 19, 2033 21:37 UTC   8y              no      
front-proxy-ca          Mar 19, 2033 21:37 UTC   8y              no  

# openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -text -noout  | grep -A 2 Validity
        Validity
            Not Before: Mar 22 20:37:39 2023 GMT
            Not After : Mar 21 20:37:39 2024 GMT

# ls -l /var/lib/kubelet/pki/kubelet-server-current.pem
ls: cannot access '/var/lib/kubelet/pki/kubelet-server-current.pem': No such file or directory

# cp /etc/kubernetes/admin.conf .kube/config
# kubectl cluster-info
Kubernetes control plane is running at https://10.0.0.6:6443
CoreDNS is running at https://10.0.0.6:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

# kubectl get nodes
NAME      STATUS   ROLES           AGE    VERSION
lexicon   Ready    control-plane   366d   v1.26.15

# kubectl describe node lexicon
Name:               lexicon
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    feature.node.kubernetes.io/cpu-cpuid.ADX=true
                    feature.node.kubernetes.io/cpu-cpuid.AESNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BITALG=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BW=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512CD=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512DQ=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512IFMA=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VL=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VNNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VP2INTERSECT=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VPOPCNTDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.CETIBT=true
                    feature.node.kubernetes.io/cpu-cpuid.CETSS=true
                    feature.node.kubernetes.io/cpu-cpuid.CMPXCHG8=true
                    feature.node.kubernetes.io/cpu-cpuid.FLUSH_L1D=true
                    feature.node.kubernetes.io/cpu-cpuid.FMA3=true
                    feature.node.kubernetes.io/cpu-cpuid.FSRM=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSR=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSROPT=true
                    feature.node.kubernetes.io/cpu-cpuid.GFNI=true
                    feature.node.kubernetes.io/cpu-cpuid.IA32_ARCH_CAP=true
                    feature.node.kubernetes.io/cpu-cpuid.IA32_CORE_CAP=true
                    feature.node.kubernetes.io/cpu-cpuid.IBPB=true
                    feature.node.kubernetes.io/cpu-cpuid.LAHF=true
                    feature.node.kubernetes.io/cpu-cpuid.MD_CLEAR=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVBE=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVDIR64B=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVDIRI=true
                    feature.node.kubernetes.io/cpu-cpuid.OSXSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.SHA=true
                    feature.node.kubernetes.io/cpu-cpuid.SPEC_CTRL_SSBD=true
                    feature.node.kubernetes.io/cpu-cpuid.SRBDS_CTRL=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSCALL=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSEE=true
                    feature.node.kubernetes.io/cpu-cpuid.VAES=true
                    feature.node.kubernetes.io/cpu-cpuid.VMX=true
                    feature.node.kubernetes.io/cpu-cpuid.VPCLMULQDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.X87=true
                    feature.node.kubernetes.io/cpu-cpuid.XGETBV1=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEC=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEOPT=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVES=true
                    feature.node.kubernetes.io/cpu-cstate.enabled=true
                    feature.node.kubernetes.io/cpu-hardware_multithreading=true
                    feature.node.kubernetes.io/cpu-model.family=6
                    feature.node.kubernetes.io/cpu-model.id=140
                    feature.node.kubernetes.io/cpu-model.vendor_id=Intel
                    feature.node.kubernetes.io/cpu-pstate.scaling_governor=powersave
                    feature.node.kubernetes.io/cpu-pstate.status=active
                    feature.node.kubernetes.io/cpu-pstate.turbo=true
                    feature.node.kubernetes.io/cpu-rdt.RDTL2CA=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ_IDLE=true
                    feature.node.kubernetes.io/kernel-version.full=5.15.0-101-generic
                    feature.node.kubernetes.io/kernel-version.major=5
                    feature.node.kubernetes.io/kernel-version.minor=15
                    feature.node.kubernetes.io/kernel-version.revision=0
                    feature.node.kubernetes.io/pci-0300_8086.present=true
                    feature.node.kubernetes.io/pci-0300_8086.sriov.capable=true
                    feature.node.kubernetes.io/storage-nonrotationaldisk=true
                    feature.node.kubernetes.io/system-os_release.ID=ubuntu
                    feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.major=22
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=04
                    feature.node.kubernetes.io/usb-ef_046d_0892.present=true
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=lexicon
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"26:f0:f4:2a:36:8a"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.0.0.6
                    kubeadm.alpha.kubernetes.io/cri-socket: unix:/run/containerd/containerd.sock
                    nfd.node.kubernetes.io/extended-resources: 
                    nfd.node.kubernetes.io/feature-labels:
                      cpu-cpuid.ADX,cpu-cpuid.AESNI,cpu-cpuid.AVX,cpu-cpuid.AVX2,cpu-cpuid.AVX512BITALG,cpu-cpuid.AVX512BW,cpu-cpuid.AVX512CD,cpu-cpuid.AVX512DQ...
                    nfd.node.kubernetes.io/master.version: v0.12.1
                    nfd.node.kubernetes.io/worker.version: v0.12.1
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 22 Mar 2023 22:37:43 +0100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  lexicon
  AcquireTime:     <unset>
  RenewTime:       Sat, 23 Mar 2024 10:45:03 +0100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Mon, 18 Mar 2024 21:03:00 +0100   Mon, 18 Mar 2024 21:03:00 +0100   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Sat, 23 Mar 2024 10:41:21 +0100   Mon, 18 Mar 2024 21:02:41 +0100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Sat, 23 Mar 2024 10:41:21 +0100   Mon, 18 Mar 2024 21:02:41 +0100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Sat, 23 Mar 2024 10:41:21 +0100   Mon, 18 Mar 2024 21:02:41 +0100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Sat, 23 Mar 2024 10:41:21 +0100   Mon, 18 Mar 2024 21:02:41 +0100   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  10.0.0.6
  Hostname:    lexicon
Capacity:
  cpu:                4
  ephemeral-storage:  47684Mi
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32479888Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  45000268112
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             32377488Ki
  pods:               110
System Info:
  Machine ID:                 2056272131924fa6bd1ad9d2922ee4cc
  System UUID:                3214261f-3ee2-1169-841c-88aedd021b0e
  Boot ID:                    6db4e730-968b-4864-a85c-a1de937de9a3
  Kernel Version:             5.15.0-101-generic
  OS Image:                   Ubuntu 22.04.4 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.28
  Kubelet Version:            v1.26.15
  Kube-Proxy Version:         v1.26.15
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (26 in total)
  Namespace                   Name                                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                      ------------  ----------  ---------------  -------------  ---
  atuin-server                atuin-585c6b96f8-htnvj                                    500m (12%)    500m (12%)  2Gi (6%)         2Gi (6%)       201d
  audiobookshelf              audiobookshelf-867f885b49-rpbpr                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         38h
  cert-manager                cert-manager-64f9f45d6f-qx4hs                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         341d
  cert-manager                cert-manager-cainjector-56bbdd5c47-ltjgx                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         341d
  cert-manager                cert-manager-webhook-d4f4545d7-tf4l6                      0 (0%)        0 (0%)      0 (0%)           0 (0%)         341d
  code-server                 code-server-5768c9d997-cr858                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         192d
  default                     inteldeviceplugins-controller-manager-7994555cdb-gjmfm    100m (2%)     100m (2%)   100Mi (0%)       120Mi (0%)     190d
  ingress-nginx               ingress-nginx-controller-6b58ffdc97-rt5lm                 100m (2%)     0 (0%)      90Mi (0%)        0 (0%)         355d
  kube-flannel                kube-flannel-ds-nrrg6                                     100m (2%)     0 (0%)      50Mi (0%)        0 (0%)         356d
  kube-system                 coredns-787d4945fb-67z8g                                  100m (2%)     0 (0%)      70Mi (0%)        170Mi (0%)     366d
  kube-system                 coredns-787d4945fb-gsx6h                                  100m (2%)     0 (0%)      70Mi (0%)        170Mi (0%)     366d
  kube-system                 etcd-lexicon                                              100m (2%)     0 (0%)      100Mi (0%)       0 (0%)         17d
  kube-system                 kube-apiserver-lexicon                                    250m (6%)     0 (0%)      0 (0%)           0 (0%)         17d
  kube-system                 kube-controller-manager-lexicon                           200m (5%)     0 (0%)      0 (0%)           0 (0%)         17d
  kube-system                 kube-proxy-qlt8c                                          0 (0%)        0 (0%)      0 (0%)           0 (0%)         366d
  kube-system                 kube-scheduler-lexicon                                    100m (2%)     0 (0%)      0 (0%)           0 (0%)         17d
  kubernetes-dashboard        dashboard-metrics-scraper-7bc864c59-p27cg                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         355d
  kubernetes-dashboard        kubernetes-dashboard-6c7ccbcf87-vwxnt                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         355d
  local-path-storage          local-path-provisioner-8bc8875b-lbjh8                     0 (0%)        0 (0%)      0 (0%)           0 (0%)         342d
  metallb-system              controller-68bf958bf9-79mpk                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         355d
  metallb-system              speaker-t8f7k                                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         355d
  metrics-server              metrics-server-74c749979-wd278                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         355d
  node-feature-discovery      nfd-node-feature-discovery-master-5f56c75d-xjkb8          0 (0%)        0 (0%)      0 (0%)           0 (0%)         190d
  node-feature-discovery      nfd-node-feature-discovery-worker-xmzwk                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         190d
  plexserver                  plexserver-85f7bf866-7c8gv                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         174d
  telegraf                    telegraf-7f7c9db469-fmkd7                                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         313d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                1650m (41%)  600m (15%)
  memory             2528Mi (7%)  2508Mi (7%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
Events:              <none>
```

This looks like at least `kubectl` finally works again, so now may be
the time to 1. upgrade and 2. enable automatic certificate renewal.

Now we try to *restart the control plane Pods*:

```
# kubectl -n kube-system delete pod -l 'component=kube-apiserver'
# kubectl -n kube-system delete pod -l 'component=kube-controller-manager'
# kubectl -n kube-system delete pod -l 'component=kube-scheduler'
# kubectl -n kube-system delete pod -l 'component=etcd'
```

It's not entirely clear what may have changed, but now the
Kubernetes console and everything else seems to work, except that
one job shows as pending:

```
# kubectl get jobs -n kube-system
NAME                   COMPLETIONS   DURATION   AGE
upgrade-health-check   0/1                      9m43s
```

After a node reboot, the minecraft server was working again
(and all clients were able to connect to it) but the
`metrics-server` is now failed:

```
$ kubectl get all -n metrics-server
NAME                                 READY   STATUS    RESTARTS        AGE
pod/metrics-server-74c749979-wd278   0/1     Running   31 (118m ago)   355d

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/metrics-server   ClusterIP   10.101.79.149   <none>        443/TCP   355d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/metrics-server   0/1     1            0           355d

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/metrics-server-74c749979   1         1         0       355d

$ kubectl logs metrics-server-74c749979-wd278 -n metrics-server
Error from server: Get "https://10.0.0.6:10250/containerLogs/metrics-server/metrics-server-74c749979-wd278/metrics-server": remote error: tls: internal error
```

Kubernetes dashboard shows event
`metrics-server-74c749979-wd278.17bf5cdf3efdd640`
with message
`Readiness probe failed: HTTP probe failed with statuscode: 500`

The above error to see the logs is happening everywhere,
also to the minecraft server, both when trying to show logs
and when trying to communicate with the server to send messages:

```
$ kubectl -n minecraft-server get pods
NAME                               READY   STATUS    RESTARTS   AGE
minecraft-server-7f847b6b7-tv6tw   1/1     Running   0          124m

$ kubectl -n minecraft-server logs minecraft-server-7f847b6b7-tv6tw
Error from server: Get "https://10.0.0.6:10250/containerLogs/minecraft-server/minecraft-server-7f847b6b7-tv6tw/minecraft-server": remote error: tls: internal error

$ kubectl -n minecraft-server exec deploy/minecraft-server -- mc-send-to-console "Hello"
Error from server: error dialing backend: remote error: tls: internal error
```

Journal logs from kubelet show a *TLS handshake error* several times per minute:

```
# journalctl -xeu kubelet | grep -Ev 'Nameserver limits exceeded' | head -3
Mar 23 11:46:36 lexicon kubelet[6055]: I0323 11:46:36.036164    6055 log.go:245] http: TLS handshake error from 10.244.0.44:45274: no serving certificate available for the kubelet
Mar 23 11:46:51 lexicon kubelet[6055]: I0323 11:46:51.042664    6055 log.go:245] http: TLS handshake error from 10.244.0.44:40012: no serving certificate available for the kubelet
Mar 23 11:47:06 lexicon kubelet[6055]: I0323 11:47:06.042088    6055 log.go:245] http: TLS handshake error from 10.244.0.44:50504: no serving certificate available for the kubelet

# journalctl -xeu kubelet | grep -Ev 'Nameserver limits exceeded' | grep -c 'http: TLS handshake error'
366
```

Filtering those out, there are signs that rotating the certs is stuck:

```
# journalctl -xeu kubelet | grep -Ev 'Nameserver limits exceeded' | grep -v 'http: TLS handshake error'
Mar 23 11:51:57 lexicon kubelet[6055]: E0323 11:51:57.493069    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
Mar 23 12:07:05 lexicon kubelet[6055]: E0323 12:07:05.531775    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
Mar 23 12:22:21 lexicon kubelet[6055]: E0323 12:22:21.802759    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
Mar 23 12:22:21 lexicon kubelet[6055]: E0323 12:22:21.802782    6055 certificate_manager.go:354] kubernetes.io/kubelet-serving: Reached backoff limit, still unable to rotate certs: timed out waiting for the condition
Mar 23 12:37:53 lexicon kubelet[6055]: E0323 12:37:53.816829    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
Mar 23 12:53:21 lexicon kubelet[6055]: E0323 12:53:21.816677    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
Mar 23 13:08:49 lexicon kubelet[6055]: E0323 13:08:49.816195    6055 certificate_manager.go:488] kubernetes.io/kubelet-serving: certificate request was not signed: timed out waiting for the condition
```

This is because, after restart, one needs to approve **the most recent** `csr` from kubernetes:

```
# kubectl get csr
NAME        AGE     SIGNERNAME                      REQUESTOR             REQUESTEDDURATION   CONDITION
csr-454fv   7m21s   kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-4mtnx   69m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-fljjd   22m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-n5bvr   114m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-plnf8   99m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-rg4cs   38m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-wzg9g   53m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-xlmlk   129m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-zj52z   84m     kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending

# kubectl certificate approve csr-454fv
certificatesigningrequest.certificates.k8s.io/csr-454fv approved

# kubectl get csr
NAME        AGE    SIGNERNAME                      REQUESTOR             REQUESTEDDURATION   CONDITION
csr-454fv   10m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Approved,Issued
csr-4mtnx   72m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-fljjd   25m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-n5bvr   117m   kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-plnf8   102m   kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-rg4cs   41m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-wzg9g   56m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-xlmlk   132m   kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
csr-zj52z   87m    kubernetes.io/kubelet-serving   system:node:lexicon   <none>              Pending
```

After approving the most recent CSR (and it's **Issued**),
`kubectl` operations are back to work and the `metrics-server`
deployment is functional again:

```
$ kubectl get all -n metrics-server
                      READY   STATUS    RESTARTS        AGE
pod/metrics-server-74c749979-wd278   1/1     Running   31 (132m ago)   355d

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/metrics-server   ClusterIP   10.101.79.149   <none>        443/TCP   355d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/metrics-server   1/1     1            1           355d

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/metrics-server-74c749979   1         1         1       355d

$ kubectl -n minecraft-server logs minecraft-server-7f847b6b7-tv6tw | tail -4
[11:23:46] [User Authenticator #2/INFO]: UUID of player M________t is bbe841f4-06de-4e86-86cd-de0061cf8db1
[11:23:47] [Server thread/INFO]: M________t joined the game
[11:23:47] [Server thread/INFO]: M________t[/10.244.0.1:12945] logged in with entity id 327 at ([world]-179.5, 71.0, -312.5)
[11:23:55] [Server thread/INFO]: M________t lost connection: Disconnected
[11:23:55] [Server thread/INFO]: M________t left the game
```

## Maybe Update

```
# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.26.3
[upgrade/versions] kubeadm version: v1.26.15
I0323 10:44:05.356608 3686397 version.go:256] remote version is much newer: v1.29.3; falling back to: stable-1.26
[upgrade/versions] Target version: v1.26.15
[upgrade/versions] Latest version in the v1.26 series: v1.26.15

Upgrade to the latest version in the v1.26 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.26.3   v1.26.15
kube-controller-manager   v1.26.3   v1.26.15
kube-scheduler            v1.26.3   v1.26.15
kube-proxy                v1.26.3   v1.26.15
CoreDNS                   v1.9.3    v1.9.3
etcd                      3.5.6-0   3.5.10-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.26.15

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
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

Reload and restart `kubelet` with:

```
sudo systemctl daemon-reload
sudo service kubelet restart
```

After this restart, one needs to approve the `csr` from kubernetes,
but this requires using `kubectl` which is precisely what was broken:

```
kubectl get csr
```

There will see the certificate waiting to be approved:

```
kubectl certificate approve csr-dlcf6
```

References:
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/
- https://devopstales.github.io/kubernetes/k8s-cert/
- https://kubernetes.io/docs/tasks/tls/certificate-rotation/
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs

## Time To Upgrade Maybe

Another reason to manually renew the server certificate
is that the node cannot be upgraded without it:

```
# kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[upgrade/config] FATAL: failed to get config map: Get "https://10.0.0.6:6443/api/v1/namespaces/kube-system/configmaps/kubeadm-config?timeout=10s": tls: failed to verify certificate: x509: certificate has expired or is not yet valid: current time 2024-03-23T10:16:27+01:00 is after 2024-03-21T21:37:37Z
To see the stack trace of this error execute with --v=5 or higher
```
