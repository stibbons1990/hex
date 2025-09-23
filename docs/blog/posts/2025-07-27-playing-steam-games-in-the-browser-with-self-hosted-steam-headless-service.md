---
date: 2025-07-27
categories:
 - kubernetes
 - gaming
 - streaming
 - steam
title: Playing Steam games in the browser with self-hosted Headless Steam Service
---

[Headless Steam](https://github.com/Steam-Headless/docker-steam-headless/tree/master?tab=readme-ov-file#headless-steam-service)
is like a self-hosted [GeForce NOW](https://play.geforcenow.com/mall/),
which can be useful to play games in a browser while away on holidays.
Although mainly intended to play Steam games, it also supports EmeDeck, Heroic and Lutris,
all easy to install via Flatpak, and supports Intel GPU which is already setup for
[Jellyfin on Kubernetes with Intel GPU](./2025-04-29-jellyfin-on-kubernetes-with-intel-gpu.md)
and not actually getting a lot of use; running games would probably be a better use of that Intel UHD GPU.

<!-- more -->

## Installation

Installing and running the Headless Steam service only requires a regular user
to run steam as (e.g. `ponder` with UID 1000) and a few persistent directories:

``` console
# mkdir -p /home/k8s/steam-headless/{home,.X11-unix,pulse}
# chown -R 1000:1000 /home/k8s/steam-headless
```

### Headless Steam deployment

The following deployment is based on the provided
[docker-steam-headless/docs/k8s-files/statefulset.yaml](https://github.com/Steam-Headless/docker-steam-headless/blob/master/docs/k8s-files/statefulset.yaml),
with the addition of a `Service` and an `Ingress` for remote access:

??? k8s "Steam Headless deployment: `steam-headless.yaml`"

    ``` yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: steam-headless
    ---
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: home-dir-pv
      namespace: steam-headless
      labels:
        app: steam-headless
    spec:
      storageClassName: manual
      capacity:
        storage: 50Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: /home/k8s/steam-headless/home/default
      persistentVolumeReclaimPolicy: Retain
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: home-dir-pvc
      namespace: steam-headless
      labels:
        app: steam-headless
    spec:
      storageClassName: manual
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi
      volumeName: home-dir-pv
      volumeMode: Filesystem
    ---
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: steam-headless
      namespace: steam-headless
    spec:
      serviceName: "steam-headless"
      replicas: 1
      selector:
        matchLabels:
          app: steam-headless
      template:
        metadata:
          labels:
            app: steam-headless
        spec:
          hostNetwork: true
          securityContext:
            fsGroup: 1000
          containers:
          - name: steam-headless
            securityContext:
              privileged: true
            image: josh5/steam-headless:latest
            resources:
              requests:
                memory: "12G"
                cpu: "4"
                gpu.intel.com/i915: "1"
              limits:
                memory: "16G"
                cpu: "8"
                gpu.intel.com/i915: "1"
            volumeMounts:
            - name: dev-dri-renderd128
              mountPath: /dev/dri/renderD128
            - name: dshm
              mountPath: /dev/shm
            - name: home-dir
              mountPath: /home/default/
            - name: input-devices
              mountPath: /dev/input/
            env:
            - name: NAME
              value: 'SteamHeadless'
            - name: TZ
              value: 'Europe/Madrid'
            - name: USER_LOCALES
              value: 'en_US.UTF-8 UTF-8'
            - name: DISPLAY
              value: ':55'
            - name: DISPLAY_CDEPTH
              value: '24'
            - name: DISPLAY_REFRESH
              value: '30'
            - name: DISPLAY_SIZEH
              value: '1080'
            - name: DISPLAY_SIZEW
              value: '2560'
            - name: SHM_SIZE
              value: '2G'
            - name: DOCKER_RUNTIME
              value: 'nvidia'
            - name: PUID
              value: '1000'
            - name: PGID
              value: '1000'
            - name: UMASK
              value: '000'
            - name: USER_PASSWORD
              value: '****************************'
            - name: MODE
              value: 'primary'
            - name: WEB_UI_MODE
              value: 'vnc'
            - name: ENABLE_VNC_AUDIO
              value: 'true'
            - name: PORT_NOVNC_WEB
              value: '8083'
            - name: NEKO_NAT1TO1
              value: ''
            - name: 'ENABLE_STEAM'
              value: 'true'
            - name: ENABLE_SUNSHINE
              value: 'true'
            - name: SUNSHINE_USER
              value: 'ponder'
            - name: SUNSHINE_PASS
              value: '****************************'
            - name: ENABLE_EVDEV_INPUTS
              value: 'true'
          volumes:
          - name: dev-dri-renderd128
            hostPath:
              path: /dev/dri/renderD128
          - name: dshm
            emptyDir:
              medium: Memory
          - name: home-dir
            persistentVolumeClaim:
              claimName: home-dir-pvc
          - name: input-devices
            hostPath:
              path: /dev/input/
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: steam-headless-svc
      namespace: steam-headless
      labels:
        app: steam-headless
    spec:
      type: ClusterIP
      ports:
        - port: 80
          targetPort: 8083
      selector:
        app: steam-headless
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: steam-headless-ingress
      namespace: steam-headless
      labels:
        app: steam-headless
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
        nginx.ingress.kubernetes.io/websocket-services: steam-headless-svc
    spec:
      ingressClassName: nginx
      rules:
        - host: steam.very-very-dark-gray.top
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: steam-headless-svc
                    port:
                      number: 80
      tls:
        - secretName: tls-secret
          hosts:
            - steam.very-very-dark-gray.top
    ```

!!! note

    Default screen resolution is 1920x1080 and this *should* be changed by adding
    `DISPLAY_SIZEH` and `DISPLAY_SIZEW` to the `env` section, and refresh rate
    *should* also be adjusted by adding `DISPLAY_REFRESH`, but these settings
    appear to have no effect (no root cause found yet).

Apply this deployment and check that the pod and ingress are up and running:

``` console
$ kubectl apply -f steam-headless.yaml 
namespace/steam-headless created
persistentvolume/home-pv created
persistentvolumeclaim/home-pvc created
statefulset.apps/steam-headless created
service/steam-headless-svc created
ingress.networking.k8s.io/steam-headless-ingress created

$ kubectl get all -n steam-headless 
NAME                   READY   STATUS    RESTARTS   AGE
pod/steam-headless-0   1/1     Running   0          89s

NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/steam-headless-svc   ClusterIP   10.100.43.61   <none>        80/TCP    39m

NAME                              READY   AGE
statefulset.apps/steam-headless   1/1     39m

$ kubectl get ingress -n steam-headless 
NAME                     CLASS   HOSTS                           ADDRESS         PORTS     AGE
steam-headless-ingress   nginx   steam.very-very-dark-gray.top   192.168.0.171   80, 443   39m
```

Check the pod logs if it's not `Running` after a couple of minutes;
if all goes well the pod should reach `Starting supervisord`:

??? terminal "`kubectl logs steam-headless-0 -n steam-headless`"

    ``` console
    $ kubectl logs steam-headless-0 -n steam-headless 
    Build: [2025-07-26 03:44:23] [master] [23e5ec9fa4747ea05219b66ec938112c2a0fa110] [debian]

    [ /etc/cont-init.d/10-setup_user.sh: executing... ]
    **** Configure default user ****
      - Setting default user uid=1000(default) gid=1000(default)
    usermod: no changes
      - Adding default user to any additional required device groups
      - Adding user 'default' to group: 'video'
      - Adding user 'default' to group: 'audio'
      - Adding user 'default' to group: 'input'
      - Adding user 'default' to group: 'pulse'
      - Adding user 'default' to group: 'polkitd' for device: /dev/input/event0
      - Adding user 'default' to group: 'video' for device: /dev/dri/card0
      - Adding user 'default' to group: 'user-gid-993' for device: /dev/dri/renderD128
      - Setting umask to 000
      - Create the user XDG_RUNTIME_DIR path '/tmp/.X11-unix/run'
      - Setting ownership of all log files in '/home/default/.cache/log'
      - Setting root password
      - Setting user password
    DONE

    [ /etc/cont-init.d/11-setup_sysctl_values.sh: executing... ]
    **** Configure some system kernel parameters ****
      - Setting the maximum number of memory map areas a process can create to 524288
    DONE

    [ /etc/cont-init.d/30-configure_dbus.sh: executing... ]
    **** Configure container dbus ****
      - Container configured to run its own dbus
    DONE

    [ /etc/cont-init.d/30-configure_udev.sh: executing... ]
    **** Configure udevd ****
      - Disable udevd - /run/udev does not exist
      - Enable dumb-udev service
      - Ensure the default user has permission to r/w on input devices
    DONE

    [ /etc/cont-init.d/40-setup_locale.sh: executing... ]
    **** Configure local ****
      - Locales already set correctly to en_US.UTF-8 UTF-8
    DONE

    [ /etc/cont-init.d/50-configure_pulseaudio.sh: executing... ]
    **** Configure pulseaudio ****
      - Enable pulseaudio service.
      - Configure pulseaudio to pipe audio to a socket
    DONE

    [ /etc/cont-init.d/60-configure_gpu_driver.sh: executing... ]
    **** Found Intel device 'Intel Corporation Raptor Lake-P [UHD Graphics] (rev 04)' ****
      - Enable i386 arch
      - Install mesa vulkan drivers
    **** No AMD device found ****
    **** No NVIDIA device found ****
    DONE

    [ /etc/cont-init.d/70-configure_desktop.sh: executing... ]
    **** Configure Desktop ****
      - Enable Desktop service.
      - Ensure home directory template is owned by the default user.
      - Installing default home directory template
    DONE

    [ /etc/cont-init.d/70-configure_xorg.sh: executing... ]
    **** Generate default xorg.conf ****
      - Configure Xwrapper.config
      - Configure container as primary the X server
      - Enabling evdev input class on pointers, keyboards, touchpads, touch screens, etc.
      - No monitors connected. Installing dummy xorg.conf
    DONE

    [ /etc/cont-init.d/80-configure_flatpak.sh: executing... ]
    **** Configure Flatpak ****
      - Flatpak configured for running inside a Docker container
    DONE

    [ /etc/cont-init.d/90-configure_neko.sh: executing... ]
    **** Configure Neko ****
      - Disable Neko server
    DONE

    [ /etc/cont-init.d/90-configure_steam.sh: executing... ]
    **** Configure Steam ****
      - Enable Steam auto-start script
      - Initializing Steam config
      - Initializing Steam library
    DONE

    [ /etc/cont-init.d/90-configure_sunshine.sh: executing... ]
    **** Configure Sunshine ****
      - Enable Sunshine server
    DONE

    [ /etc/cont-init.d/90-configure_vnc.sh: executing... ]
    **** Configure VNC ****
      - Configure VNC service port '32036'
      - Configure pulseaudio encoded stream port '32037'
      - Enable VNC server
    DONE

    [ /etc/cont-init.d/95-setup_wol.sh: executing... ]
    **** Configure WoL Manager ****
      - Disable WoL Manager service.

    **** Starting supervisord ****
      - Logging all root services to '/var/log/supervisor/'
      - Logging all user services to '/home/default/.cache/log/'

    2025-07-28 00:45:19,856 INFO Included extra file "/etc/supervisor.d/dbus.ini" during parsing
    2025-07-28 00:45:19,856 INFO Included extra file "/etc/supervisor.d/desktop.ini" during parsing
    2025-07-28 00:45:19,856 INFO Included extra file "/etc/supervisor.d/neko.ini" during parsing
    2025-07-28 00:45:19,856 INFO Included extra file "/etc/supervisor.d/pulseaudio.ini" during parsing
    2025-07-28 00:45:19,856 INFO Included extra file "/etc/supervisor.d/steam.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/sunshine.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/udev.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/vnc-audio.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/vnc.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/wol-power-manager.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/xorg.ini" during parsing
    2025-07-28 00:45:19,857 INFO Included extra file "/etc/supervisor.d/xvfb.ini" during parsing
    2025-07-28 00:45:19,857 INFO Set uid to user 0 succeeded
    2025-07-28 00:45:19,858 INFO RPC interface 'supervisor' initialized
    2025-07-28 00:45:19,859 CRIT Server 'unix_http_server' running without any HTTP authentication checking
    2025-07-28 00:45:19,859 INFO supervisord started with pid 1
    2025-07-28 00:45:20,860 INFO spawned: 'dbus' with pid 485
    2025-07-28 00:45:20,862 INFO spawned: 'udev' with pid 486
    2025-07-28 00:45:20,863 INFO spawned: 'xorg' with pid 487
    2025-07-28 00:45:20,864 INFO spawned: 'audiostream' with pid 488
    2025-07-28 00:45:20,865 INFO spawned: 'frontend' with pid 489
    2025-07-28 00:45:20,866 INFO spawned: 'pulseaudio' with pid 490
    2025-07-28 00:45:20,867 INFO spawned: 'x11vnc' with pid 491
    2025-07-28 00:45:20,867 INFO spawned: 'desktop' with pid 493
    2025-07-28 00:45:20,868 INFO spawned: 'sunshine' with pid 495
    PULSEAUDIO: Starting pulseaudio service
    2025-07-28 00:45:20,882 INFO reaped unknown pid 512 (exit status 0)
    2025-07-28 00:45:21,887 INFO success: dbus entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: udev entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: xorg entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: audiostream entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: frontend entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: pulseaudio entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: x11vnc entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: desktop entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:21,887 INFO success: sunshine entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2025-07-28 00:45:22,016 INFO reaped unknown pid 574 (exit status 1)
    2025-07-28 00:45:22,714 INFO reaped unknown pid 580 (exit status 0)
    2025-07-28 00:45:22,714 INFO reaped unknown pid 582 (exit status 0)
    2025-07-28 00:45:22,714 INFO reaped unknown pid 584 (exit status 0)
    2025-07-28 00:45:22,714 INFO reaped unknown pid 586 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 588 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 590 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 592 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 594 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 597 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 599 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 601 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 603 (exit status 0)
    2025-07-28 00:45:22,715 INFO reaped unknown pid 605 (exit status 0)
    2025-07-28 00:45:22,930 INFO reaped unknown pid 609 (exit status 0)
    2025-07-28 00:45:22,930 INFO reaped unknown pid 611 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 628 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 630 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 632 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 634 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 636 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 638 (exit status 0)
    2025-07-28 00:45:24,566 INFO reaped unknown pid 640 (exit status 0)
    ```

After a couple of minutes the web UI is ready to use at
<https://steam.very-very-dark-gray.top>

![](https://github.com/Steam-Headless/docker-steam-headless/raw/master/docs/images/web_connect.png)

To verify that the pod has full access to the GPU, open a Terminal and run
`glxinfo | grep -i 'direct render'`; the onput should be
`direct rendering: Yes`.

!!! todo "Install Heroic and Lutris via Flatpak"

## Closed Issues

### Authentication

There is a big **Connect** button but **no authentication** at all,
so this web UI is **not ready** to be deployed **for remote use** just yet.
While authentication is not ready; scale the `StatefulSet` down to zero replicas:

``` console
$ kubectl scale --replicas=0 sts steam-headless -n steam-headless
statefulset.apps/steam-headless scaled
```

[VNC Password Authentication](https://github.com/Steam-Headless/docker-steam-headless/issues/29)
can be added, as a workaround, by adding the following script under
`/home/default/init.d` (under `/home/k8s/steam-headless`):

``` bash
#!/bin/bash

# Define the file path
FILE_PATH="/usr/bin/start-x11vnc.sh"

# Define the password part to insert
PASSWORD_PART="-passwd $USER_PASSWORD"

# Check if the -passwd option is already in the script
if grep -q "\-passwd" "$FILE_PATH"; then
    echo "VNC Password flag already exists."
else
    echo "Adding VNC password flag..."
    # Use sed to insert the password part before the '&' at the end of the command line
    sed -i "/x11vnc.*\&/s/\&/ $PASSWORD_PART&/" "$FILE_PATH"
    echo "VNC Password flag added successfully."
fi
```

This relies on `USER_PASSWORD` environment variable existing, which should be in places from
the deployment above. With this, clicking on the big **Connect** button prompts for the pasword.

## Open Issues

### Audio over SSL

[Bug #171: The audio web socket incorrectly uses SSL on reverse proxies](https://github.com/Steam-Headless/docker-steam-headless/issues/171)
makes audio not available in the current setup with NGinx reverse proxy.
The only workaround available seems to be forwarding port `32041` to the host IP.
The `nginx.ingress.kubernetes.io/websocket-services: steam-headless-svc`
annotation on the `Ingress` does not seem to help.

So far there have been no visible attempts from the browser to connect to port 32041,
while audio is definitely not working for either Steam games or video playback on Firefox.

