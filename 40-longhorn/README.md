# Installing Longhorn

Longhorn:  A cloud-native replicated storage engine for Kubernetes.  It's main benefit
is relative simplicity in setup in administration.  Downside, it's not particularly
fast.  Let's give it a try...

## Architecture

In the initial deployment, we will assign one NVMe disk from each node as the sole
backing disk for Longhorn. The "dev" path to these disks is inconsistent across nodes,
unfortunately, so we plan to use "labels" to provide a consistent path to the ext4
filesystem that we will create for use by Longhorn.

Paths observed on different nodes:

> /dev/nvme1n1
> /dev/nvme0n1
> /dev/disk/by-path/pci-0000:01:00.0-nvme-1
> ?

We have a second gigabit ethernet port on each node.  We plan to dedicate this port to
storage networking using the Multus CNI plugin in "thin" mode.

Note that Lonhorn uses a "Settings" CRD for storing service settings in Kubernetes.
Weird.

## Installation

1. Update the kubernetes node labels for Longhorn by applying the patch file.  The node
   annotations will instruct Longhorn how to configure storage on each node.

    ```bash
    kubectl patch node beehive1 --patch-file 10-disk-annotations.yaml
    kubectl patch node beehive2 --patch-file 10-disk-annotations.yaml
    kubectl patch node beehive3 --patch-file 10-disk-annotations.yaml
    ```

2. Format and mount NVMe storage for Longhorn. We will create a mount point:
   `/mnt/longhorn-nvme-1` to serve as the default disk for creation of Longhorn volumes.

    ```bash
    # YOU MUST VERIFY the name of the block device that will be used for longhorn.
    # IT IS NOT THE SAME NAME ON ALL NODES!!!
    $ lsblk
    NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    nvme0n1                   259:0    0   1.8T  0 disk

    # Prepare an ext4 filesystem on the target block device:
    $ sudo mkfs.ext4 -L longhorn-nvme-1 /dev/nvme0n1

    # Add a permanent mount for the new volume:
    $ sudo -s
    $ echo "/dev/disk/by-label/longhorn-nvme-1 /mnt/longhorn-nvme-1 ext4 defaults 0 1" >> /etc/fstab

    # reboot, then verify that the mount is present:
    sudo reboot
    # OR...  you can just force application of the fstab file:
    sudo systemctl daemon-reload
    sudo mount -a

    # Then...
    $ lsblk
    NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    nvme1n1                   259:0    0   1.8T  0 disk /mnt/longhorn-nvme-1
    ```

3. Prepare an HTAccess secret to protect the longhorn UI:

    ```shell
    # You will need to create an htaccess entry, add it to this secret manifest, then apply.

    # Install htpasswd utility
    sudo apt install -y apache2-utils

    # Create an HTPASSWD value:
    htpasswd -nb "USERNAME" "PASSWORD"

    # Create the longhorn-system namespace:
    kubectl create ns longhorn-system

    # Add the generated HTPASSWD value to your secret manifest, then apply it:
    kubectl apply -f 10-Secret-basic-auth.yaml
    ```

4. Install Longhorn:

    ```bash
    helm repo add longhorn https://charts.longhorn.io
    helm install longhorn longhorn/longhorn \
        -n longhorn-system \
        --create-namespace \
        --version 1.8.1 \
        -f longhorn/20-longhorn-custom-values.yaml

    # Check for created resources:
    kubectl -n longhorn-system get pod
    # (You will see a "driver deployer" running, after which a "ton" of daemonset pods
    #  will be created)
    # You also can scan for errors in the kubernetes event stream:
    kubectl events -n longhorn-system -f
    ```

## Post-Install Configuration

If we are going to use RWX access to Longhorn storage, be sure to enabled the following settings in the "settings" UI
of the Longhorn web console:

- Automatically Delete Workload Pod when The Volume Is Detached Unexpectedly: true
- Storage Network for RWX Volume Enabled: true

However, we should be aware that RWX volume support under Longhorn is, well, kinda wonky.  So use with caution.

(Note that you also can use `kubectl edit -n longhorn-system setting <setting-name>`)

Note also that any use of block devices may require filesystem permission remapping in
the pods that will access the block volume, at least if the pod is not running as root,
as per:  
<https://longhorn.io/docs/1.7.0/nodes-and-volumes/volumes/pvc-ownership-and-permission/#longhorn-pvc-with-block-volume-mode>

```yaml
spec:
  ...
  securityContext:
    ...
    supplementalGroups:
    - 6
```

## Uninstall

Can't get it working?  Try wiping it out and re-intalling.  Have you turned it off and
back on again?

```bash
kubectl -n longhorn-system patch -p '{"value": "true"}' --type=merge lhs deleting-confirmation-flag
helm uninstall longhorn -n longhorn-system

# You might want to wipe out the longhorn-system namespace:
kubectl delete ns longhorn-system

# You might need to remove CRDs, especially if you will not be re-installing or if you
# will be installing a different version:
???

# De-provision storage:
sudo umount /mnt/longhorn-nvme-1
# (Then remove the mountpoint for /dev/disk/by-label/longhorn-nvme-1 from /etc/fstab)

# Remove any dangling storage classes:
kubectl get storageclass
kubectl delete storageclass longhorn-XXXXXX
```

## References

Installing Multus networking support:  
<https://github.com/k8snetworkplumbingwg/multus-cni#comprehensive-documentation>

Installing with Helm:  
<https://longhorn.io/docs/1.8.1/deploy/install/install-with-helm/>

Securing the Longhorn dashboard with Ingress and Basic Auth:  
<https://longhorn.io/docs/1.8.1/deploy/accessing-the-ui/longhorn-ingress/>

Multipathd warnings and errors:  
<https://longhorn.io/kb/troubleshooting-volume-with-multipath/>

Setting KUBELET_ROOT_DIR to allow kubelet detection:  
<https://longhorn.io/docs/1.8.1/advanced-resources/os-distro-specific/csi-on-k3s/>

Platform specific notes for OkD / OpenShift  
*Even though this doc is for OpenShift, not k3s, it is the only page in the
documentation that demonstrates how to format and mount storage for use by Longhorn!*  
<https://longhorn.io/docs/1.8.1/advanced-resources/os-distro-specific/okd-support/>
