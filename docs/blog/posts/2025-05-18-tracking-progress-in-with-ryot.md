---
date: 2025-05-18
categories:
 - kubernetes
 - self-hosted
 - time-tracking
 - media-tracking
title: Tracking progress with Ryot
---

Sometimes I wish for a centralized, automatically updated and moderately fancy-looking
application to keep track of multiple activities; mostly around digital media.

*   [Audiobookshelf](../../projects/self-hosting.md#audiobookshelf) is pretty good but
    separates podcasts from books and only shows yearly summary at the end of the year.
    Audible does not offer even that, and no export options.
*   [Jellyfin](../../projects/self-hosting.md#jellyfin) (and previously
    [Plex](../../projects/self-hosting.md#plex-media-server)) don't go beyond marking
    things as "done".    Besides, movies and TV shows are not the kind of videos
    I'm intersted in tracking progress with; video lectures are
    (*where was I with this Inkscape course?*).
*   Paper books are very nearly not even a thing anymore, but it would still be nice
    to be able to track progress on them, as well as reading e-Books in
    [Komga](../../projects/self-hosting.md#komga).
*   Video games are absurdly difficult to track progress for. Naturally grown from need,
    a spreadsheet is works well enough to collect data across multiple platforms, but
    it is limited, ugly and increasing slow as the library grows.
    *   [Steam](https://store.steampowered.com/) shows only total and recent (last
        2 weeks) gameplay, and probress is tracked in terms of achievements, not how
        close you are to finish the main story. At least there is the option to query the
        [Steam Web API](https://steamcommunity.com/dev) to periodically fetch gameplay
        stats, so they can be kept at a higher resolution (daily, hourly, etc.).
    *   [Nintendo Switch Parental Control](https://play.google.com/store/apps/details?id=com.nintendo.znma) (Android app).
        shows only gameplay time per game (and per user) in the current month, after that
        it shows only montly summaries. There is no option to export any of this.
    *   [GOG](https://www.gog.com/) requires installing their own (Windows-only) 
        Galaxy 2.0 client and the possiblity of *exporting or even seeing* your personal
        gameplay stats appears to be *not even a question*.

Looking around for tracking applications in the awesome directory of
[awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted), two
applications look promising and worth a try: [Ryot](#ryot) and [Yamtrack](#yamtrack).

<!-- more -->

## Ryot

[Ryot](https://github.com/IgnisDa/ryot?tab=readme-ov-file#ryot)
is *a self hosted platform for tracking various facets of your life - media, fitness, etc.* which seems to include everything that [Yamtrack](#yamtrack) can track, *plus*
other activities outside of media; it is focused in fitness but it could be used for
other activities like studying, music practice, workshop time, other hobbies, sleep, etc.

It supports Integration with Jellyfin, Plex, **Audiobookshelf** and
[many more](https://docs.ryot.io/importing.html),
[OpenID Connect](https://docs.ryot.io/guides/authentication.html),
sending notifications to Discord and Ntfy. While it's not yet ready to
*easily* [track everything](https://github.com/IgnisDa/ryot/issues/73),
it [recently added](https://github.com/IgnisDa/ryot/issues/994) a
*webhook endpoint that supports generic json* which can be used to
*build integrations with other services that Ryot does not support natively*
by importing [Generic Json](https://docs.ryot.io/integrations.html#generic-json)
generated from any non-supported systems (e.g. Steam API).

### Ryot deployment

Ryot documentation starts off with an [Installation](https://docs.ryot.io/) based on
`docker-compose`, on which the following basic deployment is based:

??? k8s "Basic Ryot deployment: `ryot.yaml`"

    ``` yaml title="ryot.yaml"
    apiVersion: v1
    kind: Namespace
    metadata:
      name: ryot
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: postgres-pv
      labels:
        type: local
      namespace: ryot
    spec:
      storageClassName: manual
      capacity:
        storage: 3Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /home/k8s/ryot/postgres
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: postgres-pv-claim
      namespace: ryot
    spec:
      storageClassName: manual
      volumeName: postgres-pv
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: ryot
      labels:
        app: postgres
      name: postgres
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: postgres
      template:
        metadata:
          labels:
            app: postgres
        spec:
          hostname: postgres
          containers:
          - image: postgres:16-alpine
            env:
            - name: "POSTGRES_PASSWORD"
              value: "____________________"
            name: postgres
            volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-data
          volumes:
          - name: postgres-data
            persistentVolumeClaim:
              claimName: postgres-pv-claim
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: postgres
      name: postgres-svc
      namespace: ryot
    spec:
      ports:
      - port: 5432
        protocol: TCP
        targetPort: 5432
        nodePort: 30543
      selector:
        app: postgres
      type: NodePort
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: ryot
      labels:
        app: ryot
      name: ryot
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ryot
      template:
        metadata:
          labels:
            app: ryot
        spec:
          containers:
          - image: ghcr.io/ignisda/ryot:v8
            env:
            - name: "TZ"
              value: "Europe/Amsterdam"
            - name: "SERVER_ADMIN_ACCESS_TOKEN"
              value: "___________________"
            - name: "DATABASE_URL"
              value: "postgres://postgres:___________________@postgres-svc:5432/postgres"
            name: ryot
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: ryot
      name: ryot-svc
      namespace: ryot
    spec:
      ports:
      - port: 8000
        protocol: TCP
        targetPort: 8000
        nodePort: 30380
      selector:
        app: ryot
      type: NodePort
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ryot-ingress
      namespace: ryot
      annotations:
        acme.cert-manager.io/http01-edit-in-place: "true"
        cert-manager.io/issue-temporary-certificate: "true"
        cert-manager.io/cluster-issuer: letsencrypt-prod
    spec:
      ingressClassName: nginx
      rules:
        - host: ryot.very-very-dark-gray.top
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: ryot-svc
                    port:
                      number: 8000
      tls:
        - secretName: tls-secret-cloudflare
          hosts:
            - ryot.very-very-dark-gray.top
    ```

Before deploying the application, create a dedicated system user to own the local
storage for the Postgres database:

``` console
root@octavo ~ # groupadd ryot -g 120
root@octavo ~ # useradd  ryot -u 120 -g 120 -s /usr/sbin/nologin
root@octavo ~ # mkdir -p /home/k8s/ryot/postgres
root@octavo ~ # chown -R ryot:ryot /home/k8s/ryot
root@octavo ~ # ls -hal /home/k8s/ryot
total 0
drwxr-xr-x 1 ryot ryot  16 May 17 14:13 .
drwxr-xr-x 1 root root 330 May 17 14:13 ..
drwxr-xr-x 1 ryot ryot   0 May 17 14:13 postgres
```

Then deploy the application:

``` console
$ kubectl apply -f ryot.yaml
namespace/ryot created
persistentvolume/postgres-pv created
persistentvolumeclaim/postgres-pv-claim created
deployment.apps/postgres created
service/postgres-svc created
deployment.apps/ryot created
service/ryot-svc created
ingress.networking.k8s.io/ryot-ingress created

$ kubectl get all -n ryot
NAME                            READY   STATUS    RESTARTS   AGE
pod/postgres-5f649d875c-82fh4   1/1     Running   0          3s
pod/ryot-55c5845667-gplxd       1/1     Running   0          3s

NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/postgres-svc   NodePort   10.96.122.54   <none>        5432:30543/TCP   3s
service/ryot-svc       NodePort   10.98.120.72   <none>        8000:30380/TCP   3s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/postgres   1/1     1            1           3s
deployment.apps/ryot       1/1     1            1           3s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/postgres-5f649d875c   1         1         1       3s
replicaset.apps/ryot-55c5845667       1         1         1       3s

$ kubectl get ingress -n ryot
NAME           CLASS   HOSTS                          ADDRESS         PORTS   AGE
ryot-ingress   nginx   ryot.very-very-dark-gray.top   192.168.0.171   80      59s
```

After a couple of minutes the application will be available at
<https://ryot.very-very-dark-gray.top/>

!!! warning

    By default, **Ryot allows anyone to create a new account**. Gating access behind
    [Cloudflare Access](./2025-03-23-remote-access-options-for-self-hosted-services.md#cloudflare-access) is advisable, at least initially, to prevent strangers from
    creating accounts if they *accidentally* find the application.

### Ryot setup

First, create a user to gain admin access to the application. Additional users can be
created but will not have the *Admin* role, only the first user can add it.

#### Import

The first step each user should probaly do is to
[import](https://docs.ryot.io/importing/overview.html)
their progress so far in the supported systems. This is done under
**Settings > Imports and Exports > Import data** using the relevant importers.

##### Audiobookshelf

The [Audiobookshelf importer](https://docs.ryot.io/importing/audiobookshelf.html)
supports importing all media that have a valid Audible ID or iTunes ID or ISBN.
Unfortunately this means those audiobooks not available in Audible or iTunes,
e.g. those from [LibriVox](https://librivox.org/), won't be imported.

This will only import media that are already finished, neither books in progress nor 
those that have not been started yet. Those will be added later, as progress is made,
via the [Audiobookshelf integration](#audiobookshelf_1).


- *Select a source*: **Audiobookshelf**
- *Instance URL*: <https://audiobookshelf.very-very-dark-gray.top> or whatever the
  ingress is for [Audiobookshelf](./2025-04-12-kubernetes-homelab-server-with-ubuntu-server-24-04-octavo.md#audiobookshelf)
- *API Key*: the API token for the user, as described in the Audiobookshelf
  [authentication](https://api.audiobookshelf.org/#authentication) docs

The import will take a little while; in my case it took about 7 minutes to import
212 books.

![](../media/2025-05-18-tracking-progress-in-with-ryot/ryot-audiobooks-imported.png)

##### Jellyfin

The [Jellyfin importer](https://docs.ryot.io/importing/jellyfin.html)
can import watched movies and shows from Jellyfin and it seems very simple to setup:

- *Select a source*: **Jellyfin**
- *Instance URL*: <https://jellyfin.very-very-dark-gray.top/> or whatever the
  ingress is for [Jellyfin](./2025-04-29-jellyfin-on-kubernetes-with-intel-gpu.md)
- *Username*: the user to import progress for
- *Password*: their password

However, the import always fails immediately and seems to stay in the *still running*
state, so the UI won't allow even looking at the logs. Inspecing the pod's logs in
Kubernetes doesn't add much:

``` console
$ klogs ryot ryot
[frontend] POST /settings/imports-and-exports.data?intent=deployImport 200 - - 14.458 ms
[frontend] GET /settings/imports-and-exports.data 200 - - 36.602 ms
[backend] 
[backend] thread 'main' panicked at /home/runner/work/ryot/ryot/crates/utils/external/src/lib.rs:93:14:
[backend] called `Result::unwrap()` on an `Err` value: reqwest::Error { kind: Decode, source: Error("EOF while parsing a value", line: 1, column: 0) }
```

??? note "Deployed with debug logs and full backtrace, still found nothing."

    To enable debug log with full backtrace, re-deploy with these environment variables:

    ``` yaml linenums="101" hl_lines="5-8"
        spec:
          containers:
          - image: ghcr.io/ignisda/ryot:v8
            env:
            - name: "RUST_BACKTRACE"
              value: "full"
            - name: "RUST_LOG"
              value: "ryot=debug"
    ```

    Even with this, the full backtrace only shows the panic happened during
    user authentication:

    ``` log hl_lines="4 20"
    [backend] 2025-06-20T19:33:31.890311Z DEBUG ryot: Deployed import job
    [backend] 2025-06-20T19:33:31.897664Z DEBUG ryot: Started import job with id imp_LvSHrV0UsANo
    [frontend] POST /settings/imports-and-exports.data?intent=deployImport 200 - - 23.617 ms
    [backend] 2025-06-20T19:33:31.902374Z DEBUG ryot: Authentication request: RequestBuilder { method: POST, url: Url { scheme: "https", cannot_be_a_base: false, username: "", password: None, host: Some(Domain("jellyfin.very-very-dark-gray.top")), port: None, path: "//Users/AuthenticateByName", query: None, fragment: None }, headers: {"authorization": "MediaBrowser , Client=\"other\", Device=\"script\", DeviceId=\"script\", Version=\"0.0.0\"", "content-type": "application/json"} }
    [backend] 
    [backend] thread 'main' panicked at /home/runner/work/ryot/ryot/crates/utils/external/src/lib.rs:93:14:
    [backend] called `Result::unwrap()` on an `Err` value: reqwest::Error { kind: Decode, source: Error("EOF while parsing a value", line: 1, column: 0) }
    [backend] stack backtrace:
    [backend]    0:     0x57740e0313b9 - <std::sys::backtrace::BacktraceLock::print::DisplayBacktrace as core::fmt::Display>::fmt::he089f96442833f67
    [backend]    1:     0x57740d020783 - core::fmt::write::h2f210ed4c94745cb
    [backend]    2:     0x57740e030c42 - std::io::Write::write_fmt::h7de08171ab770fb2
    [backend]    3:     0x57740e031213 - std::sys::backtrace::BacktraceLock::print::h810fbd31421329e6
    [backend]    4:     0x57740e030a66 - std::panicking::rust_panic_with_hook::ha9131beeb2ddc506
    [backend]    5:     0x57740e06c3a8 - std::panicking::begin_panic_handler::{{closure}}::h1bba0eaeb6da506f
    [backend]    6:     0x57740e06c309 - std::sys::backtrace::__rust_end_short_backtrace::h1d1ca3eade483f4c
    [backend]    7:     0x57740e06c8ec - rust_begin_unwind
    [backend]    8:     0x57740b75047f - core::panicking::panic_fmt::h896a0727a1a943f9
    [backend]    9:     0x57740b7507b5 - core::result::unwrap_failed::h1b5ed8541c7bebd6
    [backend]   10:     0x57740c8ee80b - external_utils::jellyfin::get_authenticated_client::{{closure}}::h7ebb0283c0bf16fb
    [backend]   11:     0x57740c8dabee - importer_service::jellyfin::import::{{closure}}::h049974b7245d3458
    [backend]   12:     0x57740c787c7f - importer_service::ImporterService::perform_import::{{closure}}::h18f02c43074026d7
    [backend]   13:     0x57740c76fdfd - backend::job::perform_mp_application_job::{{closure}}::h03b73e257a7cc114
    [backend]   14:     0x57740cd2ea24 - <apalis_core::worker::call_all::CallAllUnordered<Svc,S> as futures_core::stream::Stream>::poll_next::h797c80ebe682728c
    [backend]   15:     0x57740cb920fa - <futures_util::stream::stream::map::Map<St,F> as futures_core::stream::Stream>::poll_next::h6ef911d9c3578db2
    [backend]   16:     0x57740bb77a9d - <futures_util::future::select::Select<A,B> as core::future::future::Future>::poll::h99325282f343d4f9
    [backend]   17:     0x57740bb7912e - <apalis_core::worker::Runnable as core::future::future::Future>::poll::h6b1a144e4fe197f8
    [backend]   18:     0x57740c40f365 - <core::future::poll_fn::PollFn<F> as core::future::future::Future>::poll::hf686955acb0d887a
    [backend]   19:     0x57740cb6dfc4 - backend::main::{{closure}}::hc65b9eef1642855f.51389
    [backend]   20:     0x57740c43858b - backend::main::he4a8daadc1a11119
    [backend]   21:     0x57740c9a4a43 - std::sys::backtrace::__rust_begin_short_backtrace::hf7af6e2575e20db0
    [backend]   22:     0x57740c437b61 - main
    [backend]   23:     0x7ba286c4424a - <unknown>
    [backend]   24:     0x7ba286c44305 - __libc_start_main
    [backend]   25:     0x57740b7826de - _start
    [backend]   26:                0x0 - <unknown>
    [frontend] GET /settings/imports-and-exports.data 200 - - 42.700 ms
    ```

    Kubernetes logs for the Jellyfin pod don't show any activity at all when the import
    is started, suggesting authentication is somehow not being even *attempted*.

##### Plex

The [Plex importer](https://docs.ryot.io/importing/plex.html) can import watched
movies and shows from [Plex](./2023-09-16-migrating-a-plex-media-server-to-kubernetes.md)
but it may actually require having an active self-hosted Plex server, which is not
running in the new server ([octavo](./2025-04-12-kubernetes-homelab-server-with-ubuntu-server-24-04-octavo.md)).

No longer having my own self-hosted Plex Media Server running, it seems the process to
*obtain a `Plex-Token`* as described
[here](https://www.plexopedia.com/plex-media-server/general/plex-token/#getcurrentusertoken)
no longer works, and at this point I'm past *done* with Plex anyway.

#### Integrations

##### Audiobookshelf

The [Audiobookshelf integration](#audiobookshelf_1) must be setup to keep syncing
progress made on audiobooks. The setup is essentially the same as for the
[Audiobookshelf importer](#audiobookshelf):

- *Instance URL*: <https://audiobookshelf.very-very-dark-gray.top> or whatever the
  ingress is for [Audiobookshelf](./2025-04-12-kubernetes-homelab-server-with-ubuntu-server-24-04-octavo.md#audiobookshelf)
- *API Key*: the API token for the user, as described in the Audiobookshelf
  [authentication](https://api.audiobookshelf.org/#authentication) docs

##### Komga

There is no [importer](#import) option for
[Komga](./2024-05-26-self-hosted-ebook-library-with-komga.md) so instead the
[Komga integration](https://docs.ryot.io/integrations.html#komga) can be setup with

- *Base URL*: <https://jellyfin.very-very-dark-gray.top/> or whatever the
  ingress is for [Jellyfin](./2025-04-29-jellyfin-on-kubernetes-with-intel-gpu.md)
- *Username*: the user to import progress for
- *Password*: their password
- *Provider*: Anlist

Unfortunately this integration does not seem to do anything visible. The integration
shows (under **Settings > Integrations**) as working correctly, logs show no errors,
but activity in Komga is not reflected anywhere in Ryot.

#### Generic JSON

For everything else, "Generic Json" can be used to
[import](https://docs.ryot.io/importing/generic-json.html) and
[sync progress](https://docs.ryot.io/integrations.html#generic-json) from any
other sources that Ryot. The format of the JSON file should be `CompleteExport`
as described in the documentation for
[exporting](https://docs.ryot.io/guides/exporting.html#type-definitions).

This should made it possible to import and sync video games, at least from Steam.
However, having to implement this from scratch is not particularly appealing for now.

### Ryot impressions

Without spending many hours on it, all I seem to get out of Ryot is but a limited
subset of what Audiobookshelf shows. While the app looks promising and pleasant to
navigate, the funcionality achieved is too limited to justify regular use.

## Yamtrack

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

This may be worth the investment *if* [Ryot](#ryot) turns out not to be the best option
in the end. For now, lets just say Yamtrack is *even easier* to deploy, based on its own
[`docker-compose.yml`](https://github.com/FuzzyGrim/Yamtrack/blob/dev/docker-compose.yml)

??? k8s "Basic Jellyfin deployment: `yamtrack.yaml`"

    ``` yaml title="yamtrack.yaml"
    apiVersion: v1
    kind: Namespace
    metadata:
      name: yamtrack
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: redis-pv
      labels:
        type: local
      namespace: yamtrack
    spec:
      storageClassName: manual
      capacity:
        storage: 3Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /home/k8s/yamtrack/redis
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: redis-pv-claim
      namespace: yamtrack
    spec:
      storageClassName: manual
      volumeName: redis-pv
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: yamtrack
      labels:
        app: redis
      name: redis
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: redis
      template:
        metadata:
          labels:
            app: redis
        spec:
          hostname: redis
          containers:
          - image: redis:7-alpine
            name: redis
            volumeMounts:
            - mountPath: /data
              name: redis-data
          volumes:
          - name: redis-data
            persistentVolumeClaim:
              claimName: redis-pv-claim
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: redis
      name: redis-svc
      namespace: yamtrack
    spec:
      ports:
      - port: 6379
        protocol: TCP
        targetPort: 6379
        nodePort: 30379
      selector:
        app: redis
      type: NodePort
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: yamtrack-pv
      labels:
        type: local
      namespace: yamtrack
    spec:
      storageClassName: manual
      capacity:
        storage: 3Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /home/k8s/yamtrack/db
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: yamtrack-pv-claim
      namespace: yamtrack
    spec:
      storageClassName: manual
      volumeName: yamtrack-pv
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 3Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      namespace: yamtrack
      labels:
        app: yamtrack
      name: yamtrack
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: yamtrack
      template:
        metadata:
          labels:
            app: yamtrack
        spec:
          containers:
          - image: ghcr.io/fuzzygrim/yamtrack
            env:
            - name: "SECRET"
              value: "____________________"
            - name: "REDIS_URL"
              value: "redis://redis-svc:6379"
            name: yamtrack
            volumeMounts:
              - name: yamtrack-data
                mountPath: /yamtrack/db
          volumes:
          - name: yamtrack-data
            persistentVolumeClaim:
              claimName: yamtrack-pv-claim
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: yamtrack
      name: yamtrack-svc
      namespace: yamtrack
    spec:
      ports:
      - port: 8000
        protocol: TCP
        targetPort: 8000
        nodePort: 30380
      selector:
        app: yamtrack
      type: NodePort
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: yamtrack-ingress
      namespace: yamtrack
      annotations:
        acme.cert-manager.io/http01-edit-in-place: "true"
        cert-manager.io/issue-temporary-certificate: "true"
        cert-manager.io/cluster-issuer: letsencrypt-prod
    spec:
      ingressClassName: nginx
      rules:
        - host: yamtrack.very-very-dark-gray.top
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: yamtrack-svc
                    port:
                      number: 8000
      tls:
        - secretName: tls-secret-cloudflare
          hosts:
            - yamtrack.very-very-dark-gray.top
    ```

Prepare a local user and directory in the similar to the one for Ryot:

``` console
root@octavo ~ # groupadd yamtrack -g 120
root@octavo ~ # useradd  yamtrack -u 120 -g 120 -s /usr/sbin/nologin
root@octavo ~ # mkdir /home/k8s/yamtrack/redis /home/k8s/yamtrack/db
root@octavo ~ # chown -R yamtrack:yamtrack /home/k8s/yamtrack
root@octavo ~ # ls -hal /home/k8s/yamtrack
total 0
drwxr-xr-x 1 yamtrack yamtrack  14 May 15 23:06 .
drwxr-xr-x 1 root     root     322 May 15 23:01 ..
drwxr-xr-x 1 yamtrack yamtrack   0 May 15 23:06 db
drwxr-xr-x 1 yamtrack yamtrack   0 May 15 23:06 redis
```

Then deploy the application:

``` console
$ kubectl apply -f yamtrack.yaml 
namespace/yamtrack created
persistentvolume/redis-pv created
persistentvolumeclaim/redis-pv-claim created
deployment.apps/redis created
service/redis-svc created
persistentvolume/yamtrack-pv created
persistentvolumeclaim/yamtrack-pv-claim created
deployment.apps/yamtrack created
service/yamtrack-svc created
ingress.networking.k8s.io/yamtrack-ingress created

$ kubectl get all -n yamtrack
NAME                            READY   STATUS    RESTARTS   AGE
pod/redis-549b9c9b6f-2xq94      1/1     Running   0          98s
pod/yamtrack-668755cf84-xbt9v   1/1     Running   0          97s

NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/redis-svc      NodePort   10.96.239.219   <none>        6379:30379/TCP   98s
service/yamtrack-svc   NodePort   10.111.29.62    <none>        8000:30380/TCP   98s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/redis      1/1     1            1           98s
deployment.apps/yamtrack   1/1     1            1           98s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/redis-549b9c9b6f      1         1         1       98s
replicaset.apps/yamtrack-668755cf84   1         1         1       98s

$ kubectl get ingress -n yamtrack
NAME               CLASS   HOSTS                              ADDRESS         PORTS   AGE
yamtrack-ingress   nginx   yamtrack.very-very-dark-gray.top   192.168.0.171   80      36s
```

After a couple of minutes the application will be available at
<https://yamtrack.very-very-dark-gray.top>; otherwise inspect the logs to debug.

??? console "`$ klogs yamtrack yamtrack`"

    ``` console
    $ klogs yamtrack yamtrack
    Operations to perform:
      Apply all migrations: account, admin, app, auth, contenttypes, db, django_celery_beat, django_celery_results, events, lists, mfa, sessions, socialaccount, users
    Running migrations:
      Applying contenttypes.0001_initial... OK
      Applying contenttypes.0002_remove_content_type_name... OK
      Applying auth.0001_initial... OK
      Applying auth.0002_alter_permission_name_max_length... OK
      Applying auth.0003_alter_user_email_max_length... OK
      Applying auth.0004_alter_user_username_opts... OK
      Applying auth.0005_alter_user_last_login_null... OK
      Applying auth.0006_require_contenttypes_0002... OK
      Applying auth.0007_alter_validators_add_error_messages... OK
      Applying auth.0008_alter_user_username_max_length... OK
      Applying auth.0009_alter_user_last_name_max_length... OK
      Applying auth.0010_alter_group_name_max_length... OK
      Applying auth.0011_update_proxy_permissions... OK
      Applying auth.0012_alter_user_first_name_max_length... OK
      Applying users.0001_squashed_0008_user_game_layout_alter_user_last_search_type... OK
      Applying account.0001_initial... OK
      Applying account.0002_email_max_length... OK
      Applying account.0003_alter_emailaddress_create_unique_verified_email... OK
      Applying account.0004_alter_emailaddress_drop_unique_email... OK
      Applying account.0005_emailaddress_idx_upper_email... OK
      Applying account.0006_emailaddress_lower... OK
      Applying account.0007_emailaddress_idx_email... OK
      Applying account.0008_emailaddress_unique_primary_email_fixup... OK
      Applying account.0009_emailaddress_unique_primary_email... OK
      Applying admin.0001_initial... OK
      Applying admin.0002_logentry_remove_auto_add... OK
      Applying admin.0003_logentry_add_action_flag_choices... OK
      Applying app.0001_squashed_0014_historicalanime_historicalepisode_historicalgame_and_more... OK
      Applying app.0015_customlist_listitem_squashed_0021_alter_item_media_type... OK
      Applying app.0022_item_source_squashed_0037_alter_item_media_type... OK
      Applying app.0038_comic_historicalcomic_and_more... OK
      Applying app.0039_anime_progress_changed_basicmedia_progress_changed_and_more... OK
      Applying db.0001_initial... OK
      Applying django_celery_beat.0001_initial... OK
      Applying django_celery_beat.0002_auto_20161118_0346... OK
      Applying django_celery_beat.0003_auto_20161209_0049... OK
      Applying django_celery_beat.0004_auto_20170221_0000... OK
      Applying django_celery_beat.0005_add_solarschedule_events_choices... OK
      Applying django_celery_beat.0006_auto_20180322_0932... OK
      Applying django_celery_beat.0007_auto_20180521_0826... OK
      Applying django_celery_beat.0008_auto_20180914_1922... OK
      Applying django_celery_beat.0006_auto_20180210_1226... OK
      Applying django_celery_beat.0006_periodictask_priority... OK
      Applying django_celery_beat.0009_periodictask_headers... OK
      Applying django_celery_beat.0010_auto_20190429_0326... OK
      Applying django_celery_beat.0011_auto_20190508_0153... OK
      Applying django_celery_beat.0012_periodictask_expire_seconds... OK
      Applying django_celery_beat.0013_auto_20200609_0727... OK
      Applying django_celery_beat.0014_remove_clockedschedule_enabled... OK
      Applying django_celery_beat.0015_edit_solarschedule_events_choices... OK
      Applying django_celery_beat.0016_alter_crontabschedule_timezone... OK
      Applying django_celery_beat.0017_alter_crontabschedule_month_of_year... OK
      Applying django_celery_beat.0018_improve_crontab_helptext... OK
      Applying django_celery_beat.0019_alter_periodictasks_options... OK
      Applying django_celery_results.0001_initial... OK
      Applying django_celery_results.0002_add_task_name_args_kwargs... OK
      Applying django_celery_results.0003_auto_20181106_1101... OK
      Applying django_celery_results.0004_auto_20190516_0412... OK
      Applying django_celery_results.0005_taskresult_worker... OK
      Applying django_celery_results.0006_taskresult_date_created... OK
      Applying django_celery_results.0007_remove_taskresult_hidden... OK
      Applying django_celery_results.0008_chordcounter... OK
      Applying django_celery_results.0009_groupresult... OK
      Applying django_celery_results.0010_remove_duplicate_indices... OK
      Applying django_celery_results.0011_taskresult_periodic_task_name... OK
      Applying django_celery_results.0012_taskresult_date_started... OK
      Applying django_celery_results.0013_taskresult_django_cele_periodi_1993cf_idx... OK
      Applying django_celery_results.0014_alter_taskresult_status... OK
      Applying events.0001_initial... OK
      Applying events.0002_alter_event_unique_together... OK
      Applying events.0003_alter_event_unique_together_and_more... OK
      Applying events.0004_fix_anime_episode_numbers... OK
      Applying events.0005_alter_event_date... OK
      Applying events.0006_alter_event_options_rename_date_event_datetime... OK
      Applying events.0007_event_notification_sent... OK
      Applying events.0008_delete_tv_events... OK
      Applying events.0009_fix_movie_episode_number... OK
      Applying events.0010_alter_event_options... OK
      Applying events.0011_remove_event_unique_item_episode_and_more... OK
      Applying events.0012_remove_event_unique_item_episode_and_more... OK
      Applying events.0013_delete_single_anime_events... OK
      Applying lists.0001_initial... OK
      Applying lists.0002_alter_customlistitem_item_alter_customlist_items_and_more... OK
      Applying lists.0003_alter_customlist_unique_together_and_more... OK
      Applying mfa.0001_initial... OK
      Applying mfa.0002_authenticator_timestamps... OK
      Applying mfa.0003_authenticator_type_uniq... OK
      Applying sessions.0001_initial... OK
      Applying socialaccount.0001_initial... OK
      Applying socialaccount.0002_token_max_lengths... OK
      Applying socialaccount.0003_extra_data_default_dict... OK
      Applying socialaccount.0004_app_provider_id_settings... OK
      Applying socialaccount.0005_socialtoken_nullable_app... OK
      Applying socialaccount.0006_alter_socialaccount_extra_data... OK
      Applying users.0009_alter_user_options_squashed_0022_alter_user_last_search_type... OK
      Applying users.0023_user_home_sort_user_home_sort_valid... OK
      Applying users.0024_remove_user_lists_sort_valid_alter_user_lists_sort_and_more... OK
      Applying users.0025_remove_user_last_search_type_valid_and_more... OK
      Applying users.0026_user_anime_sort_user_book_sort_user_game_sort_and_more... OK
      Applying users.0027_user_list_detail_sort_user_list_detail_sort_valid... OK
      Applying users.0028_user_anime_status_user_book_status_user_game_status_and_more... OK
      Applying users.0029_user_notification_urls... OK
      Applying users.0030_user_notification_excluded_items... OK
      Applying users.0031_remove_user_last_search_type_valid_and_more... OK
      Applying users.0032_alter_user_token_and_generate_missing_user_tokens... OK
      Applying users.0033_user_daily_digest_enabled_and_more... OK
    usermod: no changes
    2025-05-15 21:13:58,156 INFO Set uid to user 0 succeeded
    2025-05-15 21:13:58,168 INFO supervisord started with pid 1
    2025-05-15 21:13:59,170 INFO spawned: 'nginx' with pid 16
    2025-05-15 21:13:59,172 INFO spawned: 'gunicorn' with pid 17
    2025-05-15 21:13:59,175 INFO spawned: 'celery' with pid 18
    2025-05-15 21:13:59,176 INFO spawned: 'celery-beat' with pid 19
    celery beat v5.5.2 (immunity) is starting.
    [2025-05-15 21:13:59 +0000] [17] [INFO] Starting gunicorn 23.0.0
    [2025-05-15 21:13:59 +0000] [17] [INFO] Listening at: http://127.0.0.1:8001 (17)
    [2025-05-15 21:13:59 +0000] [17] [INFO] Using worker: sync
    [2025-05-15 21:13:59 +0000] [37] [INFO] Booting worker with pid: 37
    __    -    ... __   -        _
    LocalTime -> 2025-05-15 21:14:00
    Configuration ->
        . broker -> redis://redis-svc:6379//
        . loader -> celery.loaders.app.AppLoader
        . scheduler -> django_celery_beat.schedulers.DatabaseScheduler

        . logfile -> [stderr]@%INFO
        . maxinterval -> 5.00 seconds (5s)
    2025-05-15 21:14:00,475 INFO success: nginx entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-05-15 21:14:00,475 INFO success: gunicorn entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-05-15 21:14:00,475 INFO success: celery entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-05-15 21:14:00,475 INFO success: celery-beat entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    [2025-05-15 21:14:00 +0000] [19] [INFO] beat: Starting...
    [2025-05-15 21:14:00 +0000] [19] [INFO] DatabaseScheduler: Schedule changed.
    
    -------------- celery@yamtrack-668755cf84-xbt9v v5.5.2 (immunity)
    --- ***** ----- 
    -- ******* ---- Linux-6.8.0-58-generic-x86_64-with 2025-05-15 21:14:00
    - *** --- * --- 
    - ** ---------- [config]
    - ** ---------- .> app:         yamtrack:0x7aefc7145df0
    - ** ---------- .> transport:   redis://redis-svc:6379//
    - ** ---------- .> results:     
    - *** --- * --- .> concurrency: 1 (prefork)
    -- ******* ---- .> task events: OFF (enable -E to monitor tasks in this worker)
    --- ***** ----- 
    -------------- [queues]
                    .> celery           exchange=celery(direct) key=celery
                    

    [tasks]
      . Import from AniList
      . Import from Kitsu
      . Import from MyAnimeList
      . Import from SIMKL
      . Import from Trakt
      . Import from Yamtrack
      . Reload calendar
      . Send daily digest
      . Send release notifications

    [2025-05-15 21:14:01 +0000] [18] [INFO] Connected to redis://redis-svc:6379//
    [2025-05-15 21:14:01 +0000] [18] [INFO] celery@yamtrack-668755cf84-xbt9v ready.
    ```

??? warning "Deploy Yamtrack **without** `securityContext` or it will fail:"

    ``` console
    $ kubectl get all -n yamtrack
    NAME                            READY   STATUS    RESTARTS     AGE
    pod/redis-6768c658df-ncnrk      1/1     Running   0            19s
    pod/yamtrack-6d955f556b-hq5df   0/1     Error     1 (4s ago)   19s

    NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    service/redis-svc      NodePort   10.111.45.57    <none>        6379:30379/TCP   20s
    service/yamtrack-svc   NodePort   10.105.251.86   <none>        8000:30380/TCP   20s

    NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/redis      1/1     1            1           20s
    deployment.apps/yamtrack   0/1     1            0           20s

    NAME                                  DESIRED   CURRENT   READY   AGE
    replicaset.apps/redis-6768c658df      1         1         1       20s
    replicaset.apps/yamtrack-6d955f556b   1         1         0       20s

    $ klogs yamtrack yamtrack
    Operations to perform:
      Apply all migrations: account, admin, app, auth, contenttypes, db, django_celery_beat, django_celery_results, events, lists, mfa, sessions, socialaccount, users
    Running migrations:
      No migrations to apply.
    groupmod: /etc/group.9: Permission denied
    groupmod: cannot lock /etc/group; try again later.
    ```

## Videogame progress trackers

There doesn't seem to be any self-hosted option to keep track of multi-platform video
game libraries and game progress. The closest thing to it is the open-source project
[Playnite](https://playnite.link/), sadly only available for Windows and not possible to
run using Wine, PlayOnLinux or similar tools. The good news is,
[Playnite Will Eventually Make Its Way to Linux](https://retrohandhelds.gg/playnite-will-eventually-make-its-way-to-linux/);
the bad news is, ***not any time soon***.

In the meantime, there are several hosted services to do the above, although at most
they can importing games from Steam or GOG and few other platforms, but not including
itch.io, Nintendo or most other platforms. A quick review with the following show that
*at beast* very basic functionality may be used without some sort of paid subscription:

- [HowLongToBeat](https://howlongtobeat.com/) can be used for free. It is
  *kinda* possible to track games in different states but maintaining a wishlist is not
  a native, well-supported feature.
- [Backloggd](https://backloggd.com) can be used for free (with ads) and support keeping
  a wishlist alongside the collection, but the latter can only be seen in 3 different
  sections (Played, Playing, Backlog) and sorting options are not remembered.
- [INFINITE BACKLOG](https://infinitebacklog.net/) does support more detailled progress
  tracking, plus importing games *and progress* from Steam, GOG and a few more. Both the
  collection and wishlist can be sorted by release date and filtered by multiple fields.
  Access to some features is gated behind Patreon, but the tiers are low enough that it
  could justify a $1/mo. subscription.
- [Keep Track of My Games](https://keeptrackofmygames.com/) gates most features behind
  their own [Fair Use Price](https://ktomg.uservoice.com/knowledgebase/articles/1986487).
- [GG](https://ggapp.io/) seems unfinished and inactive, with
  [elite](https://ggapp.io/elite) features gated behind a $5/mo. subscriptions
  (apparently on their own platform) and no updates since September 2022.

Of all the above, [INFINITE BACKLOG](https://infinitebacklog.net/) seems like the only
potential winner.
