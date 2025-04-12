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

[Tailscale quickstart](https://tailscale.com/kb/1017/install) explains the basics and how to
get started as a [personal user](https://tailscale.com/kb/1017/install#personal-users).

### Create a tailnet

[Create a tailnet](https://tailscale.com/kb/1017/install#create-a-tailnet)
by singing up with your choice of identity provider; e.g. signging up with a `@gmail.com`
account automatically set the tailnet up so that other `@gmail.com` users can be invited
to form part of the "team". Choosing the **Personal Plan** keeps the service free of charge,
there is not even a requirement to setup a valid billing method (e.g. credit card).

The quickstart wizard will wait until **Tailscale** is installed on 2 systems, e.g. a
server and a PC. The installation process is as simple as running their `install.sh` script:

??? terminal "Installation of Tailscale on **alfred**"

    ``` console
    pi@alfred:~ $ curl -fsSL https://tailscale.com/install.sh | sh
    Installing Tailscale for debian bookworm, using method apt
    + sudo mkdir -p --mode=0755 /usr/share/keyrings
    + curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg
    + sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg
    + sudo chmod 0644 /usr/share/keyrings/tailscale-archive-keyring.gpg
    + curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.tailscale-keyring.list
    + sudo tee /etc/apt/sources.list.d/tailscale.list
    # Tailscale packages for debian bookworm
    deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/debian bookworm main
    + sudo chmod 0644 /etc/apt/sources.list.d/tailscale.list
    + sudo apt-get update
    Hit:1 http://deb.debian.org/debian bookworm InRelease
    Hit:2 http://archive.raspberrypi.com/debian bookworm InRelease                                               
    Hit:3 https://download.docker.com/linux/debian bookworm InRelease                                            
    Hit:4 http://deb.debian.org/debian-security bookworm-security InRelease                                      
    Hit:5 http://deb.debian.org/debian bookworm-updates InRelease                                                
    Hit:6 https://baltocdn.com/helm/stable/debian all InRelease                                                  
    Hit:7 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb  InRelease
    Get:8 https://pkgs.tailscale.com/stable/debian bookworm InRelease               
    Get:9 https://pkgs.tailscale.com/stable/debian bookworm/main armhf Packages [12.2 kB]
    Get:10 https://pkgs.tailscale.com/stable/debian bookworm/main arm64 Packages [12.3 kB]
    Get:11 https://pkgs.tailscale.com/stable/debian bookworm/main all Packages [354 B]
    Fetched 31.4 kB in 1s (29.4 kB/s)  
    Reading package lists... Done
    + sudo apt-get install -y tailscale tailscale-archive-keyring
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following packages were automatically installed and are no longer required:
      linux-headers-6.6.51+rpt-common-rpi linux-kbuild-6.6.51+rpt
    Use 'sudo apt autoremove' to remove them.
    The following NEW packages will be installed:
      tailscale tailscale-archive-keyring
    0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
    Need to get 29.6 MB of archives.
    After this operation, 56.1 MB of additional disk space will be used.
    Get:2 https://pkgs.tailscale.com/stable/debian bookworm/main all tailscale-archive-keyring all 1.35.181 [3,082 B]
    Get:1 https://pkgs.tailscale.com/stable/debian bookworm/main arm64 tailscale arm64 1.82.0 [29.6 MB]   
    Fetched 29.6 MB in 15s (2,040 kB/s)                                                                          
    Selecting previously unselected package tailscale.
    (Reading database ... 91338 files and directories currently installed.)
    Preparing to unpack .../tailscale_1.82.0_arm64.deb ...
    Unpacking tailscale (1.82.0) ...
    Selecting previously unselected package tailscale-archive-keyring.
    Preparing to unpack .../tailscale-archive-keyring_1.35.181_all.deb ...
    Unpacking tailscale-archive-keyring (1.35.181) ...
    Setting up tailscale-archive-keyring (1.35.181) ...
    Setting up tailscale (1.82.0) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/tailscaled.service → /lib/systemd/system/tailscaled.service.
    + [ false = true ]
    + set +x
    Installation complete! Log in to start using Tailscale by running:

    sudo tailscale up
    ```

??? terminal "Installation of Tailscale on **rapture**"

    ``` console
    $ curl -fsSL https://tailscale.com/install.sh | sh
    Installing Tailscale for ubuntu noble, using method apt
    + sudo mkdir -p --mode=0755 /usr/share/keyrings
    + curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg
    + sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg
    + sudo chmod 0644 /usr/share/keyrings/tailscale-archive-keyring.gpg
    + curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list
    + sudo tee /etc/apt/sources.list.d/tailscale.list
    # Tailscale packages for ubuntu noble
    deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/ubuntu noble main
    + sudo chmod 0644 /etc/apt/sources.list.d/tailscale.list
    + sudo apt-get update
    Hit:1 http://ch.archive.ubuntu.com/ubuntu noble InRelease
    Hit:2 https://brave-browser-apt-release.s3.brave.com stable InRelease                                        
    Hit:3 https://repo.steampowered.com/steam stable InRelease                                                   
    Hit:4 http://ch.archive.ubuntu.com/ubuntu noble-updates InRelease                                            
    Hit:5 http://archive.ubuntu.com/ubuntu noble InRelease                                                       
    Hit:6 http://ch.archive.ubuntu.com/ubuntu noble-backports InRelease                                          
    Hit:7 https://packages.microsoft.com/repos/code stable InRelease                                             
    Hit:8 https://dl.google.com/linux/chrome/deb stable InRelease                                                
    Hit:9 http://archive.ubuntu.com/ubuntu noble-updates InRelease                                               
    Hit:10 https://esm.ubuntu.com/apps/ubuntu noble-apps-security InRelease                                      
    Hit:11 https://esm.ubuntu.com/apps/ubuntu noble-apps-updates InRelease                                       
    Hit:12 https://esm.ubuntu.com/infra/ubuntu noble-infra-security InRelease                        
    Hit:13 https://esm.ubuntu.com/infra/ubuntu noble-infra-updates InRelease                         
    Hit:14 http://security.ubuntu.com/ubuntu noble-security InRelease          
    Get:15 https://pkgs.tailscale.com/stable/ubuntu noble InRelease
    Get:16 https://pkgs.tailscale.com/stable/ubuntu noble/main all Packages [354 B]
    Get:17 https://pkgs.tailscale.com/stable/ubuntu noble/main i386 Packages [12.2 kB]
    Get:18 https://pkgs.tailscale.com/stable/ubuntu noble/main amd64 Packages [12.7 kB]
    Fetched 31.8 kB in 1s (23.3 kB/s)                
    Reading package lists... Done
    N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'https://brave-browser-apt-release.s3.brave.com stable InRelease' doesn't support architecture 'i386'
    + sudo apt-get install -y tailscale tailscale-archive-keyring
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following NEW packages will be installed:
      tailscale tailscale-archive-keyring
    0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
    Need to get 31.5 MB of archives.
    After this operation, 59.1 MB of additional disk space will be used.
    Get:2 https://pkgs.tailscale.com/stable/ubuntu noble/main all tailscale-archive-keyring all 1.35.181 [3,082 B]
    Get:1 https://pkgs.tailscale.com/stable/ubuntu noble/main amd64 tailscale amd64 1.82.0 [31.5 MB]       
    Fetched 31.5 MB in 14s (2,258 kB/s)                                                                          
    Selecting previously unselected package tailscale.
    (Reading database ... 491341 files and directories currently installed.)
    Preparing to unpack .../tailscale_1.82.0_amd64.deb ...
    Unpacking tailscale (1.82.0) ...
    Selecting previously unselected package tailscale-archive-keyring.
    Preparing to unpack .../tailscale-archive-keyring_1.35.181_all.deb ...
    Unpacking tailscale-archive-keyring (1.35.181) ...
    Setting up tailscale-archive-keyring (1.35.181) ...
    Setting up tailscale (1.82.0) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/tailscaled.service → /usr/lib/systemd/system/tailscaled.service.
    + [ false = true ]
    + set +x
    Installation complete! Log in to start using Tailscale by running:

    sudo tailscale up
    ```

After installing the software, running `sudo tailscale up` will provide a URL to 
authenticate and the system will show up as ready in the wizard:

``` console
$ sudo tailscale up

To authenticate, visit:

        https://login.tailscale.com/a/______________

Success.
```

Repeat the process with the second system and the wizzard will show both as active;
along with a useful test to check that all is working well:

> Every device in your Tailscale network has a private 100.x.y.z IP address
> that you can reach no matter where you are. And every protocol works —
> SSH, RDP, HTTP, Minecraft — use whatever you want while Tailscale is running.

![Successful Tailscale Quickstart](../media/2025-03-23-remote-access-options-for-self-hosted-services/tailscale-quickstart-success.png)

And indeed that *just works*; an SSH connection to `pi@100.113.110.3` instantly connects to
`alfred` and SSH key authentication just works (after accepting this new hostname).

From this point on, one can connect more devices, by repeating the above 2 steps:
install Tailscale, then authenticate it to join this tailnet. It may be a good idea to add
all desired devices before proceeding to additional configuration changes, such as

*   [Set up DNS](https://login.tailscale.com/admin/dns), to more easily connect to devices.
*   [Share a node](https://tailscale.com/kb/1084/sharing) with users on other networks.
*   [Set up access controls](https://login.tailscale.com/admin/acls) to limit which devices
    can talk to each other.

### Set up DNS

The first step when setting up DNS, although optional, should be to rename the tailnet to a
more memorable ("fun") name. There is a limited number of 4 randomly generated names to pick
from, which one can reroll many times but will never, probably by design, contain only
dictionary words. Some of the "funniest" names offered were:

*   `blenny-godzilla.ts.net`
*   `chicken-fujita.ts.net`
*   `ocelot-betelgeuse.ts.net`
*   `raptor-penny.ts.net`
*   `royal-penny.ts.net`
*   `risk-truck.ts.net`
*   `xantu-lizard.ts.net`

The tailnet name is not too important, so just pick the first one that isn't too long or
hard to spell, then try re-rolling a few times just in a case a better one shows up.

[MagicDNS](https://tailscale.com/kb/1081/magicdns) being enabled by default, it should be
possibly to ping or SSH directly to `alfred.xantu-lizard.ts.net`, etc.

### Tailscale Serve

[Tailscale Serve](https://tailscale.com/kb/1312/serve) lets you route traffic from other
devices on your tailnet to a local service running on your device. This means the local
service is made available to other devices in your tailnet, without making it available
**publicly** on the internet.

!!! note

    [Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh) seems unnecessary for a
    single admin using SSH; so long as the tailnet IP address is reachable.

### Tailscale Funnel

[Tailscale Funnel](https://tailscale.com/kb/1223/funnel) lets you route traffic from the
broader internet to a local service, like a web app, for anyone to access—even if they
don't use Tailscale. This can be used to expose non-HTML sensitive applications over 
**HTTPS**, with the caveat that traffic is not protected from abuse as with Cloudflare.
