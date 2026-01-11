# Spectrum-X Host Configuration on OpenShift

**Goal**: The goal of this document is to document completely the steps to getting a host ready for Spectrum-X.

## Workflow Sections

- [Environment](#environment)
- [Set Core User Password for Troubleshooting](#set-core-user-password-for-troubleshooting)
- [Set Hugepages and IOMMU Off](#set-hugepages-and-iommu-off)
- [Set UDEV Rules for Rail Device Names](#set-udev-rules-for-rail-device-names)
- [Configuring NFD Operator](#configuring-nfd-operator)
- [Configuring SRIOV Operator](#configuring-sriov-operator)
- [Configuring NMState Operator](#configuring-nmstate-operator)
- [Configuring NVIDIA Network Operator](#configuring-nvidia-network-operator)
- [Configuring NVIDIA Maintenance Operator](#configuring-nvidia-maintenance-operator)
- [Install HTTP server to serve NIC Firmware files](#Install-HTTP-server-to-serve-NIC-Firmware-files)
- [Configuring Nic Firmware](#configuring-nic-firmware)
- [Configuring NVIDIA GPU Operator](#configuring-nvidia-network-operator)
- [Configuring LLDPD Daemonset](#configuring-lldpd-daemonset)
- [Configuring OVS Offload](#configuring-ovs-offload)
- [Configure Physical Rail Interface Attributes](#configure-physical-rail-interface-attributes)
- [Configure Bridges and OVS Flows](#configure-bridges-and-ovs-flows)
- [Configure Spectrum-X CNI](#configure-spectrum-x-CNI)
- [Validate Spectrum-X Topology](#validate-spectrum-x-topology)


## Environment

The host environment for this document was a single GH200 node running OpenShift 4.20 with Local Volume Storage Operator already configured.  The NFD, SR-IOV, NMState, NVIDIA Network, NVIDIA Maintenance and NVIDIA GPU operators were all installed but need to be configured.

## Set Core User Password for Troubleshooting

This section is completely optional but might be useful in the event network connectivity is lost to one of the OpenShift nodes.   Here we will configure a password for the core user so we can login via the console if necessary.  

The first step is to assign a hash password to the core user variable using the `mkpasswd` utlity.  In our example we are passing in `password` as the password.  Choose a password that is appropriate for the organizations password policy. 

~~~bash
$ export COREPASS=`mkpasswd -m SHA-512 password`
~~~

Now we can take the COREPASS variable and use it to construct a MachineConfig that we can apply to our nodes which will set the core user pass to our value.  Note that we are only specifying master node here since we are using single node.

~~~bash
$ cat <<EOF > set-core-user-password-machineconfig.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master  # Note if you want this on worker nodes then change the role.
  name: set-core-user-password
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
      - name: core 
        passwordHash: $COREPASS
EOF
~~~

Create the MachineConfig on the cluster.  Note that for this MachineConfig the nodes are not required to reboot.

~~~bash
$ oc create -f set-core-user-password-machineconfig.yaml
machineconfig.machineconfiguration.openshift.io/set-core-user-password created
~~~

We can monitor the progress of setting the core user password MachineConfig using the `oc get mcp` command.

~~~bash
$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-fefd989a138b206eab4a9a9362f67a03   False     True       False      1              0                   0                     0                      25h
worker   rendered-worker-d853a15275615906b9d23f4a28e007f4   True      False      False      0              0                   0                     0                      25h
~~~

Before completing this section make sure to test that it is possible to login as the core user on the BMC console as the password that was set.

## Set Hugepages and IOMMU Off

We need to set hughpages and disable iommu.  This can be achieved with the following machine configuration.

~~~bash
$ cat <<EOF > 99-machineconfig-nvd-srv-36.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-nvd-srv-36
spec:
  kernelArguments:
    - default_hugepagesz=1G
    - hugepagesz=1G
    - hugepages=16
    - iommu=off
EOF
~~~

Create the machine configuration on the cluster.  This will cause nodes to reboot in a rolling fashion and can be monitored with `oc get mcp`.

~~~bash
$ oc create -f 99-machineconfig-nvd-srv-36.yaml
machineconfig.machineconfiguration.openshift.io/99-master-nvd-srv-36 created
~~~

To validate this has been configured we can use `dmesg` output and `oc describe node` to see iommu and hughpages are set.

## Set UDEV Rules for Rail Device Names

We need to use udev rules to normalize the interface names on each node to eth_rail or roce_rail for ethernet and infiniband respectively.   We can do this by creating a persistent udev rules file indicating the devices which should be the same across all worker nodes in a homogenous cluster.   Below is an example of what that file might look like.  On the GH200 node we only have two interfaces to work with but it gives you an idea.

~~~bash
$ cat <<EOF > 70-persistent-net.rules 
ACTION=="add", KERNELS=="0000:18:00.0", SUBSYSTEM=="net", NAME="eth_rail0"
ACTION=="add", KERNELS=="0000:1a:00.0", SUBSYSTEM=="net", NAME="eth_rail1"
ACTION=="add", KERNELS=="0000:3a:00.0", SUBSYSTEM=="net", NAME="eth_rail2"
ACTION=="add", KERNELS=="0000:4d:00.0", SUBSYSTEM=="net", NAME="eth_rail3"
ACTION=="add", KERNELS=="0000:5d:00.0", SUBSYSTEM=="net", NAME="eth_rail4"
ACTION=="add", KERNELS=="0000:9b:00.0", SUBSYSTEM=="net", NAME="eth_rail5"
ACTION=="add", KERNELS=="0000:ba:00.0", SUBSYSTEM=="net", NAME="eth_rail6"
ACTION=="add", KERNELS=="0000:ca:00.0", SUBSYSTEM=="net", NAME="eth_rail7"
ACTION=="add", KERNELS=="0000:cc:00.0", SUBSYSTEM=="net", NAME="eth_rail8"
ACTION=="add", KERNELS=="0000:db:00.0", SUBSYSTEM=="net", NAME="eth_rail9"

ACTION=="add", KERNELS=="0000:18:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail0"
ACTION=="add", KERNELS=="0000:1a:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail1"
ACTION=="add", KERNELS=="0000:3a:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail2"
ACTION=="add", KERNELS=="0000:4d:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail3"
ACTION=="add", KERNELS=="0000:5d:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail4"
ACTION=="add", KERNELS=="0000:9b:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail5"
ACTION=="add", KERNELS=="0000:ba:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail6"
ACTION=="add", KERNELS=="0000:ca:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail7"
ACTION=="add", KERNELS=="0000:cc:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail8"
ACTION=="add", KERNELS=="0000:db:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail9"
EOF
~~~

Once we have the udev file created we need to base64 encoded it to prepare it for a machine configuration.  Below will base64 encoded it and then assign it to the variable UDV_RULES.

~~~bash
$ UDEV_RULES=`cat 70-persistent-net.rules|base64 -w 0`
~~~

Next we can cat out the machine configuration file below and it will use the UDEV_RULES variable we set above to enbedded the base64 udev rules.

~~~bash
$ cat <<EOF > 99-machine-config-udev-network.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
   labels:
     machineconfiguration.openshift.io/role: master # Change to worker when using a real multi-node cluster
   name: 99-machine-config-udev-network
spec:
   config:
     ignition:
       version: 3.2.0
     storage:
       files:
       - contents:
           source: data:text/plain;charset=utf-8;base64,${UDEV_RULES}
         filesystem: root
         mode: 420
         path: /etc/udev/rules.d/70-persistent-net.rules
EOF
~~~

Finally we can create the machine configuration on the cluster.  This will cause nodes to reboot in a rolling fashion and can be monitored with `oc get mcp`.
pay attention to the output of `oc get mcp` , sometimes is not working well and the worker node/s remain in degraded  true.

~~~bash
$ oc create -f 99-machine-config-udev-network.yaml
machineconfig.machineconfiguration.openshift.io/99-machine-config-udev-network created
~~~

Once the machine configuration has been successfully applied we can validate that its applied by spot checking nodes in a debug pod to confirm the interface names have been set appropriately.

~~~bash
$ oc debug node/nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-36nvidiaengrdu2dcredhatcom-debug-v4x47 ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
Pod IP: 10.6.135.15
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# ip link|grep rail
5: eth_rail0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
6: eth_rail1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
sh-5.1#
~~~

## Configuring NFD Operator

The Node Feature Discovery (NFD) operator manages the detection of hardware features and configuration in an OpenShift Container Platform cluster by labeling the nodes with hardware-specific information. NFD labels the host with node-specific attributes, such as PCI cards, kernel, operating system version, and so on.

We need to create the NodeFeatureDiscovery instance custom resource file. Note that we add entries to the default deviceClasseWhiteList field, so that to support more network adapters, such as the NVIDIA BlueField DPUs.

~~~bash
$ cat <<EOF > nfd-instance.yaml 
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  instance: ''
  operand:
    servicePort: 12000
  prunerOnDelete: false
  topologyUpdater: false
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist:
            - "02"
            - "03"
            - "0200"
            - "0207"
            - "12"
          deviceLabelFields:
            - "vendor"
EOF
~~~

With our instance custom resource file generated we can create it on the cluster.

~~~bash
$ oc create -f nfd-instance.yaml
nodefeaturediscovery.nfd.openshift.io/nfd-instance created
~~~

Finally we can validate our instance is up and running by again looking at the pods under the openshift-nfd namespace.

~~~bash
$ oc get pods -n openshift-nfd
NAME                                     READY   STATUS    RESTARTS   AGE
nfd-controller-manager-f4fc58bb4-k5fq2   1/1     Running   7          4d23h
nfd-gc-676786b846-6kpcz                  1/1     Running   6          4d23h
nfd-master-75fcc46558-b49jl              1/1     Running   6          4d23h
nfd-worker-l6jc6                         1/1     Running   7          6d5h
~~~

## Configuring SRIOV Operator

We need to create the default SriovOperatorConfig custom resource file.

~~~bash
$ cat <<EOF > sriov-operator-config.yaml 
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovOperatorConfig
metadata:
  name: default
  namespace: openshift-sriov-network-operator
spec:
  enableInjector: true
  enableOperatorWebhook: true
  logLevel: 2
EOF
~~~

Then we need to create it on our cluster.

~~~bash
$ oc create -f sriov-operator-config.yaml 
sriovoperatorconfig.sriovnetwork.openshift.io/default created
~~~

Validate that the pods are running.

~~~bash
$ oc get pods -n openshift-sriov-network-operator
NAME                                      READY   STATUS    RESTARTS   AGE
network-resources-injector-m4t8k          1/1     Running   0          49s
operator-webhook-w9k8n                    1/1     Running   0          49s
sriov-network-config-daemon-ql4cl         1/1     Running   0          49s
sriov-network-operator-5995bb94f6-qbsgd   1/1     Running   0          18m
~~~

Finally patch the sriovoperatorconfig to work with the NVIDIA Network Operator and DOCA/MOFED.

~~~bash
oc patch sriovoperatorconfig default --type=merge -n openshift-sriov-network-operator --patch '{ "spec": { "configDaemonNodeSelector": { "network.nvidia.com/operator.mofed.wait": "false", "node-role.kubernetes.io/master": "", "feature.node.kubernetes.io/pci-15b3.sriov.capable": "true" } } }'
~~~

## Configuring NMState Operator

There will be a need to configure network interfaces on the nodes that were not configured at initial cluster creation time and the NMState operator is designed for those use cases. The first step is to create a custom resource file that contains the namespace, operator group and subscription.

A nmstate instance is required so we will create a custom resource file for that.

~~~bash
$ cat <<EOF > nmstate-instance.yaml 
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF
~~~

Then we will create the instance on the cluster.

~~~bash
$ oc create -f nmstate-instance.yaml
nmstate.nmstate.io/nmstate created
~~~

Finally we will validate the instance is running.

~~~bash
$ oc get pods -n openshift-nmstate
NAME                                     READY   STATUS    RESTARTS   AGE
nmstate-console-plugin-8dcf98f9b-stx5x   0/1     Running   0          10s
nmstate-handler-zwhpt                    0/1     Running   0          10s
nmstate-metrics-767cb6c678-bx5ln         1/2     Running   0          10s
nmstate-operator-78777d7db8-5frht        1/1     Running   0          21m
nmstate-webhook-676f7797c4-h4nl6         0/1     Running   0          10s
~~~

We will configure NNCP policies at a later point in this document.

## Configuring NVIDIA Network Operator

With the NVIDIA Network operator up we need to create the NicClusterPolicy custom resource file. Note in this file there are values coded for my environment.  For example the `nic-fw-storage-pvc` is a persistent volume claim I have precreated in my environment using the Local Volume Storage Operator.

~~~bash
$ cat <<EOF > ncp-spectrumx.yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    env:
      - name: UNLOAD_STORAGE_MODULES
        value: "true"
    image: doca-driver
    repository: nvcr.io/nvstaging/mellanox
    version: doca3.2.0-25.10-1.1.2.0-0
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: true
        enable: true
        force: true
        timeoutSeconds: 300
      maxParallelUpgrades: 1
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 10
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
  nicConfigurationOperator:
    operator:
      image: nic-configuration-operator
      repository: nvcr.io/nvstaging/mellanox
      version: network-operator-v25.10.0-beta.4
      containerResources:
        - name: nic-configuration-operator
          limits:
            cpu: 1000m
            memory: 5000Mi
          requests:
            cpu: 1000m
            memory: 5000Mi
    configurationDaemon:
      image: nic-configuration-operator-daemon
      repository: nvcr.io/nvstaging/mellanox
      version: network-operator-v25.10.0-beta.4
      containerResources:
        - name: nic-configuration-daemon
          limits:
            cpu: 1000m
            memory: 5000Mi
          requests:
            cpu: 1000m
            memory: 5000Mi
    nicFirmwareStorage: # configure your own storage params
      create: true
      pvcName: nic-fw-storage-pvc
      storageClassName: lvms-vg1
      availableStorageSize: 1Gi
    logLevel: debug
  spectrumXOperator:
    image: spectrum-x-operator
    repository: nvcr.io/nvstaging/mellanox
    version: network-operator-v25.10.0-beta.4
EOF
~~~

Next we can create the NicClusterPolicy custom resource on the cluster.

~~~bash
$ oc create -f ncp-spectrumx.yaml 
nicclusterpolicy.mellanox.com/nic-cluster-policy created
~~~

We can validate the NicClusterPolicy by looking at the running pods.

~~~bash
$ oc get pods -n nvidia-network-operator
NAME                                                              READY   STATUS    RESTARTS   AGE
mofed-rhel9.6-7f8f74fb56-ds-xkjdv                                 2/2     Running   0          6m12s
nic-configuration-daemon-lkbc8                                    1/1     Running   0          6m11s
nic-configuration-operator-5cb54c5c89-lhbk7                       1/1     Running   0          6m11s
nvidia-network-operator-controller-manager-cb54ffcc-k44dz         1/1     Running   0          8m1s
spectrum-x-flowcontroller-w9wf9                                   1/1     Running   0          6m11s
~~~

## Configuring NVIDIA Maintenance Operator

Now we need to configure the `MaintenanceOperatorConfig` custom resource file.  In this file we can specify the log level, the number of parallel operations (ie how many nodes to take offline at once) and the time the node is kept in maintenance (a number in seconds that provides enough time for the maintenance work to happen before the operator will remove the node maintenance policy).  In our example we are just going to allow one maintenance operation at a time and that operation has 300 seconds to finish before the node is returned to schedulable.

~~~bash
$ cat <<EOF > maintenance-operator-config.yaml
apiVersion: maintenance.nvidia.com/v1alpha1
kind: MaintenanceOperatorConfig
metadata:
  name: default
  namespace: nvidia-maintenance-operator
spec:
  logLevel: info
  maxParallelOperations: 1
  maxNodeMaintenanceTimeSeconds: 300
EOF
~~~

Now let's create the `MaintenanceOperatorConfig` on the cluster.

~~~bash
$ oc create -f maintenance-operator-config.yaml
maintenanceoperatorconfig.maintenance.nvidia.com/default created
~~~

The `MaintenanceOperatorConfig` does not spin up any additional pods but we can check that it is there by running the following command.

~~~bash
$ oc get MaintenanceOperatorConfig -n nvidia-maintenance-operator -o yaml
apiVersion: v1
items:
- apiVersion: maintenance.nvidia.com/v1alpha1
  kind: MaintenanceOperatorConfig
  metadata:
    creationTimestamp: "2025-10-30T15:41:01Z"
    generation: 1
    name: default
    namespace: nvidia-maintenance-operator
    resourceVersion: "2664798"
    uid: 51d80b8b-eda3-4665-be1b-81f6fca9a64d
  spec:
    logLevel: info
    maxNodeMaintenanceTimeSeconds: 300
    maxParallelOperations: 1
kind: List
metadata:
  resourceVersion: ""

$ oc get pods -n nvidia-maintenance-operator
NAME                                                      READY   STATUS    RESTARTS   AGE
maintenance-operator-controller-manager-995859f88-vq262   1/1     Running   0          18h
~~~
## Install HTTP server to serve NIC Firmware files

In Order to serve the firmware files, we should Install HTTP server on the local cluster. Firstly by creating NFS pv

~~~bash
$ cat <<EOF > nfs-pv.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-nfs-pvc
spec:
  capacity:
    storage: 10Gi # Define the storage capacity
  accessModes:
    - ReadWriteMany # Allows the volume to be mounted as read-write by many nodes
  nfs:
    path: /vm_img/NFS # The exported path on your NFS server
    server: 10.8.231.39 # The IP address of your NFS server
  persistentVolumeReclaimPolicy: Retain # Retains the volume after the PVC is deleted
  volumeMode: Filesystem
EOF
~~~

We can apply it now with

~~~bash
$ oc apply -f nfs-pv.yaml
~~~

Now we need to create a pvc to use it

~~~bash
$ cat <<EOF > cat nfs-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-nfs-pvc 
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  volumeName: nginx-nfs-pv
  storageClassName: ""
EOF
~~~
Now let's apply it
~~~bash
oc apply -f nfs-pvc.yaml
~~~

now we will install NGINX server ( the Nic Configuration Operator can work with http )
~~~bash
$ cat <<EOF > nginx_configmap.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-autoindex
data:
  default.conf: |
    server {
      listen 80;
      server_name _;

      location / {
        root /usr/share/nginx/html;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
      }
    }
EOF
~~~

Lets apply the configmap

~~~bash
oc apply -f nginx_configmap.yaml
~~~

Nginx Server deployment
~~~bash
$ cat <<EOF > nginx-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-fileserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-fileserver
  template:
    metadata:
      labels:
        app: nginx-fileserver
    spec:
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - containerPort: 80
        volumeMounts:
        - name: files
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d  
      volumes:
      - name: files
        persistentVolumeClaim:
          claimName: nginx-nfs-pvc
      - name: nginx-config
        configMap:
          name: nginx-autoindex 
EOF
~~~
Now we will deploy it by running

~~~bash
oc apply -f nginx-deployment.yaml
~~~

Now, In order for it to be able to serve us, we need to configure service and route
~~~bash
$ cat <<EOF > nginx-service.yaml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-fileserver
spec:
  selector:
    app: nginx-fileserver
  ports:
  - name: http
    port: 80
    targetPort: 80
EOF
~~~
and apply the service yaml
~~~bash
oc apply -f nginx-service.yaml
~~~
Lets install the route now
~~~bash
$ cat <<EOF > nginx-route.yaml 
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-fileserver
spec:
  to:
    kind: Service
    name: nginx-fileserver
  port:
    targetPort: http
EOF
~~~
and apply it
~~~bash
oc apply -f nginx-route.yaml
~~~

## Configuring Nic Firmware

NVIDIA NIC Configuration Operator provides Kubernetes API(Custom Resource Definition) to allow FW configuration on Nvidia NICs in a coordinated manner. It deploys, based on settings in the NicClusterPolicy, a configuration daemon on each of the desired nodes to configure Nvidia NICs there. NVIDIA NIC Configuration Operator uses the Maintenance Operator to prepare a node for maintenance before the actual configuration.

We need to define the following NicFirmwareSource file.  This tells the operator where to get the firmware.

~~~bash
$ cat <<EOF > fwsource.yaml
apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicFirmwareSource
metadata:
  name: spc-x-doca-pcc
  namespace: nvidia-network-operator
spec:
  bfbUrlSource: https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb # the newest version I have found, you may skip this for now
  docaSpcXCCUrlSource: http://nginx-fileserver-default.apps.doca9.nvidia.eng.rdu2.dc.redhat.com/doca-spcx-cc_3.1.0105-1_amd64.deb
EOF
~~~

We can then create the resource on the cluster.

~~~bash
$ oc create -f fwsource.yaml 
nicfirmwaresource.configuration.net.nvidia.com/spc-x-doca-pcc created
~~~

In the logs of the Nic Configuration Operator pod we can confirm the download is working by seeing the following.

~~~bash
2025-10-31T18:50:42.283077413Z	INFO	firmware/provisioning.go:509	Cache directory is missing metadata file, removing it	{"cacheDir": "/nic-firmware/spc-x-doca-pcc/bfb"}
2025-10-31T18:50:42.285847653Z	INFO	firmware/provisioning.go:558	Downloading a file	{"cacheDir": "/nic-firmware/spc-x-doca-pcc/bfb", "url": "https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb"}
2025-10-31T18:50:42.285867557Z	LEVEL(-2)	firmware/utils.go:72	FirmwareUtils.DownloadFile()	{"url": "https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb", "destPath": "/nic-firmware/spc-x-doca-pcc/bfb/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb"}
2025-10-31T18:51:26.941575957Z	LEVEL(-2)	firmware/provisioning.go:574	File downloaded and validated	{"path": "/nic-firmware/spc-x-doca-pcc/bfb/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb"}
2025-10-31T18:51:26.941824053Z	INFO	firmware/provisioning.go:275	Writing metadata file to disk	{"cacheDir": "/nic-firmware/spc-x-doca-pcc/bfb", "metadata": {"https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb":"bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb"}}
2025-10-31T18:51:26.943203797Z	INFO	firmware/provisioning.go:587	File successfully processed	{"cacheName": "/nic-firmware/spc-x-doca-pcc/bfb", "url": "https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb", "filename": "bf-bundle-3.1.0-76_25.07_ubuntu-22.04_prod.bfb"}
~~~

The NicFirmwareTemplate file will instruct the Nic Configuration Operator on what network cards it should update and with what firmware source defined.

~~~bash
$ cat <<EOF > fwtemplate.yaml 
apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicFirmwareTemplate
metadata:
  name: spectrum-x-configuration
  namespace: nvidia-network-operator
spec:
  nodeSelector:
    kubernetes.io/hostname: nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com # Drop section if want on all nodes 
  nicSelector:
    nicType: "a2dc" # BlueField-3 SuperNIC Type
    pciAddresses:
      - "0002:01:00.0" # Drop for all nics to be configured or specifically set for just certain nic
  template:
    nicFirmwareSourceRef: spc-x-doca-pcc
    updatePolicy: Update
EOF
~~~

We can then create the resource on the cluster.

~~~bash
$ oc create -f fwtemplate.yaml
nicfirmwaretemplate.configuration.net.nvidia.com/spectrum-x-configuration created
~~~

We can see in the logs 

~~~bash
$ oc logs nic-configuration-operator-569d6d6f5b-9njf2 -n nvidia-network-operator

2026-01-07T07:57:45.554318301Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:61	Listed templates	{"templates": [{"kind":"NicFirmwareTemplate","apiVersion":"configuration.net.nvidia.com/v1alpha1","metadata":{"name":"spectrum-x-configuration","namespace":"nvidia-network-operator","uid":"92c6f791-4551-4b99-92ef-1a8d70d6b245","resourceVersion":"72316629","generation":1,"creationTimestamp":"2025-12-31T10:07:08Z","labels":{"machineconfiguration.openshift.io/role":"worker"},"managedFields":[{"manager":"kubectl-create","operation":"Update","apiVersion":"configuration.net.nvidia.com/v1alpha1","time":"2025-12-31T10:07:08Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:machineconfiguration.openshift.io/role":{}}},"f:spec":{".":{},"f:nicSelector":{".":{},"f:nicType":{},"f:pciAddresses":{}},"f:nodeSelector":{},"f:template":{".":{},"f:nicFirmwareSourceRef":{},"f:updatePolicy":{}}}}},{"manager":"manager","operation":"Update","apiVersion":"configuration.net.nvidia.com/v1alpha1","time":"2025-12-31T10:07:09Z","fieldsType":"FieldsV1","fieldsV1":{"f:status":{".":{},"f:nicDevices":{}}},"subresource":"status"}]},"spec":{"nicSelector":{"nicType":"a2dc","pciAddresses":["0000:18:00.0","0000:1a:00.0","0000:3a:00.0","0000:4d:00.0","0000:5d:00.0","0000:9b:00.0","0000:ba:00.0","0000:ca:00.0","0000:cc:00.0","0000:db:00.0"]},"template":{"nicFirmwareSourceRef":"spc-x-doca-pcc","updatePolicy":"Update"}},"status":{"nicDevices":["dell-h200-3-a2dc-vn0kk4nrfcbnv48b63k3","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63fv","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63bk","dell-h200-3-a2dc-vn0kk4nrfcbnv48u600r","dell-h200-2-a2dc-vn0kk4nrfcbnv48b63ch","dell-h200-2-a2dc-vn0kk4nrfcbnv48b63fz","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63ek","dell-h200-2-a2dc-vn0kk4nrfcbnv495604l","dell-h200-2-a2dc-vn0kk4nrfcbnv495600m","dell-h200-2-a2dc-vn0kk4nrfcbnv495603b","dell-h200-2-a2dc-vn0kk4nrfcbnv48b63jr","dell-h200-2-a2dc-vn0kk4nrfcbnv48b63jt","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63jw","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63j5","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63dz","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63ka","dell-h200-3-a2dc-vn0kk4nrfcbnv48b63es","dell-h200-2-a2dc-vn0kk4nrfcbnv495600o","dell-h200-2-a2dc-vn0kk4nrfcbnv48b637k","dell-h200-2-a2dc-vn0kk4nrfcbnv48b63f3"]}}]}
2026-01-07T07:57:45.554427606Z	LEVEL(-2)	controller/template_matcher.go:51	Listed devices	{"count": 24}
2026-01-07T07:57:45.554584216Z	LEVEL(-2)	controller/template_matcher.go:59	Listed nodes	{"count": 5}
2026-01-07T07:57:45.554611976Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-2-a2dc-vn0kk4nrfcbnv495604l
2026-01-07T07:57:45.55462141Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-3-a2dc-vn0kk4nrfcbnv48b63ek
2026-01-07T07:57:45.554629304Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-3-a2dc-vn0kk4nrfcbnv48b63j5
2026-01-07T07:57:45.554635832Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-3-a2dc-vn0kk4nrfcbnv48b63jw
2026-01-07T07:57:45.55464101Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-3-a2dc-vn0kk4nrfcbnv48u600r
2026-01-07T07:57:45.554645877Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-2-a2dc-vn0kk4nrfcbnv48b63ch
2026-01-07T07:57:45.554650203Z	LEVEL(-2)	controller/nicfirmwaretemplate_controller.go:104	Applying template spectrum-x-configuration to device dell-h200-2-a2dc-vn0kk4nrfcbnv48b63fz

~~~

The NicConfigurationTemplate file will instruct the Nic Configuration Operator how to configure the network cards  

~~~bash
$ cat <<EOF > configtemplate.yaml 
apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicConfigurationTemplate
metadata:
  creationTimestamp: "2025-12-31T10:09:25Z"
  generation: 4
  name: spc-x-config
  namespace: nvidia-network-operator
  resourceVersion: "72388744"
  uid: 7ee2a727-31c5-468f-956c-cd562c5ca155
spec:
  nicSelector:
    nicType: a2dc
    pciAddresses:
    - "0000:18:00.0"
    - 0000:1a:00.0
    - 0000:3a:00.0
    - 0000:4d:00.0
    - 0000:5d:00.0
    - 0000:9b:00.0
    - 0000:ba:00.0
    - 0000:ca:00.0
    - 0000:cc:00.0
    - 0000:db:00.0
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  resetToDefault: false
  template:
    linkType: Ethernet
    numVfs: 1
    spectrumXOptimized:
      enabled: true
      overlay: none
      version: RA2.0
status:
  nicDevices:
  - dell-h200-2-a2dc-vn0kk4nrfcbnv495600o
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b637k
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b63f3
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63k3
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63fv
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63bk
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48u600r
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b63ch
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b63fz
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63ek
  - dell-h200-2-a2dc-vn0kk4nrfcbnv495604l
  - dell-h200-2-a2dc-vn0kk4nrfcbnv495600m
  - dell-h200-2-a2dc-vn0kk4nrfcbnv495603b
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b63jr
  - dell-h200-2-a2dc-vn0kk4nrfcbnv48b63jt
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63jw
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63j5
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63dz
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63ka
  - dell-h200-3-a2dc-vn0kk4nrfcbnv48b63es
EOF
~~~

~~~bash

~~~

## Configuring NVIDIA GPU Operator

The NVIDIA GPU Operator is installed but we need to create a GPU cluster policy custom resource file like the one below.

~~~bash
$ cat <<EOF > gpu-cluster-policy.yaml 
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  vgpuDeviceManager:
    config:
      default: default
    enabled: true
  migManager:
    config:
      default: all-disabled
      name: default-mig-parted-config
    enabled: true
  operator:
    defaultRuntime: crio
    initContainer: {}
    runtimeClass: nvidia
    use_ocp_driver_toolkit: true
  dcgm:
    enabled: true
  gfd:
    enabled: true
  dcgmExporter:
    config:
      name: ''
    serviceMonitor:
      enabled: true
    enabled: true
  cdi:
    default: false
    enabled: false
  driver:
    licensingConfig:
      nlsEnabled: true
      configMapName: ''
    kernelModuleType: open
    certConfig:
      name: ''
    kernelModuleConfig:
      name: ''
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: false
        enable: false
        force: false
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      maxUnavailable: 25%
      podDeletion:
        deleteEmptyDir: false
        force: false
        timeoutSeconds: 300
      waitForCompletion:
        timeoutSeconds: 0
      kernelModuleType: auto
    repoConfig:
      configMapName: ''
    virtualTopology:
      config: ''
    enabled: true
    useNvidiaDriverCRD: false
  devicePlugin:
    config:
      name: ''
      default: ''
    mps:
      root: /run/nvidia/mps
    enabled: true
  gdrcopy:
    enabled: true
  kataManager:
    config:
      artifactsDir: /opt/nvidia-gpu-operator/artifacts/runtimeclasses
  mig:
    strategy: single
  sandboxDevicePlugin:
    enabled: true
  validator:
    plugin:
      env:
        - name: WITH_WORKLOAD
          value: 'false'
  nodeStatusExporter:
    enabled: true
  daemonsets:
    rollingUpdate:
      maxUnavailable: '1'
    updateStrategy: RollingUpdate
  sandboxWorkloads:
    defaultWorkload: container
    enabled: false
  gds:
    enabled: true
    image: nvidia-fs
    repository: nvcr.io/nvidia/cloud-native
    version: 2.26.6
  vgpuManager:
    enabled: false
  vfioManager:
    enabled: true
  toolkit:
    installDir: /usr/local/nvidia
    enabled: true
EOF
~~~

With the GPU ClusterPolicy custom resource generated, let's create it on the cluster on the cluster.

~~~bash
$ oc create -f gpu-cluster-policy.yaml
clusterpolicy.nvidia.com/gpu-cluster-policy created
~~~

Validate the pods are running.

~~~bash
$ oc get pods -n nvidia-gpu-operator
NAME                                           READY   STATUS      RESTARTS       AGE
gpu-feature-discovery-mpqfp                    1/1     Running     0              17h
gpu-operator-787dc9685f-g9lgj                  1/1     Running     9              2d20h
nvidia-container-toolkit-daemonset-qvb5c       1/1     Running     0              17h
nvidia-cuda-validator-7gzsk                    0/1     Completed   0              17h
nvidia-dcgm-exporter-2w6ff                     1/1     Running     0              17h
nvidia-dcgm-hclt7                              1/1     Running     0              17h
nvidia-device-plugin-daemonset-mdkzm           1/1     Running     0              17h
nvidia-driver-daemonset-9.6.20250925-0-42fxk   4/4     Running     33 (17h ago)   2d19h
nvidia-mig-manager-s6mtv                       1/1     Running     0              17h
nvidia-node-status-exporter-n2kkh              1/1     Running     6              2d19h
nvidia-operator-validator-4b5zx                1/1     Running     0              17h
~~~

Validate that `nvidia-smi` reponds and kernel modules are loaded.

~~~bash
$ oc rsh -n nvidia-gpu-operator $(oc -n nvidia-gpu-operator get pod -o name -l app.kubernetes.io/component=nvidia-driver)
sh-5.1# nvidia-smi
Fri Oct 31 16:04:06 2025       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GH200 480GB             On  |   00000009:01:00.0 Off |                    0 |
| N/A   24C    P0             85W /  700W |       0MiB /  97871MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
sh-5.1# lsmod|grep nvidia
nvidia_fs             303104  0
nvidia_modeset       1949696  0
nvidia_uvm           3305472  12
nvidia              14544896  26 nvidia_uvm,nvidia_fs,gdrdrv,nvidia_modeset
video                  69632  1 nvidia_modeset
nvidia_cspmu           40960  0
arm_cspmu_module       32768  1 nvidia_cspmu
drm                   684032  5 drm_kms_helper,ast,drm_shmem_helper,nvidia
~~~

## Configuring LLDPD Daemonset

To run our lldpd container daemonset we will need to provide a service account with privilege access similar to what we do with NMState.   The first step is to create a service account we will simply call lldp.  In this example I am creating it under the nvidia-network-operator namespace.  First craft the custom resource file.

~~~bash
$ cat <<EOF > lldp-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lldp
  namespace: nvidia-network-operator
EOF
~~~

Then use the custom resource file to create the service account on the cluster.

~~~bash
$ oc create -f lldp-serviceaccount.yaml 
serviceaccount/lldp created
~~~

Once the service account is created we can apply the privileges to it.

~~~bash
$ oc -n nvidia-network-operator adm policy add-scc-to-user privileged -z lldp
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "lldp"
~~~

With our service account created and given the permissision it needs we can now focus on creating our lldpd daemonset.   This daemonset will live in the nvidia-network-operator namespace as well.  Further in this example I am assiging a secondary resource to enable a secondary interface that is connected to the Spectrum-X switch.  That way we can demonsrate the sending and recieving of lldp packets.  Create the below custom resource file and modify as necessary for the environment.

~~~bash
$ cat <<EOF > lldpd-daemonset.yaml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: lldpd-container
  namespace: nvidia-network-operator
  labels:
    app: lldpd
spec:
  selector:
    matchLabels:
      app: lldpd
  template:
    metadata:
      labels:
        app: lldpd
    spec:
      serviceAccountName: lldp
      hostNetwork: true
      containers:
        - name: lldpd-container
          image: quay.io/redhat_emp1/ecosys-nvidia/lldpd-arm64:0.0.5
          securityContext:
            privileged: true
EOF
~~~

Once we have created the daemonset custom resource file we can create it on the cluster.

~~~bash
$ oc create -f lldpd-daemonset.yaml 
daemonset.apps/lldpd-container created
~~~

We can validate it is running by looking at the pods in the nvidia-network-operator namespace.

~~~bash
$ oc get pods -n nvidia-network-operator -l app=lldpd -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP            NODE                                       NOMINATED NODE   READINESS GATES
lldpd-container-hhslt   1/1     Running   0          4m42s   10.6.135.15   nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com   <none>           <none>
~~~

## Configuring OVS Offload

Before we configure OVS Offload let's disable Mellanox plugin.

~~~bash
$ oc patch sriovoperatorconfigs.sriovnetwork.openshift.io -n openshift-sriov-network-operator default --patch '{ "spec": { "disablePlugins": ["mellanox"] } }' --type='merge'
sriovoperatorconfig.sriovnetwork.openshift.io/default patched
~~~

In OpenShift we can achieve OVS offload by creating an SriovNetworkPoolConfig and apply it to node.  Note this will not work on a SNO node it seems.

~~~bash
$ cat <<EOF > sriovnetworkpoolconfig-offload.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkPoolConfig
metadata:
  name: sriovnetworkpoolconfig-offload
  namespace: openshift-sriov-network-operator
spec:
  # Add fields here
  ovsHardwareOffloadConfig:
    name: master
EOF
~~~

Create the SriovNetworkPoolConfig on the cluster.

~~~bash
$ oc create -f sriovnetworkpoolconfig-offload.yaml 
sriovnetworkpoolconfig.sriovnetwork.openshift.io/sriovnetworkpoolconfig-offload created
~~~

Verify that offload is set to true under other_config.

~~~bash
$ oc debug node/nvd-srv-29.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-29nvidiaengrdu2dcredhatcom-debug-v66nq ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.8
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# ovs-vsctl list open_vswitch
_uuid               : 78300d8c-9134-4ae4-8d47-066a11492e23
bridges             : [6dcbfd65-cb12-44de-8954-efe2256529e4, 9b9b41ba-22c6-4c50-a2cf-0a403816af48]
cur_cfg             : 2733
datapath_types      : [netdev, system]
datapaths           : {system=5154b9d6-3ffa-43f8-9f92-2b7c81fd86fa}
db_version          : "8.8.0"
dpdk_initialized    : false
dpdk_version        : "DPDK 24.11.2"
external_ids        : {hostname=nvd-srv-29.nvidia.eng.rdu2.dc.redhat.com, ovn-bridge-mappings="physnet:br-ex", ovn-bridge-remote-probe-interval="0", ovn-enable-lflow-cache="true", ovn-encap-ip="10.6.135.8", ovn-encap-type=geneve, ovn-is-interconn="true", ovn-memlimit-lflow-cache-kb="1048576", ovn-monitor-all="true", ovn-ofctrl-wait-before-clear="0", ovn-remote="unix:/var/run/ovn/ovnsb_db.sock", ovn-remote-probe-interval="180000", ovn-set-local-ip="true", rundir="/var/run/openvswitch", system-id="bac2184b-58eb-48f2-b517-4bd623f21a30"}
iface_types         : [bareudp, erspan, geneve, gre, gtpu, internal, ip6erspan, ip6gre, lisp, patch, srv6, stt, system, tap, vxlan]
manager_options     : []
next_cfg            : 2733
other_config        : {bundle-idle-timeout="0", hw-offload="true", ovn-chassis-idx-bac2184b-58eb-48f2-b517-4bd623f21a30="", vlan-limit="0"}
ovs_version         : "3.5.2-33.el9fdp"
ssl                 : []
statistics          : {}
system_type         : rhel
system_version      : "9.6"
~~~

## Configure Physical Rail Interface Attributes

We can provide those same settings via a SriovNetworkNodePolicy for each rail interface.  An example of the policy which provides the `switchdev` mode, mtu and vfs count is below.

~~~bash
$ cat <<EOF > rail-example-sriovnetworknodepolicy.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: snnp-eth_rail1
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  eSwitchMode: "switchdev"
  mtu: 9216
  nicSelector:
    pfNames: ["eth_rail1"]
  numVfs: 1
  isRdma: true
  linkType: eth
  resourceName: eth_rail1
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
---
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: snnp-eth_rail2
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  eSwitchMode: "switchdev"
  mtu: 9216
  nicSelector:
    pfNames: ["eth_rail2"]
  numVfs: 1
  isRdma: true
  linkType: eth
  resourceName: eth_rail2
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
EOF
~~~

Once we generate the SriovNetworkNodePolicy we can create it on the cluster.  We will repeat this for each rail.

~~~bash
$ oc create -f rail-example-sriovnetworknodepolicy.yaml 
sriovnetworknodepolicy.sriovnetwork.openshift.io/rail-x created
~~~

We can validate the configuration by using a `debug` pod and checking the values.  In this example we used emp55s0np0 as our test interface.

~~~bash
$ oc debug node/nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-30nvidiaengrdu2dcredhatcom-debug-f6thp ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.9
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host

sh-5.1# cat /sys/class/net/enp55s0np0/device/sriov_numvfs
1

sh-5.1# ip link |grep enp55
26: enp55s0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 qdisc mq state UP mode DEFAULT group default qlen 1000
29: enp55s0np0_0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    altname enp55s0npf0vf0
31: enp55s0v0: <BROADCAST,MULTICAST> mtu 9216 qdisc noop state DOWN mode DEFAULT group default qlen 1000

sh-5.1# grep PCI_SLOT_NAME /sys/class/net/enp55s0np0/device/uevent
PCI_SLOT_NAME=0000:37:00.0

sh-5.1# devlink dev eswitch show pci/0000:37:00.0
pci/0000:37:00.0: mode switchdev inline-mode none encap-mode basic
~~~

This completes the physical interface confguration section.

## Configure Bridges and OVS Flows

These manual commands can be wrapped into a script where we can programatically create the bridges and flow and recover them when the Openvswitch service restarts or reloads.  The first step is to build a script from the commands.  I envision this script as one that discovers the eth_rail(x) devices on the hosts and then proceeds to create and configure the bridges for each host.  The example script below contains some of that 

~~~bash
$ cat <<EOF > spectrum-br-flows.sh
#!/bin/bash
echo "Gathering the hostname..." 
SYSTEM=`hostname`
while IFS=':' read -r HOSTNAME INTERFACE IPADDRESS SUBNET GATEWAY TORIPADDRESS
do
  if [[ "$SYSTEM" == "$HOSTNAME" ]]
  then
    echo "Creating bridge on $SYSTEM for interface $INTERFACE and settings values..."
    ovs-vsctl --may-exist add-br br-$INTERFACE
    ovs-vsctl set bridge br-$INTERFACE fail-mode=secure
    ovs-vsctl set bridge br-$INTERFACE external-ids:rail_uplink=$INTERFACE
    ovs-vsctl set Interface br-$INTERFACE mtu_request=9216
    ovs-vsctl add-port br-$INTERFACE $INTERFACE
    ovs-vsctl set Interface $INTERFACE mtu_request=9216

    echo "Setting ovs-bridge external-ids to tor_ip for br-$INTERFACE..."
    #ovs-vsctl set bridge br-$INTERFACE external-ids:rail_peer_ip=$TORIPADDRESS
    echo "ovs-vsctl set bridge br-$INTERFACE external-ids:rail_peer_ip=$TORIPADDRESS"

    echo "Adding ip addresses to internal bridge br-$INTERFACE..."
    #ip addr add $IPADDRESS/$SUBNET dev br-$INTERFACE
    echo "ip addr add $IPADDRESS/$SUBNET dev br-$INTERFACE"
    #ip addr add $GATEWAY dev br-$INTERFACE    #### <--- Check with NVIDIA this command seems wrong
    echo "ip addr add $GATEWAY dev br-$INTERFACE" 

    echo "Bringing up br-$INTERFACE port..."
    #ip link set dev br-$INTERFACE up
    echo "ip link set dev br-$INTERFACE up"

    echo "Adding the flows to the bridge br-$INTERFACE..."
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$IPADDRESS actions=LOCAL"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$IPADDRESS actions=LOCAL""
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$GATEWAY actions=LOCAL"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$GATEWAY actions=LOCAL""
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,nw_dst=$IPADDRESS actions=LOCAL"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,nw_dst=$IPADDRESS actions=LOCAL""
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,nw_dst=$GATEWAY actions=LOCAL"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,nw_dst=$GATEWAY actions=LOCAL""
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$TORIPADDRESS actions=output:$INTERFACE"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, arp,arp_tpa=$TORIPADDRESS actions=output:$INTERFACE""
    #ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,in_port=LOCAL,nw_dst=$TORIPADDRESS/$SUBNET actions=output:$INTERFACE"
    echo "ovs-ofctl add-flow  br-$INTERFACE "cookie=0x1, ip,in_port=LOCAL,nw_dst=$TORIPADDRESS/$SUBNET actions=output:$INTERFACE""
  fi
done </etc/spectrum-config-map

echo "Completed setting up all rail bridges and flows on $SYSTEM!"
EOF
~~~

Now that we have the example script we can base64 encode it and stash it into a variable.

~~~bash
$ BASE64_SCRIPT=`cat spectrum-br-flows.sh | base64 -w 0`
$ echo $BASE64_SCRIPT
IyEvYmluL2Jhc2gKZWNobyAiR2F0aGVyaW5nIHRoZSBob3N0bmFtZS4uLiIgClNZU1RFTT1gaG9zdG5hbWVgCndoaWxlIElGUz0nOicgcmVhZCAtciBIT1NUTkFNRSBJTlRFUkZBQ0UgSVBBRERSRVNTIFNVQk5FVCBHQVRFV0FZIFRPUklQQUREUkVTUwpkbwogIGlmIFsiJFNZU1RFTSIgPT0gIiRIT1NUTkFNRSJdCiAgdGhlbgogICAgZWNobyAiQ3JlYXRpbmcgYnJpZGdlIG9uICRTWVNURU0gZm9yIGludGVyZmFjZSAkSU5URVJGQUNFIGFuZCBzZXR0aW5ncyB2YWx1ZXMuLi4iCiAgICBvdnMtdnNjdGwgLS1tYXktZXhpc3QgYWRkLWJyIGJyLSRJTlRFUkZBQ0UKICAgIG92cy12c2N0bCBzZXQgYnJpZGdlIGJyLSRJTlRFUkZBQ0UgZmFpbC1tb2RlPXNlY3VyZQogICAgb3ZzLXZzY3RsIHNldCBicmlkZ2UgYnItJElOVEVSRkFDRSBleHRlcm5hbC1pZHM6cmFpbF91cGxpbms9JElOVEVSRkFDRQogICAgb3ZzLXZzY3RsIHNldCBJbnRlcmZhY2UgYnItJElOVEVSRkFDRSBtdHVfcmVxdWVzdD05MjE2CiAgICBvdnMtdnNjdGwgYWRkLXBvcnQgYnItJElOVEVSRkFDRSAkSU5URVJGQUNFCiAgICBvdnMtdnNjdGwgc2V0IEludGVyZmFjZSAkSU5URVJGQUNFIG10dV9yZXF1ZXN0PTkyMTYKCiAgICBlY2hvICJTZXR0aW5nIG92cy1icmlkZ2UgZXh0ZXJuYWwtaWRzIHRvIHRvcl9pcCBmb3IgYnItJElOVEVSRkFDRS4uLiIKICAgICNvdnMtdnNjdGwgc2V0IGJyaWRnZSBici0kSU5URVJGQUNFIGV4dGVybmFsLWlkczpyYWlsX3BlZXJfaXA9JFRPUklQQUREUkVTUwogICAgZWNobyAib3ZzLXZzY3RsIHNldCBicmlkZ2UgYnItJElOVEVSRkFDRSBleHRlcm5hbC1pZHM6cmFpbF9wZWVyX2lwPSRUT1JJUEFERFJFU1MiCgogICAgZWNobyAiQWRkaW5nIGlwIGFkZHJlc3NlcyB0byBpbnRlcm5hbCBicmlkZ2UgYnItJElOVEVSRkFDRS4uLiIKICAgICNpcCBhZGRyIGFkZCAkSVBBRERSRVNTL1NVQk5FVCBkZXYgYnItJElOVEVSRkFDRQogICAgZWNobyAiaXAgYWRkciBhZGQgJElQQUREUkVTUy9TVUJORVQgZGV2IGJyLSRJTlRFUkZBQ0UiCiAgICAjaXAgYWRkciBhZGQgJEdBVEVXQVkgZGV2IGJyLSRJTlRFUkZBQ0UgICAgIyMjIyA8LS0tIENoZWNrIHdpdGggTlZJRElBIHRoaXMgY29tbWFuZCBzZWVtcyB3cm9uZwogICAgZWNobyAiaXAgYWRkciBhZGQgJEdBVEVXQVkgZGV2IGJyLSRJTlRFUkZBQ0UiIAoKICAgIGVjaG8gIkJyaW5naW5nIHVwIGJyLSRJTlRFUkZBQ0UgcG9ydC4uLiIKICAgICNpcCBsaW5rIHNldCBkZXYgYnItJElOVEVSRkFDRSB1cAogICAgZWNobyAiaXAgbGluayBzZXQgZGV2IGJyLSRJTlRFUkZBQ0UgdXAiCgogICAgZWNobyAiQWRkaW5nIHRoZSBmbG93cyB0byB0aGUgYnJpZGdlIGJyLSRJTlRFUkZBQ0UuLi4iCiAgICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kSU5URVJGQUNFICJjb29raWU9MHgxLCBhcnAsYXJwX3RwYT0kSVBBRERSRVNTIGFjdGlvbnM9TE9DQUwiCiAgICBlY2hvICJvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGFycCxhcnBfdHBhPSRJUEFERFJFU1MgYWN0aW9ucz1MT0NBTCIiCiAgICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kSU5URVJGQUNFICJjb29raWU9MHgxLCBhcnAsYXJwX3RwYT0kR0FURVdBWSBhY3Rpb25zPUxPQ0FMIgogICAgZWNobyAib3ZzLW9mY3RsIGFkZC1mbG93ICBici0kSU5URVJGQUNFICJjb29raWU9MHgxLCBhcnAsYXJwX3RwYT0kR0FURVdBWSBhY3Rpb25zPUxPQ0FMIiIKICAgICNvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGlwLG53X2RzdD0kSVBBRERSRVNTIGFjdGlvbnM9TE9DQUwiCiAgICBlY2hvICJvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGlwLG53X2RzdD0kSVBBRERSRVNTIGFjdGlvbnM9TE9DQUwiIgogICAgI292cy1vZmN0bCBhZGQtZmxvdyAgYnItJElOVEVSRkFDRSAiY29va2llPTB4MSwgaXAsbndfZHN0PSRHQVRFV0FZIGFjdGlvbnM9TE9DQUwiCiAgICBlY2hvICJvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGlwLG53X2RzdD0kR0FURVdBWSBhY3Rpb25zPUxPQ0FMIiIKICAgICNvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGFycCxhcnBfdHBhPSRUT1JJUEFERFJFU1MgYWN0aW9ucz1vdXRwdXQ6JElOVEVSRkFDRSIKICAgIGVjaG8gIm92cy1vZmN0bCBhZGQtZmxvdyAgYnItJElOVEVSRkFDRSAiY29va2llPTB4MSwgYXJwLGFycF90cGE9JFRPUklQQUREUkVTUyBhY3Rpb25zPW91dHB1dDokSU5URVJGQUNFIiIKICAgICNvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRJTlRFUkZBQ0UgImNvb2tpZT0weDEsIGlwLGluX3BvcnQ9TE9DQUwsbndfZHN0PSRUT1JJUEFERFJFU1MvOCBhY3Rpb25zPW91dHB1dDokSU5URVJGQUNFIgogICAgZWNobyAib3ZzLW9mY3RsIGFkZC1mbG93ICBici0kSU5URVJGQUNFICJjb29raWU9MHgxLCBpcCxpbl9wb3J0PUxPQ0FMLG53X2RzdD0kVE9SSVBBRERSRVNTLzggYWN0aW9ucz1vdXRwdXQ6JElOVEVSRkFDRSIiCiAgZmkKZG9uZSA8L2V0Yy9zcGVjdHJ1bS1jb25maWctbWFwCgplY2hvICJDb21wbGV0ZWQgc2V0dGluZyB1cCBhbGwgcmFpbCBicmlkZ2VzIGFuZCBmbG93cyBvbiAkU1lTVEVNISIK
~~~

Next we need to generate our config mapping.

~~~bash
$ cat <<EOF >spectrum-config-map 
nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com:eth_rail0:192.168.67.36:24:192.168.67.1:192.168.67.250
nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com:eth_rail1:192.168.67.37:24:192.168.67.1:192.168.67.250
~~~

We also need to base64 encode our config mapping.

~~~bash
$ BASE64_MAP=`cat spectrum-config-map | base64 -w 0`
$ echo $BASE64_MAP
bnZkLXNydi0zNi5udmlkaWEuZW5nLnJkdTIuZGMucmVkaGF0LmNvbTpldGhfcmFpbDA6MTkyLjE2OC42Ny4zNjoyNDoxOTIuMTY4LjY3LjE6MTkyLjE2OC42Ny4yNTAKbnZkLXNydi0zNi5udmlkaWEuZW5nLnJkdTIuZGMucmVkaGF0LmNvbTpldGhfcmFpbDE6MTkyLjE2OC42Ny4zNzoyNDoxOTIuMTY4LjY3LjE6MTkyLjE2OC42Ny4yNTAK
~~~

Now we need to take our script and our mapping file and embed the base64 encoding into a machine configuration that will run the script as a systemd service.  Beside initially running the script the systemd service will also restart anytime the openvswitch service is restarted or if the box is rebooted which ensures the flows are always set.

~~~bash
$ cat <<EOF >spectrum-br-flows-machineconfig.yaml
kind: MachineConfig
apiVersion: machineconfiguration.openshift.io/v1
metadata:
  name: worker-spectrum-br-flow-systemd-service
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - name: spectrum-br-flows.service
        enabled: true
        contents: |
          [Unit]
          Description=Adds the bridges and flows for Spectrum-X whenever openvswitch is started or reloaded 
          PartOf=openvswitch.service
          After=openvswitch.service
          [Service]
          RemainAfterExit=yes
          ExecStart=/etc/scripts/spectrum-br-flows.sh
          Type=oneshot
          [Install]
          WantedBy=multi-user.target
    storage:
      files:
      - filesystem: root
        path: "/etc/scripts/spectrum-br-flows.sh"
        contents:
          source: data:text/plain;charset=utf-8;base64,$BASE64_SCRIPT
          verification: {}
        mode: 0755
        overwrite: true
      - filesystem: root
        path: "/etc/scripts/spectrum-config-map"
        contents:
          source: data:text/plain;charset=utf-8;base64,$BASE64_MAP
          verification: {}
        mode: 0644
        overwrite: true
EOF
~~~

Apply the machineconfig to the cluster to install the service.

~~~bash
$ oc create -f spectrum-br-flows-machineconfig.yaml
machineconfig.machineconfiguration.openshift.io/worker-spectrum-br-flow-systemd-service created
~~~

~~~bash
$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-c619e7c9fe8956e4444d2afc6e846411   False     True       False      1              0                   0                     0                      9d
worker   rendered-worker-54885d6142f5d2cfb949378a4e7e9241   True      False      False      0              0                   0                     0                      9d
~~~

Once the `oc mcp` command comes back update we can validate the machine configuration applied appropriately with the following check in a debug pod on the node.

~~~bash
$ oc debug node/nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-36nvidiaengrdu2dcredhatcom-debug-lwdgf ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
Pod IP: 10.6.135.15
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# systemctl status spectrum-br-flows.service
● spectrum-br-flows.service - Adds the bridges and flows for Spectrum-X whenever openvswitch is started or reloaded
     Loaded: loaded (/etc/systemd/system/spectrum-br-flows.service; enabled; preset: disabled)
     Active: active (exited) since Sun 2025-11-02 17:24:14 UTC; 10min ago
   Main PID: 2316 (code=exited, status=0/SUCCESS)
        CPU: 49ms

Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip link set dev br-eth_rail1 up
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Adding the flows to the bridge br-eth_rail1...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.37 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,nw_dst=192.168.67.37 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,nw_dst=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.250 actions=output:eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,in_port=LOCAL,nw_dst=192.168.67.250/8 actions=output:eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Completed setting up all rail bridges and flows on nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com!
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com systemd[1]: Finished Adds the bridges and flows for Spectrum-X whenever openvswitch is started or reloaded.
~~~

~~~bash
[root@nvd-srv-30 core]# journalctl |grep spectrum-br-flows.sh
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Gathering the hostname...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Creating bridge on nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com for interface eth_rail0 and settings values...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Setting ovs-bridge external-ids to tor_ip for br-eth_rail0...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-vsctl set bridge br-eth_rail0 external-ids:rail_peer_ip=192.168.67.250
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Adding ip addresses to internal bridge br-eth_rail0...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip addr add 192.168.67.36/24 dev br-eth_rail0
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip addr add 192.168.67.1 dev br-eth_rail0
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Bringing up br-eth_rail0 port...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip link set dev br-eth_rail0 up
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Adding the flows to the bridge br-eth_rail0...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, arp,arp_tpa=192.168.67.36 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, arp,arp_tpa=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, ip,nw_dst=192.168.67.36 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, ip,nw_dst=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, arp,arp_tpa=192.168.67.250 actions=output:eth_rail0
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail0 cookie=0x1, ip,in_port=LOCAL,nw_dst=192.168.67.250/24 actions=output:eth_rail0
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Creating bridge on nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com for interface eth_rail1 and settings values...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Setting ovs-bridge external-ids to tor_ip for br-eth_rail1...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-vsctl set bridge br-eth_rail1 external-ids:rail_peer_ip=192.168.67.250
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Adding ip addresses to internal bridge br-eth_rail1...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip addr add 192.168.67.37/24 dev br-eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip addr add 192.168.67.1 dev br-eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Bringing up br-eth_rail1 port...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ip link set dev br-eth_rail1 up
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Adding the flows to the bridge br-eth_rail1...
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.37 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,nw_dst=192.168.67.37 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,nw_dst=192.168.67.1 actions=LOCAL
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, arp,arp_tpa=192.168.67.250 actions=output:eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: ovs-ofctl add-flow  br-eth_rail1 cookie=0x1, ip,in_port=LOCAL,nw_dst=192.168.67.250/24 actions=output:eth_rail1
Nov 02 17:24:14 nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[2316]: Completed setting up all rail bridges and flows on nvd-srv-36.nvidia.eng.rdu2.dc.redhat.com!
~~~

## Configure Spectrum-X CNI

TBD

## Validate Spectrum-X Topology

TBD
