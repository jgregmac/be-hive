# Installing cert-manager for K3s

Let's try installing via OperatorHub.io.  We may need to back out if the OLM pods chew
up too many CPU resources, as they tend to do in OpenShift:

```bash
# Install the Operator Lifecycle Manager:


# Install Krew:
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Install the operator plugin for krew:
kubectl krew install operator

# Create a namespace to host the operator and cert-manager instance:
kubectl create ns cert-manager

# Install the operator!
kubectl operator install cert-manager -n cert-manager --channel stable --approval Automatic --create-operator-group
```

For reference, we have saved the deployed resources here in [99-operator-reference.yaml](99-operator-reference.yaml)

## Resources

Operator SDK Installation:  
<https://sdk.operatorframework.io/docs/installation/>

Krew Installation:  
<https://krew.sigs.k8s.io/docs/user-guide/setup/install/>

Installing cert-manager using OLM:  
<https://cert-manager.io/docs/installation/operator-lifecycle-manager/>