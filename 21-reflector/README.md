# Reflector Installation and Configuration

Installation of the Reflector controller is straightforward, requiring no modifications
of the stock installation values.

See:  
<https://github.com/emberstack/kubernetes-reflector?tab=readme-ov-file#deployment>

```bash
helm repo add emberstack https://emberstack.github.io/helm-charts
helm repo update
helm upgrade --install reflector emberstack/reflector -n reflector --create-namespace
```

My experience with Reflector has not been overly positive.  I often need to restart the
reflector pod to force re-replicaiton of outdated or deleted secrets.  In the past I
have used the Kubernetes Replicator from Mittwald.de.  That may be the way forward.
