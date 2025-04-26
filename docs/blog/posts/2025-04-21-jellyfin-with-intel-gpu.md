---
date: 2025-04-21
draft: true
categories:
 - Hardware
 - Jellyfin
 - IntelGPU
 - Media
title: Jellyfin with Intel GPU
---

**Starting point:**

- https://merox.dev/blog/kubernetes-media-server/#jellyfin-br
- https://docs.merox.dev/operations/containerization/k3s/manifests/media-stack/


https://www.reddit.com/r/jellyfin/comments/s843w4/how_can_i_force_jf_to_generate_thumbnails_for/


<!-- more -->

``` console
# cat /proc/cpuinfo | grep 'model name' | uniq
model name      : 11th Gen Intel(R) Core(TM) i3-1115G4 @ 3.00GHz
```

``` console
# intel_gpu_top
intel-gpu-top: Intel Tigerlake (Gen12) @ /dev/dri/card0 -    0/   0 MHz; 100% RC6;  0.01/14.51 W
```

``` console
root@lexicon ~ # lshw -c video
  *-display                 
       description: VGA compatible controller
       product: Intel Corporation
       vendor: Intel Corporation
       physical id: 2
       bus info: pci@0000:00:02.0
       version: 01
       width: 64 bits
       clock: 33MHz
       capabilities: pciexpress msi pm vga_controller bus_master cap_list rom
       configuration: driver=i915 latency=0
       resources: iomemory:600-5ff iomemory:400-3ff irq:175 memory:603c000000-603cffffff memory:4000000000-400fffffff ioport:3000(size=64) memory:c0000-dffff memory:4010000000-4016ffffff memory:4020000000-40ffffffff
```

``` console
root@lexicon ~ # vainfo --display drm --device /dev/dri/renderD128
libva info: VA-API version 1.14.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_14
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.14 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 22.3.1 ()
vainfo: Supported profile and entrypoints
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileNone                   : VAEntrypointStats
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Simple            : VAEntrypointEncSlice
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointFEI
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointFEI
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline: VAEntrypointFEI
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileVP8Version0_3          : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointFEI
      VAProfileHEVCMain               : VAEntrypointEncSliceLP
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointEncSliceLP
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile1            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileVP9Profile3            : VAEntrypointVLD
      VAProfileHEVCMain12             : VAEntrypointVLD
      VAProfileHEVCMain12             : VAEntrypointEncSlice
      VAProfileHEVCMain422_10         : VAEntrypointVLD
      VAProfileHEVCMain422_10         : VAEntrypointEncSlice
      VAProfileHEVCMain422_12         : VAEntrypointVLD
      VAProfileHEVCMain422_12         : VAEntrypointEncSlice
      VAProfileHEVCMain444            : VAEntrypointVLD
      VAProfileHEVCMain444            : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_10         : VAEntrypointVLD
      VAProfileHEVCMain444_10         : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_12         : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain10          : VAEntrypointVLD
      VAProfileHEVCSccMain10          : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain444         : VAEntrypointVLD
      VAProfileHEVCSccMain444         : VAEntrypointEncSliceLP
      VAProfileAV1Profile0            : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointEncSliceLP
```

``` console
# mkdir /home/k8s/jellyfin
# chown 1000:1000 /home/k8s/jellyfin

# apt install intel-media-va-driver-non-free libva-drm2 libva-x11-2 -y
```

        new file:   intel-gpu-values.yaml
        new file:   jellyfin.yaml
        new file:   node-feature-rules.yaml


``` console

$ helm uninstall -n intel-device-plugins-gpu intel-device-plugins-gpu
release "intel-device-plugins-gpu" uninstalled


$ helm repo add node-feature-discovery https://kubernetes-sigs.github.io/node-feature-discovery/charts
"node-feature-discovery" has been added to your repositories

$ helm upgrade -i --create-namespace \
    -n node-feature-discovery node-feature-discovery \
    node-feature-discovery/node-feature-discovery
Release "node-feature-discovery" does not exist. Installing it now.
W0421 18:21:42.601848  610556 warnings.go:70] spec.template.spec.affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution[0].preference.matchExpressions[0].key: node-role.kubernetes.io/master is use "node-role.kubernetes.io/control-plane" instead
NAME: node-feature-discovery
LAST DEPLOYED: Mon Apr 21 18:21:41 2025
NAMESPACE: node-feature-discovery
STATUS: deployed
REVISION: 1
TEST SUITE: None

$ kubectl apply -f https://raw.githubusercontent.com/intel/intel-device-plugins-for-kubernetes/main/deployments/nfd/overlays/node-feature-rules/node-feature-rules.yaml
Error from server (BadRequest): error when creating "https://raw.githubusercontent.com/intel/intel-device-plugins-for-kubernetes/main/deployments/nfd/overlays/node-feature-rules/node-feature-rules.yaml": NodeFeatureRule in version "v1alpha1" cannot be handled as a NodeFeatureRule: strict decoding error: unknown field "spec.rules[6].extendedResources"

$ wget  https://raw.githubusercontent.com/intel/intel-device-plugins-for-kubernetes/main/deployments/nfd/overlays/node-feature-rules/node-feature-rules.yaml

$ kubectl apply -f node-feature-rules.yaml 
Error from server (BadRequest): error when creating "node-feature-rules.yaml": NodeFeatureRule in version "v1alpha1" cannot be handled as a NodeFeatureRule: strict decoding error: unknown field "spec.rules[6].extendedResources"

$ vi node-feature-rules.yaml +116

$ kubectl apply -f node-feature-rules.yaml 
nodefeaturerule.nfd.k8s-sigs.io/intel-dp-devices created


$ cat intel-gpu-values.yaml 
name: gpudeviceplugin

sharedDevNum: 1
logLevel: 2
resourceManager: false
enableMonitoring: true
allocationPolicy: "none"

nodeSelector:
  intel.feature.node.kubernetes.io/gpu: 'true'

nodeFeatureRule: true



$ helm upgrade -i --create-namespace \
    -n intel-device-plugins-gpu intel-device-plugins-gpu \
    -f intel-gpu-values.yaml \    
    intel/intel-device-plugins-gpu
Release "intel-device-plugins-gpu" does not exist. Installing it now.
W0421 18:33:37.524184  907255 warnings.go:70] unknown field "spec.tolerations"
W0421 18:33:37.533339  907255 warnings.go:70] unknown field "spec.rules[0].extendedResources"
NAME: intel-device-plugins-gpu
LAST DEPLOYED: Mon Apr 21 18:33:37 2025
NAMESPACE: intel-device-plugins-gpu
STATUS: deployed
REVISION: 1
TEST SUITE: None


$ kubectl get no lexicon -o json | jq .status.capacity
{
  "cpu": "4",
  "ephemeral-storage": "47684Mi",
  "gpu.intel.com/i915": "1",
  "gpu.intel.com/i915_monitoring": "1",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "32480452Ki",
  "pods": "110"
}

$ kubectl logs -n intel-device-plugins-gpu intel-gpu-plugin-gpudeviceplugin-fl8m7
I0421 16:49:54.160769       1 gpu_plugin.go:799] GPU device plugin started with none preferred allocation policy
I0421 16:49:54.161112       1 gpu_plugin.go:518] GPU (i915/xe) resource share count = 1
I0421 16:49:54.253972       1 gpu_plugin.go:540] GPU scan update: 0->1 'i915' resources found
I0421 16:49:54.253990       1 gpu_plugin.go:540] GPU scan update: 0->1 'i915_monitoring' resources found
I0421 16:49:55.254784       1 server.go:285] Start server for i915 at: /var/lib/kubelet/device-plugins/gpu.intel.com-i915.sock
I0421 16:49:55.254787       1 server.go:285] Start server for i915_monitoring at: /var/lib/kubelet/device-plugins/gpu.intel.com-i915_monitoring.sock
I0421 16:49:55.259054       1 server.go:303] Device plugin for i915_monitoring registered
E0421 16:49:55.259088       1 manager.go:146] Failed to serve gpu.intel.com/i915_monitoring: too many open files
Failed to create watcher for /var/lib/kubelet/device-plugins/gpu.intel.com-i915_monitoring.sock
github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin.watchFile
        github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin/server.go:325
github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin.(*server).setupAndServe
        github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin/server.go:307
github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin.(*server).Serve
        github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin/server.go:225
github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin.(*Manager).handleUpdate.func1
        github.com/intel/intel-device-plugins-for-kubernetes/pkg/deviceplugin/manager.go:144
runtime.goexit
        runtime/asm_amd64.s:1700


$ sysctl fs.inotify.max_user_instances
fs.inotify.max_user_instances = 128
$ cat /proc/sys/fs/file-max
9223372036854775807

$ kubectl logs -n intel-device-plugins-gpu intel-gpu-plugin-gpudeviceplugin-fl8m7
I0421 17:10:24.353783       1 gpu_plugin.go:799] GPU device plugin started with none preferred allocation policy
I0421 17:10:24.354195       1 gpu_plugin.go:518] GPU (i915/xe) resource share count = 1
I0421 17:10:24.354758       1 gpu_plugin.go:540] GPU scan update: 0->1 'i915_monitoring' resources found
I0421 17:10:24.354764       1 gpu_plugin.go:540] GPU scan update: 0->1 'i915' resources found
I0421 17:10:25.355464       1 server.go:285] Start server for i915_monitoring at: /var/lib/kubelet/device-plugins/gpu.intel.com-i915_monitoring.sock
I0421 17:10:25.355500       1 server.go:285] Start server for i915 at: /var/lib/kubelet/device-plugins/gpu.intel.com-i915.sock
I0421 17:10:25.362503       1 server.go:303] Device plugin for i915 registered
I0421 17:10:25.362532       1 server.go:303] Device plugin for i915_monitoring registered
I0421 17:10:34.164722       1 gpu_plugin.go:92] Select nonePolicy for GPU device allocation
I0421 17:10:34.164745       1 gpu_plugin.go:138] Allocate deviceIds: ["card0-0"]

$ sysctl fs.inotify.max_user_instances
fs.inotify.max_user_instances = 128


```




**Check again, maybe add ports**

- https://www.debontonline.com/2021/11/kubernetes-part-16-deploy-jellyfin.html

### jellyfin hardware transcoding intel nuc

Unused:

- https://www.reddit.com/r/jellyfin/comments/13wy58z/intel_nuc_recommendation/


### linux command to find intel gpu video codecs

Unused:

- https://www.quora.com/How-can-I-see-on-a-Linux-machine-if-video-decoding-is-done-by-CPU-or-GPU
- https://askubuntu.com/questions/5417/how-to-get-the-gpu-info
- https://bbs.archlinux.org/viewtopic.php?id=257178
- https://bbs.archlinux.org/viewtopic.php?id=213692
- https://bbs.archlinux.org/viewtopic.php?id=279799
- https://www.reddit.com/r/framework/comments/r64clb/video_decoding_acceleration_in_linux/
- https://wiki.archlinux.org/title/Hardware_video_acceleration

### video codes in "Tiger Lake"

https://dgpu-docs.intel.com/devices/hardware-table.html

- 9A78
- Intel® UHD Graphics
- Xe
- Tiger Lake
- 5.7
- 48


### kubernetes jellyfin ffmpeg exited with code 187

Unused:

- https://forum.jellyfin.org/t-solved-ffmpeg-error-code-187
- https://github.com/immich-app/immich/issues/14050
- https://forum.jellyfin.org/t-solved-subtitle-settings-not-applying?pid=56913#pid56913
- https://stackoverflow.com/questions/46519022/ffmpeg-error-in-docker-container-ffmpeg-exited-with-code-1
- https://forum.jellyfin.org/t-transcoding-fails-with-ffmpeg-exit-code-187
- https://forum.jellyfin.org/t-solved-ffmpeg-exits-with-code-187-when-playing-select-files
- https://jellyfin.org/docs/general/administration/hardware-acceleration/intel/#configure-and-verify-lp-mode-on-linux

### kubernetes intel gpu

**Used:**

- https://jonathangazeley.com/2025/02/11/intel-gpu-acceleration-on-kubernetes/
- https://artifacthub.io/packages/helm/djjudas21/jellyfin?modal=template&template=statefulset.yaml

Unused:

- https://intel.github.io/intel-device-plugins-for-kubernetes/cmd/gpu_plugin/README.html
- https://jellyfin.org/docs/general/administration/hardware-acceleration/intel/#verify-on-linux
- https://github.com/kubernetes-sigs/node-feature-discovery/blob/master/docs/usage/customization-guide.md
- https://raw.githubusercontent.com/intel/intel-device-plugins-for-kubernetes/main/deployments/nfd/overlays/node-feature-rules/node-feature-rules.yaml
- https://developers.redhat.com/articles/2024/04/05/enable-gpu-acceleration-kernel-module-management-operator#

#### NodeFeatureRule "unknown field" extendedResources

- https://github.com/intel/intel-device-plugins-for-kubernetes/blob/main/deployments/nfd/overlays/node-feature-rules/node-feature-rules.yaml

### kubernetes  /dev/dri/renderD128

Unused:

- https://www.reddit.com/r/selfhosted/comments/121vb07/plex_on_kubernetes_with_intel_igpu_passthrough/
- https://github.com/blakeblackshear/frigate/discussions/14082
- https://www.johanneskueber.com/posts/hardware_acceleration_kubernets_jellyfin/


### "failed to create fsnotify watcher: too many open files"

Unused:

- 
- https://serverfault.com/questions/1137211/failed-to-create-fsnotify-watcher-too-many-open-files
- https://github.com/kairos-io/kairos/issues/2071

### "Failed to serve" gpu.intel.com/i915_monitoring "too many open files"

Workaround: https://github.com/NVIDIA/gpu-operator/issues/441#issuecomment-1539649895

root@lexicon ~ # sysctl -w fs.inotify.max_user_watches=100000
fs.inotify.max_user_watches = 100000
root@lexicon ~ # sysctl -w fs.inotify.max_user_instances=100000
fs.inotify.max_user_instances = 100000

Unused:

- https://github.com/intel/intel-device-plugins-for-kubernetes/issues/1940
- https://www.intel.com/content/www/us/en/docs/mpi-library/developer-guide-linux/2021-6/error-message-too-many-open-files.html
- https://forum.jellyfin.org/t-too-many-open-files-in-system-error-during-library-scan
- https://stackoverflow.com/questions/15952635/debugging-the-too-many-files-open-issue
- https://hpc-community.unige.ch/t/too-many-open-files-error/3388


### Home Assistant

**DO NOT ADD trailing /**

https://github.com/home-assistant/core/issues/82558#issuecomment-1499153223
