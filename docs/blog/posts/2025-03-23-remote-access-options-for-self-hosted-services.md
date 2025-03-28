---
date: 2025-03-28
draft: true
categories:
 - privacy
 - security
title: Remote access options for self-hosted services
---

Running self-hosted services behind a router that allows port forwarding is *mostly* as
simple as forwarding a few ports, mainly 443 for everything over HTTP**S** and port 80 for
[automatically renewing Let's Encrypt certificates](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#automatic-renovation.md).

Otherwise, being behind a router that either doens't allow port forwarding, or just doesn't
work well, or being behind [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT),
may require the use of some sort of tunnels to route inbound traffic using outbound
connections. This can also be useful even in the above case, when multiple systems need to
be reachable *on port 80*.

<!-- more -->

## Options considered

Making applications (ports) reachable from outside will require a different approach and
implementation depending on the type of traffic served by the application behind each port:

- **HTML *non-sensitive* content** can easibly be made available through a
  [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).
  These are free to use and easy to setup, with the caveats that

    * Traffic **must** be decrypted by Cloudflare, to apply certain traffic filters, even
      if everything is then (re-)encrypted between Cloudflare and each service.
    * Traffic **must** be *either* HTML content or **low-bandwidth** non-HTML content.
      Cloudflare does not allow, at least when using their tunnels for free, to stream
      high-bandwidth multi-media content: audio, video, photos, videogame client-to-server
      communication, etc.

- **HTML *sensitive* content** may also be made available through a
  [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/);
  [Zero Trust Access](https://developers.cloudflare.com/cloudflare-one/policies/access/)
  can be used to limit access to each application (port) based on a number of rules,
  including user's identity as verified by 3P SSO providers (e.g. Google account).

- **Non-HTML *sensitive* content** may also be made available through a
  [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/);
  [Non-HTTP applications](https://developers.cloudflare.com/cloudflare-one/applications/non-http/)
  [require connecting your private network to Cloudflare](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/)
  which, again, means that Cloudflare is decrypting traffic (to inspected it and apply
  filters) to then re-encrypt it again. Then again, this may only be used for
  **low-bandwidth** non-HTML content, e.g. SSH sessions. Ideally with password
  authentication disabled!

- **Non-HTML *sensitive* or *high-bandwidth* content** can be made *reachable* externally,
  albeit from only a few clients, through the use of
  [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) (or similar) to establish
  [fully-encrypted](https://tailscale.com/kb/1504/encryption) (TLS passthrough) tunnels
  between *sites* (LANs) and a few trusted clients.

## Options chosen

With the above considerations, each of those options would seem to best match the
relevant self-hosted services as follows:

- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) 
  can be used for non-sensitive, or at least *not-too-sensitive*, HTML content:
    * [automatically renewing Let's Encrypt certificates](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#automatic-renovation.md) 
      over plain HTTP on port 80.
    * [InfluxDB](./2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md)
      already serving over HTTPS and *not-too-sensitive*; basic PC monitoring.

- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) 
  can also be used for sensitive HTML content, when combined with
  [Zero Trust Access](https://developers.cloudflare.com/cloudflare-one/policies/access/)
  to restrict access to each service:
    * [Grafana](./2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md)
    * [UniFi Controller](./2024-12-31-migrating-unifi-controller-to-kubernetes.md)
    * [Visual Studio Code Server](./2023-05-29-running-vs-code-server-on-kubernetes.md)
    * [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
    * [Firefly III](./2024-05-19-self-hosted-accountancy-with-firefly-iii.md)
    * [Homebox](./2024-07-10-self-hosted-inventory-with-homebox.md)

- [Tailscale Funnel](https://tailscale.com/kb/1223/funnel)
  must be used for those applications that transfer mostly non-HTML content:
    * [Audiobookshelf](./2024-02-28-audiobookshelf-on-kubernetes.md)
    * [Komga: eBook library](./2024-05-26-self-hosted-ebook-library-with-komga.md)
    * [Navidrome](./2024-10-26-self-hosted-music-streaming-with-navidrome.md)
    * [Plex Media Server](./2023-09-16-migrating-a-plex-media-server-to-kubernetes.md)
    * SSH

- Port forwarding may still be needed for **non-HTTP** services that are meant to be
  accessible by *technically not trusted* clients, i.e. without Tailscale. This would
  be mostl video game servers, of which there is so far only
    * [Minecraft Server](./2023-08-10-running-minecraft-java-server-for-bedrock-clients-on-kubernetes.md)

DNS records are setup to point the various `<service>.<node>.uu.am` to the external IP
address of the router, with port 80 redirected to **one** node's port `32080`, so that
HTTPS certificates can be renewed automatically, and port 443 pointing to Nginx on **one**
node. It is possibly to access Nginx on additional nodes by forwarding a different port,
but the automated renewals of HTTPS certificates is only possibly over port 80.

This limitation may mean the node that is rechable through the external port 80 is the
only one that can serve high-bandwidth content directly via port forwarding. All other
nodes will need to use Cloudflare tunnels, and thus be restricted to HTML-only or
low-bandwidth traffic, or use Tailscale Funnel to serve high-bandwidth traffic, and
thus be restricted to specific clients.

### Cloudflare

#### Cloudflare Tunnels

- https://tsmith.co/2023/cloudflare-zero-trust-tunnels-for-the-homelab/
- https://chriscolotti.us/technology/how-to-setup-and-use-cloudflare-tunnels/

- https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- https://developers.cloudflare.com/cloudflare-one/policies/access/

This requires using Cloudflare for DNS.

Create a tunnel in the Cloudflare Zero Trust Web interface and bring up a
docker container. You only need one container “per source IP” like your house.
You can then run multiple application hostnames over that tunnel.

To creat a tunnel in Zero Trust:

1.  Go to Access–>Tunnels on the menu.
2.  Give the tunnel a name.
3.  Install the connector; this can run in docker, docker compose, etc.
4.  Assign at least one application route
    -  Externally this exposed as `sub.domain/path`
    -  Internally this redirects to an IP+port (HTTP/S or SSH/RDP)

You can also run the connector with the same token from multiple hosts inside
the homelab network, so that one keeps the tunnel up while the other updates.

#### Cloudflare Access

https://chriscolotti.us/technology/how-to-add-cloudflare-access-to-tunnels/

#### Cloudflare HTTPS certificates

https://community.cloudflare.com/t/tunnel-encrypted/751222/2

### Tailscale

https://tailscale.com/kb/1223/funnel