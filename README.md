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
- [Configuring Nic Firmware](#configuring-nic-firmware)
- [Configuring NVIDIA GPU Operator](#configuring-nvidia-network-operator)
- [Configuring LLDPD Daemonset](#configuring-lldpd-daemonset)
- [Configuring OVS Offload](#configuring-ovs-offload)
- [Configure Physical Rail Interface Attributes](#configure-physical-rail-interface-attributes)
- [Configuring Bridges and Flows](#configuring-bridges-and-flows)


## Environment

The host environment for this document was a single GH200 node running OpenShift 4.20.  The NFD, SR-IOV, NMState, NVIDIA Network, NVIDIA Maintenance and NVIDIA GPU operators were all installed but need to be configured.

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

~~~bash
$ oc create -f 99-machineconfig-nvd-srv-36.yaml
machineconfig.machineconfiguration.openshift.io/99-master-nvd-srv-36 created
~~~

## Set UDEV Rules for Rail Device Names

~~~bash
$ cat <<EOF > 70-persistent-net.rules 
ACTION=="add", KERNELS=="0002:01:00.0", SUBSYSTEM=="net", NAME="eth_rail0"
ACTION=="add", KERNELS=="0002:01:00.1", SUBSYSTEM=="net", NAME="eth_rail1"

ACTION=="add", KERNELS=="0002:01:00.0", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail0"
ACTION=="add", KERNELS=="0002:01:00.1", SUBSYSTEM=="infiniband", PROGRAM="rdma_rename %k NAME_FIXED roce_rail1"
EOF
~~~

~~~bash
$ UDEV_RULES=`cat 70-persistent-net.rules|base64 -w 0`
~~~

~~~bash
$ cat <<EOF > 99-machine-config-udev-network.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
   labels:
     machineconfiguration.openshift.io/role: master
   name: 99-machine-config-udev-network
spec:
   config:
     ignition:
       version: 3.2.0
     storage:
       files:
       - contents:
           source: data:text/plain;base64,$UDEV_RULES
         filesystem: root
         mode: 420
         path: /etc/udev/rules.d/70-persistent-net.rules
EOF
~~~

~~~bash
$ oc create -f 99-machine-config-udev-network.yaml
machineconfig.machineconfiguration.openshift.io/99-machine-config-udev-network created
~~~

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

~~~bash
$ oc create -f nfd-instance.yaml
nodefeaturediscovery.nfd.openshift.io/nfd-instance created
~~~

~~~bash
$ oc get pods -n openshift-nfd
NAME                                     READY   STATUS    RESTARTS   AGE
nfd-controller-manager-f4fc58bb4-k5fq2   1/1     Running   7          4d23h
nfd-gc-676786b846-6kpcz                  1/1     Running   6          4d23h
nfd-master-75fcc46558-b49jl              1/1     Running   6          4d23h
nfd-worker-l6jc6                         1/1     Running   7          6d5h
~~~

## Configuring SRIOV Operator

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

~~~bash
$ oc create -f sriov-operator-config.yaml 
sriovoperatorconfig.sriovnetwork.openshift.io/default created
~~~

~~~bash
$ oc get pods -n openshift-sriov-network-operator
NAME                                      READY   STATUS    RESTARTS   AGE
network-resources-injector-m4t8k          1/1     Running   0          49s
operator-webhook-w9k8n                    1/1     Running   0          49s
sriov-network-config-daemon-ql4cl         1/1     Running   0          49s
sriov-network-operator-5995bb94f6-qbsgd   1/1     Running   0          18m
~~~

~~~bash
oc patch sriovoperatorconfig default --type=merge -n openshift-sriov-network-operator --patch '{ "spec": { "configDaemonNodeSelector": { "network.nvidia.com/operator.mofed.wait": "false", "node-role.kubernetes.io/master": "", "feature.node.kubernetes.io/pci-15b3.sriov.capable": "true" } } }'
~~~

## Configuring NMState Operator

~~~bash
$ cat <<EOF > nmstate-instance.yaml 
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF
~~~

~~~bash
$ oc create -f nmstate-instance.yaml
nmstate.nmstate.io/nmstate created
~~~

~~~bash
$ oc get pods -n openshift-nmstate
NAME                                     READY   STATUS    RESTARTS   AGE
nmstate-console-plugin-8dcf98f9b-stx5x   0/1     Running   0          10s
nmstate-handler-zwhpt                    0/1     Running   0          10s
nmstate-metrics-767cb6c678-bx5ln         1/2     Running   0          10s
nmstate-operator-78777d7db8-5frht        1/1     Running   0          21m
nmstate-webhook-676f7797c4-h4nl6         0/1     Running   0          10s
~~~

## Configuring NVIDIA Network Operator

~~~bash
$ cat <<EOF > ncp-spectrumx.yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
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

## Configuring NVIDIA GPU Operator

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
~~~

~~~bash

~~~

## Enable Offloading in OVS

In OpenShift we can achieve OVS offload by creating an SriovNetworkPoolConfig and apply it to node.

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
  name: rail-x
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  eSwitchMode: "switchdev"
  mtu: 9216
  nicSelector:
    pfNames: ["eth_rail(x)"]
  numVfs: 1
  isRdma: true
  linkType: eth
  resourceName: rail-x
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
echo "Gathering the interfaces..." 
mapfile -t interfaces < <( ip link | awk -F': ' '/enp55/ && !/br/ {print $2}' )  # replace enp55 with Mellanox when done testing

# Instead of discovering the interfaces above I believe on a true rail system we will always have eth_rail[1-8].   I also believe we need to have some kind of topology awareness.   I believe when we create this service we probably need to point to a topology map file that we would then parse into the array.
# something like hostname:interface:ipaddress:tor_ipaddress
# then in the array below we would enumerate through split out the 4 values into variables and then if running on the right host execute the commands.

for i in "${interfaces[@]}"
do
#i="enp55s0np0"
  echo "Creating bridge for interface $i and settings values..."
  ovs-vsctl --may-exist add-br br-$i
  ovs-vsctl set bridge br-$i fail-mode=secure
  ovs-vsctl set bridge br-$i external-ids:rail_uplink=$i
  ovs-vsctl set Interface br-$i mtu_request=9216
  ovs-vsctl add-port br-$i $i
  ovs-vsctl set Interface $i mtu_request=9216

  echo "Setting ovs-bridge external-ids to tor_ip for br-$1..."
  #ovs-vsctl set bridge br-$i external-ids:rail_peer_ip={{rail_1_tor_ip}}

  echo "Adding ip addresses to internal bridge br-$i..."
  #ip addr add rail_1_host_ip/infra_rail_subnet dev br-$i
  #ip addr add rail_1_pod_gw_ip dev br-$i

  echo "Bringing up br-$i port..."
  ip link set dev br-$i up

  echo "Adding the flows to the bridge br-$i..."
  #ovs-ofctl add-flow  br-$i "cookie=0x1, arp,arp_tpa= rail_1_host_ip actions=LOCAL"
  #ovs-ofctl add-flow  br-$i "cookie=0x1, arp,arp_tpa= rail_1_pod_gw_ip actions=LOCAL"
  #ovs-ofctl add-flow  br-$i "cookie=0x1, ip,nw_dst= rail_1_host_ip actions=LOCAL"
  #ovs-ofctl add-flow  br-$i "cookie=0x1, ip,nw_dst= rail_1_pod_gw_ip actions=LOCAL"
  #ovs-ofctl add-flow  br-$i "cookie=0x1, arp,arp_tpa=rail_1_tor_ip actions=output:$i"
  #ovs-ofctl add-flow  br-$i "cookie=0x1, ip,in_port=LOCAL,nw_dst=rail_1_tor_ip/8 actions=output:$i"
done

echo "Completed setting up all rail bridges and flows!"
EOF
~~~

Now that we have the example script we can base64 encode it and stash it into a variable.

~~~bash
$ BASE64_SCRIPT=`cat spectrum-br-flows.sh | base64 -w 0`
$ echo $BASE64_SCRIPT
IyEvYmluL2Jhc2gKZWNobyAiR2F0aGVyaW5nIHRoZSBpbnRlcmZhY2VzLi4uIiAKbWFwZmlsZSAtdCBpbnRlcmZhY2VzIDwgPCggaXAgbGluayB8IGF3ayAtRic6ICcgJy9lbnA1NS8ge3ByaW50ICQyfScgKSAgIyByZXBsYWNlIGVucDU1IHdpdGggTWVsbGFub3ggd2hlbiBkb25lIHRlc3RpbmcKCmZvciBpIGluICIke2ludGVyZmFjZXNbQF19IgpkbwojaT0iZW5wNTVzMG5wMCIKICBlY2hvICJDcmVhdGluZyBicmlkZ2UgZm9yIGludGVyZmFjZSAkaSBhbmQgc2V0dGluZ3MgdmFsdWVzLi4uIgogIG92cy12c2N0bCAtLW1heS1leGlzdCBhZGQtYnIgYnItJGkKICBvdnMtdnNjdGwgc2V0IGJyaWRnZSBici0kaSBmYWlsLW1vZGU9c2VjdXJlCiAgb3ZzLXZzY3RsIHNldCBicmlkZ2UgYnItJGkgZXh0ZXJuYWwtaWRzOnJhaWxfdXBsaW5rPSRpCiAgb3ZzLXZzY3RsIHNldCBJbnRlcmZhY2UgYnItJGkgbXR1X3JlcXVlc3Q9OTIxNgogIG92cy12c2N0bCBhZGQtcG9ydCBici0kaSAkaQogIG92cy12c2N0bCBzZXQgSW50ZXJmYWNlICRpIG10dV9yZXF1ZXN0PTkyMTYKCiAgZWNobyAiU2V0dGluZyBvdnMtYnJpZGdlIGV4dGVybmFsLWlkcyB0byB0b3JfaXAgZm9yIGJyLSQxLi4uIgogICNvdnMtdnNjdGwgc2V0IGJyaWRnZSBici0kaSBleHRlcm5hbC1pZHM6cmFpbF9wZWVyX2lwPXt7cmFpbF8xX3Rvcl9pcH19CgogIGVjaG8gIkFkZGluZyBpcCBhZGRyZXNzZXMgdG8gaW50ZXJuYWwgYnJpZGdlIGJyLSRpLi4uIgogICNpcCBhZGRyIGFkZCByYWlsXzFfaG9zdF9pcC9pbmZyYV9yYWlsX3N1Ym5ldCBkZXYgYnItJGkKICAjaXAgYWRkciBhZGQgcmFpbF8xX3BvZF9nd19pcCBkZXYgYnItJGkKCiAgZWNobyAiQnJpbmdpbmcgdXAgYnItJGkgcG9ydC4uLiIKICBpcCBsaW5rIHNldCBkZXYgYnItJGkgdXAKCiAgZWNobyAiQWRkaW5nIHRoZSBmbG93cyB0byB0aGUgYnJpZGdlIGJyLSRpLi4uIgogICNvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRpICJjb29raWU9MHgxLCBhcnAsYXJwX3RwYT0gcmFpbF8xX2hvc3RfaXAgYWN0aW9ucz1MT0NBTCIKICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kaSAiY29va2llPTB4MSwgYXJwLGFycF90cGE9IHJhaWxfMV9wb2RfZ3dfaXAgYWN0aW9ucz1MT0NBTCIKICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kaSAiY29va2llPTB4MSwgaXAsbndfZHN0PSByYWlsXzFfaG9zdF9pcCBhY3Rpb25zPUxPQ0FMIgogICNvdnMtb2ZjdGwgYWRkLWZsb3cgIGJyLSRpICJjb29raWU9MHgxLCBpcCxud19kc3Q9IHJhaWxfMV9wb2RfZ3dfaXAgYWN0aW9ucz1MT0NBTCIKICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kaSAiY29va2llPTB4MSwgYXJwLGFycF90cGE9cmFpbF8xX3Rvcl9pcCBhY3Rpb25zPW91dHB1dDokaSIKICAjb3ZzLW9mY3RsIGFkZC1mbG93ICBici0kaSAiY29va2llPTB4MSwgaXAsaW5fcG9ydD1MT0NBTCxud19kc3Q9cmFpbF8xX3Rvcl9pcC84IGFjdGlvbnM9b3V0cHV0OiRpIgpkb25lCgplY2hvICJDb21wbGV0ZWQgc2V0dGluZyB1cCBhbGwgcmFpbCBicmlkZ2VzIGFuZCBmbG93cyEiCg==
~~~

Next we need to generate our config mapping.

~~~bash
TBD
~~~

We also need to base64 encode our config mapping.

~~~bash
$ BASE64_MAP=`cat spectrum-config-map | base64 -w 0`
$ echo $BASE64_MAP
~~~

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
master   rendered-master-5e7c05365dad5cb1e5f96a9c14f6cee9   True      False      False      3              3                   3                     0                      21d
worker   rendered-worker-213acfcc255f377939857a512e933ff0   False     True       False      2              0                   0                     0                      21d
~~~

~~~bash
$ ssh core@nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com
Red Hat Enterprise Linux CoreOS 9.6.20250826-1
  Part of OpenShift 4.19, RHCOS is a Kubernetes-native operating system
  managed by the Machine Config Operator (`clusteroperator/machine-config`).

WARNING: Direct SSH access to machines is not recommended; instead,
make configuration changes via `machineconfig` objects:
  https://docs.openshift.com/container-platform/4.19/architecture/architecture-rhcos.html

---
Last login: Wed Oct  8 20:11:37 2025 from 10.22.66.79
[systemd]
Failed Units: 1
  NetworkManager-wait-online.service
[core@nvd-srv-30 ~]$ sudo bash
[systemd]
Failed Units: 1
  NetworkManager-wait-online.service
[root@nvd-srv-30 core]# systemctl status spectrum-br-flows.service 
● spectrum-br-flows.service - Adds the bridges and flows for Spectrum-X whenever openvswitch is started or reloaded
     Loaded: loaded (/etc/systemd/system/spectrum-br-flows.service; enabled; preset: disabled)
     Active: active (exited) since Wed 2025-10-08 20:58:34 UTC; 2min 25s ago
   Main PID: 3523 (code=exited, status=0/SUCCESS)
        CPU: 31ms

Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com ovs-vsctl[3536]: ovs|00001|vsctl|INFO|Called as ovs-vsctl set bridge br-enp55s0np0 external-ids:rail_uplink=enp55s0np0
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com ovs-vsctl[3538]: ovs|00001|vsctl|INFO|Called as ovs-vsctl set Interface br-enp55s0np0 mtu_request=9216
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com ovs-vsctl[3539]: ovs|00001|vsctl|INFO|Called as ovs-vsctl add-port br-enp55s0np0 enp55s0np0
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com ovs-vsctl[3541]: ovs|00001|vsctl|INFO|Called as ovs-vsctl set Interface enp55s0np0 mtu_request=9216
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Setting ovs-bridge external-ids to tor_ip for br-...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Adding ip addresses to internal bridge br-enp55s0np0...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Bringing up br-enp55s0np0 port...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Adding the flows to the bridge br-enp55s0np0...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Completed setting up all rail bridges and flows!
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com systemd[1]: Finished Adds the bridges and flows for Spectrum-X whenever openvswitch is started or reloaded.
~~~

~~~bash
[root@nvd-srv-30 core]# journalctl |grep spectrum-br-flows.sh
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Gathering the interfaces...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Creating bridge for interface enp55s0np0 and settings values...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Setting ovs-bridge external-ids to tor_ip for br-...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Adding ip addresses to internal bridge br-enp55s0np0...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Bringing up br-enp55s0np0 port...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Adding the flows to the bridge br-enp55s0np0...
Oct 08 20:58:34 nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com spectrum-br-flows.sh[3523]: Completed setting up all rail bridges and flows!
~~~
