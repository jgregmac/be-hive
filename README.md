# The Be-hive: A Self-Hosting Experiment

- [The Be-hive: A Self-Hosting Experiment](#the-be-hive-a-self-hosting-experiment)
  - [Overview](#overview)
  - [Environment](#environment)
    - [Hardware](#hardware)
    - [Budget](#budget)
    - [System Information](#system-information)
    - [Software](#software)
    - [Networking](#networking)
  - [Host Install Process](#host-install-process)
    - [Network configuration verification and repair](#network-configuration-verification-and-repair)
  - [K3S Configuration](#k3s-configuration)
    - [Install additional nodes, or re-install a node](#install-additional-nodes-or-re-install-a-node)
    - [Install MetalLB](#install-metallb)
    - [Install Reflector](#install-reflector)
    - [Install Cert-Manager](#install-cert-manager)
    - [Install Ingress](#install-ingress)
    - [Install Dashboard](#install-dashboard)
    - [Install Multus](#install-multus)
    - [Install Longhorn](#install-longhorn)
    - [Install KubeVirt](#install-kubevirt)
    - [Next steps: Nextcloud Helm Chart?](#next-steps-nextcloud-helm-chart)
  - [~~MicroK8s Configuration~~](#microk8s-configuration)
  - [~~Ingress Traffic and Load Balancing~~](#ingress-traffic-and-load-balancing)
  - [Troubleshooting](#troubleshooting)
    - [Name resolution problems](#name-resolution-problems)
    - [Reconfiguring K3s after installation](#reconfiguring-k3s-after-installation)
  - [Resources](#resources)

## Overview

Let's try using K3S Kuberenetes on NUC-class Micro PCs with a replicated storage engine
to host cloud-like services entirely at home.  Because why would you NOT want to take on
the burden of running an entire cloud suite of services ALONG WITH hardware maintenance
obligations, entirely in your "copious" spare time?

The configuration steps documented here are the current choices resulting from a fair
amount of trial and error.  This project stated with an effort to deploy a MicroK8s
cluster on Ubuntu Core OS and Mayastor, for the purpose of self-hosting a replacement
"OneDrive.  It has morphed to this current K3s/Longhorn project.  It may change again in
the future as I discover more stable or maintainable platform options.

I don't know that this ever will have been a good idea, but it did give opportunities
to learn.

## Environment

It's really just three BeeLink EQ6 Mini PCs, a 16-port managed switch, and an existing
OpnSense router/firewall.

### Hardware

- 16 port 1GbE TP-Link Managed Switch
- 3x EQ6 Mini PCs:
  - 2x 1GbE network interfaces
  - 500Gb NVMe Boot Drive
  - 2Tb NVMe Storage Drive
  - AMD Ryzen 5 6600U low-power CPU, 6c/12t
- OpnSense FW4B Firewall Appliance (repurposed, purchased 2020)

### Budget

| Item                     | Model         | Cost     | Qt  | Total     |
| ---                      | ---           | ---      | --- | ---       |
| TP-Link Switch           | TL-SG116E     | $  72    | 1   | $   72    |
| Beelink Mini-PC          | EQR6          | $ 289    | 3   | $  867    |
| Crucual 2Tb M.2 SSD      | CT2000P3PSSD8 | $ 118    | 3   | $  354    |
| 3-ft Cat6 Ethernet Cable | 5-pack (red)  | $  12.50 | 1   | $   12.50 |
| 3-ft Cat6 Ethernet Cable | 5-pack (blue) | $  12.50 | 1   | $   12.50 |
| ProtectLi OpnSense FW    | FW4B          | $ 280    | 1   | $  280    |
| Total                    |               |          |     | $ 1598    |

...But that does not include the cost to my sanity.

### System Information

Keep a table of your hardware identifiers:  
It is useful when building out your infrastructure-as-code.

| Hostname | S/N | eno1 MAC          | enp5s0 MAC        | wlp3s0 MAC        | 500Gb SSD | 2Tb SSD |
| -------- | --- | ----------------- | ----------------- | ----------------- | --------- | ------- |
| beehive1 |     | e8:ff:1e:d5:2b:09 | e8:ff:1e:d5:2b:0a | e4:c7:67:98:11:ab | nvme1n1   | nvme0n1 |
| beehive2 |     | e8:ff:1e:d5:2d:6f | e8:ff:1e:d5:2d:70 | e4:c7:67:97:d2:a4 | nvme1n1   | nvme0n1 |
| beehive3 |     | e8:ff:1e:d5:2b:37 | e8:ff:1e:d5:2b:38 | e4:c7:67:ee:1e:ed | nvme1n1   | nvme0n1 |
|          |     |                   |                   |                   |           |         |

### Software

Some plans and ideas for what to do with this cluster...

- Ubuntu Server 24.04  
  (Maybe I should have used [Elemental](https://elemental.docs.rancher.com/)?)
- K3s Kubernetes
- Longhorn Storage Engine
- Nginx ingress
- MetalLB Load Balancer
- cert-manager
- Nextcloud
- Accounting Package?
- NAS service (CIFS)
- Backup Service (Velero, with which target?  S3?  What will that cost?)
  
### Networking

- Off-LAN access to services will require TailScale:
  - Tailscale client will need to be configured on each node.
- Kubernetes Nodes will have their data network interface set in the following network/range:
  - Network: 192.168.2.0/23
  - Netmask: 255.255.254.0
  - Default Route: 192.168.2.1
  - DNS Servers: 192.168.2.1,9.9.9.9
  - IP Range: 192.168.3.1 - 192.168.3.30 (CIDR: 192.168.3.0/27)
- Storage network interfaces will be set in the following network/range:
  - Network: 192.168.4.0/23
  - Netmask: 255.255.254.0
  - IP Range: 192.168.5.1 - 192.168.5.30 (CIDR: 192.168.5.0/27)
- We will try using MetalLB for load balancing of resources running in the cluster:
  - Network: 192.168.2.0/23
  - IP Range: 192.168.3.33 - 192.168.62 (CIDR: 192.168.3.32/27)
  - MicroK8s uses the following networks by default (for reference):
  - Cluster CIDR: 10.1.0.0/16
  - Service Cluster IP Range: 10.152.183.0/24
  - Cluster DNS: 10.152.183.10
- Longhorn Storage Engine will need a network range for use of Multus:
  - 192.168.5.129-192.168.5.254

## Host Install Process

1. Configure UEFI settings on the system to support Core OS install:
   1. Setting: Boot -> Quiet Boot -> DIsable
   2. Setting: Advanced -> AMD CBS -> FCH Common -> AC Power Loss -> Previous
   3. Setting: Security -> Secure Boot Mode -> Set to "Standard"
   4. Setting: Security -> Secure Boot -> Enabled
2. Create an Ubuntu Server 24.04.1 USB Boot device using Balena Etcher or Rufus.  Insert into the Beelink PC, press "F7" at the boot screen to select a boot device.  Select to boot from the "Generic Mass Storage" device.
3. In Ubuntu Installer:
   1. For "Choose Type of Installation", select "Ubuntu Server (minimized)"
   2. Configure static IP addresses for all interfaces, using config table.
   3. For "Guided storage configuration", select "Use an entire disk", and be sure to select the 500Gb (465.761 GiB) drive for the install.
   4. For profile configuration:
      1. Name: Hive Administration Professional
      2. Server name: beehive\[1-3\]
      3. Username: queenbee
      4. Password: REDACTED
   5. Ubuntu Pro: Skip for now
   6. SSH Config:
      1. Select to install SSH Server
      2. For authorized keys, select "Launchpad" as the import source, select "jgregmac" as the username.
      3. ~~Featured Server Snaps: Select Microk8s.  May as well...~~
      4. Run installation to completion, then reboot, verify console login.
4. Post install:

    ```shell
    # Install commonly needed diagnostic tools:
    sudo apt -y install git iputils-ping net-tools vim zsh

    # Improve the shell using oh-my-zsh, for your own sanity:
    sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

    # Apply latest patches:
    sudo apt update
    sudo apt -y upgrade

    # Reboot!
    sudo reboot
    
    ### SKIP THESE OBSOLETE PREREQUISITES!
    # Allow access to microk8s commands:
    # sudo usermod -a -G microk8s queenbee
    # newgrp microk8s

    # We will need to enable hugepages in the kernel for Mayastor.
    # (Also needed for Longhorn v2 storage engines, which we are not planning to use.)
    # On each node:
    # $ echo vm.nr_hugepages = 1024 \
    #     | sudo tee -a /etc/sysctl.d/20-microk8s-hugepages.conf
    # vm.nr_hugepages = 1024

    # Install extra modules for nvme-tcp module
    # (Its probably already present):
    # sudo apt-get install linux-modules-extra-$(uname -r)
    
    # Load the module:
    # sudo modprobe nvme-tcp
    
    # Configure the module to load on startup:
    # echo 'nvme-tcp' \
    #     | sudo tee -a /etc/modules-load.d/microk8s-mayastor.conf
    ```

### Network configuration verification and repair

1. After hooking up the Mini PC to the Ethernet switch, ensure that wifi adapters are disabled.  Use the Netplan file to fix the interface configuration. Note that all search domains are set to null `[]`. (See 10-netplan directory)
2. Apply the revised netplan:
   `sudo netplan apply`
3. Make sure that there are no unwanted search domains in the resolver config:

    ```shell
    $ resolvectl domain
    Global:
    Link 2 (eno1): duckdns.org      <-- Bad!
    Link 3 (enp5s0): duckdns.org    <-- Bad!
    Link 4 (wlp3s0):

    $ sudo resolvectl domain eno1 ""
    $ sudo resolvectl domain enp5s0 ""
    $ resolvectl domain
    Global:
    Link 2 (eno1):                  <-- Good!
    Link 3 (enp5s0):                <-- Good!
    Link 4 (wlp3s0): 
    ```

## K3S Configuration

### Install additional nodes, or re-install a node

```bash
# If reinstalling a node, you will need to clean up any artifacts from previous installs:
/usr/local/bin/k3s-uninstall.sh
sudo rm /etc/rancher/node/password
# From a working node:
kubectl delete node beehive[#]

# (Then see 40-longhorn/README.md for instructions on preparing the node for Longhorn.)

# Prepare the k3s install directory:
sudo mkdir /etc/rancher
sudo mkdir /etc/rancher/k3s

# Copy in the required configuration files:
sudo cp config-beehive[#].yaml /etc/rancher/k3s/config.yaml
sudo cp registries.yaml /etc/rancher/k3s/

# Install and start K3s (which should use the config.yaml file to configure k3s)
# (This command was taken from a second cluster node.  Apparently I did not capture
# the original cluster setup commands.)
curl -sfL https://get.k3s.io | sh -s -

# NOTE that we can install a specific version by injecting an ENV var into the CLI:
# (This is necessary if we are not running the current k3s release when re-creating a 
# node in the cluster.)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.30.5+k3s1 sh -s -

# Check the service status:
systemctl status k3s
journalctl -xeu k3s.service
k3s kubectl get nodes

# Make the K3S config accessible to other utilities, such as `cmctl`:
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml >> ~/.zshrc
```

### Install MetalLB

A load balancer is needed to enable traffic destined to services in the cluster to
reach their respective "Ingress" resources.  We will use "MetalLB" to avoid the need to
deploy an external hardware load balancer.  This should work well on small, simple
networks like home networks or small office environments.

See [20-metallb](/20-metallb/README.md)

### Install Reflector

We want to minimize the number of TLS certificates required to operate our cluster. We
will create one wildcard certificate for our cluster, then use "Reflector" (one of many
K8S resource replicators) to copy our issued certs between K8s namespaces.

Reflector is a prerequisite for cert-manager.  We will use it to replicate certificates
provisioned by cert-manager between Kubernetes namespaces.

See [21-reflector](/21-reflector/README.md)

### Install Cert-Manager

We will use cert-manager to automate the issuance of TLS certificates within our cluster
using the ACME protocol as implemented by the "Let's Encrypt" service, in combination
with DNS01 services provided by CloudFlare.

Cert-Manager is a prerequisite for Ingress installation, as the NGinx Ingress will
use cert-manager-provisioned certificates for connection security.

See [22-cert-manager](/22-cert-manager/README.md)

### Install Ingress

We will use the NGinx Ingress for Kubernetes to handle routing of external requests to
services running in K8s.  While it may be possible to get by without an Ingress, using
one allows us more flexibility in how we secure access to services within K8S.  Services
such as the Longhorn UI do not natively support TLS *or* authentication.  The Ingress
allows us to add both layers of security to a resource that otherwise would be "wide
open" on the LAN.

See [25-nginx-ingress](/25-nginx-ingress/README.md)

### Install Dashboard

See [30-dashboard](/30-dashboard/README.md)

### Install Multus

Multus is required to give pods access to alternate network interfaces on out K8S nodes.

Multus is a prerequisite for Longhorn because we plan to use Multus to allow storage
replication services to run on a different interface from user-facing traffic.  This
will help prevent heavy network workloads from degrading services running on the LAN.

See [30-multus](/30-multus/README.md)

### Install Longhorn

We will use longhorn to provide SAN-like services using replicated local storage.
Longhorn is not the highest-performance replicated storage option, but it is well
documented for use with K3S, and it is relatively simple to install and configure.

In the near future, Longhorn also will support the use of NVMe-over-TCP, which WILL
provide SIGNIFICANTLY higher performance, if we feel we need it.

Longhorn is a prerequisite for any workloads that we will run that require persistent
storage.

See [40-longhorn](/40-longhorn/README.md)

### Install KubeVirt

I don't know if we actually need this, but KubeVirt allows us to run traditional virtual
machines within the K8S framework.  This can be very useful when running services that
are not feasible to run within containers.

Support for live migration of VMs when using Longhorn is doubtful, but I don't know that
we particularly care.

See [45-kubevirt](/45-kubevirt/README.md)

### Next steps: Nextcloud Helm Chart?

## ~~MicroK8s Configuration~~

MicroK8S is a problem: Looks convenient, but does not actually work well.  Pretty much
all of the supported plugin services are poorly maintained and out-of-date.  I think we
are better off with a more "vanilla" Kubernetes service.

1. Form an HA Cluster:

    ```bash
    # The Ubuntu installer did not install the latest stable MicroK8s snap.
    # Lets update!  On each node:
    snap refresh microk8s --channel=1.31/stable

    # Run the microk8s add-node command:
    $ microk8s add-node
    From the node you wish to join to this cluster, run the following:
    microk8s join 192.168.3.1:25000/[RANDOMISH_ONETIME_CODE]
    ...

    # From an un-joined node paste the generated command:
    $ microk8s join 192.168.3.1:25000/[RANDOMISH_ONETIME_CODE]
    Contacting cluster at 192.168.3.1
    Waiting for this node to finish joining the cluster. .. .. .. ..
    Successfully joined the cluster.

    # Repeat the process for the remaining node.

    ### Install add-ons...

    # MetalLB, a Kubernetes-native load balancer
    # We will use this to serve ingress IP addresses. Feed the desired IP address range
    # for Metal LB to use at the time of enablement:
    microk8s enable metallb:192.168.3.33-192.168.3.62

    # Ingress, a... reverse proxy? web application router? for K8S
    # microk8s uses the popular nginx-ingress project
    microk8s enable ingress

    # cert-manager, for generating TLS certificates
    # Can be configure to use the LetsEncrypt API!
    microk8s enable cert-manager
    

    # Registry: Local storage for container images.
    # I dont think we need a registry, and if we did want a registry, it probably would
    # not be this built-in thing, which does not appear to work with clustering.

    # Dashboard: A web console and for Kubernetes
    microk8s enable dashboard

    # Install Mayastor:
    # What are you thinking? DO NOT install Mayastor!
    ```

## ~~Ingress Traffic and Load Balancing~~

Currently we are using CloudFlare as a DNS provider with Let's Encrypt for certificates.
These steps were used when attempting to use DuckDns with Let's Encrypt, which was
pretty difficult to support.

1. Create a Service for the Engine Ingress that gives a MetalLB static IP to the controller
2. Obtain a duckdns.org address, get the auth token for the address.
3. Configure local router to redirect \*.mackinnon.duckdns.org queries to the ingress IP
4. Install the cert-manager-duckdns-webhook using Helm
5. Create ingresses! Nginx annotations of note:  
   <https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#backend-protocol>

## Troubleshooting

### Name resolution problems

- Disable all DHCP assignments in `/etc/netplan/*` then apply using `sudo netplan apply`
- If the local DNS server (opnsense) is causing problems, we can bypass it by editing `/run/systemd/resolve/resolv.conf` then running `sudo systemctl restart systemd-resolved`

### Reconfiguring K3s after installation

1. Edit the local `/etc/rancher/k3s/config.yaml` with the desired changes.
2. Run: `sudo systemctl daemon-reload && sudo systemctl restart k3s`. Changes in the local config are applied on restart!

## Resources

DNS Troubleshooting in Kubernetes:  
<https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/>

Nginx Ingress Annotations:  
<https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/>

Cert-Manager Ingress Annotation:  
<https://cert-manager.io/docs/usage/ingress/>

Troubleshooting ACME Certificate Issuance with cert-manager:  
<https://cert-manager.io/docs/troubleshooting/acme/>

Source / Documentation for the DuckDNS webhook for cert-manager:  
<https://github.com/csp33/cert-manager-duckdns-webhook/tree/master>

Example: Exposing the Kubernetes Dashboard using Ingress on MicroK8s:  
<https://jonathangazeley.com/2020/09/16/exposing-the-kubernetes-dashboard-with-an-ingress/>

MicroK8s cluster upgrade procedure:  
<https://microk8s.io/docs/upgrade-cluster>

Ubuntu snap management: purging or deleting snapshot backups:  
<https://askubuntu.com/questions/1354549/how-do-you-purge-a-removed-app-or-delete-snapshot-after-app-removal>

Removing a node from a MicroK8s cluster:  
(Dirtly little trick for solving node problems... leave and re-join.)  
<https://microk8s.io/docs/clustering#removing-a-node>
