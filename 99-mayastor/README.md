# OpenEBS Mayastor configuration

OpenEBS / Mayastor has not worked out for us. Can't mount volumes, can't figure out what
is wrong, and Hugepage support that is required for Mayastor creates other problems that
I really don't want to deal with.  Too bad, since it looks to be really fast...

If Longhorn does not work out, we can revisit this storage option.

NOTE:  
MicroK8s has a built-in add on for Mayastor, the NVM-over-TCP engine for OpenEBS.  You
might be tempted to use it, but DO NOT. It is hopelessly outdated.  This add on is
based on a much over code branch of Mayastor, from before it was fully integrated into
OpenEBS. The built-in version has no support for topology labels, making it impossible
to link Kubernetes StorageClasses to DiskPools.  This is a problem if you plan to have
different classes of storage accessible by OpenEBS.

See:  
<https://openebs.netlify.app/docs/quickstart-guide/installation>

## Base OpenEBS Install

```shell
helm repo add openebs https://openebs.github.io/openebs
helm repo update
# We have prepared a Helm values file that disables OpenEBS engines that we do not
# intend to use, but note that the values no not appear to be respected at install time.
# Boo!
helm install openebs -n openebs openebs/openebs --create-namespace -f 00-helm-values.yaml
```

## Mayastor enablement

```shell
# Label all nodes as being valid for use with mayastor:
for NODE in $(kubectl get nodes -o name); do 
    kubectl label $NODE openebs.io/engine=mayastor;
done
# Create Mayastor disk pools for our local NVMe drives:
kubectl apply -f 10-DiskPools-nvme.yaml
# Create a storage class that uses the disk pool (based on topology labels):
kubectl apply -f 20-StorageClass.yaml
```
