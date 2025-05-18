# Nextcloud Installation

1. Provision secrets for use by NextCloud and its database:

    ```bash
    kubectl apply -f 00-Secret-nextcloud-credentials.yaml
    ```

2. Install Nextcloud using Helm:

    ```bash
    helm repo add nextcloud https://nextcloud.github.io/helm/
    helm install behive nextcloud/nextcloud -n nextcloud \
        -f 10-helm-values.yaml
    ```

Mimic Liveness/Readiness probes:

```bash
# From another pod in the cluster: (You need to look up the current podIP):
curl -XGET -H "Host: nexcloud.be-hive.cloud" -XGET http://10.1.221.152:80/status.php

# From the nextcloud pod itself:
curl -XGET -H "Host: nexcloud.be-hive.cloud" -XGET http://localhost:80/status.php
```
