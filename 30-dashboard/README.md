# Setup and Access for the Kubernetes Dashboard

The Kubernetes dashboard included with MicroK8s is v2.6.0, from 2022.  Literally 2.5
years old at the time of the K8s 1.31 release!

## Install

```bash
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
    -f 10-dashboard-values.yaml \
    --create-namespace -n kubernetes-dashboard
```

## Deploy an Ingress for dashbaord access

```shell
kubectl apply -f 10-Ingress-dashboard.yaml
```

## Configure Private DNS

At the router, access the DNS service config, and add a new A record override:

dashboard.mackinnon.duckdns.org -> 192.168.3.33

## Create a service account and token

See:  
<https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md>

```shell
kubectl apply -f 10-DashboardAccountToken.yaml
```

## Accessing the long-lived token

```shell
$ kubectl get secret dashboard-admin -n kube-system -o jsonpath={".data.token"} \
    | base64 -d
ACCESS_TOKEN_WILL_BE_DISPLAYED_HERE
```

## Access the dashboard

In a browser, visit:  
<https://dashboard.be-hive.cloud/>

## References

Installation:  
<https://github.com/kubernetes/dashboard?tab=readme-ov-file#installation>