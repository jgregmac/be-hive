# Installing Longhorn

Longhorn:  A cloud-native replicated storage engine for Kubernetes.  It's main benefit is relative simplicity in setup in administration.  Downside, it's not particularly fast.  Let's give it a try...

## Architecture

In the initial deployment, we will assign one NVMe disk from each node as the sole backing disk for Longhorn. The "dev" for these disks is inconsistent, unfortunately, so we plan to use "labels"
to provide a consistent path to the ext4 filesystem that we will create for use by Longhorn.

> /dev/nvme1n1
> /dev/nvme0n1
> /dev/disk/by-path/pci-0000:01:00.0-nvme-1
> ?

We have a second gigabit ethernet port on each node.  We plan to dedicate this port to storage networking using the Multus CNI plugin in "thin" mode.

Note that Lonhorn uses a "Settings" CRD for storing service settings in Kubernetes.  Weird.

## Installation

1. Install Multus CNI using the "thin mode" manifests:
   `kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml`
2. Update node labels for Longhorn and apply:

    ```bash
    kubectl patch node beehive1 -f 10-disk-annotations.yaml
    kubectl patch node beehive2 -f 10-disk-annotations.yaml
    kubectl patch node beehive3 -f 10-disk-annotations.yaml
    ```

3. Format and mount NVMe storage for Longhorn:

    ```bash
    $ lsblk
    NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    nvme0n1                   259:0    0   1.8T  0 disk
    $ sudo mkfs.ext4 -L longhorn /dev/nvme0n1
    $ sudo -s
    $ echo "/dev/disk/by-label/longhorn /var/mnt/longhorn ext4 defaults 0 1" >> /etc/fstab
    $ reboot
    ```

## References

Installing Multus networking support:  
<https://github.com/k8snetworkplumbingwg/multus-cni#comprehensive-documentation>

Installing with Helm:  
<https://longhorn.io/docs/1.7.1/deploy/install/install-with-helm/>

Securing the Longhorn dashboard with Ingress and Basic Auth:  
<https://longhorn.io/docs/1.7.1/deploy/accessing-the-ui/longhorn-ingress/>

Multipathd warnings and errors:  
<https://longhorn.io/kb/troubleshooting-volume-with-multipath/>

Setting KUBELET_ROOT_DIR to allow kubelet detection:  
<https://longhorn.io/docs/1.7.1/advanced-resources/os-distro-specific/csi-on-k3s/>

Platform specific notes for OkD / OpenShift  
*Even though this doc is for OpenShift, not MicroK8s, it is the only page in the documentation that demonstrates how to format and mount storage for use by Longhorn!*  
<https://longhorn.io/docs/1.7.1/advanced-resources/os-distro-specific/okd-support/>
