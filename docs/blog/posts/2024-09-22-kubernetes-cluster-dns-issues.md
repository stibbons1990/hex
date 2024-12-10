---
date: 2024-09-22
categories:
 - linux
 - ubuntu
 - server
 - kubernetes
 - docker
 - dns
title: Kubernetes cluster DNS issues
---

**TL;DR:**
[Forwarding IPv4 and letting iptables see bridged traffic](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
is **not optional**; even if it [looks like it](2024-05-12-single-node-kubernetes-cluster-on-ubuntu-studio-desktop-rapture.md#prerequisites).
Skipping that crucial step that takes a few minutes eventually led
to wasting over 3 hours troubleshooting a issue that, apparently,
*nobody has ever solved on the Internet before*. Naturally,
because *nobody should ever need to*.

<!-- more -->

### The Problem(s)

When I tried to *push* to Github from Visual Studio Code, after
[upgrading the Kubernetes on Rapture](2024-09-22-upgrading-single-node-kubernetes-cluster-on-ubuntu-studio-24-04.md)
(which is the article I was trying to publish), it failed with:

*Git: fatal: unable to access 'https://github.com/stibbons1990/hex.git/': Could not resolve host: github.com*

Showing command output only confirmed the error came from `git`:

```
> git push origin main:main
fatal: unable to access 'https://github.com/stibbons1990/hex.git/': Could not resolve host: github.com
```

At this moment several other issues immediately came to mind, as I
have been having DNS-related issues other pods *and* in the other
cluster:

1. Plex Media Server in Rapture first appeared unavailable, then
   would show itself as not authorized, eventually when trying to
   unclaim and reclaim the it the very last step failed silently.
   In hindsight, this looks like it tried to send a request to the
   Plex network and it fails due to the name not resolving.
2. Monitoring suddently *stopped working* after a reboot the
   previous day, but the *failure* was only between Grafana and
   InfluxDB and *only* when using Grafana reached for InfluxDB on
   its internal service name (`influxdb-svc`) and port 
   [http://influxdb-svc:18086](http://influxdb-svc:18086), even
   though the service was reachable on its external HTTPS address 
   (which actually relies on the same internal service name).
3. Audiobookshelf is *still now* unable to resolve the address of
   `api.audnex.us` even (several days) after the issue
   ([#3385](https://github.com/advplyr/audiobookshelf/issues/3385))
   was resolved. This fails in Rapture too, when deploying there.
4. Minecraft server is no longer running and stuck in 
   `CrashLoopBackOff` because it cannot resolve `api.papermc.io`.

The Minecraft server issue was discovered while troubleshooting
the others, and the logs offer a little more details:

```
$ kubectl -n minecraft-server logs minecraft-server-76f44bd597-kpw58 -f
[init] Running as uid=1003 gid=1003 with /data as 'drwxrwxr-x 1 1003 1003 886 Sep 20 04:10 /data'
[init] Resolving type given PAPER
[mc-image-helper] 10:53:56.316 ERROR : 'install-paper' command failed. Version is 1.39.11
reactor.core.Exceptions$ReactiveException: io.netty.resolver.dns.DnsResolveContext$SearchDomainUnknownHostException: Failed to resolve 'api.papermc.io' [A(1)] and search domain query for configured domains failed as well: [minecraft-server.svc.cluster.local, svc.cluster.local, cluster.local]
...
Caused by: io.netty.resolver.dns.DnsNameResolverTimeoutException: [9444: /10.96.0.10:53] DefaultDnsQuestion(api.papermc.io.minecraft-server.svc.cluster.local. IN A) query '9444' via UDP timed out after 5000 milliseconds (no stack trace available)
```

### Kube-DNS is healthy

The first thing to check would be whether Kube DNS (CoreDNS) is
running and healthy (answers queries correctly). This apperas to be
the case in both clusters:

#### Rapture

The service is running on ClusterIP 10.96.0.10 and it answers
correctly (104.21.44.87 is a known-good address of api.audnex.us):

```
$ kubectl get all -A | grep -i dns
kube-system            pod/coredns-5d78c9869d-9xbh4                    1/1     Running   0               160m
kube-system            pod/coredns-5d78c9869d-w8954                    1/1     Running   0               160m
kube-system            service/kube-dns                             ClusterIP      10.96.0.10       <none>          53/UDP,53/TCP,9153/TCP                                                                          132d
kube-system            deployment.apps/coredns                     2/2     2            2           132d
kube-system            replicaset.apps/coredns-5d78c9869d                    2         2         2       160m
kube-system            replicaset.apps/coredns-787d4945fb                    0         0         0       132d

$ nslookup k8s.io 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   k8s.io
Address: 34.107.204.206
Name:   k8s.io
Address: 2600:1901:0:26f3::

$ nslookup k8s.io 
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   k8s.io
Address: 34.107.204.206
Name:   k8s.io
Address: 2600:1901:0:26f3::

$ nslookup api.audnex.us 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57

$ nslookup api.audnex.us
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57

$ nslookup api.papermc.io 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   api.papermc.io
Address: 104.26.12.138
Name:   api.papermc.io
Address: 172.67.72.198
Name:   api.papermc.io
Address: 104.26.13.138
Name:   api.papermc.io
Address: 2606:4700:20::681a:c8a
Name:   api.papermc.io
Address: 2606:4700:20::ac43:48c6
Name:   api.papermc.io
Address: 2606:4700:20::681a:d8a

$ nslookup api.papermc.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   api.papermc.io
Address: 104.26.13.138
Name:   api.papermc.io
Address: 172.67.72.198
Name:   api.papermc.io
Address: 104.26.12.138
Name:   api.papermc.io
Address: 2606:4700:20::681a:d8a
Name:   api.papermc.io
Address: 2606:4700:20::681a:c8a
Name:   api.papermc.io
Address: 2606:4700:20::ac43:48c6
```

#### Rapture

The service is running on ClusterIP 10.96.0.10 and it answers
correctly (104.21.44.87 is a known-good address of `api.audnex.us`)

```
$ kubectl get all -A | grep -i dns
kube-system              pod/coredns-787d4945fb-67z8g                                 1/1     Running            54 (28h ago)      549d
kube-system              pod/coredns-787d4945fb-gsx6h                                 1/1     Running            54 (28h ago)      549d
kube-system              service/kube-dns                                                ClusterIP      10.96.0.10       <none>          53/UDP,53/TCP,9153/TCP                                                                          549d
kube-system              deployment.apps/coredns                                 2/2     2            2           549d
kube-system              replicaset.apps/coredns-787d4945fb                                 2         2         2       549d

$ nslookup k8s.io 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   k8s.io
Address: 34.107.204.206
Name:   k8s.io
Address: 2600:1901:0:26f3::

$ nslookup k8s.io 
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   k8s.io
Address: 34.107.204.206
Name:   k8s.io
Address: 2600:1901:0:26f3::

$ nslookup api.audnex.us 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637

$ nslookup api.audnex.us 
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637

$ nslookup api.papermc.io 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   api.papermc.io
Address: 104.26.12.138
Name:   api.papermc.io
Address: 172.67.72.198
Name:   api.papermc.io
Address: 104.26.13.138
Name:   api.papermc.io
Address: 2606:4700:20::681a:c8a
Name:   api.papermc.io
Address: 2606:4700:20::ac43:48c6
Name:   api.papermc.io
Address: 2606:4700:20::681a:d8a

$ nslookup api.papermc.io
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   api.papermc.io
Address: 104.26.13.138
Name:   api.papermc.io
Address: 172.67.72.198
Name:   api.papermc.io
Address: 104.26.12.138
Name:   api.papermc.io
Address: 2606:4700:20::681a:d8a
Name:   api.papermc.io
Address: 2606:4700:20::681a:c8a
Name:   api.papermc.io
Address: 2606:4700:20::ac43:48c6
```

### Pods are not reaching Kube-DNS

The error from the Minecraft server suggests the issue is with the
pods not being able to *reach* the DNS server; at least over UDP:

```
Caused by: io.netty.resolver.dns.DnsNameResolverTimeoutException: [9444: /10.96.0.10:53] DefaultDnsQuestion(api.papermc.io.minecraft-server.svc.cluster.local. IN A) query '9444' via UDP timed out after 5000 milliseconds (no stack trace available)
```

Worried about both clusters having the same ClusterIP for Kube DNS
(10.96.0.10); I tried stopping the whole customer in Rapture to see
if that helps Lexicon:

```
# systemctl stop kubelet
# systemctl stop containerd.service
# systemctl stop docker.service
# systemctl stop docker.socket
```

Once everything is stopped, Rapture Kube DNS is no longer available
and Lexicon Kube DNS still works, but audiobookshelf is still not
able to resolve `api.audnex.us`:

```
root@rapture:~# nslookup api.audnex.us 10.96.0.10
;; communications error to 10.96.0.10#53: connection refused
;; communications error to 10.96.0.10#53: connection refused
;; communications error to 10.96.0.10#53: connection refused
;; no servers could be reached

root@lexicon:~# nslookup api.audnex.us 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
```

Trying to reach the DNS server from the pod always fails:

```
$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- cat /etc/resolv.conf 
search audiobookshelf.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- ping api.audnex.us
ping: bad address 'api.audnex.us'
command terminated with exit code 1

$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- nslookup api.audnex.us
;; connection timed out; no servers could be reached

$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- nslookup api.audnex.us 10.96.0.10
;; connection timed out; no servers could be reached
```

However, the pods are able to reach Kube DNS on its **endpoints**:

```
$ kubectl get ep kube-dns --namespace=kube-system
NAME       ENDPOINTS                                                     AGE
kube-dns   10.244.0.178:53,10.244.0.183:53,10.244.0.178:53 + 3 more...   549d

$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- nslookup api.audnex.us 10.244.0.178
Server:         10.244.0.178
Address:        10.244.0.178:53

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55

Non-authoritative answer:
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637

$ kubectl -n audiobookshelf exec audiobookshelf-987b955fb-64gxm -- nslookup api.audnex.us 10.244.0.183
Server:         10.244.0.183
Address:        10.244.0.183:53

Non-authoritative answer:
Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 104.21.44.87

Non-authoritative answer:
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637
```

### The Solution

At this point the problem is pretty clear: CoreDNS works just fine,
it's just unreachable by its ClusterIP. On to searching for threads
discussing similiar issues.

After about 3 hours of
[unsuccessfully trying other approaches](#what-did-not-work),
the solution was found *quite indirectly* from a single check
mentioned in the opening of
[kubernetes issue #57096: kube-dns (10.96.0.10) unavailable on minion nodes](https://github.com/kubernetes/kubernetes/issues/57096):

> I ensure to set `net.ipv4.ip_forward=1` and 
> `net.bridge.bridge-nf-call-iptables=1` via `sysctl`

That caught my attention because it was not mentioned by any of the
other threads I had been reading in the previous 3 hours. Indeed,
going back to
[Forwarding IPv4 and letting iptables see bridged traffic](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
it turned out that *somehow* bridged traffic was no longer allowed:

```
# sysctl -a | egrep 'net.ipv4.ip_forward|net.bridge.bridge-nf-call-ip'
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_update_priority = 1
net.ipv4.ip_forward_use_pmtu = 0
```

Comparing with what **was** there when checking [prerequisites](2024-05-12-single-node-kubernetes-cluster-on-ubuntu-studio-desktop-rapture.md#prerequisites) in May, `net.bridge.*` variables are missing:

```
root@rapture:~# sysctl -a | egrep 'net.ipv4.ip_forward|net.bridge.bridge-nf-call-ip'
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_update_priority = 1
net.ipv4.ip_forward_use_pmtu = 0

root@rapture:~# lsmod | egrep 'overlay|bridge|br_netfilter'
bridge                311296  0
stp                    16384  1 bridge
llc                    16384  2 bridge,stp
overlay               151552  30
```

These are missing now:

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

And they are missing in Lexicon too. Finally, a likely culprit!

So I went back and added these explicitly, with the commands from
[Forwarding IPv4 and letting iptables see bridged traffic](https://v1-27.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)

```
root@rapture:~# cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
overlay
br_netfilter

root@rapture:~# sudo modprobe overlay
root@rapture:~# sudo modprobe br_netfilter

root@rapture:~# ls -l /etc/sysctl.d
total 40
-rw-r--r-- 1 root root   77 Feb 25  2022 10-console-messages.conf
-rw-r--r-- 1 root root  490 Feb 25  2022 10-ipv6-privacy.conf
-rw-r--r-- 1 root root 1229 Feb 25  2022 10-kernel-hardening.conf
-rw-r--r-- 1 root root 1184 Feb 25  2022 10-magic-sysrq.conf
-rw-r--r-- 1 root root  158 Feb 25  2022 10-network-security.conf
-rw-r--r-- 1 root root 1292 Feb 25  2022 10-ptrace.conf
-rw-r--r-- 1 root root  506 Feb 25  2022 10-zeropage.conf
-rw-rw-r-- 1 root root  146 Aug  1 15:14 30-brave.conf
-rw-r--r-- 1 root root  597 Mar 19  2022 50-ubuntustudio.conf
lrwxrwxrwx 1 root root   14 Nov 21  2023 99-sysctl.conf -> ../sysctl.conf
-rw-r--r-- 1 root root  798 Feb 25  2022 README.sysctl

root@rapture:~# grep bridge  /etc/sysctl.d/*.conf
root@rapture:~# grep forward /etc/sysctl.d/*.conf
/etc/sysctl.d/99-sysctl.conf:# Uncomment the next line to enable packet forwarding for IPv4
/etc/sysctl.d/99-sysctl.conf:#net.ipv4.ip_forward=1
/etc/sysctl.d/99-sysctl.conf:# Uncomment the next line to enable packet forwarding for IPv6
/etc/sysctl.d/99-sysctl.conf:#net.ipv6.conf.all.forwarding=1

root@rapture:~# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1

root@rapture:~# sudo sysctl --system
* Applying /etc/sysctl.d/10-console-messages.conf ...
kernel.printk = 4 4 1 7
* Applying /etc/sysctl.d/10-ipv6-privacy.conf ...
net.ipv6.conf.all.use_tempaddr = 2
net.ipv6.conf.default.use_tempaddr = 2
* Applying /etc/sysctl.d/10-kernel-hardening.conf ...
kernel.kptr_restrict = 1
* Applying /etc/sysctl.d/10-magic-sysrq.conf ...
kernel.sysrq = 176
* Applying /etc/sysctl.d/10-network-security.conf ...
net.ipv4.conf.default.rp_filter = 2
net.ipv4.conf.all.rp_filter = 2
* Applying /etc/sysctl.d/10-ptrace.conf ...
kernel.yama.ptrace_scope = 1
* Applying /etc/sysctl.d/10-zeropage.conf ...
vm.mmap_min_addr = 65536
* Applying /etc/sysctl.d/30-brave.conf ...
* Applying /usr/lib/sysctl.d/50-bubblewrap.conf ...
kernel.unprivileged_userns_clone = 1
* Applying /usr/lib/sysctl.d/50-default.conf ...
kernel.core_uses_pid = 1
net.ipv4.conf.default.rp_filter = 2
net.ipv4.conf.default.accept_source_route = 0
sysctl: setting key "net.ipv4.conf.all.accept_source_route": Invalid argument
net.ipv4.conf.default.promote_secondaries = 1
sysctl: setting key "net.ipv4.conf.all.promote_secondaries": Invalid argument
net.ipv4.ping_group_range = 0 2147483647
net.core.default_qdisc = fq_codel
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_regular = 1
fs.protected_fifos = 1
* Applying /usr/lib/sysctl.d/50-pid-max.conf ...
kernel.pid_max = 4194304
* Applying /etc/sysctl.d/50-ubuntustudio.conf ...
vm.swappiness = 10
* Applying /usr/lib/sysctl.d/99-protect-links.conf ...
fs.protected_fifos = 1
fs.protected_hardlinks = 1
fs.protected_regular = 2
fs.protected_symlinks = 1
* Applying /etc/sysctl.d/99-sysctl.conf ...
kernel.dmesg_restrict = 0
* Applying /etc/sysctl.d/k8s.conf ...
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
* Applying /etc/sysctl.conf ...
kernel.dmesg_restrict = 0

root@rapture:~# lsmod | egrep 'overlay|bridge|br_netfilter'
br_netfilter           32768  0
bridge                311296  1 br_netfilter
stp                    16384  1 bridge
llc                    16384  2 bridge,stp
overlay               151552  30

root@rapture:~# sysctl -a | egrep 'net.ipv4.ip_forward|net.bridge.bridge-nf-call-ip'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_update_priority = 1
net.ipv4.ip_forward_use_pmtu = 0
```

This made all the difference; pods can now reach the CoreDNS
service on its ClusterIP:

```
$ kubectl -n ingress-nginx exec ingress-nginx-controller-7c7754d4b6-w4w5g -- nslookup api.audnex.us 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637

Name:   api.audnex.us
Address: 172.67.198.55
Name:   api.audnex.us
Address: 104.21.44.87
```

To validate this fix, started Audiobookshelf in Rapture and tried
to match audiobooks. Not only it worked, it also was able to find
books by title and author (which had stopped working days ago).
The test with `nslookup` was also successful in this new pod.
also when not specifying which DNS server to query:

```
$ kubectl -n audiobookshelf exec pod/audiobookshelf-59c5f7c9f5-m5ztg -- nslookup api.audnex.us 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10:53

Non-authoritative answer:
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55

$ kubectl -n audiobookshelf exec pod/audiobookshelf-59c5f7c9f5-m5ztg -- nslookup api.audnex.us 
Server:         10.96.0.10
Address:        10.96.0.10:53

Non-authoritative answer:
Name:   api.audnex.us
Address: 104.21.44.87
Name:   api.audnex.us
Address: 172.67.198.55

Non-authoritative answer:
Name:   api.audnex.us
Address: 2606:4700:3037::6815:2c57
Name:   api.audnex.us
Address: 2606:4700:3030::ac43:c637

$ kubectl -n audiobookshelf exec pod/audiobookshelf-59c5f7c9f5-m5ztg -- cat /etc/resolv.conf 
search audiobookshelf.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

After applying the same fix to Lexicon, all the above issues are
finally solved:

1. Plex Media Server in Rapture is successfully claimed and is now
   available to stream media.
2. Grafana can reach InfluxDB on its internal service name
   (`influxdb-svc`) and port 
   [http://influxdb-svc:18086](http://influxdb-svc:18086).
3. Audiobookshelf is again able to find book matches by title and
   author, just as in Rapture.
4. Minecraft is running again; it started up pretty much as soon
   as the DNS service was reachable again.

### Not Over Yet

Just a couple days later I noticed in the Kubernetes dashboard that
the `coredns` deployment was broken because both pods were crash-looping:

```
# kubectl get all -A | grep -i dns
kube-system            pod/coredns-5d78c9869d-9xbh4                    0/1     CrashLoopBackOff   81 (2m14s ago)   2d12h
kube-system            pod/coredns-5d78c9869d-w8954                    0/1     CrashLoopBackOff   81 (2m30s ago)   2d12h
kube-system            service/kube-dns                             ClusterIP      10.96.0.10       <none>          53/UDP,53/TCP,9153/TCP                                                                          135d
kube-system            deployment.apps/coredns                     0/2     2            0           135d
kube-system            replicaset.apps/coredns-5d78c9869d                    2         2         0       2d12h
kube-system            replicaset.apps/coredns-787d4945fb                    0         0         0       135d

# get ep kube-dns --namespace=kube-system
NAME       ENDPOINTS   AGE
kube-dns               135d

# kubectl -n kube-system logs coredns-5d78c9869d-9xbh4 -f
.:53
[INFO] plugin/reload: Running configuration SHA512 = c0af6acba93e75312d34dc3f6c44bf8573acff497d229202a4a49405ad5d8266c556ca6f83ba0c9e74088593095f714ba5b916d197aa693d6120af8451160b80
CoreDNS-1.10.1
linux/amd64, go1.20, 055b2c3
[FATAL] plugin/loop: Loop (127.0.0.1:42071 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 5799046874759025118.6581212788663693097."

# kubectl -n kube-system logs coredns-5d78c9869d-w8954 -f
[INFO] plugin/ready: Still waiting on: "kubernetes"
.:53
[INFO] plugin/reload: Running configuration SHA512 = c0af6acba93e75312d34dc3f6c44bf8573acff497d229202a4a49405ad5d8266c556ca6f83ba0c9e74088593095f714ba5b916d197aa693d6120af8451160b80
CoreDNS-1.10.1
linux/amd64, go1.20, 055b2c3
[FATAL] plugin/loop: Loop (127.0.0.1:42908 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 426107543664332041.3956332948985369701."

# kubectl get ep kube-dns --namespace=kube-system
NAME       ENDPOINTS   AGE
kube-dns               135d

# nslookup k8s.io 10.96.0.10
;; communications error to 10.96.0.10#53: connection refused
;; communications error to 10.96.0.10#53: connection refused
;; communications error to 10.96.0.10#53: connection refused
;; no servers could be reached
```

The
[Troubleshooting](https://coredns.io/plugins/loop#troubleshooting)
page explains this is because a DNs loop has been detected and
recommend making sure that `kubelet` configuration points to the
*real* `resolve.conf`... which it already does:

```
# grep -iA2 resolv /var/lib/kubelet/config.yaml
resolvConf: /run/systemd/resolve/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 0s

# cat /run/systemd/resolve/resolv.conf
nameserver 62.2.24.158
nameserver 62.2.17.61
search .
```

At least the bridging of traffic seems to be fine:

```
# sysctl -a | egrep 'net.ipv4.ip_forward|net.bridge.bridge-nf-call-ip'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_update_priority = 1
net.ipv4.ip_forward_use_pmtu = 0

# cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search .

# iptables -L

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp has no endpoints */ reject-with icmp-port-unreachable
```

[Troubleshooting Loops In Kubernetes Clusters](https://coredns.io/plugins/loop/#troubleshooting-loops-in-kubernetes-clusters)
seems very generic so I turn to searching for more answer and find
a few clues... but not enough.

Maybe I should just
[disable local dns cache on ubuntu](https://www.google.com/search?q=disable+local+dns+cache+on+ubuntu&oq=disable+local+dns+cache+on+ubuntu&gs_lcrp=EgZjaHJvbWUyBggAEEUYOTIICAEQABgWGB4yCggCEAAYgAQYogTSAQg0NDYxajBqN6gCALACAA&sourceid=chrome&ie=UTF-8)?

```
# netstat -tulpn | grep '\<53\>'
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      2280/systemd-resolv 
udp        0      0 127.0.0.53:53           0.0.0.0:*                           2280/systemd-resolv
```

Comparing `/etc/resolv.conf` between the two servers, it seems odd
that the one in Rapture has `search .`; that doesn't seem right.

This can be overriden via Netplan:

```
# tail -3 /etc/resolv.conf 
nameserver 127.0.0.53
options edns0 trust-ad
search .

# vi /etc/netplan/01-network-manager-all.yaml 
```

Add `search` under `nameservers` and apply the change:

```yaml
      nameservers:
      # Set DNS name servers
        search: [v.cable.com]
```

```
# netplan apply

# tail -3 /etc/resolv.conf 
nameserver 127.0.0.53
options edns0 trust-ad
search v.cable.com
```

**Note:** `netplan apply` showed a couple of warnings, one due to
[Ubuntu bug #2041727](https://bugs.launchpad.net/ubuntu/+source/netplan.io/+bug/2041727)
(solved with `apt install openvswitch-switch`) and another one due
to having the same default route on 2 interfaces (it was needed in
only one of them).

All the above still did not help; both `coredns` pods keep
crash-looping with the same error about a lop being detected for
zone "."; even after forcing them to restart with

```
# kubectl scale --replicas=0 deployment.apps/coredns -n kube-system
# sleep 10
# kubectl scale --replicas=2 deployment.apps/coredns -n kube-system
```

[Flushing the local DNS cache](https://www.howtogeek.com/how-to-flush-dns-cache-in-ubuntu/)
also did not help:

```
root@rapture:~# resolvectl statistics
DNSSEC supported by current servers: no

Transactions              
Current Transactions: 0
  Total Transactions: 4486
                          
Cache                     
  Current Cache Size: 56
          Cache Hits: 61
        Cache Misses: 339
                          
DNSSEC Verdicts           
              Secure: 0
            Insecure: 0
               Bogus: 0
       Indeterminate: 0
root@rapture:~# resolvectl flush-caches
root@rapture:~# resolvectl statistics
DNSSEC supported by current servers: no

Transactions              
Current Transactions: 0
  Total Transactions: 4490
                          
Cache                     
  Current Cache Size: 0
          Cache Hits: 63
        Cache Misses: 341
                          
DNSSEC Verdicts           
              Secure: 0
            Insecure: 0
               Bogus: 0
       Indeterminate: 0
```

The next thing to try is entirely
[Disabling Local DNS Caching](https://tecadmin.net/disable-local-dns-caching-ubuntu/).

```
# systemctl list-units --type=service | grep -E 'systemd-resolved|dnsmasq' 
  systemd-resolved.service
             loaded active running Network Name Resolution

# vi /etc/systemd/resolved.conf
```

```ini
[Resolve]
Domains=v.cablecom.net
Cache=no
```

```
# systemctl restart systemd-resolved
root@rapture:~# dig tecadmin.net

; <<>> DiG 9.18.28-0ubuntu0.22.04.1-Ubuntu <<>> tecadmin.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14739
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;tecadmin.net.                  IN      A

;; ANSWER SECTION:
tecadmin.net.           150     IN      A       104.21.25.106
tecadmin.net.           150     IN      A       172.67.134.5

;; Query time: 39 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Wed Sep 25 20:37:28 CEST 2024
;; MSG SIZE  rcvd: 73

# kubectl get pods -n kube-system | grep -i dns
coredns-5d78c9869d-gv9q5          0/1     CrashLoopBackOff   4 (14s ago)   107s
coredns-5d78c9869d-zv5rt          0/1     CrashLoopBackOff   4 (8s ago)    107s

# kubectl -n kube-system logs coredns-5d78c9869d-zv5rt
.:53
[INFO] plugin/reload: Running configuration SHA512 = c0af6acba93e75312d34dc3f6c44bf8573acff497d229202a4a49405ad5d8266c556ca6f83ba0c9e74088593095f714ba5b916d197aa693d6120af8451160b80
CoreDNS-1.10.1
linux/amd64, go1.20, 055b2c3
[FATAL] plugin/loop: Loop (127.0.0.1:48053 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 2605680300388702341.222935474922528127."

```

This does change the TTL but is still making DNS queries go to the
local DNS service on `127.0.0.53`; so this still does not help.

To get the local DNS service on `127.0.0.53` *out of the way* the
configuration in `/etc/systemd/resolved.conf` must disable the
DNS sbut listener:

```ini
[Resolve]
DNSStubListener=no
```

This gets DNS requests answered directly by the non-local DNS
servers, but even this still does not help removing the loop!

```
# systemctl restart systemd-resolved

# nslookup k8s.io 127.0.0.53
;; communications error to 127.0.0.1#53: connection refused
;; communications error to 127.0.0.1#53: connection refused
;; communications error to 127.0.0.1#53: connection refused
;; no servers could be reached

# dig tecadmin.net

; <<>> DiG 9.18.28-0ubuntu0.22.04.1-Ubuntu <<>> tecadmin.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46975
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;tecadmin.net.                  IN      A

;; ANSWER SECTION:
tecadmin.net.           300     IN      A       104.21.25.106
tecadmin.net.           300     IN      A       172.67.134.5

;; Query time: 19 msec
;; SERVER: 62.2.24.158#53(62.2.24.158) (UDP)
;; WHEN: Wed Sep 25 20:48:57 CEST 2024
;; MSG SIZE  rcvd: 73

# kubectl get pods -n kube-system | grep -i dns
coredns-5d78c9869d-489wl          0/1     CrashLoopBackOff   4 (61s ago)    2m49s
coredns-5d78c9869d-pwn9x          0/1     CrashLoopBackOff   4 (68s ago)    2m49s

# kubectl -n kube-system logs coredns-5d78c9869d-489wl
.:53
[INFO] plugin/reload: Running configuration SHA512 = c0af6acba93e75312d34dc3f6c44bf8573acff497d229202a4a49405ad5d8266c556ca6f83ba0c9e74088593095f714ba5b916d197aa693d6120af8451160b80
CoreDNS-1.10.1
linux/amd64, go1.20, 055b2c3
[FATAL] plugin/loop: Loop (127.0.0.1:43216 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 7289571247349402084.8563501253473529468."
```

The next thing I can think of trying is the *quick and dirty fix*
[Troubleshooting Loops In Kubernetes Clusters](https://coredns.io/plugins/loop/#troubleshooting-loops-in-kubernetes-clusters)
mentions as the last resort; there is just **no** DNS listening
on 127.0.0.53 anyway so may as well forward it to a real one.

```
# kubectl get -n kube-system cm/coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2024-05-12T10:52:00Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "5838688"
  uid: d29d50a6-7f42-4046-b157-f93876807af1

# kubectl -n kube-system edit configmaps coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . 62.2.24.158 {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2024-05-12T10:52:00Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "5959049"
  uid: d29d50a6-7f42-4046-b157-f93876807af1

# systemctl restart kubelet

# kubectl get pods -n kube-system | grep dns
coredns-55cb58b774-ph8qz          1/1     Running   33 (5m39s ago)   23h
coredns-55cb58b774-rd6wg          1/1     Running   33 (5m30s ago)   23h
```

***Finally!!!*** Now we wait and see if this *really fixed it*...

### What Did Not Work

#### Workaround around IP tables rules

The answer to
[coredns do not resolve service name correctly](https://stackoverflow.com/a/58788222)
[kubeadm issue #1056: Fresh deploy with CoreDNS not resolving any dns lookup](https://github.com/kubernetes/kubeadm/issues/1056)
where it is noted that firewall rules are blocking access to the
DNS service ClusterIP:

```
root@rapture:~# iptables -L | grep 10.96.0.10
REJECT     udp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics has no endpoints */ reject-with icmp-port-unreachable
```

That would explain why the DNS is unreachable on its Cluster IP,
but why are those rules there?

[Prodian0013 points out](https://github.com/kubernetes/kubernetes/issues/83063#issuecomment-535687028) 
that these rules are added by `kube-proxy` when it sees that
`kube-dns` has no endpoints... but it does!

```
$ kubectl describe ep kube-dns -n kube-system
Name:         kube-dns
Namespace:    kube-system
Labels:       k8s-app=kube-dns
              kubernetes.io/cluster-service=true
              kubernetes.io/name=CoreDNS
Annotations:  <none>
Subsets:
  Addresses:          10.244.0.62,10.244.0.63
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    dns-tcp  53    TCP
    dns      53    UDP
    metrics  9153  TCP

Events:  <none>
```

So those rules should not be there.
In fact, when I went back to check again, they were gone.

Before:

```
# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-NODEPORTS  all  --  anywhere             anywhere             /* kubernetes health check service ports */
KUBE-EXTERNAL-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes externally-visible service portals */
KUBE-FIREWALL  all  --  anywhere             anywhere            

Chain FORWARD (policy DROP)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-FORWARD  all  --  anywhere             anywhere             /* kubernetes forwarding rules */
KUBE-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
KUBE-EXTERNAL-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes externally-visible service portals */
DOCKER-USER  all  --  anywhere             anywhere            
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            
ACCEPT     all  --  anywhere             anywhere            
FLANNEL-FWD  all  --  anywhere             anywhere             /* flanneld forward */

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
KUBE-FIREWALL  all  --  anywhere             anywhere            

Chain DOCKER (1 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination         
DOCKER-ISOLATION-STAGE-2  all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere            
RETURN     all  --  anywhere             anywhere            

Chain DOCKER-USER (1 references)
target     prot opt source               destination         
RETURN     all  --  anywhere             anywhere            

Chain FLANNEL-FWD (1 references)
target     prot opt source               destination         
ACCEPT     all  --  rapture/16           anywhere             /* flanneld forward */
ACCEPT     all  --  anywhere             rapture/16           /* flanneld forward */

Chain KUBE-EXTERNAL-SERVICES (2 references)
target     prot opt source               destination         
REJECT     tcp  --  anywhere             nginx.rapture.uu.am  /* ingress-nginx/ingress-nginx-controller:https has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:https has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:discovery-udp has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:discovery-udp has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:gdm-32412 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:gdm-32412 has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:gdm-32413 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:gdm-32413 has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-tcp:pms-web has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* plexserver/plex-tcp:pms-web has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:dlna-udp has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:dlna-udp has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-tcp:plex-roku has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* plexserver/plex-tcp:plex-roku has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             photoprism.rapture.uu.am  /* photoprism/photoprism:http has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* photoprism/photoprism:http has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-tcp:plex-companion has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* plexserver/plex-tcp:plex-companion has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-tcp:dlna-tcp has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* plexserver/plex-tcp:dlna-tcp has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             nginx.rapture.uu.am  /* ingress-nginx/ingress-nginx-controller:http has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:http has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:gdm-32410 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:gdm-32410 has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             plex.rapture.uu.am   /* plexserver/plex-udp:gdm-32414 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             anywhere             /* plexserver/plex-udp:gdm-32414 has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             k8s.rapture.uu.am    /* kubernetes-dashboard/kubernetes-dashboard has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             anywhere             /* kubernetes-dashboard/kubernetes-dashboard has no endpoints */ ADDRTYPE match dst-type LOCAL reject-with icmp-port-unreachable

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination         
DROP       all  -- !localhost/8          localhost/8          /* block incoming localnet connections */ ! ctstate RELATED,ESTABLISHED,DNAT

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere             ctstate INVALID
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding rules */
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding conntrack rule */ ctstate RELATED,ESTABLISHED

Chain KUBE-KUBELET-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-NODEPORTS (1 references)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:https health check node port */
ACCEPT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:http health check node port */

Chain KUBE-PROXY-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-PROXY-FIREWALL (3 references)
target     prot opt source               destination         

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
REJECT     tcp  --  anywhere             10.97.59.176         /* ingress-nginx/ingress-nginx-controller:https has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.98.208.0          /* ingress-nginx/ingress-nginx-controller-admission:https-webhook has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:discovery-udp has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:gdm-32412 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:gdm-32413 has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.97.83.71          /* plexserver/plex-tcp:pms-web has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:dlna-udp has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.97.83.71          /* plexserver/plex-tcp:plex-roku has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:dns-tcp has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.109.211.96        /* metallb-system/metallb-webhook-service has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.108.135.112       /* photoprism/photoprism:http has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.97.83.71          /* plexserver/plex-tcp:plex-companion has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.97.83.71          /* plexserver/plex-tcp:dlna-tcp has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.97.59.176         /* ingress-nginx/ingress-nginx-controller:http has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.96.0.10           /* kube-system/kube-dns:metrics has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.107.131.104       /* kubernetes-dashboard/dashboard-metrics-scraper has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:gdm-32410 has no endpoints */ reject-with icmp-port-unreachable
REJECT     udp  --  anywhere             10.100.30.101        /* plexserver/plex-udp:gdm-32414 has no endpoints */ reject-with icmp-port-unreachable
REJECT     tcp  --  anywhere             10.107.235.155       /* kubernetes-dashboard/kubernetes-dashboard has no endpoints */ reject-with icmp-port-unreachable
```

After:

```
# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-NODEPORTS  all  --  anywhere             anywhere             /* kubernetes health check service ports */
KUBE-EXTERNAL-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes externally-visible service portals */
KUBE-FIREWALL  all  --  anywhere             anywhere            

Chain FORWARD (policy DROP)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-FORWARD  all  --  anywhere             anywhere             /* kubernetes forwarding rules */
KUBE-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
KUBE-EXTERNAL-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes externally-visible service portals */
FLANNEL-FWD  all  --  anywhere             anywhere             /* flanneld forward */

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
KUBE-PROXY-FIREWALL  all  --  anywhere             anywhere             ctstate NEW /* kubernetes load balancer firewall */
KUBE-SERVICES  all  --  anywhere             anywhere             ctstate NEW /* kubernetes service portals */
KUBE-FIREWALL  all  --  anywhere             anywhere            

Chain DOCKER (0 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (0 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-2 (0 references)
target     prot opt source               destination         

Chain DOCKER-USER (0 references)
target     prot opt source               destination         

Chain FLANNEL-FWD (1 references)
target     prot opt source               destination         
ACCEPT     all  --  rapture/16           anywhere             /* flanneld forward */
ACCEPT     all  --  anywhere             rapture/16           /* flanneld forward */

Chain KUBE-EXTERNAL-SERVICES (2 references)
target     prot opt source               destination         

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination         
DROP       all  -- !localhost/8          localhost/8          /* block incoming localnet connections */ ! ctstate RELATED,ESTABLISHED,DNAT

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             anywhere             ctstate INVALID
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding rules */
ACCEPT     all  --  anywhere             anywhere             /* kubernetes forwarding conntrack rule */ ctstate RELATED,ESTABLISHED

Chain KUBE-KUBELET-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-NODEPORTS (1 references)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:http health check node port */
ACCEPT     tcp  --  anywhere             anywhere             /* ingress-nginx/ingress-nginx-controller:https health check node port */

Chain KUBE-PROXY-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-PROXY-FIREWALL (3 references)
target     prot opt source               destination         

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
```

Even then, I tried the commands in 
[that comment](https://github.com/kubernetes/kubernetes/issues/83063#issuecomment-535687028)
but they didn't help.

**Note:** had to make a symlink for `crictl` commands to work well:

```
# ln -s /var/run/containerd/containerd.sock /var/run/dockershim.sock
# ls -l /var/run/containerd/containerd.sock /var/run/dockershim.sock
srw-rw---- 1 root root  0 Sep 22 13:34 /var/run/containerd/containerd.sock
lrwxrwxrwx 1 root root 35 Sep 22 13:53 /var/run/dockershim.sock -> /var/run/containerd/containerd.sock
```

#### Debugging DNS Resolution

Kubernetes guide for
[Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/),
which was linked from
[Kubernetes : kube-dns service not accessible via ClusterIP](https://serverfault.com/questions/1050450/kubernetes-kube-dns-service-not-accessible-via-clusterip),
did not help either. Naturally, that documentation would
*assume the installation didn't miss a crucial step*.

#### Override `clusterDNS` in Kubelet configuration

[pkeuter's comment](https://github.com/kubernetes/kubeadm/issues/1056#issuecomment-417579533)
in
[kubeadm issue #1056: Fresh deploy with CoreDNS not resolving any dns lookup](https://github.com/kubernetes/kubeadm/issues/1056)
suggesting to override the `clusterDNS` variable in
`/var/lib/kubelet/config.yaml` also did not help.

This was also reported to be a working solution for
[Pods cannot resolve kubernetes DNS](https://stackoverflow.com/questions/76556493/pods-cannot-resolve-kubernetes-dns).

However, this variable **must** be set to the ClusterIP, since
that is the one that won't change over time. This may be an easy
*workaround* in other situations, but in my case fixing the missed
prerequisite seems to have been **the** correct solution.