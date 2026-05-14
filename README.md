# Spectrum-X Host Configuration on OpenShift

**Goal**: The goal of this document is to document completely the steps to getting a host ready for Spectrum-X.

## Workflow Sections

- [Environment](#environment)
- [Set Core User Password for Troubleshooting](#set-core-user-password-for-troubleshooting)
- [Set Hugepages and IOMMU off](#set-hugepages-and-iommu-off)
- [Set UDEV Rules for Rail Device Names](#set-udev-rules-for-rail-device-names)
- [Configuring NFD Operator](#configuring-nfd-operator)
- [Configuring SRIOV Operator](#configuring-sriov-operator)
- [Configuring NMState Operator](#configuring-nmstate-operator)
- [Configuring NVIDIA Network Operator](#configuring-nvidia-network-operator)
- [Configuring NVIDIA Maintenance Operator](#configuring-nvidia-maintenance-operator)
- [Configuring Nic Firmware](#configuring-nic-firmware)
- [Configuring NVIDIA GPU Operator](#configuring-nvidia-network-operator)
- [Configuring LLDPD Daemonset](#configuring-lldpd-daemonset)
- [Configuring OVS Offload](#configuring-ovs-offload)
- [Configure Physical Rail Interface Attributes](#configure-physical-rail-interface-attributes)
- [Configure MTU](#configure-MTU)
- [Configure Spectrum-X CNI](#configure-spectrum-x-CNI)
- [Solve missing kernel modules in NIC Configuration Daemon](#Solve-missing-kernel-modules-in-NIC-Configuration-Daemon)
- [Validate Spectrum-X Topology](#validate-spectrum-x-topology)
- [Performance Testing and Troubleshooting](#performance-testing-and-troubleshooting)


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

## Set Hugepages and IOMMU off

We need to set hughpages and iommu.  This can be achieved with the following machine configuration.

~~~bash
$ cat <<EOF > 99-machineconfig-nvd-srv-36.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-master-nvd-srv-36
spec:
  kernelArguments:
    - default_hugepagesz=1G
    - hugepagesz=1G
    - hugepages=16
    - intel_iommu=on
    - iommu=pt
    - pci=realloc=on
EOF
~~~

Create the machine configuration on the cluster.  This will cause nodes to reboot in a rolling fashion and can be monitored with `oc get mcp`.

~~~bash
$ oc create -f 99-machineconfig-nvd-srv-36.yaml
machineconfig.machineconfiguration.openshift.io/99-master-nvd-srv-36 created
~~~

To validate this has been configured we can use `dmesg` output and `oc describe node` to see iommu and hughpages are set or even better with `cat /proc/cmdline` in each node.

## Set RDMA Subsystem Namespace Awareness

enable RDMA device namespace separation, which is essential for proper resource isolation in containerized environments.

~~~bash
IB_CORE=$(echo "options ib_core netns_mode=0" | base64 -w 0)
~~~

~~~bash
cat <<EOF > 99-worker-ib-core-netns.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-ib-core-netns
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;base64,${IB_CORE}
        mode: 420
        overwrite: true
        path: /etc/modprobe.d/ib_core.conf
EOF
~~~

Create the machine configuration on the cluster.  This will cause nodes to reboot in a rolling fashion and can be monitored with `oc get mcp`.

~~~bash
oc create -f 99-worker-ib-core-netns.yaml 
machineconfig.machineconfiguration.openshift.io/99-worker-ib-core-netns created
~~~

## Set UDEV Rules for Rail Device Names

We need to use udev rules to normalize the interface names on each node to eth_rail or roce_rail for ethernet and infiniband respectively.   We can do this by creating a persistent udev rules file indicating the devices which should be the same across all worker nodes in a homogenous cluster.   Below is an example of what that file might look like.  On the GH200 node we only have two interfaces to work with but it gives you an idea.

~~~bash
$ cat <<EOF > 70-persistent-net.rules 
ACTION=="add", KERNELS=="0000:18:00.0", SUBSYSTEM=="net", NAME="eth_rail0"
ACTION=="add", KERNELS=="0000:3a:00.0", SUBSYSTEM=="net", NAME="eth_rail1"
ACTION=="add", KERNELS=="0000:4d:00.0", SUBSYSTEM=="net", NAME="eth_rail2"
ACTION=="add", KERNELS=="0000:5d:00.0", SUBSYSTEM=="net", NAME="eth_rail3"
ACTION=="add", KERNELS=="0000:9b:00.0", SUBSYSTEM=="net", NAME="eth_rail4"
ACTION=="add", KERNELS=="0000:ba:00.0", SUBSYSTEM=="net", NAME="eth_rail5"
ACTION=="add", KERNELS=="0000:ca:00.0", SUBSYSTEM=="net", NAME="eth_rail6"
ACTION=="add", KERNELS=="0000:db:00.0", SUBSYSTEM=="net", NAME="eth_rail7"

ACTION=="add", KERNELS=="0000:18:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail0"
ACTION=="add", KERNELS=="0000:3a:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail1"
ACTION=="add", KERNELS=="0000:4d:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail2"
ACTION=="add", KERNELS=="0000:5d:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail3"
ACTION=="add", KERNELS=="0000:9b:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail4"
ACTION=="add", KERNELS=="0000:ba:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail5"
ACTION=="add", KERNELS=="0000:ca:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail6"
ACTION=="add", KERNELS=="0000:db:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail7"

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
  configDaemonNodeSelector:
    feature.node.kubernetes.io/pci-15b3.sriov.capable: 'true'
    network.nvidia.com/operator.mofed.wait: 'false'
    node-role.kubernetes.io/worker: ''
  enableInjector: true
  enableOperatorWebhook: true
  featureGates:
    manageSoftwareBridges: true
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

At this point we can move onto the next section of the document.

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
  nicConfigurationOperator:
    configurationDaemon:
      containerResources:
      - limits:
          cpu: 1000m
          memory: 10000Mi
        name: nic-configuration-daemon
        requests:
          cpu: 1000m
          memory: 10000Mi
      image: nic-configuration-operator-daemon
      repository: nvcr.io/nvidia/mellanox
      version: network-operator-v26.1.0
    logLevel: debug
    operator:
      containerResources:
      - limits:
          cpu: 1000m
          memory: 10000Mi
        name: nic-configuration-operator
        requests:
          cpu: 1000m
          memory: 10000Mi
      image: nic-configuration-operator
      repository: nvcr.io/nvidia/mellanox
      version: network-operator-v26.1.0
  nvIpam:
    image: nvidia-k8s-ipam
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.1.0
    enableWebhook: false
  ofedDriver:
    env:
    - name: UNLOAD_STORAGE_MODULES
      value: "true"
    forcePrecompiled: false
    image: doca-driver
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
    repository: nvcr.io/nvstaging/mellanox
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 20
    terminationGracePeriodSeconds: 300
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: true
        enable: true
        force: true
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      safeLoad: false
    version: doca3.3.0-26.01-0.4.6.0-1
  spectrumXOperator:
    image: spectrum-x-operator
    repository: nvcr.io/nvidia/mellanox
    version: network-operator-v26.1.0
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

## Configuring Nic Firmware

NVIDIA NIC Configuration Operator provides Kubernetes API(Custom Resource Definition) to allow FW configuration on Nvidia NICs in a coordinated manner. It deploys, based on settings in the NicClusterPolicy, a configuration daemon on each of the desired nodes to configure Nvidia NICs there. NVIDIA NIC Configuration Operator uses the Maintenance Operator to prepare a node for maintenance before the actual configuration.

We need to define the following NicFirmwareSource file.  This tells the operator where to get the firmware. for RA2.1 you don't need to specify the doca spcx CC. It's built into the nic daemon image

~~~bash
$ cat <<EOF > fwsource.yaml
apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicFirmwareSource
metadata:
  name: spc-x-doca-pcc
  namespace: nvidia-network-operator
spec:
  bfbUrlSource: https://content.mellanox.com/BlueField/FW-Bundle/bf-fwbundle-3.3.0-202_26.01-prod.bfb 
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
  name: spc-x-config
  namespace: nvidia-network-operator
spec:
  nicSelector:
    nicType: a2dc
    pciAddresses:
    - 0000:18:00.0
    - 0000:3a:00.0
    - 0000:4d:00.0
    - 0000:5d:00.0
    - 0000:9b:00.0
    - 0000:ba:00.0
    - 0000:ca:00.0
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
      version: RA2.1
EOF
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
$ oc rsh -n nvidia-gpu-operator $(oc -n nvidia-gpu-operator get pod -o name -l app.kubernetes.io/component=nvidia-driver | head -1)
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

Pay Attention !! according to the RA2.1 the externallyManaged need to be true. when applying set it to false , because in other case the vf will be remain at 0 and eSwitchMode will remain at legacy ! 

Give it some time and change back the externallyManaged to true . we can use patching for that for example:
~~~bash
for p in $(kubectl get sriovnetworknodepolicy -n openshift-sriov-network-operator -o name); do echo "Patching $p..."; kubectl patch $p -n openshift-sriov-network-operator --type merge -p '{"spec":{"externallyManaged":true}}' 2>&1; done
Patching sriovnetworknodepolicy.sriovnetwork.openshift.io/eth-rail0-node2...
sriovnetworknodepolicy.sriovnetwork.openshift.io/eth-rail0-node2 patched
Patching sriovnetworknodepolicy.sriovnetwork.openshift.io/eth-rail0-node3...
sriovnetworknodepolicy.sriovnetwork.openshift.io/eth-rail0-node3 patched
.
.
~~~

~~~bash
$ cat <<EOF > snnp-eth-rail0.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: eth-rail0
  namespace: openshift-sriov-network-operator
spec:
  bridge:
    ovs: {}
  deviceType: netdevice
  eSwitchMode: switchdev
  isRdma: true
  linkType: ETH
  mtu: 9216
  nicSelector:
    pfNames:
    - eth_rail0
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  numVfs: 1
  priority: 99
  resourceName: eth_rail0
EOF
~~~


Once we generate the SriovNetworkNodePolicy we can create it on the cluster.  We will repeat this for each rail.

~~~bash
$ oc apply -f snnp-eth-rail0.yaml
sriovnetworknodepolicy.sriovnetwork.openshift.io/eth-rail0 created
~~~

We can validate the configuration by using a `debug` pod and checking the values.  In this example we used emp55s0np0 as our test interface.

~~~bash
$ oc debug node/dell-h200-2 -- chroot /host bash -c 'for i in 0 1 2 3 4 5 6 7; do PCI=$(readlink -f /sys/class/net/eth_rail${i}/device | xargs basename); MODE=$(devlink dev eswitch show pci/$PCI 2>&1); echo "eth_rail${i} ($PCI): $MODE"; done' 2>&1
Starting pod/dell-h200-2-debug-jjc96 ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
eth_rail0 (0000:18:00.0): pci/0000:18:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail1 (0000:3a:00.0): pci/0000:3a:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail2 (0000:4d:00.0): pci/0000:4d:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail3 (0000:5d:00.0): pci/0000:5d:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail4 (0000:9b:00.0): pci/0000:9b:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail5 (0000:ba:00.0): pci/0000:ba:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail6 (0000:ca:00.0): pci/0000:ca:00.0: mode switchdev inline-mode none encap-mode basic
eth_rail7 (0000:db:00.0): pci/0000:db:00.0: mode switchdev inline-mode none encap-mode basic

Removing debug pod ...
~~~

This completes the physical interface confguration section.

## Configure MTU
The MTU part of configuration could be done using NMState Policy, The following example configures port eth_rail0.

No need to configure ovs flows, The SPX Operator configures the OVS flows.

~~~bash
$ cat <<EOF > nncp-mtu-rail0.yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: nncp-mtu-rail0
spec:
  desiredState:
    interfaces:
    - description: mtu 9216 eth_rail0
      mtu: 9216
      name: eth_rail0
      state: up
      type: ethernet
  nodeSelector:
    node-role.kubernetes.io/worker: ""
EOF

~~~
Create the NodeNetworkConfigurationPolicy on the cluster

~~~bash
$ oc create -f nncp-mtu-rail0.yaml
~~~ 


## Configure Spectrum-X CNI

The next thing we need to configure is the Spectrum-X CNI.  This is composed of two components: nv-ipam CIDRPool and an OVSNetwork.   These configuration files will need to be created for each rail interface in the system.   

The nv-ipam CIDRPool looks similar to the following example.  Please adjust the ipaddressing and nodeName information to match the environment and create one for each rail interface.

~~~bash
$ cat <<EOF > cidrpool_rail0.yaml
apiVersion: nv-ipam.nvidia.com/v1alpha1
kind: CIDRPool
metadata:
  name: eth-rail0
  namespace: nvidia-network-operator
spec:
  cidr: 172.16.0.0/15
  gatewayIndex: 0
  perNodeNetworkPrefix: 31
  routes:
  - dst: 172.16.0.0/15
  - dst: 172.16.0.0/12
  staticAllocations:
  - gateway: 172.16.0.5
    nodeName: dell-h200-2
    prefix: 172.16.0.4/31
  - gateway: 172.16.0.7
    nodeName: dell-h200-3
    prefix: 172.16.0.6/31
EOF
~~~

The OVSNetwork custom resource file looks similar to the following example.   Each rail will require a OVSNetwork configuration.

~~~bash
apiVersion: sriovnetwork.openshift.io/v1
kind: OVSNetwork
metadata:
  name: eth-rail0
  namespace: openshift-sriov-network-operator
spec:
  ipam: |
    {
      "type": "nv-ipam",
      "poolName": "eth-rail0",
      "poolType": "cidrpool"
    }
  metaPlugins: |
    {
      "type": "rdma"
    }
  networkNamespace: default
  resourceName: eth_rail0
~~~
## Solve missing kernel modules in NIC Configuration Daemon 

There is an issue that we can see in NIC Configuration Daemon logs, that fwctl.ko and mlx5_fwctl.ko modules arn't loaded on the worker nods in order for dms client commands to work ,In order to solve it we should load them using machine       config 

~~~bash
$ cat <<EOF > mc-load-fwctl.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-load-fwctl
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - name: load-fwctl.service
        enabled: false
        contents: |
          [Unit]
          Description=Load fwctl kernel modules from MOFED container
          After=crio.service kubelet.service

          [Service]
          Type=oneshot
          RemainAfterExit=yes
          Environment=CONTAINER_RUNTIME_ENDPOINT=unix:///run/crio/crio.sock
          ExecStart=/bin/bash -c '\
            if lsmod | grep -q mlx5_fwctl; then echo "fwctl already loaded"; exit 0; fi; \
            CID=$$(crictl ps --name mofed-container --state running -q 2>/dev/null | head -1); \
            if [ -z "$$CID" ]; then echo "MOFED container not found"; exit 1; fi; \
            echo "Found MOFED container: $$CID"; \
            KERN=$$(uname -r); \
            MOD=/lib/modules/$${KERN}/extra/mlnx-ofa_kernel/drivers/fwctl; \
            crictl exec $$CID insmod $${MOD}/fwctl.ko; \
            crictl exec $$CID insmod $${MOD}/mlx5/mlx5_fwctl.ko; \
            lsmod | grep -q mlx5_fwctl && echo "fwctl modules loaded successfully" || { echo "Failed to load fwctl"; exit 1; }'

          [Install]
          WantedBy=multi-user.target
      - name: load-fwctl.timer
        enabled: true
        contents: |
          [Unit]
          Description=Timer to load fwctl kernel modules after MOFED is ready

          [Timer]
          OnBootSec=90s
          OnUnitInactiveSec=60s

          [Install]
          WantedBy=timers.target
EOF
~~~
Create the machine config on the cluster
~~~bash
$ oc create -f mc-load-fwctl.yaml
~~~
We can monitor the progress of setting the core user password MachineConfig using the `oc get mcp` command.

## Validate Spectrum-X Topology

Once we believe we have all the configurations in place for Spectrum-X on OpenShift and at the Spectrum switch level we should then start with validations.   These validations will include using the rcp-tool validate topology and switch configurations along with an NCCL test run.

To do the topology validation we can run the following rcp-tool command.   

~~~bash
rcp-tool topology validate
~~~

To validate the Spectrum switch configuration we can run the following rcp-tool command.

~~~bash
rcp-tool validation switch-config
~~~

To validate the OpenShift host side configuration we should run the following rcp-tool command.

~~~bash
rcp-tool validation host-config
~~~ 

To initiate ping checks in the topology we can use the following rcp-tool command.

~~~bash
rcp-tool validation ping
~~~

If all of the rcp-tool validations check out we can move onto performance testing.

## Performance Testing and Troubleshooting

In this section we are going to set up the cluster to run a performance NCCL job.   This job will run in two pods across two worker nodes and use the NVIDIA Tooling image.   We will setup the required privileges, create the workload pod, validate connectivity between the two hosts on the infinband fabric and then run a performance test.

First let's generate a service account CRD to use in the `default` namespace.

~~~bash
$ cat <<EOF > default-serviceaccount.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rdma
  namespace: default
EOF
~~~

Next we can create it on our cluster.

~~~bash
$ oc create -f default-serviceaccount.yaml 
serviceaccount/rdma created
~~~

Finally with the service account create we can add privleges to it.

~~~bash
$ oc -n default adm policy add-scc-to-user privileged -z rdma
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "rdma"
~~~

With the service account setup we now need to create a workload pod that contains all the tooling for our testing.  We can generate a custom pod CRD as follows to meet that requirement.

~~~bash
$ cat <<EOF > nvidiatools-dell-h200-3-workload.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nvidiatools-dell-h200-3-workload
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: eth-rail0, eth-rail1, eth-rail2, eth-rail3, eth-rail4, eth-rail5, eth-rail6, eth-rail7
spec:
  serviceAccountName: rdma
  nodeSelector:
    kubernetes.io/hostname: dell-h200-3
  volumes:
    - name: shmem
      emptyDir: {
          medium: 'Memory',
          sizeLimit: '16Gi'
      }
  containers:
    - name: nvidiatools-dell-h200-3-workload
      image: quay.io/redhat_emp1/ecosys-nvidia/nvidia-tools:0.2.2
      imagePullPolicy: IfNotPresent
      securityContext:
        privileged: true
        capabilities:
          add: ["IPC_LOCK"]
      resources:
        limits:
          nvidia.com/gpu: 8
          openshift.io/eth_rail0: "1"
          openshift.io/eth_rail1: "1"
          openshift.io/eth_rail2: "1"
          openshift.io/eth_rail3: "1"
          openshift.io/eth_rail4: "1"
          openshift.io/eth_rail5: "1"
          openshift.io/eth_rail6: "1"
          openshift.io/eth_rail7: "1"
        requests:
          nvidia.com/gpu: 8
          openshift.io/eth_rail0: "1"
          openshift.io/eth_rail1: "1"
          openshift.io/eth_rail2: "1"
          openshift.io/eth_rail3: "1"
          openshift.io/eth_rail4: "1"
          openshift.io/eth_rail5: "1"
          openshift.io/eth_rail6: "1"
          openshift.io/eth_rail7: "1"
      volumeMounts:
        - mountPath: /dev/shm
          name: shmem
EOF

$ cat <<EOF > nvidiatools-dell-h200-2-workload.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nvidiatools-dell-h200-2-workload
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: eth-rail0, eth-rail1, eth-rail2, eth-rail3, eth-rail4, eth-rail5, eth-rail6, eth-rail7
spec:
  serviceAccountName: rdma
  nodeSelector:
    kubernetes.io/hostname: dell-h200-2
  volumes:
    - name: shmem
      emptyDir: {
          medium: 'Memory',
          sizeLimit: '16Gi'
      }
  containers:
    - name: nvidiatools-dell-h200-2-workload
      image: quay.io/redhat_emp1/ecosys-nvidia/nvidia-tools:0.2.2
      imagePullPolicy: IfNotPresent
      securityContext:
        privileged: true
        capabilities:
          add: ["IPC_LOCK"]
      resources:
        limits:
          nvidia.com/gpu: 8
          openshift.io/eth_rail0: "1"
          openshift.io/eth_rail1: "1"
          openshift.io/eth_rail2: "1"
          openshift.io/eth_rail3: "1"
          openshift.io/eth_rail4: "1"
          openshift.io/eth_rail5: "1"
          openshift.io/eth_rail6: "1"
          openshift.io/eth_rail7: "1"
        requests:
          nvidia.com/gpu: 8
          openshift.io/eth_rail0: "1"
          openshift.io/eth_rail1: "1"
          openshift.io/eth_rail2: "1"
          openshift.io/eth_rail3: "1"
          openshift.io/eth_rail4: "1"
          openshift.io/eth_rail5: "1"
          openshift.io/eth_rail6: "1"
          openshift.io/eth_rail7: "1"
      volumeMounts:
        - mountPath: /dev/shm
          name: shmem
EOF
~~~

Then we can create the pods on the cluster.

~~~bash
$ oc create -f nvidiatools-dell-h200-3-workload.yaml
pod/nvidiatools-dell-h200-3-workload created

$ oc create -f nvidiatools-dell-h200-2-workload.yaml
pod/nvidiatools-dell-h200-2-workload created
~~~

Let's validate the pods is running.

~~~bash
$ oc get pods
NAME                               READY   STATUS    RESTARTS   AGE
nvidiatools-dell-h200-2-workload   1/1     Running   0          9s
nvidiatools-dell-h200-3-workload   1/1     Running   0          18s
~~~

Now open two terminals and rsh into each of the pods.  We need to grab the ip addresses to populate the `mpirun` command with the ip addresses of the pods.

~~~bash
$ oc rsh nvidiatools-dell-h200-3-workload
sh-5.1# source ./.bashrc
[root@nvidiatools-dell-h200-3-workload ~]#

$ oc rsh nvidiatools-dell-h200-2-workload
sh-5.1# source ./.bashrc
[root@nvidiatools-dell-h200-2-workload ~]#
~~~

We need to grab the ip addresses to populate the `mpirun` command with the ip addresses of the pods.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1
10.129.2.27

[root@nvidiatools-dell-h200-3-workload ~]# ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1
10.128.2.44
~~~

Now let's run the following command (update the ip addresses based on the output from previous command) in one of the pods.  The command will use ssh on port 20024 to connect back to both the pod where the command is executed and the other pod on the remote worker.

~~~bash
mpirun --allow-run-as-root \
-H 10.129.2.27:1,10.128.2.44:1 \
-np 2 \
-bind-to none -map-by slot \
-x NCCL_IB_GID_INDEX=3 \
-x NCCL_IB_TC=96 \
-x NCCL_IB_HCA=mlx5_ \
-x NCCL_IB_ADAPTIVE_ROUTING=1 \
-x NCCL_IB_SPLIT_DATA_ON_QPS=0 \
-x NCCL_IBEXT_DISABLE=0 \
-x UCX_IB_GID_INDEX=3 \
-x UCX_TLS=tcp \
-x UCC_CLS=basic \
-x UCC_TLS=ucp,self,sm \
-x UCC_TL_NCCL_TUNE=0 \
-x UCC_TL_UCP_TUNE=allgather:@0 \
-x HYDRA_FULL_ERROR=1 \
-x NCCL_DEBUG=warn \
-x NCCL_IB_QPS_PER_CONNECTION=2 \
-x NCCL_P2P_NET_CHUNKSIZE=524288 \
-x NCCL_MIN_NCHANNELS=32 \
-mca btl tcp,self \
-mca btl_tcp_if_include net1,net2,net3,net4,net5,net6,net7,net8 \
-mca plm_rsh_args "-p 20024" \
all_reduce_perf --ngpus 8 \
--minbytes 1k --maxbytes 16G --stepfactor 2 \
--stepbytes 1M --op sum --datatype float --root 0 \
--iters 100 --warmup_iters 50 \
--agg_iters 1 --average 1 --parallel_init 0 --check 1 --blocking 0 --cudagraph 0
~~~

When the command is run above the following will be displayed on the terminal.  The busbw number is what we are interested in.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# mpirun --allow-run-as-root \
-H 10.129.2.27:1,10.128.2.44:1 \
-np 2 \
-bind-to none -map-by slot \
-x NCCL_IB_GID_INDEX=3 \
-x NCCL_IB_TC=96 \
-x NCCL_IB_HCA=mlx5_ \
-x NCCL_IB_ADAPTIVE_ROUTING=1 \
-x NCCL_IB_SPLIT_DATA_ON_QPS=0 \
-x NCCL_IBEXT_DISABLE=0 \
-x UCX_IB_GID_INDEX=3 \
-x UCX_TLS=tcp \
-x UCC_CLS=basic \
-x UCC_TLS=ucp,self,sm \
-x UCC_TL_NCCL_TUNE=0 \
-x UCC_TL_UCP_TUNE=allgather:@0 \
-x HYDRA_FULL_ERROR=1 \
-x NCCL_DEBUG=warn \
-x NCCL_IB_QPS_PER_CONNECTION=2 \
-x NCCL_P2P_NET_CHUNKSIZE=524288 \
-x NCCL_MIN_NCHANNELS=32 \
-mca btl tcp,self \
-mca btl_tcp_if_include net1,net2,net3,net4,net5,net6,net7,net8 \
-mca plm_rsh_args "-p 20024" \
all_reduce_perf --ngpus 8 \
--minbytes 1k --maxbytes 16G --stepfactor 2 \
--stepbytes 1M --op sum --datatype float --root 0 \
--iters 100 --warmup_iters 50 \
--agg_iters 1 --average 1 --parallel_init 0 --check 1 --blocking 0 --cudagraph 0
Warning: Permanently added '[10.128.2.44]:20024' (ED25519) to the list of known hosts.
[1777406131.465474] [nvidiatools-dell-h200-2-workload:10204:0]          parser.c:2305 UCX  WARN  unused environment variable: UCX_IB_GID_INDEX
[1777406131.465474] [nvidiatools-dell-h200-2-workload:10204:0]          parser.c:2305 UCX  WARN  (set UCX_WARN_UNUSED_ENV_VARS=n to suppress this warning)
[1777406131.464522] [nvidiatools-dell-h200-3-workload:9750 :0]          parser.c:2305 UCX  WARN  unused environment variable: UCX_IB_GID_INDEX
[1777406131.464522] [nvidiatools-dell-h200-3-workload:9750 :0]          parser.c:2305 UCX  WARN  (set UCX_WARN_UNUSED_ENV_VARS=n to suppress this warning)
# nccl-tests version 2.18.3 nccl-headers=23004 nccl-library=23004
# Collective test starting: all_reduce_perf
# nThread 1 nGpus 8 minBytes 1024 maxBytes 17179869184 step: 2(factor) warmup iters: 50 iters: 100 agg iters: 1 validation: 1 graph: 0 unalign: 0
#
# Using devices
#  Rank  0 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  0 [0000:1b:00] NVIDIA H200
#  Rank  1 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  1 [0000:3c:00] NVIDIA H200
#  Rank  2 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  2 [0000:4b:00] NVIDIA H200
#  Rank  3 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  3 [0000:5c:00] NVIDIA H200
#  Rank  4 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  4 [0000:9a:00] NVIDIA H200
#  Rank  5 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  5 [0000:bb:00] NVIDIA H200
#  Rank  6 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  6 [0000:cd:00] NVIDIA H200
#  Rank  7 Group  0 Pid  10204 on nvidiatools-dell-h200-2-workload device  7 [0000:dc:00] NVIDIA H200
#  Rank  8 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  0 [0000:1b:00] NVIDIA H200
#  Rank  9 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  1 [0000:3c:00] NVIDIA H200
#  Rank 10 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  2 [0000:4b:00] NVIDIA H200
#  Rank 11 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  3 [0000:5c:00] NVIDIA H200
#  Rank 12 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  4 [0000:9a:00] NVIDIA H200
#  Rank 13 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  5 [0000:bb:00] NVIDIA H200
#  Rank 14 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  6 [0000:cd:00] NVIDIA H200
#  Rank 15 Group  0 Pid   9750 on nvidiatools-dell-h200-3-workload device  7 [0000:dc:00] NVIDIA H200
NCCL version 2.30.4+cuda13.2
#
#                                                              out-of-place                       in-place          
#       size         count      type   redop    root     time   algbw   busbw  #wrong     time   algbw   busbw  #wrong 
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)             (us)  (GB/s)  (GB/s)         
        1024           256     float     sum      -1    45.06    0.02    0.04       0    43.74    0.02    0.04       0
        2048           512     float     sum      -1    43.84    0.05    0.09       0    43.70    0.05    0.09       0
        4096          1024     float     sum      -1    43.51    0.09    0.18       0    44.16    0.09    0.17       0
        8192          2048     float     sum      -1    47.64    0.17    0.32       0    47.41    0.17    0.32       0
       16384          4096     float     sum      -1    53.65    0.31    0.57       0    53.76    0.30    0.57       0
       32768          8192     float     sum      -1    64.93    0.50    0.95       0    64.98    0.50    0.95       0
       65536         16384     float     sum      -1    70.32    0.93    1.75       0    65.48    1.00    1.88       0
      131072         32768     float     sum      -1    81.24    1.61    3.03       0    83.01    1.58    2.96       0
      262144         65536     float     sum      -1    63.59    4.12    7.73       0    63.42    4.13    7.75       0
      524288        131072     float     sum      -1    80.78    6.49   12.17       0    79.93    6.56   12.30       0
     1048576        262144     float     sum      -1    76.86   13.64   25.58       0    77.16   13.59   25.48       0
     2097152        524288     float     sum      -1    86.37   24.28   45.53       0    86.03   24.38   45.71       0
     4194304       1048576     float     sum      -1   106.13   39.52   74.10       0   105.58   39.73   74.48       0
     8388608       2097152     float     sum      -1   149.76   56.02  105.03       0   149.06   56.28  105.52       0
    16777216       4194304     float     sum      -1   304.32   55.13  103.37       0   207.61   80.81  151.52       0
    33554432       8388608     float     sum      -1   276.88  121.19  227.23       0   277.00  121.13  227.13       0
    67108864      16777216     float     sum      -1   461.01  145.57  272.94       0   460.78  145.64  273.08       0
   134217728      33554432     float     sum      -1   742.13  180.85  339.10       0   747.72  179.50  336.57       0
   268435456      67108864     float     sum      -1  1275.21  210.50  394.69       0  1278.78  209.92  393.59       0
   536870912     134217728     float     sum      -1  2327.12  230.70  432.57       0  2322.37  231.17  433.45       0
  1073741824     268435456     float     sum      -1  4388.55  244.67  458.75       0  4385.35  244.85  459.09       0
  2147483648     536870912     float     sum      -1  8497.37  252.72  473.86       0  8501.88  252.59  473.60       0
  4294967296    1073741824     float     sum      -1  16715.2  256.95  481.78       0  16716.5  256.93  481.74       0
  8589934592    2147483648     float     sum      -1  33169.3  258.97  485.57       0  33208.0  258.67  485.01       0
 17179869184    4294967296     float     sum      -1  66326.9  259.02  485.66       0  66345.5  258.95  485.52       0
# Out of bounds values : 0 OK
# Avg bus bandwidth    : 178.222 
#
# Collective test concluded: all_reduce_perf
#
~~~

If everything checks out we should have a peek busbw over the actual line rate.  In our cluster the line rate is 400GB/s and from the results above we can see the last few iterations were above ~480G/s.   However if those numbers are lower for example below ~400GB/s then there are a few things we can check to ensure we have our setup configured correctly.

First check that ATS_ENABLED is set to zero which implies disabled.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# for device in `mst status -v|grep BlueField|awk {'print $3'}` ; do mlxconfig -d $device --yes q ATS_ENABLED; done

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             cc:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             ba:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             3a:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3B6-00CV-A_Ax 
Description:        NVIDIA BlueField-3 B3220 P-Series FHHL DPU; 200GbE (default mode) / NDR200 IB; Dual-port QSFP112; PCIe Gen5.0 x16 with x16 PCIe extension option; 16 Arm cores; 32GB on-board DDR; integrated BMC; Crypto Enabled
Device:             bc:00.1             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             5d:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             ca:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3B6-00CV-A_Ax 
Description:        NVIDIA BlueField-3 B3220 P-Series FHHL DPU; 200GbE (default mode) / NDR200 IB; Dual-port QSFP112; PCIe Gen5.0 x16 with x16 PCIe extension option; 16 Arm cores; 32GB on-board DDR; integrated BMC; Crypto Enabled
Device:             5f:00.1             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             db:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             18:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             1a:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             9b:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3B6-00CV-A_Ax 
Description:        NVIDIA BlueField-3 B3220 P-Series FHHL DPU; 200GbE (default mode) / NDR200 IB; Dual-port QSFP112; PCIe Gen5.0 x16 with x16 PCIe extension option; 16 Arm cores; 32GB on-board DDR; integrated BMC; Crypto Enabled
Device:             bc:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3B6-00CV-A_Ax 
Description:        NVIDIA BlueField-3 B3220 P-Series FHHL DPU; 200GbE (default mode) / NDR200 IB; Dual-port QSFP112; PCIe Gen5.0 x16 with x16 PCIe extension option; 16 Arm cores; 32GB on-board DDR; integrated BMC; Crypto Enabled
Device:             5f:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)            

Device #1:
----------

Device type:        BlueField3          
Name:               900-9D3D4-00EN-HA0_Ax
Description:        Nvidia BlueField-3 B3140H E-series HHHL SuperNIC; 400GbE (default mode) / NDR IB; Single-port QSFP112; PCIe Gen5.0 x16; 8 Arm cores; 16GB on board DDR; integrated BMC; Crypto Enabled
Device:             4d:00.0             

Configurations:                                          Next Boot
        ATS_ENABLED                                 False(0)   
~~~

Next check that the interfaces have the MaxReadReq set to 4k.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# lspci -vv -d 15b3:a2dc| grep -i MaxReadReq
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
			MaxPayload 256 bytes, MaxReadReq 4096 bytes
~~~

Finally make sure that ACS is disabled.   We can check it by running the following command.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# lspci -vv | grep -i "acsctl"
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
		ACSCtl:	SrcValid- TransBlk- ReqRedir- CmpltRedir- UpstreamFwd- EgressCtrl- DirectTrans-
~~~

If any values have a `+` symbol next to the word then ACS might be enabled.  To disable permanently, at least on Dell systems go into the BIOS and disable virtualization.   As a temporary fix we can use the following script to disable ACS as well.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# cat <<EOF > disable-acs.sh 
#!/bin/bash
# must be root to access extended PCI config space
if [ "$EUID" -ne 0 ]; then
  echo "ERROR: $0 must be run as root"
  exit 1
fi
 
for BDF in `lspci -d "*:*:*" | awk '{print $1}'`; do
 
    # skip if it doesn't support ACS
    setpci -v -s ${BDF} ECAP_ACS+0x6.w > /dev/null 2>&1
    if [ $? -ne 0 ]; then
            echo "${BDF} does not support ACS, skipping"
            continue
    fi
 
    echo "Disabling ACS on ${BDF}"
    setpci -v -s ${BDF} ECAP_ACS+0x6.w=0000
 
done
exit 0
EOF
~~~

Set the execute bit on the script and run it.  Note this needs to be done on all nodes.

~~~bash
[root@nvidiatools-dell-h200-2-workload ~]# chmod +x disable-acs.sh 
[root@nvidiatools-dell-h200-2-workload ~]# ./disable-acs.sh |grep Disabling
Disabling ACS on 00:0a.0
Disabling ACS on 00:0c.0
Disabling ACS on 00:0e.0
Disabling ACS on 15:01.0
Disabling ACS on 17:00.0
Disabling ACS on 17:01.0
Disabling ACS on 17:02.0
Disabling ACS on 17:03.0
Disabling ACS on 17:1f.0
Disabling ACS on 18:00.0
Disabling ACS on 18:00.1
Disabling ACS on 1a:00.0
Disabling ACS on 1a:00.1
Disabling ACS on 37:01.0
Disabling ACS on 39:00.0
Disabling ACS on 39:01.0
Disabling ACS on 39:02.0
Disabling ACS on 3a:00.0
Disabling ACS on 3a:00.1
Disabling ACS on 48:01.0
Disabling ACS on 4a:00.0
Disabling ACS on 4a:01.0
Disabling ACS on 4a:02.0
Disabling ACS on 4d:00.0
Disabling ACS on 4d:00.1
Disabling ACS on 59:01.0
Disabling ACS on 5b:00.0
Disabling ACS on 5b:01.0
Disabling ACS on 5b:02.0
Disabling ACS on 5b:03.0
Disabling ACS on 5b:1f.0
Disabling ACS on 5d:00.0
Disabling ACS on 5d:00.1
Disabling ACS on 5f:00.0
Disabling ACS on 5f:00.1
Disabling ACS on 5f:00.2
Disabling ACS on 80:01.0
Disabling ACS on 81:00.0
Disabling ACS on 81:00.1
Disabling ACS on 82:00.0
Disabling ACS on 82:01.0
Disabling ACS on 82:02.0
Disabling ACS on 82:03.0
Disabling ACS on 97:01.0
Disabling ACS on 99:00.0
Disabling ACS on 99:01.0
Disabling ACS on 99:02.0
Disabling ACS on 99:1f.0
Disabling ACS on 9b:00.0
Disabling ACS on 9b:00.1
Disabling ACS on b7:01.0
Disabling ACS on b9:00.0
Disabling ACS on b9:01.0
Disabling ACS on b9:02.0
Disabling ACS on b9:03.0
Disabling ACS on ba:00.0
Disabling ACS on ba:00.1
Disabling ACS on bc:00.0
Disabling ACS on bc:00.1
Disabling ACS on bc:00.2
Disabling ACS on c7:01.0
Disabling ACS on c9:00.0
Disabling ACS on c9:01.0
Disabling ACS on c9:02.0
Disabling ACS on c9:03.0
Disabling ACS on c9:1f.0
Disabling ACS on ca:00.0
Disabling ACS on ca:00.1
Disabling ACS on cc:00.0
Disabling ACS on cc:00.1
Disabling ACS on d7:01.0
Disabling ACS on d9:00.0
Disabling ACS on d9:01.0
Disabling ACS on d9:02.0
Disabling ACS on db:00.0
Disabling ACS on db:00.1
~~~

After all of this rerun the NCCL job.  If it still does not have performance numbers expected go back and review the entire configuration.




If everything works we should see near line speeds for the data transfer between GPU worker nodes.
