# Installing cert-manager for K3s

## Install the cmctl command line tool

Some cert-manager operations require the `cmctl` command line tool.  Install it:

1. Get Homebrew:  
   <https://brew.sh/>  
   `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`  
   (Then follow prompts to update your shell environment to support the `brew` command.)
2. Install `cmctl`:  
   `brew install cmctl`

## Install cert-manager using Helm

See: <https://cert-manager.io/docs/installation/helm/>

```bash
brew install helm
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.1 \
  --set crds.enabled=true
```

## Create a ClusterIssuer

In order for cert-manager to do anything, you need to configure Issuers to instruct the
operator the services to use for certificate requests and signing.  We are using
"letsencrypt" as our certificate issuer, and the "cloudflare" provider for ACME-DNS01
validation.

Before proceeding we will need:

1. A registered domain in CloudFlare
2. An API key in CloudFlare for updating this domain's TXT records.

```bash
# Apply the secret that contains the CloudFlare API key:
oc apply -f 00-Secret-cloudflare-api-key.yaml

# Create a cluster issuer for LetsEncrypt/CloudFlare:
kubectl apply -f 20-ClusterIssuer-cloudflare-prod.yaml

# Create any namespace to which the certificates that you wish to mint will be reflected:
kubectl create ns ingress
kubectl create ns kubernetes-dashboard
kubectl create ns longhorn-system

# Trigger cert-manager to create wildcard certs for our cluster:
kubectl apply -f 30-Certificate-be-hive.yaml
```

## Observing Cert-Manager

```bash
# Observe the state of a Certificate:
$ kubectl get certificate -n cert-manager
NAME            READY   SECRET              AGE
be-hive-cloud   False   be-hive-cloud-tls   26s

# If READY is not True, you can get more detailed info about the request using "describe":
kubectl describe certificate -n cert-manager

# cert-manager will have created a "CertificateRequest" resource for your cert:
kubectl get certificaterequest -n cert-manager be-hive-cloud-1 -o yaml

# which will, in turn, create an "Order" for the cert:
kubectl get order -n cert-manager be-hive-cloud-1-3335386594 -o yaml

# Finally, the "Order" is dependent on "Challenges" for completion:
kubectl get challenges -n cert-manager
# (You can inspect the individual challenges associated with this order.)
```

In one case we observed challenges hanging due to this issue:
<https://github.com/cert-manager/cert-manager/issues/7540>

There was a change in the CloudFlare API which has broken the cert-manager integration.
The workaround is to manually delete the TXT records created by cert-manager in the
CloudFlare console.

This issue supposedly is resolved in cert-manager 1.17.1:
<https://github.com/cert-manager/cert-manager/releases/tag/v1.17.1>

## Resources

Operator SDK Installation:  
<https://sdk.operatorframework.io/docs/installation/>

Krew Installation:  
<https://krew.sigs.k8s.io/docs/user-guide/setup/install/>

Installing cert-manager using OLM:  
<https://cert-manager.io/docs/installation/operator-lifecycle-manager/>

## Retired: Install cert-manager using the OLM Operator

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

# Uninstall the dang operator:
# Cue deletion of all "operands" (cert-manager resources) and deletion of the operator:
kubectl operator uninstall cert-manager -n cert-manager -s delete

# clean up resources not removed by the operator uninstall process:
for CRD in $(kubectl get crd -o name | grep cert-manager); do kubectl delete $CRD; done
for role in $(kubectl get clusterrole -o name | grep cert-manager); do kubectl delete $role; done
```

For reference, we have saved the deployed resources here in [99-operator-reference.yaml](99-operator-reference.yaml)