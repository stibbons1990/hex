---
date: 2025-03-28
draft: true
categories:
 - privacy
 - security
title: Remote access options for self-hosted services
---

## Remote access

Although during the instalation this sytem was behind a router that allowed port
forwarding, and it worked, the final destination for this sytem may be behind a different
router that either doens't allow port forwarding, or just doesn't work well. Even then,
[CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT) may also replace the currently
*de-facto* static IP addresses available so far.

Making ports reachable from outside will require a different approach and implementation
depending on the type of traffic served by the application behind each port:

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

341234

- CF only for cert renewals, multiple ngnix behind single ip, and backup for bad routers
- 433 port + ngnix for everything else (open ports in routers)
  - dyndns not tet needed
- tailscale maybe for secure tunnel rib->thl


1.  A DNS record pointing `alfred.uu.am` to the external IP address of the router,
    with port 80 redirected to the node's port `32080`, so that certe  (nothing listening there yet)
    and the `NodePort` of Nginx respectively. The same can work with a port other than 443.
2.  
    redirecting alfred.something-else.uu.am via the `cloudflared` connector with ports 80
    and 443 redirected to the local port 80 (nothing listening there yet) and the
    `NodePort` of Nginx respectively. This requires using Cloudflare DNS; probably best
    with done with a different domain regirested with Cloudflare.
3.  [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) or similar to establish a
    a tunnel with real privacy and unrestricted use (media streaming, e.g. audiobooks).


1.  A DNS record pointing `alfred.uu.am` to the external IP address of the router,
    with port 80 redirected to the node's port `32080`, so that certe  (nothing listening there yet)
    and the `NodePort` of Nginx respectively. The same can work with a port other than 443.
2.  [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
    redirecting alfred.something-else.uu.am via the `cloudflared` connector with ports 80
    and 443 redirected to the local port 80 (nothing listening there yet) and the
    `NodePort` of Nginx respectively. This requires using Cloudflare DNS; probably best
    with done with a different domain regirested with Cloudflare.
3.  [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) or similar to establish a
    a tunnel with real privacy and unrestricted use (media streaming, e.g. audiobooks).

### Cloudflare

#### Cloudflare Tunnels

https://tsmith.co/2023/cloudflare-zero-trust-tunnels-for-the-homelab/
https://chriscolotti.us/technology/how-to-setup-and-use-cloudflare-tunnels/

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
