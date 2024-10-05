# Install Multus CNI

Previously we installed Multus on KicroK8s using the "authoritative" chart:

1. Install Multus CNI using the "thin mode" manifests:
   `kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml`

But for K3s, it is recommended to use the Rancher chart:

1. Add the repo:

    ```shell
    helm repo add rke2-charts https://rke2-charts.rancher.io
    helm repo update
    ```

2. Prepare a values file with whereabouts sub-chart.
3. Install:

    ```shell
    helm install multus rke2-charts/rke2-multus -n kube-system \
        -f 10-multus-values.yaml
    ```

4. Once installed, we can create a network attachment definition that will
allow pods that need to use the storage network to attach an interface 
automatically, when needed:

    ```shell
    kubectl apply -f 20-networkAttachmentDefinition.yaml
    ```