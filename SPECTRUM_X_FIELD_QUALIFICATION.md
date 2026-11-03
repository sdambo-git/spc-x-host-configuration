# Spectrum-X Field Qualification Process

## 1. Purpose and scope

This process qualifies a Red Hat OpenShift and NVIDIA Spectrum-X east-west GPU fabric for field deployment. It covers configuration assurance, host and driver validation, persistent interface naming, NIC policy, synthetic RDMA testing, NCCL testing, and OpenShift AI workload validation.

Use this process for:

- Single-plane and multi-plane Spectrum-X reference architectures.
- Bare-metal OpenShift worker nodes with ConnectX or BlueField SuperNICs.
- RoCEv2 east-west GPU traffic using RDMA, GPUDirect RDMA, and NCCL.
- Red Hat OpenShift AI training and distributed inference workloads.

The process is based on:

- `Spectrum-X Comprehensive Workflow.pdf`.
- Spectrum-X field discussions in Slack channels `C0BGWJDR85C`, `C0A8GTYJY3C`, `C08HG1FHKDJ`, and `C09F5CR4HDL`.
- The host-configuration approach described by `schmaustech/spc-x-host-configuration` and the related `sdambo-git/spc-x-host-configuration` workflow referenced in the PDF.
- RA 2.1 and RA 2.3.1 practices discussed by the field and engineering teams.

## 2. Qualification principles

1. **Configuration is a gate, not a troubleshooting step.** Do not run performance tests until topology, firmware, kernel, drivers, udev, YAML, and policy checks pass.
2. **Every result is reproducible.** Store manifests, commands, logs, versions, topology, timestamps, and test parameters in a signed evidence bundle.
3. **Rail identity must be deterministic.** `mlx5_X` device names are not reliable cross-node rail identifiers. Use explicit rail names, PCI addresses, and resource selectors.
4. **No silent fallback is acceptable.** NCCL must show the expected RoCE/RDMA transport. A workload that passes over the default OpenShift network does not qualify Spectrum-X.
5. **Failure isolation is mandatory.** Every rail must pass independently before a multi-rail test is considered valid.
6. **Use supported combinations.** Match OpenShift, RHEL, NVIDIA Network Operator, DOCA-OFED, GPU driver, CUDA, NCCL, BlueField or ConnectX firmware, switch firmware, and reference architecture versions.
7. **Field changes are controlled.** Use GitOps and Ansible for host and OpenShift configuration. Use Podman for local test tooling and container execution.

## 3. Qualification gates

| Gate | Name | Entry condition | Exit condition |
|---|---|---|---|
| G0 | Engagement and topology baseline | Hardware is racked, cabled, powered, and accessible | Approved topology, BOM, versions, IP plan, rail map, test matrix, and rollback plan |
| G1 | Configuration and manifest validation | Golden configuration repository exists | All YAML, switch, host, and operator configurations validate against the approved design |
| G2 | Host validation | Nodes boot the approved RHEL/OpenShift build | Kernel, firmware, PCI, NUMA, hugepages, IOMMU, interfaces, and console access pass on every node |
| G3 | Driver and subsystem validation | G2 passes | Approved DOCA/OFED, RDMA, fwctl, GPU, and device-plugin versions load and remain healthy |
| G4 | udev and interface persistence | G3 passes | Rail names and RDMA device mappings remain correct after daemon reload and reboot |
| G5 | NIC and operator policy validation | G4 passes | All NIC policy CRDs reconcile, firmware/configuration state is correct, and Spectrum-X CNI is ready |
| G6 | Synthetic RDMA east-west suite | G5 passes | Every rail and every required node pair pass bandwidth, latency, bidirectional, and GPUDirect RDMA tests |
| G7 | NCCL qualification suite | G6 passes | Multi-node NCCL collectives use the expected RDMA/GDRDMA transport and meet the approved baseline |
| G8 | OpenShift AI workload qualification | G7 passes | Representative training and inference workloads use Spectrum-X rails and meet functional, performance, and stability criteria |
| G9 | Failure and resilience qualification | G8 passes | Planned rail, link, pod, node, and switch failure cases behave according to the agreed support model |
| G10 | Field sign-off | Evidence is complete | Customer, Red Hat, and NVIDIA or partner owners sign the qualification record |

Do not proceed to the next gate with a blocking defect. A non-blocking defect requires an owner, severity, target date, workaround, and customer acknowledgement.

## 4. Roles

| Role | Responsibility |
|---|---|
| Qualification lead | Owns the test plan, gate decisions, evidence bundle, exception log, and final sign-off |
| Network fabric engineer | Owns switch configuration, rail topology, LLDP, congestion control, telemetry, and fabric health |
| OpenShift engineer | Owns cluster operators, CRDs, NADs, SR-IOV, OVS hardware offload, GitOps, and workload scheduling |
| RHEL host engineer | Owns kernel parameters, firmware, drivers, udev, NUMA, storage, and host observability |
| AI workload engineer | Owns NCCL, OpenShift AI, training, inference, and application performance analysis |
| Customer owner | Provides environment access, maintenance window, success criteria, and business acceptance |

## 5. G0: Baseline and topology qualification

### 5.1 Required inputs

- Approved Spectrum-X reference architecture and version.
- Physical topology and rail diagram.
- Switch model, firmware, port map, port speed, transceivers, DAC or AOC details, and VLAN plan.
- Host inventory, BMC inventory, GPU model, SuperNIC model, PCI slot map, and memory size.
- OpenShift, RHEL CoreOS or RHEL worker, GPU Operator, NVIDIA Network Operator, SR-IOV Operator, and OpenShift AI versions.
- DOCA-OFED, DOCA, NIC firmware, BlueField BMC firmware, GPU driver, CUDA, and NCCL versions.
- IP addressing plan, NV-IPAM or CIDRPool plan, route policy, and DNS records.
- Expected rail count, plane count, MTU, congestion-control profile, and PCC package.
- Success thresholds and workload models.

### 5.2 Topology validation

Create a node-by-rail matrix:

| Node | GPU | NIC | PCI address | Rail | Switch | Switch port | Speed | MTU | LLDP peer |
|---|---|---|---|---|---|---|---|---|---|
| worker-01 | GPU 0 | BF3/CX8 | `0000:18:00.0` | rail 0 | leaf-01 | swp1 | 400 Gb/s | 9000 | pending |

Validate:

- Every expected rail is present on every expected node.
- No rail is crossed, duplicated, or missing.
- Every rail reaches the intended leaf, spine, plane, and rail pool.
- Port speed, lane count, FEC, transceiver type, MTU, VLAN, and switch profile match the design.
- LLDP data confirms the physical peer on every port.
- The design explicitly handles shared or overlapping IP prefix behavior, such as the RA 2.1 leaf-level `/25` model with interleaved `/31` host links.
- A documented route or steering model exists for rail affinity. Do not rely on assumptions when multiple rails share one address block.

### 5.3 Required tools

Use:

- `dnf` for RHEL package installation.
- Podman for validation containers and local tooling.
- Ansible for repeatable host configuration and preflight checks.
- `oc` and `kubectl` for OpenShift validation.
- `ethtool`, `lspci`, `ip`, `rdma`, `ibstat`, and `ibv_devinfo`.
- NVIDIA `rcp-tool`, launch kit, Minerva, or the approved Spectrum-X readiness tool when available.

## 6. G1: Configuration and YAML validation

### 6.1 Golden repository

Store all declarative configuration in a Git repository:

- OpenShift manifests and OperatorSubscription objects.
- `NicClusterPolicy`.
- `NicFirmwareSource`.
- `NicFirmwareTemplate`.
- `NicConfigurationTemplate`.
- `NicInterfaceNameTemplate`.
- SR-IOV policies and network attachment definitions.
- NV-IPAM or CIDRPool resources.
- Spectrum-X CNI and rail pool configuration.
- NodeFeatureDiscovery rules and labels.
- Tuned and MachineConfig objects.
- OpenShift AI workloads and serving or training runtimes.

Tag the exact commit used for qualification. Do not apply untracked manual changes.

### 6.2 Static validation

Run static validation before cluster application:

```bash
yamllint .
kubeconform -strict -summary -ignore-missing-schemas manifests/
oc apply --dry-run=server -f manifests/
```

For custom NVIDIA CRDs, install or reference their schemas. If `kubeconform` cannot resolve a CRD schema, record the CRD version and validate required fields with a dedicated policy test.

### 6.3 Manifest checks

Validate that every manifest contains:

- Intended namespace.
- Explicit resource requests and limits.
- Node selectors and affinity aligned to GPU and rail placement.
- Approved image repository and immutable version or digest.
- Image pull secrets where required.
- Operator upgrade policy and drain settings.
- No debug values or staging repositories unless explicitly approved.
- No secrets in plain text.

For Network Operator policy, validate:

- DOCA-OFED driver image and version.
- Driver upgrade policy, safe load setting, drain timeout, and parallelism.
- RDMA shared device plugin selectors.
- SR-IOV device plugin selectors.
- NIC configuration operator deployment state.
- Spectrum-X operator deployment state.
- Proxy or air-gapped image settings.

Example fields to inspect:

```yaml
ofedDriver:
  repository: <approved-registry>
  image: doca-driver
  version: <approved-version>
  upgradePolicy:
    autoUpgrade: <approved-policy>
    drain:
      enable: true
      timeoutSeconds: <approved-timeout>
rdmaSharedDevicePlugin:
  config: |
    {
      "configList": [
        {
          "resourceName": "rdma_shared_device_eth",
          "rdmaHcaMax": 63,
          "selectors": {
            "ifNames": ["<approved-rail-names>"]
          }
        }
      ]
    }
```

### 6.4 Switch configuration validation

Capture and validate:

- Switch configuration snapshot.
- Firmware version.
- Rail and plane assignment.
- Port profile and Ethernet profile.
- MTU.
- PFC, ECN, and congestion-control settings.
- DOCA PCC or approved PCC algorithm.
- Breakout and lane configuration.
- LLDP and telemetry enablement.
- Route policy and ECMP policy.
- Buffer and queue profile.

Run the switch vendor or NVIDIA configuration validator. A manual visual inspection is not sufficient for qualification.

### 6.5 Exit criteria

- Configuration repository is tagged and immutable.
- All YAML passes linting, schema validation, and server-side dry run.
- Switch configuration matches the approved profile.
- Configuration drift is zero against the approved source.
- A rollback plan and last-known-good configuration exist.

## 7. G2: Host-side validation

### 7.1 RHEL and OpenShift baseline

Collect from every worker:

```bash
uname -a
cat /etc/os-release
rpm -qa | sort
subscription-manager status
systemctl --failed --no-pager
journalctl -p err -b --no-pager
```

Validate:

- Supported RHEL or RHCOS build.
- Required packages are installed with `dnf`.
- Time synchronization is healthy.
- No failed systemd units.
- No storage, filesystem, or BMC error that can interrupt qualification.
- Secure local console access and a documented break-glass user.
- BMC or Redfish access is available.
- Host firewall and SELinux settings match the supported configuration.

### 7.2 Kernel parameters

Capture:

```bash
cat /proc/cmdline
sysctl -a
```

Validate the approved boot profile. Common Spectrum-X and SR-IOV settings include:

```text
pci=realloc=on intel_iommu=on iommu=pt default_hugepagesz=2M hugepagesz=2M hugepages=<approved-count>
```

Do not copy this value blindly:

- IOMMU must match the supported SR-IOV and reference-architecture mode.
- `iommu=pt` is commonly used to reduce translation overhead while retaining IOMMU support for devices that require it.
- If an RA requires a different IOMMU mode, record the RA requirement and the validation evidence.
- `hugepages=32768` with 2 MiB pages reserves 64 GiB. Size hugepages from GPU memory, framework requirements, and host memory, not from a fixed field value.

Validate:

```bash
grep Huge /proc/meminfo
lsblk
lspci -vvv
```

Confirm:

- Hugepages are available after reboot.
- GPU and NIC DMA works under the selected IOMMU mode.
- PCI device allocation and BAR resources are correct.
- ACS or bridge settings do not introduce unexpected latency. If ACS changes are required, capture before and after evidence.

### 7.3 Hardware inventory and health

Collect:

```bash
lscpu
numactl -H
dmidecode -t bios
dmidecode -t memory
lspci -nn | grep -E 'NVIDIA|Mellanox'
journalctl -b --no-pager | grep -Ei 'mce|pcie|mlx5|nvidia|bluefield|iommu'
```

Validate:

- BIOS, BMC, and firmware versions match the approved matrix.
- GPU and NIC are on the expected PCI bus and NUMA node.
- GPU-to-NIC NUMA affinity matches the topology model.
- Power and cooling profiles are set for sustained line-rate traffic.
- CPU governor and performance profile are suitable for qualification.
- There are no correctable-error rates above the approved threshold.
- There are no uncorrectable PCI, memory, GPU, or NIC errors.

### 7.4 Interface and link health

For each host interface:

```bash
ip -d link show
ethtool <interface>
ethtool -i <interface>
ethtool -k <interface>
ethtool -S <interface>
ip -br addr show
rdma link show
```

Validate:

- Link is up.
- Speed, duplex, lanes, and FEC match the design.
- MTU is 9000 or the approved value on host, switch, NAD, OVS, and workload interfaces.
- No interface errors, drops, pauses, or retransmissions before testing.
- RDMA link state is active.
- Switchdev or SR-IOV mode matches the architecture.
- BlueField rshim or BMC ownership is the approved mode.

### 7.5 Reboot persistence

After a controlled reboot:

- Kernel command line is unchanged.
- Hugepages are still allocated.
- Interface names are stable.
- RDMA devices are visible.
- OVS bridges and offload state are restored.
- NADs and device plugins rediscover resources.
- Nodes return to Ready without manual intervention.

## 8. G3: Driver and subsystem validation

### 8.1 Version matrix

Record the exact version and source of:

- Linux kernel.
- DOCA-OFED or DOCA driver container.
- `mlx5_core`.
- `mlx5_ib`.
- `fwctl`.
- GPU driver.
- CUDA runtime.
- NCCL.
- rdma-core and perftest.
- ConnectX or BlueField firmware.
- BlueField BMC firmware.
- Switch firmware.
- NVIDIA Network Operator.
- GPU Operator.
- SR-IOV Operator.
- Spectrum-X operator and CNI.

Validate one supported matrix. Mixed versions require an explicit exception and a separate qualification run.

### 8.2 Driver loading

Collect:

```bash
lsmod | sort
modinfo mlx5_core
modinfo mlx5_ib
modinfo fwctl
ibstat
ibv_devinfo -v
rdma statistic -l
```

Validate:

- Required modules load without errors.
- Storage modules unloaded by the DOCA-OFED driver are expected and documented.
- Conflicting RDMA modules are absent or blacklisted according to the approved policy, including unsupported combinations such as `irdma` or `bnx_ib` where applicable.
- Firmware and driver versions are compatible.
- RDMA devices and ports are active.
- `fwctl` loads if NIC configuration daemons require it.
- No kernel log contains driver initialization, timeout, reset, or firmware errors.

### 8.3 Operator and daemon health

Run:

```bash
oc get csv -A
oc get pods -n nvidia-network-operator
oc get pods -n nvidia-gpu-operator
oc get pods -n openshift-sriov-network-operator
oc describe nicclusterpolicy <name>
oc get nicclusterpolicy -o yaml
```

Validate:

- All required CSVs are `Succeeded`.
- All operator and daemon pods are `Running` or `Completed`.
- No crash loops, image pull errors, OOM kills, or SCC denials.
- `NicClusterPolicy` reports ready.
- Applied state for OFED, RDMA plugin, SR-IOV, Spectrum-X, and NIC configuration is correct.
- Driver upgrade policy is safe and uses drain behavior.

### 8.4 GPU and GPUDirect readiness

Run inside the approved GPU validation container:

```bash
nvidia-smi -q
nvidia-smi topo -m
nvidia-smi dmabuf -s
```

Validate:

- All expected GPUs are visible.
- Peer-to-peer and GPUDirect topology is as expected.
- GPU NUMA affinity is mapped correctly to NIC rails.
- DCGM reports no active hardware faults.
- Workload containers can access the expected GPU and RDMA resources.

## 9. G4: udev and rail naming validation

### 9.1 Preferred RA 2.3.1 method

Use `NicInterfaceNameTemplate` instead of hand-written persistent udev rules when the supported Network Operator and NIC Configuration Operator versions are available.

The template should derive deterministic names from:

- `railPciAddresses`.
- `netDevicePrefix`.
- `rdmaDevicePrefix`.
- `%nic_id%`, `%plane_id%`, and `%rail_id%` placeholders.

Validate:

```bash
oc get nicinterfacenametemplate -A -o yaml
oc describe nicinterfacenametemplate <name>
```

Confirm:

- PCI addresses match the approved rail matrix.
- Prefixes are unique for Ethernet and RDMA devices.
- Rail and plane IDs are generated as intended.
- Every selected node is covered by the node selector.
- Reconciliation succeeds.

### 9.2 Manual fallback

Only use a manual `70-persistent-net.rules` file when the declarative method is unavailable and the exception is approved.

Validate:

- Rules use stable physical identifiers such as PCI path and device ID.
- Rules map to explicit `eth_railN` and `roce_railN` names.
- Rules do not use non-deterministic `mlx5_X` names as identity.
- File ownership and permissions are correct.
- Rules are generated by Ansible or GitOps, not edited by hand.

Apply and test:

```bash
udevadm control --reload
udevadm trigger
ip -d link show
rdma link show
```

### 9.3 Persistence test

On every node:

1. Record the PCI address, MAC, PCI path, Ethernet name, and RDMA name for each rail.
2. Reload udev.
3. Reboot.
4. Compare the complete mapping.
5. Run one RDMA probe per rail.

Pass criteria:

- No mapping changes after reboot.
- No two rails map to the same logical name.
- Every node uses the same logical rail number for the same physical topology role.
- Application resource selectors use logical rail names or resource names, not raw `mlx5_X` names.

## 10. G5: NIC policy and Spectrum-X CNI validation

### 10.1 Firmware and configuration templates

Validate each custom resource:

```bash
oc get nicfirmwaresource -A -o yaml
oc get nicfirmwaretemplate -A -o yaml
oc get nicconfigurationtemplate -A -o yaml
oc get nicinterfacenametemplate -A -o yaml
```

For each NIC:

- PCI selector matches the intended device.
- NIC type is correct, for example `a2dc` for BlueField-3 SuperNIC where applicable.
- Firmware source and PCC package are approved.
- Update policy is approved.
- Link type is Ethernet.
- VF count and switchdev mode match the architecture.
- Spectrum-X optimized profile is enabled.
- Overlay setting and RA version match the design.

### 10.2 Spectrum-X rail and CNI state

Validate:

```bash
oc get pods -A | grep -E 'spectrum|nic-config|flow-controller|rdma'
oc get network-attachment-definitions -A
oc get sriovnetworknodestate -A
oc get sriovnetwork -A
```

Confirm:

- Spectrum-X operator and CNI are ready.
- Flow-controller DaemonSet is ready on every GPU worker.
- Per-rail OVS flows are programmed.
- Rail pool configuration matches the expected plane and rail count.
- NAD names are stable and documented.
- Pods can attach to each expected rail NAD.
- No OVS flow programming errors are present.

### 10.3 OVS and hardware offload

Run on an approved privileged debug pod or host shell:

```bash
ovs-vsctl show
ovs-vsctl get Open_vSwitch . other_config
ovs-appctl dpctl/dump-flows type=offloaded
ovs-appctl dpctl/show
```

Validate:

- OVS-DOCA or approved OVS build is installed.
- Hardware offload is enabled where required.
- Expected flows are offloaded.
- Host CPU usage during line-rate traffic is within the approved baseline.
- No partial offload or unsupported flow warning is ignored.
- OVS MTU and per-rail bridges match the design.

### 10.4 Resource injection test

Create a small validation pod annotated with every required rail NAD.

Validate inside the pod:

```bash
ip -br addr
rdma link show
ibv_devinfo
```

Pass criteria:

- The pod sees every expected rail interface and RDMA device.
- Interface MTUs are correct.
- The default route does not unexpectedly steer east-west traffic over the cluster network.
- Resource requests expose the intended GPU and RDMA devices.

## 11. G6: Synthetic RDMA east-west test suite

### 11.1 Test matrix

Create a complete matrix:

| Test | Scope | Mode | Required result |
|---|---|---|---|
| Link probe | Every rail and node | Unidirectional and bidirectional | Connection and transfer complete |
| `ib_write_bw` | Every node pair and rail | Default and `-R` | Meets baseline, no errors |
| `ib_read_bw` | Every node pair and rail | Default and `-R` | Meets baseline, no errors |
| `ib_send_bw` | Every node pair and rail | Default and `-R` | Meets baseline, no errors |
| `ib_write_lat` | Every node pair and rail | Default | Meets latency baseline |
| `ib_read_lat` | Every node pair and rail | Default | Meets latency baseline |
| GPUDirect RDMA | Every GPU-to-rail pair | `--use_cuda` or approved method | Meets baseline, no CUDA or RDMA errors |
| Multi-rail concurrency | All rails simultaneously | Concurrent jobs | Aggregate bandwidth meets baseline, no rail stalls |
| MTU test | Every rail | 9000-byte and approved sizes | No fragmentation or silent path change |
| Route test | Every rail | With and without `-R` | Both modes behave as documented |
| Congestion test | Full fabric | Multi-node fan-in | No congestion collapse or excessive retransmission |

Minimum coverage:

- Every rail on every node.
- Every directed node pair for two-node qualification.
- Full mesh for small clusters.
- Representative node pairs, plus all suspected failure domains, for large clusters.
- Every GPU-to-rail NUMA combination.
- Each rail in isolation and all rails together.

### 11.2 Example perftest commands

On the receiver:

```bash
podman run --rm --privileged --network host \
  <approved-perftest-image> ib_write_bw \
  -d <rdma-device> -R --report_gbits -D 30
```

On the sender:

```bash
podman run --rm --privileged --network host \
  <approved-perftest-image> ib_write_bw \
  -d <rdma-device> -R --report_gbits -D 30 <receiver-ip>
```

For latency:

```bash
ib_write_lat -d <rdma-device> --report_gbits
ib_read_lat -d <rdma-device> --report_gbits
```

For GPUDirect RDMA, use the approved perftest build with CUDA support and the approved device selection syntax. Capture the GPU ID used for every run.

### 11.3 RoCE and route-specific checks

The field evidence showed that overlapping `/25` addressing can affect path selection in non-`-R` perftest while bidirectional `-R` traffic may behave differently.

Therefore every rail must be tested:

- With server bind on the specific rail IP.
- With explicit source IP and source interface.
- With default routing.
- With explicit route or policy routing.
- With and without `-R`.

Do not approve rail affinity based only on an `ib_write_bw -R` result. Record which path each test used.

### 11.4 Metrics and counters

Before and after every test, capture:

```bash
ethtool -S <interface>
rdma statistics link <rdma-device>
ip -s link show <interface>
```

Also capture switch-side counters:

- TX and RX packets and bytes.
- Discards and drops.
- Pause frames.
- ECN marks.
- Congestion notifications.
- Out-of-order or retransmitted segments.
- CRC, FEC, symbol, and link errors.

Pass criteria:

- No failed rail.
- No timeout or stalled connection after RDMA establishment.
- No silent use of an unintended rail.
- No RDMA transport errors.
- No unexplained pause or drop counter increase.
- Bandwidth is at or above the approved reference for the link speed, message size, and topology.
- Latency is at or below the approved reference.

Set the reference from the RA or a validated golden system. For a new field system, require at least 90 percent of the validated same-topology baseline unless the approved RA provides a stricter or workload-specific threshold.

## 12. G7: NCCL qualification suite

### 12.1 Test image

Build one approved NCCL test image containing:

- CUDA and NCCL versions matching the workload matrix.
- Matching rdma-core and perftest userspace.
- NCCL tests.
- Required GPUDirect RDMA libraries.
- OpenShift-compatible security context.

Run the image with Podman during host validation and through OpenShift for cluster validation. Record the image digest.

### 12.2 NCCL functional tests

Run at minimum:

```bash
all_reduce_perf
all_gather_perf
reduce_scatter_perf
broadcast_perf
sendrecv_perf
alltoall_perf
```

Recommended matrix:

| Test | Nodes | GPUs per node | Rails | Messages |
|---|---|---|---|---|
| Single-node baseline | 1 | All | N/A | 8 B to 1 GiB |
| Two-node functional | 2 | 1, 2, 4, 8 | All | 8 B to 1 GiB |
| Two-node per-rail | 2 | All | One rail at a time | Selected large messages |
| Multi-node scale | 4 or more | All | All | 8 B to 1 GiB |
| Multi-rail concurrency | 2 or more | All | All | Large message sweep |
| Failure isolation | 2 or more | All | Healthy and degraded rail | Selected messages |

Use:

```bash
NCCL_DEBUG=INFO
NCCL_DEBUG_SUBSYS=INIT,GRAPH,TUNING,NET,ENV
NCCL_IB_DISABLE=0
```

Do not force NCCL to use a raw `mlx5_X` name as cross-node rail identity. Use logical rail interfaces, approved device selectors, or the NVIDIA-supported all-rail discovery model.

### 12.3 Required log evidence

Every successful NCCL run must show:

- Correct number of ranks, GPUs, and nodes.
- Expected RoCE or IB transport, for example `NET/IB`.
- Expected GDRDMA capability.
- The number of HCAs discovered.
- Successful topology and ring or graph construction.
- Successful transport connection establishment.
- No socket fallback for the GPU data path.
- No transport timeout, retry storm, or connection reset.
- Expected collective completion and zero wrong results.

Correlate NCCL log timestamps with host and switch counters to prove which rail carried traffic.

### 12.4 Performance criteria

For every collective:

- Compare bus bandwidth against the approved golden baseline.
- Compare against the synthetic RDMA ceiling for the same rail set.
- Record the scaling efficiency as node count increases.
- Record CPU overhead.
- Record the stability of repeated runs.

Suggested pass rules:

- Functional pass requires zero errors and zero failed collectives.
- Performance pass requires at least 90 percent of the approved same-hardware baseline, or the customer-approved workload threshold.
- Multi-rail pass requires no rail to remain idle in an all-rail test unless that behavior is documented and approved by NVIDIA.
- Stability pass requires three consecutive runs within the approved variance.

### 12.5 Failure and resilience tests

After the healthy baseline:

1. Disable one rail at a time.
2. Run a two-node all-reduce test.
3. Record whether NCCL selects the remaining healthy rail, fails, or stalls.
4. Restore the rail and confirm recovery.
5. Repeat for a planned link failure and, if the topology permits, one switch or plane failure.

Do not invent an unsupported resilience guarantee. Qualify the observed behavior, compare it with the NVIDIA-supported model, and record the support statement.

## 13. G8: OpenShift AI workload qualification

### 13.1 Network attachment requirement

OpenShift AI workloads do not automatically use Spectrum-X just because the fabric exists. Every distributed training or inference workload must be explicitly attached to the intended rail NADs or use an approved platform mechanism that exposes those interfaces.

Validate before running the workload:

```bash
oc get pod <pod> -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/networks}'
oc exec <pod> -- ip -br addr
oc exec <pod> -- rdma link show
oc exec <pod> -- nvidia-smi
```

If the OpenShift AI dashboard cannot attach rail NADs, use a GitOps-managed YAML workload or patch the generated workload. Record this limitation in the qualification report.

### 13.2 Training test suite

Run:

1. A PyTorch distributed data parallel smoke test.
2. A Kubeflow Trainer or TrainJob distributed test.
3. A representative customer training model or approved model proxy.
4. A sustained run for the approved duration.

Validate:

- Pods are scheduled on the intended nodes.
- Pods consume the expected GPU and RDMA resources.
- Every rank can communicate over the expected rail interfaces.
- NCCL uses RoCE and GDRDMA, not the default OpenShift network.
- Training loss and step time are stable.
- No rank timeout, NCCL abort, OOM, or network reset occurs.
- East-west traffic counters increase on the expected rails.

### 13.3 Distributed inference test suite

For OpenShift AI distributed inference:

- Validate the approved ServingRuntime.
- Validate `tensorParallelSize`, `pipelineParallelSize`, and worker placement.
- Validate that workerSpec fields survive admission and are not silently removed.
- Use multi-node vLLM or the approved OpenShift AI runtime.
- Attach pods to Spectrum-X rail NADs.
- Capture NCCL or other approved transport logs.

Validate:

- Head and worker pods start on the intended nodes.
- GPU and RDMA resources are correct.
- Model load and inference complete.
- Inter-node traffic uses Spectrum-X.
- No silent fallback to the cluster network.
- Throughput and latency meet the customer baseline.
- The workload survives the approved restart or failure scenario.

### 13.4 OpenShift AI acceptance tests

Run each workload three times:

| Test | Duration | Pass condition |
|---|---|---|
| Pod and NAD validation | Until ready | Expected rail and GPU resources visible |
| Training smoke test | Short | Correct completion and RDMA transport |
| Training stability | Approved soak | No failures and stable throughput |
| Inference smoke test | Short | Correct completion and RDMA transport |
| Inference stability | Approved soak | Stable throughput and latency |
| Failure test | Approved scenario | Documented resilience behavior |

## 14. Evidence bundle

Store evidence in one immutable location for each qualification run:

- Git commit and diff of all manifests.
- Host inventory and Ansible run output.
- Switch configuration snapshots.
- LLDP and topology export.
- Kernel command lines and hugepage data.
- PCI and NUMA topology.
- Firmware and driver versions.
- Operator status and pod logs.
- NIC policy and custom-resource YAML.
- udev or interface-template mapping before and after reboot.
- OVS configuration and offloaded flows.
- Perftest command lines, raw output, and counters.
- NCCL logs and performance output.
- OpenShift AI workload YAML, logs, and metrics.
- Prometheus or Minerva metrics where available.
- Exception list and final sign-off.

Name evidence files with gate, node, rail, test, timestamp, and run ID. Do not replace failed output with a later passing run; retain both.

## 15. Defect severity

| Severity | Definition | Gate effect |
|---|---|---|
| S1 | Fabric, node, GPU, or workload is unusable or data is incorrect | Stop qualification |
| S2 | Required rail or transport fails, or workload silently uses wrong network | Stop current gate |
| S3 | Performance below threshold but function is correct | Stop performance sign-off |
| S4 | Cosmetic or documentation defect | May continue with owner and date |

Examples of S2:

- Any rail fails synthetic RDMA after connection establishment.
- NCCL selects a failed rail instead of a healthy rail and the collective fails.
- OpenShift AI workload passes only through the default OVN network.
- Raw `mlx5_X` names are used as cross-node rail identity.

## 16. Automation model

### 16.1 Ansible preflight

Use Ansible roles for:

- Host package installation with `dnf`.
- Kernel argument management.
- Hugepage sizing.
- Module blacklist policy.
- BMC and firmware inventory.
- udev fallback deployment.
- Reboot and persistence checks.
- Log and counter collection.

Run preflight and post-test collection as idempotent playbooks.

### 16.2 GitOps

Use OpenShift GitOps for:

- Operator installation.
- Custom resource deployment.
- NAD and rail pool configuration.
- OpenShift AI workload deployment.
- Controlled rollback.

No cluster change should exist only in the cluster database.

### 16.3 Container validation

Use Podman for:

- Local test image builds.
- Host-side perftest containers.
- NCCL test image validation before OpenShift execution.
- Debug and evidence collection containers.

Example:

```bash
podman build -t spectrum-x-qualification:<version> .
podman run --rm --privileged --network host spectrum-x-qualification:<version> ibv_devinfo
```

## 17. Field sign-off

Sign-off requires:

1. All mandatory gates passed.
2. All S1 and S2 defects closed.
3. All S3 defects closed or formally accepted with a remediation plan.
4. Synthetic RDMA, NCCL, and OpenShift AI tests pass on the same configuration commit.
5. Evidence bundle is complete and immutable.
6. Supported versions, known limitations, and operational ownership are documented.
7. Customer, Red Hat, and NVIDIA or partner owner signs the qualification record.

## 18. Minimum one-day field qualification

When time is limited, use this subset and clearly mark the result as limited qualification:

| Gate | Minimum coverage |
|---|---|
| G0 | Two nodes, all rails, full topology and version inventory |
| G1 | YAML and switch configuration validation |
| G2 | Kernel, hugepages, IOMMU, PCI, NUMA, links, and reboot persistence |
| G3 | Driver, GPU, RDMA, and operator health |
| G4 | Interface-name mapping across nodes and reboot |
| G5 | NIC policy, CNI, OVS offload, and resource injection |
| G6 | Every rail independently, then all rails concurrently |
| G7 | Two-node NCCL all-reduce and one large-message collective |
| G8 | One OpenShift AI training smoke test with NAD and NCCL evidence |

A limited qualification is not sufficient for production sign-off if resilience, sustained stability, multi-node scaling, or representative inference workloads were not tested.
   
