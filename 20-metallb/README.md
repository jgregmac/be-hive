# Installing and configuring MetalLB

```shell
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl apply -f 10-IPAddressPool.yaml
kubectl apply -f 10-L2Advertisement.yaml
```

## References

Installation using Helm:  
<https://metallb.universe.tf/installation/#installation-with-helm>