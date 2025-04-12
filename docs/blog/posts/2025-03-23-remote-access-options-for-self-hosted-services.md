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
[automatically renewing Let's Encrypt certificates](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#automatic-renovation).

Otherwise, being behind a router that either doens't allow port forwarding, or just doesn't
work well, or being behind [CGNAT](https://en.wikipedia.org/wiki/Carrier-grade_NAT),
may require the use of some sort of tunnels to route inbound traffic using outbound
connections. This can also be useful even in the above case, when multiple systems need to
be reachable *on port 80*.

??? note "Cloudflare tunnels do not enable access on port 80."

    Cloudflare redirects port 80 to 443, to upgrade HTTP connections to HTTPS. That means
    ACME HTTP-01 challenges to renew Let's Encrypt certificates need to be routed to the
    relevant port (80 or 32080) based on the request **path**; see
    [Let's Encrypt via tunnel](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#lets-encrypt-via-tunnel).

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
node. While it is possible to access Nginx on additional nodes by forwarding a different
port, the automated renewals of HTTPS certificates is only possible over port 80.

This limitation may mean the node that is rechable through the external port 80 is the
only one that can serve high-bandwidth content directly via port forwarding. All other
nodes will need to use Cloudflare tunnels, and thus be restricted to HTML-only or
low-bandwidth traffic, or use Tailscale Funnel to serve high-bandwidth traffic, and
thus be restricted to specific clients.

## Cloudflare

To get started with Cloudflare, create account and a team (e.g. `high-energy-building`),
select the **Free plan**, enter billing details and checkout. Make sure to enable 2FA
authentiction and take a look around for other settings to personalize.

### Add the first site

[Adding a site](https://developers.cloudflare.com/fundamentals/setup/manage-domains/add-site/)
is required before creating a tunnel. This can be an external domain, although it requires
replacing that domain's DNS with the one from Cloudflare. It seems most convenient to have
a dedicated domain for self-hosted applications, to be used mostly, if not exclusively,
through Cloudflare tunnels. For this purpose, I've registered `very-very-dark-gray.top`
with Porkbun and cleared the default DNS records, which removed their
[parking domain](https://kb.porkbun.com/article/239-cname-alias-record-with-that-host-already-exists-error).

Start at the **Account Home** in
[dash.cloudflare.com](https://dash.cloudflare.com/0041a4dbd1c161fe785c4f63fa8fcc06/home),
enter the new domain and select the option to *Manually enter DNS records* since
there is none anyway. Scroll down to select the **Free plan** and continue to
start adding DNS records.

Having no DNS records yet, 3 warnings are shown about missing an MX record to for
emails to `@very-very-dark-gray.top` addresses and missing A/AAAA/CNAME records
for the root domain and `www`. All that is fine, there is no need for those.

As a quick test, add a subdomain for `alfred` pointing to the router's external
IPv4 address and let it be proxied by Cloudflare to see how that goes. Then
replace the DNS for this domain with the ones from Cloudflare's, and wait.

Once the change is live, go under **DNS > Settings** in Cloudflare and
[Enable DNSSEC](https://developers.cloudflare.com/dns/dnssec/).

### Cloudflare Tunnels

Tunnels are very easy to setup; accessing applications behind them not necessarily so much.
[Cloudflare Tunnels in Alfred](./2025-02-22-home-assistant-on-kubernetes-on-raspberry-pi-5-alfred.md#cloudflare-tunnel)
involved an *unfair* amount of troubleshooting to get the Kubernetes dashboard to work at
[https://kubernetes-alfred.very-very-dark-gray.top/](https://kubernetes-alfred.very-very-dark-gray.top/),
eventually using it's own Let's Encrypt certificates.

### Cloudflare Access

[Zero Trust Web Access](https://developers.cloudflare.com/learning-paths/zero-trust-web-access/)
builds on top of the previous setup; once an applications is made available through a
tunnel, restricting access to a fixed set of users (e.g. Google accounts) requires only
a few more steps:

1.  [Set up Google as an identity provider](https://developers.cloudflare.com/cloudflare-one/identity/idp-integration/google/),
    which requires creating an OAuth Client ID in the Google Cloud Console.

    The OAuth client will be limited to a list of (max 100) test users, which is more than
    enough but those users have to be manually added in the Google Cloud Console, under
    [/auth/audience?project=very-very-dark-gray](https://console.cloud.google.com/auth/audience?project=very-very-dark-gray).

1.  Create a **Policy** (e.g *Google Account*) to **Allow** only a few users, with
    **only one Include rule** to match those users' email addresses.

1.  [Create an Access application](https://developers.cloudflare.com/learning-paths/zero-trust-web-access/access-application/create-access-app/); more precisely a
    **self-hosted** application (e.g. `kubernetes-alfred`) for a **Public hostname**
    ([https://kubernetes-alfred.very-very-dark-gray.top/](https://kubernetes-alfred.very-very-dark-gray.top/)) and add **only** the **Google Account** policy, so that
    only users that match that policy are allowed. To authenicate those users, enable
    **only** the *Google* authentication in **Login methods** ane enable **Instant Auth**.

So long as **only one policy** is added to the applications *and* that policy has
**only one Include rule**, only those users added to that Include rule are allowed in.

Additional sources of inspiration (not reference):

- [Cloudflare Tunnels with SSO/OAuth working for immich](https://github.com/immich-app/immich/discussions/8299)
- [HOWTO: Secure Cloudflare Tunnels remote access](https://community.home-assistant.io/t/howto-secure-cloudflare-tunnels-remote-access/570837/1)

## Tailscale

TODO: start with <https://tailscale.com/kb/1223/funnel> to learn about Tailscale Funnel,
then set one up to access an SSH server that is not otherwise externally reachable.