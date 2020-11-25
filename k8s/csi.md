
# Kubernetes storage and CSI

https://kubernetes-csi.github.io/docs/introduction.html
https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/

## What is CSI?
The Container Storage Interface (CSI) is a standard for exposing arbitrary block and file storage systems to containerized workloads on Container Orchestration Systems (COs) like Kubernetes. Using CSI third-party storage providers can write and deploy plugins exposing new storage systems in Kubernetes without ever having to touch the core Kubernetes code.

Design: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md

So, CSI Driver need be developed and deployed before a Pod can make use of it. Before looking at how a CSI Driver get developed, let's have a look at how a Pod make use of it first.

## PV, PVC, StorageClass and CSI Driver

A csi volume can be used in a Pod in three different ways:
- through a reference to a PersistentVolumeClaim
- with a generic ephemeral volume (alpha feature)
- with a CSI ephemeral volume if the driver supports that (beta feature)

Let's take PVC for example, a Pod will use a PVC like below:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-request-for-storage
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-storage
```

Where if `my-request-for-storage` PV already exist, The Pod will try to attach and mount it directly, if the PV does not exist, the CSI drive will provision it first. So, the CSI driver is required, which is exposed in `StorageClass` like below:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast-storage
provisioner: csi-driver.example.com
parameters:
  type: pd-ssd
  csi.storage.k8s.io/provisioner-secret-name: mysecret
  csi.storage.k8s.io/provisioner-secret-namespace: mynamespace
```

`provisioner` defines the CSI Driver name `csi-driver.example.com` for `StorageClass` fast-storage.

So, the CSI Driver `csi-driver.example.com` will be used to provisioning a PV and then use it as a PVC in Pod, the Pod descripto tells kubelet which CSI Driver shuld be used via `storageClassName`. `StorageClass` defines the `provisioner` -- CSI Driver. (We'll see how Kubelet discover and make use of this CSI Driver in later part.) BTW, PV can also be provisioned before hand using a descriptor like:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-manually-created-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: csi-driver.example.com
    volumeHandle: existingVolumeName
    readOnly: false
    fsType: ext4
    volumeAttributes:
      foo: bar
    controllerPublishSecretRef:
      name: mysecret1
      namespace: mynamespace
    nodeStageSecretRef:
      name: mysecret2
      namespace: mynamespace
    nodePublishSecretRef
      name: mysecret3
      namespace: mynamespace
```

a PV and PVS always have 1:1 map.

Once known the usage of a CSI Driver, let's have a look at how does it work.

## CSI Plugin, CSI Driver and Sidecar containers
We call it CSI Plugin if it's an in-tree build in Kubernetes, we call it CSI Driver if it's a out-tree code separately from Kubernetes.

Before we can make use of a CSI Driver, it should be regsitered to kubelet first, CSI Driver will be implemented and registered following the `Device Plugin` spec: https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md, the arch looks like 
![Device Plugin Arch](https://raw.githubusercontent.com/kubernetes/community/master/contributors/design-proposals/resource-management/device-plugin-overview.png)

So, the CSI Driver will register it to kubelet when startup, Kubelet (responsible for mount and unmount) will communicate with an external “CSI volume driver” running on the same host machine (whether containerized or not) via a Unix Domain Socket. CSI volume drivers should create a socket at the following path on the node machine: `/var/lib/kubelet/plugins/[SanitizedCSIDriverName]/csi.sock`. An example can be found in `csi-driver-nfs` at https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/pkg/nfs/server.go#L72

CSI Driver works along side with some sidecar containers, the algorithm is illustrated in section `Recommended Mechanism for Deploying CSI Drivers on Kubernetes` in the design https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md, as below:
![CSI Driver Implement Flow](https://raw.githubusercontent.com/kubernetes/community/master/contributors/design-proposals/storage/container-storage-interface_diagram1.png)

To deploy a containerized third-party CSI volume driver, it is recommended that storage vendors:

    - Create a “CSI volume driver” container that implements the volume plugin behavior and exposes a gRPC interface via a unix domain socket, as defined in the CSI spec (including Controller, Node, and Identity services).
    - Bundle the “CSI volume driver” container with helper containers (external-attacher, external-provisioner, node-driver-registrar, cluster-driver-registrar, external-resizer, external-snapshotter, livenessprobe) that the Kubernetes team will provide (these helper containers will assist the “CSI volume driver” container in interacting with the Kubernetes system). More specifically, create the following Kubernetes objects:
        - To facilitate communication with the Kubernetes controllers, a StatefulSet or a Deployment (depending on the user's need; see Cluster-Level Deployment) that has:
            - The following containers
                - The “CSI volume driver” container created by the storage vendor.
                - Containers provided by the Kubernetes team (all of which are optional):
                    - cluster-driver-registrar (refer to the README in cluster-driver-registrar repository for when the container is required)
                    - external-provisioner (required for provision/delete operations)
                    - external-attacher (required for attach/detach operations. If you wish to skip the attach step, CSISkipAttach feature must be enabled in Kubernetes in addition to omitting this container)
                    - external-resizer (required for resize operations)
                    - external-snapshotter (required for volume-level snapshot operations)
                    - livenessprobe
            - The following volumes:
                - emptyDir volume
                    - Mounted by all containers, including the “CSI volume driver”.
                    - The “CSI volume driver” container should create its Unix Domain Socket in this directory to enable communication with the Kubernetes helper container(s).
        - A DaemonSet (to facilitate communication with every instance of kubelet) that has:
            - The following containers
                - The “CSI volume driver” container created by the storage vendor.
                - Containers provided by the Kubernetes team:
                    - node-driver-registrar - Responsible for registering the unix domain socket with kubelet.
                    - livenessprobe (optional)
            - The following volumes:
                - hostpath volume
                    - Expose /var/lib/kubelet/plugins_registry from the host.
                    - Mount only in node-driver-registrar container at /registration
                    - node-driver-registrar will use this unix domain socket to register the CSI driver’s unix domain socket with kubelet.
                - hostpath volume
                    - Expose /var/lib/kubelet/ from the host.
                    - Mount only in “CSI volume driver” container at /var/lib/kubelet/
                    - Ensure bi-directional mount propagation is enabled, so that any mounts setup inside this container are propagated back to the host machine.
                - hostpath volume
                    - Expose /var/lib/kubelet/plugins/[SanitizedCSIDriverName]/ from the host as hostPath.type = "DirectoryOrCreate".
                    - Mount inside “CSI volume driver” container at the path the CSI gRPC socket will be created.
                    - This is the primary means of communication between Kubelet and the “CSI volume driver” container (gRPC over UDS).
    - Have cluster admins deploy the above StatefulSet and DaemonSet to add support for the storage system in their Kubernetes cluster.

Alternatively, deployment could be simplified by having all components (including external-provisioner and external-attacher) in the same pod (DaemonSet). Doing so, however, would consume more resources, and require a leader election protocol (likely https://git.k8s.io/contrib/election) in the external-provisioner and external-attacher components.

Regard the sidecar containers, can refer: https://kubernetes-csi.github.io/docs/sidecar-containers.html

## Provision Flow as a sample
Let's have a look at an example of the flow described in above diagram 

We need two containers to explain the flow:
 - A mock CSI Driver container (This should be implemented by vendor): https://github.com/kubernetes-csi/csi-test
 - A provisioning container (Provided by Kubernetes CSI Sig) https://github.com/kubernetes-csi/external-provisioner
 
- Mock CSI

The mock CSI driver is implemented in https://github.com/kubernetes-csi/csi-test

Create mock service
```
s := service.New(config)
```
https://github.com/kubernetes-csi/csi-test/blob/af2d21f66c88d7b4956c17d8123cdc563ca0ba0a/cmd/mock-driver/main.go#L72

New CSI Driver server with the services
```
d := driver.NewCSIDriver(servers)
```
When starting the server
```
if err := d.Start(l); err != nil {
```
it will register service controller to the grpc server:
```
csi.RegisterControllerServer(c.server, c.servers.Controller)
```
https://github.com/kubernetes-csi/csi-test/blob/af2d21f66c88d7b4956c17d8123cdc563ca0ba0a/driver/driver.go#L107 https://github.com/kubernetes-csi/csi-test/blob/af2d21f66c88d7b4956c17d8123cdc563ca0ba0a/vendor/github.com/container-storage-interface/spec/lib/go/csi/csi.pb.go#L5490

The service CreateVolume func will be invoked for the CreateVolumeRequest https://github.com/kubernetes-csi/csi-test/blob/af2d21f66c88d7b4956c17d8123cdc563ca0ba0a/mock/service/controller.go#L22 from Provisiner.

- Provisioner
The provisioner is implemented in https://github.com/kubernetes-csi/external-provisioner
In main func of https://github.com/kubernetes-csi/external-provisioner/blob/2a52c60e4a17696b7270a39281b2e27f64ad17c0/cmd/csi-provisioner/csi-provisioner.go#L101,

create ProvisionController object and run it:
```
provisionController = controller.NewProvisionController(
		clientset,
		provisionerName,
		csiProvisioner,
		serverVersion.GitVersion,
		provisionerOptions...,
	)
...
provisionController.Run(ctx)
```
`NewProvisionController` will initialize an `Informer` to add a handler to handle `PersistentVolumeClaims` Add/Update/Delete events in the cluster: https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner/blob/203b2c9cdf9c44e504af7d8a15d9df6642cd9ea5/controller/controller.go#L689

The handler will add the events to `claimQueue`, and handle the events asynchronously in `processNextClaimWorkItem`

Finally, it will invoke CSI provisioner's `Provision` func to create the volume

Send CreateVolumeRequest
external-provisioner implements the `Provision` func: https://github.com/kubernetes-csi/external-provisioner/blob/2a52c60e4a17696b7270a39281b2e27f64ad17c0/pkg/controller/controller.go#L432

It will create a CSI CreateVolumeRequest and invoke CreateVolume func of CSI client to send the request
```
// Create a CSI CreateVolumeRequest and Response
req := csi.CreateVolumeRequest{
	Name:               pvName,
	Parameters:         options.StorageClass.Parameters,
	VolumeCapabilities: volumeCaps,
	CapacityRange: &csi.CapacityRange{
		RequiredBytes: int64(volSizeBytes),
	},
}
...
rep, err = p.csiClient.CreateVolume(createCtx, &req)
```
CSI client will send the request to CSI driver endpoints: https://github.com/container-storage-interface/spec/blob/baa71a34651e5ee6cb983b39c03097d7aa384278/lib/go/csi/csi.pb.go#L5312


## How is provision secret used?

According to the [doc](https://kubernetes-csi.github.io/docs/secrets-and-credentials-storage-class.html), we can set secret name and namespace in `StorageClass` parameters. It could be used to provision volume:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast-storage
provisioner: csi-driver.team.example.com
parameters:
  type: pd-ssd
  csi.storage.k8s.io/provisioner-secret-name: fast-storage-provision-key
  csi.storage.k8s.io/provisioner-secret-namespace: pd-ssd-credentials
```

The keys are defined in [controller.go](https://github.com/kubernetes-csi/external-provisioner/blob/2a52c60e4a17696b7270a39281b2e27f64ad17c0/pkg/controller/controller.go#L79):
```
prefixedProvisionerSecretNameKey      = csiParameterPrefix + "provisioner-secret-name"
prefixedProvisionerSecretNamespaceKey = csiParameterPrefix + "provisioner-secret-namespace"

...

provisionerSecretParams = secretParamsMap{
		name:                         "Provisioner",
		deprecatedSecretNameKey:      provisionerSecretNameKey,
		deprecatedSecretNamespaceKey: provisionerSecretNamespaceKey,
		secretNameKey:                prefixedProvisionerSecretNameKey,
		secretNamespaceKey:           prefixedProvisionerSecretNamespaceKey,
	}
```

The secrets values will be retrived via API in [getCredentials](https://github.com/kubernetes-csi/external-provisioner/blob/2a52c60e4a17696b7270a39281b2e27f64ad17c0/pkg/controller/controller.go#L1257) func, and set as `CreateVolumeRequest.Secrets`:
```
provisionerSecretRef, err := getSecretReference(provisionerSecretParams, options.StorageClass.Parameters, pvName, &v1.PersistentVolumeClaim{
		ObjectMeta: metav1.ObjectMeta{
			Name:      options.PVC.Name,
			Namespace: options.PVC.Namespace,
		},
	})

provisionerCredentials, err := getCredentials(ctx, p.client, provisionerSecretRef)

req.Secrets = provisionerCredentials
``` 

In the mock CSI driver, the secrets will be used for authentication:
```
func authenticateCreateVolume(req *csi.CreateVolumeRequest, creds *CSICreds) (bool, error) {
	return credsCheck(req.GetSecrets(), creds.CreateVolumeSecret)
}
```
https://github.com/kubernetes-csi/csi-test/blob/af2d21f66c88d7b4956c17d8123cdc563ca0ba0a/driver/driver.go#L267


## How does Kubelet mount and unmount attached volume?

`Kubelet` will create the `volumemanager` and reconcile the volumes that used by Pod.
```
RunKubelet -> https://github.com/kubernetes/kubernetes/blob/71331d8596575d58bc9089168447be26d5d1a877/cmd/kubelet/app/server.go#L1077
  k = createAndInitKubelet
    NewMainKubelet  -> https://github.com/kubernetes/kubernetes/blob/b2dc35dab2cdd2aa33ba8240bb6e53478e63c435/pkg/kubelet/kubelet.go#L720
      klet.volumeManager = volumemanager.NewVolumeManager ->https://github.com/kubernetes/kubernetes/blob/c678434623be4957d892a9865e5649f887a40c49/pkg/kubelet/volumemanager/volume_manager.go#L198
        vm.reconciler = reconciler.NewReconciler -> https://github.com/kubernetes/kubernetes/blob/c678434623be4957d892a9865e5649f887a40c49/pkg/kubelet/volumemanager/reconciler/reconciler.go#L174
  startKubelet
    go k.Run(podCfg.Updates())
      go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)
        vm.reconciler.Run() -> https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/volumemanager/reconciler/reconciler.go#L202
          reconciliationLoopFunc()
            reconcile()
              mountAttachVolumes()
```

## A summary
- Sidecar containers generated by Kubernetes community implemented the main algorith for the CSI logic, like provision/deprovision (create/delete), attach/detach, the container should be deployed before hand, these containers spec can be found in https://kubernetes-csi.github.io/docs/sidecar-containers.html, code can be found in https://github.com/kubernetes-csi
- CSI Driver should be implemented by vendor, and implement the interfaces that the CSI spec defined, include:
  - Register
    - Register uses the Device Plugin framework described in https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md
  - Service
    - CSI will run a gRPC service and listen on a UNIX Socket on Node, default is: `/var/lib/kubelet/plugins/[SanitizedCSIDriverName]/csi.sock`, Kubelet will communicate via this Unix Socket.
  - Provision/Deprovision (Create/Delete)
  - Attach/Detach
  - Code example can be found inn https://github.com/kubernetes-csi/csi-test/blob/master/mock/service/controller.go
- The Volume Mount happens in Kubelet code once the volume created and attached. (Attach can also happens in Kubelet code if CSI Driver did not implement it or choose to skip it)