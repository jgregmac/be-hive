# Install Multus CNI

We will install Multus to allow pods in the cluster to attach directly to alternate host
network interfaces.  The primary use case for this is to allow Longhorn storage
replication to attach to an alternate network interface to isolate storage replication
traffic from user data traffic.

We will need to use the "whereabouts" IPAM plugin for Multus to manage IP addresses to
be used by such pods.

Documentation of Multus in the primary project pages is pretty poor.  The best support
resources on Multus come out of Red Hat, who likely leads the development of the tool.

Arriving at a working config was... painful.  I need to add a testing config to the end
of the install process as it is _critical_ to validate Multus before deploying any
future resources that will depend on it.

## Configuration choices

1. We will use the "whereabouts" IPAM plugin to manage IP addresses used by pods that
   call Multus.
2. The network range 192.168.4.1-192.168.4.255 will be assigned to Multus.  This is a
   lot more than is needed, but we have no other plans for this IP range.
3. We will use the "macvlan" multus plugin as it appears to be the simplest option that
   will work best on a traditional LAN.

## Install Process

Previously we installed Multus on KicroK8s using the "authoritative" chart:

1. Install Multus CNI using the "thin mode" manifests:
   `kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml`

But for K3s, it is recommended to use the Rancher chart:

1. Add the repo:

    ```shell
    helm repo add rke2-charts https://rke2-charts.rancher.io
    helm repo update
    ```

2. Prepare a values file with whereabouts sub-chart.  
   (NOTE: Post October 2024, the multus helm values recommended for install with
   whereabouts have changed.  A more stable "bin" path has been supplied.)
3. Install:

    ```shell
    helm install multus rke2-charts/rke2-multus -n kube-system \
        -f 10-multus-values.yaml
    ```

4. Once installed, we can create a network attachment definition that will
allow pods that need to use the storage network to attach an interface
automatically, when called using pod annotations.

    ```shell
    kubectl apply -f 20-networkAttachmentDefinition.yaml

    ```

## Testing

## References

Installing Multus on K3s:  
<https://docs.k3s.io/networking/multus-ipams>

Whereabouts configration:  
<https://github.com/k8snetworkplumbingwg/whereabouts>

Red Hat doc eplaining Multus plugins: macvlan, ipvlan, etc:  
<https://www.redhat.com/en/blog/using-the-multus-cni-in-openshift>
