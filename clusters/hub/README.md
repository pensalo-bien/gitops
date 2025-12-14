# Hub DR setup

* Step 1: Create a machine somewhere with at least 50GB of storage.
* Step 2: install k3s `curl -sfL https://get.k3s.io | sh -`
* Step 3: create some secrets

```
kubectl create namespace sfera &&
kubectl create secret docker-registry ghcr -n sfera --docker-server=https://ghcr.io --docker-username=the-username --docker-password=$GITHUB_TOKEN --docker-email=none.of@yourconce.rn
```

* Step 3: setup flux via helm

Adjust the version as needed (from [helm releases page](https://github.com/helm/helm/releases))
```curl https://get.helm.sh/helm-v4.0.4-linux-amd64.tar.gz```

untar it into /usr/local/bin, check for +x and then

```helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace \
  --wait
```

Have flux-cli installed: 
```curl -s https://fluxcd.io/install.sh | sudo bash```

Install secrets manually using a ssh key and apply it:

```flux create secret git my-secret-name \
 --private-key-file=/my/absolute/path/to/key/file.pem \
 --url=ssh://git@github.com/my-org-name/my-repo-name \
 --export > flux-secrets.yaml```

kubectl-apply the fluxInstance
```
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
  annotations:
    fluxcd.controlplane.io/reconcileEvery: "1h"
    fluxcd.controlplane.io/reconcileArtifactEvery: "10m"
    fluxcd.controlplane.io/reconcileTimeout: "5m"
spec:
  distribution:
    version: "2.7.3"
    registry: "ghcr.io/fluxcd"
    artifact: "oci://ghcr.io/controlplaneio-fluxcd/flux-operator-manifests"
  components:
    - source-controller
    - kustomize-controller
    - helm-controller
  cluster:
    type: kubernetes
    size: medium
    multitenant: false
    networkPolicy: true
    domain: "cluster.local"
  kustomize:
    patches:
      - target:
          kind: Deployment
        patch: |
          - op: replace
            path: /spec/template/spec/nodeSelector
            value:
              kubernetes.io/os: linux
          - op: add
            path: /spec/template/spec/tolerations
            value:
              - key: "CriticalAddonsOnly"
                operator: "Exists"
```

Once the deployments have settled go on with the repo and kustomization. Kubectl-apply this as well.

Watch out for the repo, branch and path
```
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  url: ssh://git@github.com/pensalo-bien/gitops
  ref:
    branch: wireguard
  secretRef:
    name: gitops-auth
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  path: ./clusters/hub
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

* Step 4: manually start the restore job (check credentials for OVH's S3)
* Step 5: Check that wireguard, Caddy and Sfera use the shared folders for saving

