# Installing NginX Ingress

There is a documented method for installing the NGinx Ingress Controller using OLM, but don't try to use it.  If you
are not using OpenShift, you have to compile the operator before installing it, and that just does not save anyone any
time at all. Why not install from OperatorHub.io?  Because its not there!  The Operator only is published in the Red
Hat operator catalog for OpenShift.

```shell
# Install from the helm chart into the "ingress" namespace.
# use the values file to force the ingress to run on a pre-selected IP address.
helm install my-release oci://ghcr.io/nginx/charts/nginx-ingress \
    --version 2.1.0\
    -n ingress --create-namespace \
    -f 10-ingress-values.yaml
```

Odd as it may seem, we are going to deploy separate Ingress resources for each service that we wish to expose outside
of the cluster. This seems goofy to an OpenShift admin, but here we don't have the concept of "default Routes" under
which all resources will be exposed.  Unless I am missing something?

## References

Installation with Helm:  
<https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/>