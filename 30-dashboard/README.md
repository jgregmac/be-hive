# Setup and Access for the Kubernetes Dashboard

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
<https://dashboard.mackinnon.duckdns.org/>
