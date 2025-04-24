# KubeVirt on K3s

## Install

Validate the host is ready for virtualization:

```bash
$ sudo apt install -y libvirt-clients
$ sudo  virt-host-validate qemu
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : PASS
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
  QEMU: Checking for cgroup 'cpu' controller support                         : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
  QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
  QEMU: Checking for cgroup 'memory' controller support                      : PASS
  QEMU: Checking for cgroup 'devices' controller support                     : PASS
  QEMU: Checking for cgroup 'blkio' controller support                       : PASS
  QEMU: Checking for device assignment IOMMU support                         : PASS
  QEMU: Checking if IOMMU is enabled by kernel                               : PASS
  QEMU: Checking for secure guest support                                    : WARN (Unknown if this platform has Secure Guest support)
```

Install the operator:  
(see: <https://kubevirt.io/user-guide/cluster_admin/installation/>)

```bash
# Point at latest release
$ export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
# Deploy the KubeVirt operator
$ kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml"
# Create the KubeVirt CR (instance deployment request) which triggers the actual installation
$ kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml"
# wait until all KubeVirt components are up
$ kubectl -n kubevirt wait kubevirt kubevirt --for condition=Available
```

Well I'll be danged... that actually worked on the first try.

Install the CLI:

```bash
kubectl krew install virt

# Instead of "virtclt" run "kubectl virt":
$ kubectl virt version
Client Version: version.Info{GitVersion:"v1.5.0", GitCommit:"522b44c0ce8d1909618324cb083d69e5c7a0a234", GitTreeState:"clean", BuildDate:"2025-03-13T18:12:11Z", GoVersion:"go1.23.4 X:nocoverageredesign", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{GitVersion:"v1.5.0", GitCommit:"522b44c0ce8d1909618324cb083d69e5c7a0a234", GitTreeState:"clean", BuildDate:"2025-03-13T19:53:21Z", GoVersion:"go1.23.4 X:nocoverageredesign", Compiler:"gc", Platform:"linux/amd64"}
```

## Test

It even works, too... at least at a basic level.  Run through the lab here to verify:  
<https://kubevirt.io/labs/kubernetes/lab1>

## Install Container Data Importer (CDI)

...

## Enable Live Migration

```bash
kubectl apply -f 20-ConfigMap-live-migration.yaml
```

Then test live migration using this lab:  
<https://kubevirt.io/labs/kubernetes/migration.html>

It actually works!  Holy $***!
