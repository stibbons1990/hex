---
title: Self-hosting
hide:
  - footer
---

*Self-hosting is the practice of hosting and managing applications on your own server(s)*.

Source: [awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted).

## Why? *Because I Can!*

Self-Hosting is *a choice*.
It is primarily about choice and control.

Hosted services are still valuable as they provide access
to content and tools that would otherwise be unavailable.
Self-hosted services are not necessarily are replacement to
those, most of the times they are an extension or addition.

But sometimes it is *nice* to replace a hosted service with a
self-hosted alternative; I find listening to audiobooks and podcasts
*substantially* more enjoyable since I started using
[Audiobookshelf](#audiobookshelf) and eventually removed the Audible
app since all I was getting out of it was annoying push notifications.

### Not a Question of Price

Self-Hosting is a hobby, not a way to save money, or a business plan.
Like every hobby, it is practiced *for the joy of it* within the
limits allowed by restrictions impossed by the environment.

[The Cost of Self-Hosting](https://kevquirk.com/blog/the-cost-of-self-hosting)
is always a consideration, and it's not just the monetary cost; a
significant amount of time goes into learning how to get things to
work, and then later into dealing with things breaking down, because
*every thing breaks down eventually*. For now, I am still happy to
take that as an *investment* because I *enjoy* the learning process.

Last but not least,
[Self-Hosting Isn't a Solution; It's A Patch](https://matduggan.com/self-hosting-isnt-a-solution-its-a-patch/).

## Applications Installed

These are the applications that have been installed and used,
at least enough to determine whether they are a good match for
my intended purpose/s.

### Used Often

These are the applications I find myself using often, most of them
on a daily basis, otherwise at least once or twice a week.

#### Audiobookshelf

[Audiobookshelf on Kubernetes](../blog/posts/2024-02-28-audiobookshelf-on-kubernetes.md)
may be the one application I use every single day, to listen to
podcasts during the day and to audiobooks in the evening, sometimes
also offline while traveling.

#### Continuous Monitoring

[Continuous Monitoring](conmon.md) is also used very nearly on a
daily basis. It started in early 2020 as an ad-hoc implementation of
[detailed system and process monitoring](../blog/posts/2020-03-21-detailed-system-and-process-monitoring.md)
and 4 years later remained my preferred setup for
[monitoring with InfluxDB and Grafana on Kubernetes](../blog/posts/2024-04-20-monitoring-with-influxdb-and-grafana-on-kubernetes.md).

[Continuous Monitoring for TP-Link Tapo devices](../blog/posts/2024-12-28-continuous-monitoring-for-tp-link-tapo-devices.md)
made this application a *mission critical* tool that I used
*all the time*, every single day, for a few weeks every year.

#### Komga

[Self-hosted eBook library with Komga](../blog/posts/2024-05-26-self-hosted-ebook-library-with-komga.md)
has made it *substantially* easier, and thus more likely, for me
to read digital books. Not only me either, since some of the books
were really purchased for the kids, having a central library we all
can use, from any and every device, is a lot easier than sharing
files in an inevitably more disorganized fashion.

#### Navidrome

[Self-hosted music streaming with Navidrome](../blog/posts/2024-10-26-self-hosted-music-streaming-with-navidrome.md)
may not be perfect but it works quite well enough for listening to
music while working or chilling out. I like to listen to the same music
again and again anyway, I only buy albums from my favorite artists at
[Bandcamp](https://bandcamp.com/) and listen to them on an infinite loop.

#### UniFi Network Server

[Migrating UniFi Controller to Kubernetes](../blog/posts/2024-12-31-migrating-unifi-controller-to-kubernetes.md)
means no longer having to manually update the
[UniFi Network Server](https://ui.com/download/releases/network-server)
*plus* its dependencies (MongoDB and Java). The alternatives to
self-hosting are pricey, starting at $15/month or $29/month depending
on the provider.

#### Visual Studio Code Server

[Running Visual Studio Code Server on Kubernetes](../blog/posts/2023-05-29-running-vs-code-server-on-kubernetes.md)
was the first self-hosted service on the first
[single-node Kubernetes cluster on Ubuntu Server (lexicon)](../blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md).
It remains frequently used for the ability to edit Kubernetes deployment
files directly on the server, even after installing 
[Visual Studio Code on desktop PCs](../blog/posts/2024-12-27-ubuntu-studio-24-04-on-raven-gaming-pc-and-more.md#visual-studio-code)
which does work better for developing for desktop PCs.

### Used Occasionally

These are the applications I still consider *in use*, even though use
is less frequent. Most of them still get used on a weekly basis,
otherwise at least once or twice per month.

#### ActivityWatch

[Self-hosted time tracking with ActivityWatch](../blog/posts/2024-06-30-self-hosted-time-tracking-with-activitywatch.md)
is limited to the one machine where it is deployed, by design.
This has made its usefulness somewhat limited but not as much as
how hard it really is to categorize and aggregate "activities"
into groups to represent *real-life activities*.

#### Kubernetes Dashboard

[The built-in Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
was deployed as part of the 
[single-node Kubernetes cluster on Ubuntu Server (lexicon)](../blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#dashboard)
and is a nice UI to see how the cluster is doing, although when it
comes to root-causing problems for real it doesn't seem to provide
quite enough details.

#### Minecraft Server

[Running Minecraft Java Server for Bedrock clients on Kubernetes](../blog/posts/2023-08-10-running-minecraft-java-server-for-bedrock-clients-on-kubernetes.md)
is a convenient method to keep the Minecraft Java Edition server up to
date *and* make it available to multiple kids, including friends playing
remotely.

Sometimes docker images are released several days later than the original
server, which leads to a temporary version mismatch between the server
and the clients, but when the server is lagging one version behind,
(or, rarely, down) the kids will just use one of their own PCs as the
secondary server and play on that one until the primary server is fixed.

#### Plex Media Server

[Migrating a Plex Media Server to Kubernetes](../blog/posts/2023-09-16-migrating-a-plex-media-server-to-kubernetes.md)
was very convenient to let Kubernetes take care of updating the
Plex Media Server itself. However, as popular as Plex is, it is
barely used:

*   [Audiobookshelf](#audiobookshelf) has already replaced it for
    audiobooks and podcasts.
*   [Navidrome](#navidrome) has already replaced it for music.
*   [Jellyfin](#jellyfin) and [Immich](#immich) may replace it for
    video lectures and family videos respectively.

### Not Really Used

These are applications I used *for a bit* or *tried out*, but then
did not become frequently used.

#### Firefly III

[Self-hosted accountancy with Firefly III](../blog/posts/2024-05-19-self-hosted-accountancy-with-firefly-iii.md)
works well and feels agile enough to use, yet the most important
ingredient to keep using such an application is *perseverance*;
that's what I don't have.

#### Homebox

[Self-hosted inventory with Homebox](../blog/posts/2024-07-10-self-hosted-inventory-with-homebox.md)
looks promising and easy enough to use, yet again without a
good motivation to invest the *hours* to fill it in, there
is only so much you can do with it. It will probably make
more sense after establishing a criteria for *what goes in*,
because it hardly makes sense to *try and get it all in*.

### Not Really Useful

These applications turned out to be not really useful for the
intended purpose/s. This may be due to open issues (bugs) or
their design and intended behavior not matching intended purpose/s.

#### Kavita

[Kavita](https://www.kavitareader.com/) looked promising and was
installed during the process that lead to
[self-hosted eBook library with Komga](../blog/posts/2024-05-26-self-hosted-ebook-library-with-komga.md#alternatives).
What kept me from using this one long-term was that it really is
built for comic *series* and not so much for individual books.

#### PhotoPrism®

[Self-hosted photo albums with PhotoPrism®](../blog/posts/2024-08-24-self-hosted-photo-albums-with-photoprism.md)
was very promising with its *use of the latest technologies to
tag and find pictures automatically without getting in your way*,
but turned out to present one
[Big Problem](../blog/posts/2024-08-24-self-hosted-photo-albums-with-photoprism.md#big-problem)
that kept it from being really useful (and that was not the only one).

There may be solutions for those problems but, even then, the whole
navigation and UI experience was not entirely satisfactory.
The current plan is to try [Immich](#immich) next.

## Applications Considered

These applications have not been installed *yet*.

### Desired

These applications have been *briefly* evaluated and look like they
may be a good match for my intended purpose/s.

#### AdGuardHome

[AdGuardHome](https://github.com/AdguardTeam/AdGuardHome) may be a
good tool to mitigate phishing and malware attacks on the web and
possibly leverage blocking of adult domains; provided it works better
than [Pi-hole®](#pi-hole).

#### AFFiNE

[AFFiNE](https://affine.pro/) *is a workspace with fully merged docs,
whiteboards and databases*, a *privacy-focused, local-first,
open-source, and ready-to-use alternative for Notion & Miro*.

To [self-host AFFiNE](https://docs.affine.pro/docs/self-host-affine)
in a Kubernetes cluster, a deployment including AFFiNE and its
dependencies can be created from their example
[`compose.yaml`](https://github.com/toeverything/AFFiNE/blob/canary/.github/deployment/self-host/compose.yaml).

#### Authentik

[Authentik](https://goauthentik.io/) allows restricting access to a
specific set of users based on their email addresses, so that each
applications can only be accessed by their legit users and their
authentication is enforced by their respective identity providers.

In particular, it is clearly documented that the rather popular
`@gmail.com` addresses are supported by the
[Google](https://docs.goauthentik.io/docs/users-sources/sources/social-logins/google/)
identity provider (see also 
[authentik/discussions/1776](https://github.com/goauthentik/authentik/discussions/1776)).

#### Fail2Ban

[Fail2Ban](https://github.com/fail2ban/fail2ban?tab=readme-ov-file#fail2ban-ban-hosts-that-cause-multiple-authentication-errors)
*scans log files and bans IP addresses conducting too many failed login
attempts*.

This is already setup in the host OS but is limited to the
SSH service, which is has
[password authentication disabled](../blog/posts/2022-07-03-low-effort-homelab-server-with-ubuntu-server-on-intel-nuc.md#disable-ssh-password-authentication).
The next step would be to set it up to ban IPs that fail to
authenticate through [Authentik](#authentik).

#### Gatus

[Gatus](https://github.com/TwiN/gatus) is *a developer-oriented health
dashboard to monitor your services, evaluate the result of queries
based on conditions and health checks can be paired with alerting*,
e.g. via 
[Ntfy alerts](https://github.com/TwiN/gatus?tab=readme-ov-file#configuring-ntfy-alerts)
to push notifications to your phone via [ntfy](https://ntfy.sh/).

#### Gitea

[Gitea](https://about.gitea.com/products/gitea/) is a
*painless, self-hosted Git service*, although the last (first)
time I tried it, it was painfully finicky to use,
[setting up Nginx ingress with HTTPs took me a while](../blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#add-ingress-for-the-first-pod)
and I could never figure out how to use it as the (only) remote
repository when working from
[Visual Studio Code Server](#visual-studio-code-server).

#### Heimdall

[Heimdall Application Dashboard](https://heimdall.site/) is
*a dashboard for all your web applications and links to anything else*
which seems more versatile than a applications-only dashboard like
[Homepage](#homepage).

#### Homepage

[Homepage](https://gethomepage.dev/) is a modern, highly customizable
application dashboard that could be useful to have a *big picture* view
of all services in one place, should there ever be too many of them.

#### Home Assistant

[Home Assistant](https://www.home-assistant.io/) may become a necessary upgrade from
[Continuous Monitoring for TP-Link Tapo devices](../blog/posts/2024-12-28-continuous-monitoring-for-tp-link-tapo-devices.md),
especially for purposes of automating changes in around the house.

There are quite a few ways to install and run Home Assistant, such as imaging a whole
[Raspberry Pi](https://www.home-assistant.io/installation/raspberrypi)
with its Home Assistant's own distribution, which seems a bit overkill, or with
[docker-compose](https://www.home-assistant.io/installation/linux#docker-compose),
which comes closer to fitting my preferred setup of running in Kubernetes.
[abalage/home-assistant-k8s](https://github.com/abalage/home-assistant-k8s/tree/main?tab=readme-ov-file#home-assistant-k8s)
implements exactly this and would probably be my preferred method, although it might need
[a trick or two to make discovery work](https://www.reddit.com/r/homeassistant/comments/ygmcpg/anyone_running_it_in_kubernetes_and_if_yes_how/).

#### Immich

[Immich](https://immich.app/) is a *self-hosted photo and video
management solution* that should make it easy to *browse, search and
organize photos and videos with ease, without sacrificing privacy*.

#### Jellyfin

[Jellyfin](https://jellyfin.org/) *is the volunteer-built media solution
that puts you in control of your media*. It has the appeal of being open
source, unlike [Plex Media Server](#plex-media-server), but the features,
navigation and even the UI look very similar. There is not much to see
in the [live demo](https://demo.jellyfin.org/stable/web/#/home.html)
so I would need to test it thorougly to determine whether it would make
a good replacement for [Plex Media Server](#plex-media-server). 

There is hope that Jellyfin will handles private videos, such as
family videos and purchased video lectures, none of which would be
found in a public database like IMDB,
[better than others](https://forum.jellyfin.org/t-jellyfin-for-family-videos?pid=21889#pid21889).

There is no offically documented Kubernetes deployment for Jellyfin
([jellyfin/discussions/11180](https://github.com/jellyfin/jellyfin/discussions/11180))
but there is a simple enough deployment
[here](https://merox.dev/blog/kubernetes-media-server/#jellyfin-br)
and a more detailed, albeit older one
[here](https://www.debontonline.com/2021/11/kubernetes-part-16-deploy-jellyfin.html).

#### Netdata

[Netdata](https://www.netdata.cloud/) *could* replace
[Continuous Monitoring](#continuous-monitoring),
at the cost of **$90 billed yearly**
([Homelab pricing](https://www.netdata.cloud/pricing/)),
probably a significant amount of time to set it up in all hosts
*and* sending all telemetry **off-site** to Netdata Cloud, where
it can only be visualized in the **closed-source** Netdata UI.

Otherwise, the free plan is limited to *Max **5** Active Connected
Nodes* (in total), not enough to monitor all the *active* hosts in
our home network. It may be enough to monitor the *most active* hosts,
to get a sense of how much more desirable an upgrade may be.

In addition to the limitation on the number of hosts, metrics are
[aggregated past 14 days](https://learn.netdata.cloud/docs/netdata-agent/database)
so it still requires an external database for long-term storage.
[Export metrics to external time-series databases](https://learn.netdata.cloud/docs/exporting-metrics/exporting-quickstart)
supports
[InfluxDB](https://learn.netdata.cloud/docs/exporting-metrics/influxdb)
via
[Graphite](https://learn.netdata.cloud/docs/exporting-metrics/graphite)
and
[VictoriaMetrics](https://learn.netdata.cloud/docs/exporting-metrics/victoriametrics)
via
[Prometheus Remote Write](https://learn.netdata.cloud/docs/exporting-metrics/prometheus-remote-write).

That said, Netdata offers superior monitoring functionalities, with
[Top Monitoring (Netdata Functions)](https://learn.netdata.cloud/docs/top-monitoring-netdata-functions)
including *customizable*
[Applications CPU Utilization](https://www.netdata.cloud/blog/understanding-linux-cpu-consumption-load-and-pressure-for-performance-optimisation/#applications-cpu-utilization)
and
[Aggregating CPU Consumption Across Process Trees](https://www.netdata.cloud/blog/accurate-process-monitoring-with-netdata/#netdatas-appsplugin-aggregating-cpu-consumption-across-process-trees),
[better than other console based tools](https://www.netdata.cloud/blog/netdata-processes-monitoring-comparison-with-console-tools/#what-does-netdata-report)
which is what is used under the hood by
[Continuous Monitoring](#continuous-monitoring).
Also, there are hundreds of integrations, including
[HDD temperature](https://learn.netdata.cloud/docs/collecting-metrics/hardware-devices-and-sensors/hdd-temperature),
[Intel GPU](https://learn.netdata.cloud/docs/collecting-metrics/hardware-devices-and-sensors/intel-gpu),
[Linux Sensors](https://learn.netdata.cloud/docs/collecting-metrics/hardware-devices-and-sensors/linux-sensors),
[Nvidia GPU](https://learn.netdata.cloud/docs/collecting-metrics/hardware-devices-and-sensors/nvidia-gpu)
and even
[TP-Link P110](https://learn.netdata.cloud/docs/collecting-metrics/iot-devices/tp-link-p110).

#### NewsBlur

[NewsBlur](https://www.newsblur.com/) *is a personal news reader
bringing people together to talk about the world*, something I've
been missing since
[Google Reader](https://en.wikipedia.org/wiki/Google_Reader)
shut down on July 1, 2013.

#### Ollama

[Ollama](https://ollama.com/) allows you to *get up and running with
large language models*, including Llama 3.3, Phi 3, Mistral, Gemma 2,
and others. Whether this can actually be useful or fun, that is to be
determined; it should be at least some fun for
[things with object / audio detection](https://www.reddit.com/r/selfhosted/comments/1bjdbjb/comment/kvrtkgx/).

There is a Helm chart in
[otwld/ollama-helm](https://github.com/otwld/ollama-helm)
and a user-friendly self-hosted WebUI at
[open-webui/open-webui](https://github.com/open-webui/open-webui).
You can even install
[both Ollama and Open WebUI using Helm](https://github.com/open-webui/open-webui/blob/main/INSTALLATION.md#installing-both-ollama-and-open-webui-using-helm).

#### Pterodactyl®

[Pterodactyl®](https://pterodactyl.io/) is a *free, open-source game server management panel designed with security in mind*, *which runs 
all game servers in isolated Docker containers while exposing a
beautiful and intuitive UI to end users*.

There is an example `docker-compose.yml`
[here](https://technotim.live/posts/pterodactyl-game-server/).
The full list of supported games is split between
[pelican-eggs/games-standalone](https://github.com/pelican-eggs/games-standalone)
and
[pelican-eggs/games-steamcmd](https://github.com/pelican-eggs/games-steamcmd).

#### Pi-hole®

[Pi-hole®](https://pi-hole.net/) is a renowned *Network-wide Ad Blocking*
and is very simple to run. However, blocking ads is not the main concern,
but instead blocking phishing and malware domains. This requires using
[custom blocklists](https://marcelbootsman.nl/securing-my-home-network-why-i-use-pi-hole#:~:text=Customization)
manually, like
[tweedge/emerging-threats-pihole](https://github.com/tweedge/emerging-threats-pihole).

#### Ryot

[Ryot](https://github.com/IgnisDa/ryot?tab=readme-ov-file#ryot)
is *a self hosted platform for tracking various facets of your life - media, fitness, etc.* which seems to include everything that
[Yamtrack](#yamtrack) and [MediaTracker](#mediatracker) can track,
*plus* other activities outside of media; it is focused in fitness
but it could be used for other activities like studying, music
practice, workshop time, other hobbies, sleep, etc.

It supports Integration with Jellyfin, Plex, **Audiobookshelf** and
[many more](https://docs.ryot.io/importing.html),
[OpenID Connect](https://docs.ryot.io/guides/authentication.html),
sending notifications to Discord and Ntfy. However, it's not yet ready
to [track everything](https://github.com/IgnisDa/ryot/issues/73),
and without an open import format like [Yamtrack](#yamtrack),
it may more sense to invest in the latter.

#### Scrutiny

[scrutiny](https://github.com/AnalogJ/scrutiny?tab=readme-ov-file#scrutiny)
is a *WebUI for smartd S.M.A.R.T monitoring* that includes a
[collector](https://github.com/AnalogJ/scrutiny/blob/master/example.collector.yaml)
that can run on a 
[Hub & Spoke model, with multiple Hosts](https://github.com/AnalogJ/scrutiny?tab=readme-ov-file#hubspoke-deployment).
The *Hub* host needs to run all 3 images, as illustrated in the example
[`docker-compose.yml`](https://github.com/AnalogJ/scrutiny/blob/master/docker/example.hubspoke.docker-compose.yml).

#### Taiga

[Taiga](https://taiga.io/) *free and open-source project management tool*
could be useful keep track of projects, tasks and their depedencies,
although it is not yet clear whether a Kanban dashboard is what would
help organizing hobby projects.

#### UniFi Poller

[UniFi Poller](https://unpoller.com/) *allows you to collect data from
your UniFi network controller, save it to a database, and then display
it on pre-supplied attractive and data-rich Grafana dashboards* and
you can also *re-use existing database or Grafana installations*.

#### VictoriaMetrics

Migrating [Continuous Monitoring](#continuous-monitoring),
[from InfluxDB 1.x to InfluxDB 2.7](https://docs.influxdata.com/influxdb/v2/install/upgrade/v1-to-v2/)
[may be too much trouble](https://www.reddit.com/r/influxdb/comments/mq08og/upgrading_influxdb_from_v1x_to_v2x_is_it_even/),
[it may turn out to be easier](https://www.reddit.com/r/grafana/comments/151wpsn/anyone_migrated_from_influxdb_18x_to_some_newer/)
[to replace InfluxDB with VictoriaMetrics](https://community.home-assistant.io/t/influxdb-vs-victoriametrics/453361/1).
[VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics?tab=readme-ov-file#victoriametrics)
*is a fast, cost-saving, and scalable solution for monitoring and
managing time series data*.

#### Yamtrack

[Yamtrack](https://github.com/FuzzyGrim/Yamtrack?tab=readme-ov-file#yamtrack)
*is a self hosted media tracker for movies, tv shows, anime and manga*
but, more interestingly to me, also books and video games. It even has
*integration with Jellyfin, to automatically track new media watched*,
which is nice, but what I'd really love to see is integration with
[Audiobookshelf](#audiobookshelf) and [Komga](#komga), for books,
and [Steam](https://store.steampowered.com/) for games.

There is perhaps enough information available about the
[Yamtrack CSV import format](https://github.com/FuzzyGrim/Yamtrack/issues/246)
that it may be not too hard to hack something together to import
listening history and play time history using the available APIs:

*   [Audiobookshelf API](https://api.audiobookshelf.org/#sessions)
    exposes listening sessions and libraries.
*   [Komga REST API](https://komga.org/docs/api/rest)
    exposes progression and metadata for each book.
*   [Steam Web API](https://developer.valvesoftware.com/wiki/Steam_Web_API)
    exposes recent (2-weeks) and total (*forever*) play time,
    and the list of owned games. These can be used to track
    progress and purchases of games.

### Discarded

These applications were evaluated based on their documentation and/or
live demos, and deemed not a good match for my intended purpose/s.

#### MediaTracker

[MediaTracker](https://github.com/bonukai/MediaTracker?tab=readme-ov-file#mediatracker--------)
is a *self hosted platform for tracking movies, tv shows, video games,
books* **and audiobooks**, which *would* make it more interesting
than [Yamtrack](#yamtrack) *if only* it would allowe you to
[add media manually](https://github.com/bonukai/MediaTracker/issues/652).

#### Nginx Proxy Manager

[Nginx Proxy Manager](https://nginxproxymanager.com/) would be nice
to have a GUI, but the current setup with
[ingress Nginx](../blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#ingress-controller).
paired with
[ACME cert manager](../blog/posts/2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md#https-with-lets-encrypt)
deployment already provides the same functionality.

IP restrictions and other advance settings can be deployed by making
use of `nginx.ingress.kubernetes.io` annotation snippets.

#### Outline

[Outline](https://www.getoutline.com/) is
*a blazing fast editor with markdown support*.
Discarded in favor of
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/)
[hosted on GitHub Pages](../blog/posts/2024-12-03-starting-a-blog-with-mkdocs-material-on-github-pages.md)
because it looks like a better fit teams rather than one individual.
Previously,
[Jekyll on GitHub pages](../blog/posts/2023-09-21-starting-a-blog-with-jekyll-on-github-pages.md)
filled the same role.

It may still be an interesting learning exercise, to create a Kubernetes
deployment based on their recommended method to self-host with
[Docker Compose](https://docs.getoutline.com/s/hosting/doc/docker-7pfeLP5a8t).
