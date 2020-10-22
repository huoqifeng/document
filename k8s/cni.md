
# Kubernetes network and CNI

## What is CNI?
Kubernetes leverage CNI (Container Network Interface) for network management of the Pods. It defines the indeterface and independent approach can be separated but not need bundled in Kubelet, the spec is: https://github.com/containernetworking/cni/blob/master/SPEC.md

There is bundle CNI approach within kubelet of cause, but it's not necessary, we can always leverage a 3rd party CNI implememt. Like Flannel, Calico...

Then, how can Kubelet uses a 3rd party CNI?

## How kubelet know which 3rd party CNI plugin can be used?

Kubelet starts up with some parameters to identify which CNI can be used and initialize it through the configuration, e.g.
```
kubelet --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin
```

Where `--network-plugin=cni` tells kubelet CNI is used for network, `--cni-conf-dir=/etc/cni/net.d` tells kubelet where to find the CNI configuration files, `--cni-bin-dir=/opt/cni/bin` tells kubelet where to get the CNI binaries. Refer https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/ for details.

Take Calico for example, we can find the config files via:
```
# ls -l /etc/cni/net.d
total 12
-rw-r--r-- 1 root root  700 Oct 15 14:45 10-calico.conflist
-rw------- 1 root root 2623 Oct 15 14:45 calico-kubeconfig
drwxr-xr-x 2 root root 4096 Oct 15 14:45 calico-tls
```

Get the CNI plugin binaries via:
```
# ls -l /opt/cni/bin
total 144900
-rwxr-xr-x 1 root root  4879102 Oct 15 14:45 bandwidth
-rwxr-xr-x 1 root root  4836019 Aug 15  2019 bridge
-rwxr-xr-x 1 root root 35389344 Oct 15 14:45 calico
-rwxr-xr-x 1 root root 34602752 Oct 15 14:45 calico-ipam
-rwxr-xr-x 1 root root 12558442 Aug 15  2019 dhcp
-rwxr-xr-x 1 root root  6258512 Aug 15  2019 firewall
-rwxr-xr-x 1 root root  3361245 Oct 15 14:45 flannel
-rwxr-xr-x 1 root root  4418541 Aug 15  2019 host-device
-rwxr-xr-x 1 root root  4309911 Oct 15 14:45 host-local
-rwxr-xr-x 1 root root  4485779 Aug 15  2019 ipvlan
-rwxr-xr-x 1 root root  3971467 Oct 15 14:45 loopback
-rwxr-xr-x 1 root root  4622603 Aug 15  2019 macvlan
-rwxr-xr-x 1 root root  4600630 Oct 15 14:45 portmap
-rwxr-xr-x 1 root root  4832551 Aug 15  2019 ptp
-rwxr-xr-x 1 root root  3654165 Aug 15  2019 sbr
-rwxr-xr-x 1 root root  3097381 Aug 15  2019 static
-rwxr-xr-x 1 root root  3976185 Oct 15 14:45 tuning
-rwxr-xr-x 1 root root  4485585 Aug 15  2019 vlan
```

Once kubelet get the config and binaries path, the plugin could be initialized.

Then, where these files come from? It's installed by Calico:
```
https://github.com/projectcalico/cni-plugin/blob/master/pkg/install/install.go#L118
```

## How kubelet initialize a CNI plugin?

### For dockershim
```
https://github.com/kubernetes/kubernetes/pkg/kubelet/dockershim/docker_service.go
  NewDockerService
     cniPlugins := cni.ProbeNetworkPlugins(pluginSettings.PluginConfDir, pluginSettings.PluginCacheDir, pluginSettings.PluginBinDirs)
         https://github.com/kubernetes/kubernetes/pkg/kubelet/dockershim/network/cni/cni.go#L133
     network.InitNetworkPlugin
         https://github.com/kubernetes/kubernetes/pkg/kubelet/dockershim/network/plugins.go
```

### For remote CRI like containerd
```
https://github.com/containerd/cri/cri.go
init
  initCRIService
     NewCRIService   https://github.com/containerd/cri/pkg/server/service.go#L107
        newCNINetConfSyncer  https://github.com/containerd/cri/pkg/server/cni_conf_syncer.go#L43
           syncer.netPlugin.Load https://github.com/containerd/go-cni/cni.go
```

Where https://github.com/containerd/go-cni/cni.go initialize the CNI plugin from configuaration file `/etc/cni/net.d/10-calico.conflist`
```
# cat /etc/cni/net.d/10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "etcd_endpoints": "https://20.22.110.80:2379",
      "etcd_key_file": "/etc/cni/net.d/calico-tls/etcd-key",
      "etcd_cert_file": "/etc/cni/net.d/calico-tls/etcd-cert",
      "etcd_ca_cert_file": "/etc/cni/net.d/calico-tls/etcd-ca",
      "mtu": 1440,
      "ipam": {
          "type": "calico-ipam"
      },
      "policy": {
          "type": "k8s"
      },
      "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
```

As we know containerd criService can be configured as criRuntime via `--container-runtime=remote` and `--containerd=containerd/containerd.sock`, so the CNI plugin can be initialized when initialize the CRI runtime.

So far, we know how the CNI plugin is chosen and initialized in kubelet. And, how the IP get assigned in a Pod?

## Allocate IP address to a Pod
The CNI plugin has the responsinility to manage the networks of the Pods, usually it's a overlay network. Take Calico for example, here is a list of the componenets https://docs.projectcalico.org/getting-started/kubernetes/managed-public-cloud/iks. Let's have a look at from code perspective at how an IP address get assigned for a Pod.

A Pod is always start from the function `RunPodSandbox`, for containerd which is in https://github.com/containerd/cri/pkg/server/sandbox_run.go, the flow looks like:
```
RunPodSandbox
    setupPodNetwork
        criService.netPlugin.Setup  https://github.com/containerd/go-cni/blob/master/cni.go --> it was initialized when criService get initialized.
            network.Attach    https://github.com/containerd/go-cni/namespace.go#L32
                n.cni.AddNetworkList    https://github.com/containernetworking/cni/blob/master/libcni/api.go --> init from /etc/cni/net.d/10-calico.conflist
                    CNIConfig.addNetwork (..., c.args("ADD", rt))
                        invoke.ExecPluginWithResult  https://github.com/containernetworking/cni/blob/master/pkg/invoke/exec.go#L76
                            RawExec.ExecPlugin  https://github.com/containernetworking/cni/blob/master/pkg/invoke/raw_exec.go#L34
```


Where `ExecPlugin` will fall in to teh binaries identified in `/etc/cni/net.d/10-calico.conflist` and `/opt/cni/bin`. `"ADD"` to trigger the `cmdAdd` interface defined in CNI spec here: https://github.com/containernetworking/cni/blob/master/SPEC.md#Parameters. So kubernetes and the CRI runtime don't need care about the IPAM (IP Address Allocations) and the policies.

- What happens within Calico plugin?

```
https://github.com/projectcalico/cni-plugin/blob/master/pkg/k8s/k8s.go#L51
CmdAddK8s
```

- What IP range is used in CNI plugin?
Defined in kubelet parameters and can be retrieved by CNI plugin
```
kubelet --pod-cidr 172.168.0.0/16
```

So, the parameters to support CNI looks like:
```
kubelet --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime=remote --containerd=containerd/containerd.sock --pod-cidr 172.168.0.0/16
```

Let's have a looks at the Pod that get assigned an IP:
```
root@32bdb3f8d189:~# kubectl get po cert-manager-5c59fcdbd5-mdjgl -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP              NODE           NOMINATED NODE   READINESS GATES
cert-manager-5c59fcdbd5-mdjgl   1/1     Running   0          2d16h   172.168.66.10   32bdb3f8d189   <none>           <none>
```

Check the IP of the host:
```
root@32bdb3f8d189:~# hostname -I
172.168.66.0 172.17.0.1 20.22.110.83
```

You can see, the Pod is on the same network with Node (Host), and the Host takes the sub network IP `172.168.66.0`.

## Bonus

Want to implement a CNI by yourself? it's difinitely a bonus can be found here: https://github.com/containernetworking/plugins/tree/master/plugins/sample
