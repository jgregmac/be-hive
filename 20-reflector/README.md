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
