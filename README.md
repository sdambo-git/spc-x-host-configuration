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
- [Configuring NVIDIA GPU Operator](#configuring-nvidia-network-operator)
- [Configuring LLDPD Daemonset](#configuring-lldpd-daemonset)
- [Configuring OVS Offload](#configuring-ovs-offload)
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

~~~


## Enable Offloading in OVS
