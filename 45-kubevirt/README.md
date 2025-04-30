# KubeVirt on K3s

## Prerequisites

Use of Kubevirt with Longhorn has some quirks.

It appears we need to add a configuration option to the containerd runtime in K3s, as
per:  
<https://github.com/kubevirt/containerized-data-importer/blob/main/doc/block_cri_ownership_config.md>  
and:  
<https://github.com/kubevirt/containerized-data-importer/blob/main/doc/datavolumes.md#block-volume-mode>

(Note that K3s uses containerd 2, with the "v3" configuraiton syntax)

We should be able to affect this change by setting the containerd config options:

```toml
[plugins.'io.containerd.grpc.v1.cri']
    device_ownership_from_security_context = true
```

K3s only offers the following on how to do this:  
<https://docs.k3s.io/advanced?_highlight=containerd#configuring-containerd>

The problem there is that the setting we want already is in the template, and the
documentation is for NEW settings, not ones we wish to override.  Claude AI suggested
that we try this nonsense to override default settings in the `ContainerdConfig` struct,
but doing so does not work.  We get an error that "merge" is not a supported function:

```text
#/var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl

{{- $nonrootDevices := true -}}
{{- $updatedDot := merge . (dict "NonrootDevices" $nonrootDevices) -}}
{{ template "base" $updatedDot }}
```

A different AI give us this nonsense:

Create a configuration file in the containerd config directory:

sudo mkdir -p /var/lib/rancher/k3s/agent/etc/containerd/config.toml.d/

Create a file with your specific configuration setting:

cat << EOF | sudo tee /var/lib/rancher/k3s/agent/etc/containerd/config.toml.d/kubevirt.toml
[plugins."io.containerd.grpc.v1.cri"]
  device_ownership_from_security_context = true
EOF

Restart K3s to apply the changes:

sudo systemctl restart k3s

This AI then has the nerve to say that this is documented. What bullshit.

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
