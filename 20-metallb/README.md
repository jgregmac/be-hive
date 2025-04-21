# Installing and configuring MetalLB

```shell
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb -n metallb-system --create-namespace
kubectl apply -f 10-IPAddressPool.yaml
kubectl apply -f 10-L2Advertisement.yaml
```

## Monitoring and Troubleshooting

You can inspect pod logs for metallb:

```bash
# Get logs for the controller pod:
# (Responsible for deploying, updating, and restarting metallb resources)
kubectl logs -n metallb-system deploy/metallb-controller

# Get logs for the "speaker" daemonSet:
# (Responsible for load balancing work)
kubectl logs -n metallb-system ds/metallb-speaker
```

## References

Installation using Helm:  
<https://metallb.universe.tf/installation/#installation-with-helm>