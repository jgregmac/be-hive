# Installing NginX Ingress

There is a documented method for installing the NGinx Ingress Controller using OLM, but don't try to use it.  If you
are not using OpenShift, you have to compile the operator before installing it, and that just does not save anyone any
time at all. Why not install from OperatorHub.io?  Because its not there!  The Operator only is published in the Red
Hat operator catalog for OpenShift.

```shell
# Install from the helm chart into the "ingress" namespace.
# use the values file to force the ingress to run on a pre-selected IP address.
helm install nginx-ingress oci://ghcr.io/nginxinc/charts/nginx-ingress \
    --version 1.4.0 \
    -n ingress --create-namespace \
    -f 10-ingress-values.yaml
```

## References

Installation with Helm:  
<https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/>