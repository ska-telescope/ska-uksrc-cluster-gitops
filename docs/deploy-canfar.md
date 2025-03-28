![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### CANFAR

Create a new overlay `ral` directory for your site in the `apps/canfar/overlays` directory.

Create a `kustomization.yaml` as per the example,  updating the values for your cluster.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - sealed-oidc-secret.yaml
patches:
  - target:
      kind: Ingress
      name: canfar-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: canfar.ral.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value:
          - canfar.ral.uksrc.org
  - target:
      kind: HelmRelease
      name: base
    patch: |-
      - op: replace
        path: /spec/values/traefik/service/spec/loadBalancerIP
        value: "130.246.80.70"
  - target:
      kind: HelmRelease
      name: cavern
    patch: |-
      - op: replace
        path: /spec/values/deployment/hostname    
        value: "canfar.ral.uksrc.org"
      - op: replace
        path: /spec/values/deployment/cavern/resourceID
        value: ivo://canfar.ral.uksrc.org/posix-mapper
      - op: replace
        path: /spec/values/deployment/cavern/posixMapperResourceID
        value: ivo://canfar.ral.uksrc.org/posix-mapper
  - target:
      kind: HelmRelease
      name: posix-mapper
    patch: |-
      - op: replace
        path: /spec/values/deployment/hostname
        value: "canfar.ral.uksrc.org"
      - op: replace
        path: /spec/values/posixMapper/resourceID
        value: ivo://canfar.ral.uksrc.org/posix-mapper
  - target:
      kind: HelmRelease
      name: scienceportal
    patch: |-
      - op: replace
        path: /spec/values/deployment/hostname
        value: "canfar.ral.uksrc.org"
      - op: replace
        path: /spec/values/deployment/sciencePortal/oidc/redirectURI
        value: "https://canfar.ral.uksrc.org/science-portal/oidc-callback"
      - op: replace
        path: /spec/values/deployment/sciencePortal/oidc/callbackURI
        value: "https://canfar.ral.uksrc.org/science-portal/"
  - target:
      kind: HelmRelease
      name: skaha
    patch: |-
      - op: replace
        path: /spec/values/deployment/hostname
        value: "canfar.ral.uksrc.org"
      - op: replace
        path: /spec/values/deployment/skaha/posixMapperResourceID
        value: ivo://canfar.ral.uksrc.org/posix-mapper
  - target:
      kind: HelmRelease
      name: storage-ui
    patch: |-
      - op: replace
        path: /spec/values/deployment/hostname
        value: "canfar.ral.uksrc.org"
      - op: replace
        path: /spec/values/deployment/storageUI/oidc/redirectURI
        value: "https://canfar.ral.uksrc.org/storage/oidc-callback"
      - op: replace
        path: /spec/values/deployment/storageUI/oidc/callbackURI
        value: "https://canfar.ral.uksrc.org/storage/list"
      - op: replace
        path: /spec/values/deployment/storageUI/backend/service/cavern/resourceID
        value: ivo://canfar.ral.uksrc.org/cavern
      - op: replace
        path: /spec/values/deployment/storageUI/backend/service/cavern/nodeURIPrefix
        value: "vos://canfar.ral.uksrc.org~cavern"
```

Create the `secrets/uksrc-ral-prod/canfar-oidc-secret.yaml` secret in the secrets folder, as per the example using the Client ID & Client Secret from the SKA-IAM client.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: oidc-credentials
  namespace: skaha-system
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true"
stringData:
  clientID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  clientSecret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secrets using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-prod/canfar-oidc-secret.yaml --sealed-secret-file apps/canfar/overlays/ral-prod/sealed-oidc-secret.yaml
```

Another secret must be created, create the `secrets/uksrc-ral-prod/canfar-oidc-secret.yaml` secret in the secrets folder, as per the example using the Client ID & Client Secret from the SKA-IAM client.

```
---
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: storage-ui-credentials
  namespace: skaha-system
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true" 
stringData:
  clientID: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  clientSecret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secrets using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-prod/canfar-storage-ui-secret.yaml --sealed-secret-file apps/canfar/overlays/ral-prod/sealed-storage-ui-secret.yaml
```

Edit the `clusters/uksrc-ral-prod/kustomization.yaml` for the cluster.

Add the new `jupyterhub`resource to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`, note the `issuers.yaml` is common to all clusters but not deployed with bootstrap so added now. Note we point our resources to `overlays` for the apps and services that need site or cluster specific values.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../components/flux
  - ../../components/sealed-secrets
  - ../../components/cluster-api
  - ../../components/cluster
  - configmap.yaml
  - credentials-sealed.yaml

  # After bootstrap add the apps and services here
  - ../../infra/issuers
  - ../../infra/ceph-share/overlays/ral
  - ../../infra/monitoring/overlays/ral
  - ../../apps/soda/overlays/ral
  - ../../apps/gatekeeper/overlays/ral
  - ../../apps/jupyterhub/overlays/ral
  - ../../apps/canfar/overlays/ral
```


Check your build using the `kustomize build` command. 

```sh
kustomize build apps/canfar/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that CANFAR has been deployed.

[Deploy prepareData](./deploy-preparedata.md)

[Document Home](./readme.md)