![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
# UKSRC Apps and Services Deployment

This guide will demonstrate how to deploy each of the applications & services required for srcNet0.1 and subsequent versions.

Once you have deployed the Kubernetes cluster and are satisfied it is ready, you will need to make the following changes to deploy the necessary apps and services for srcNet0.1

Cluster configuration is managed in the clusters directory and the components directory contains all the components for managing the cluster. We therefore do not add any of our apps or services to these diretories.

Applications are managed under the apps directory and services are managed under the infra directory. If you are adding new applications or services to the srcNet0.1 deployment they should be placed in the applicable directory.

Each application or service may also have a base and overlays folder. The base directory contains the base application and the overlays folders will contain site / cluster specific values or secrets for that site.

Each Cluster will require Kustomization to deploy the applications and services and some applications and services will in turn require Kustomization for site specific values and secrets.

All secrets are sealed using kubeseal which uses the cluster certificate to encrypt & decrypt the secret, allowing it to be stored in the Gitlab repository. Please note the `secrets` folder will not be part of this directoy structure and must be added manually, this is because it is purposefully ignore using `.gitignore` to prevent unencrypted secrets being push to the repository. This will be explained in further detail later in the document.

## Preparing for the Deployment

We will be deploying the following Apps and Services.

- Infrastructure Services
    - Ceph CSI Driver
    - Monitoring
- Apps
    - GateKeeper
    - SODA
    - Jupyterhub
    - CANFAR


For the purposes of this deployment we've deployed the cluster at RAL, so references and file structure will reflect that. Make the necessary changes for your site / cluster. 

### Secrets

In order to keep our unsealed secrets secure and not have them accidentally commited to the Git repo, create a secrets directory and create the unencrypted secret files under the cluster secrets directory, this is protected by `.gitignore` rules.

Using the structure as per example below.

```sh
secrets/
│
└── uksrc-ral-prod
    ├── cloud-credentials.yaml
    ├── csi-cephfs-secret.yaml
    ├── gatekeeper-secret.yaml
    ├── jupyterhub-secret-config.yaml   
    └── secret-oidc.yaml
```

### Configure the ceph-csi-cephfs Driver

Create a new overlay `ral` directory for your site in the `infra/ceph-share/overlays` directory.

```sh
infra/
└──  ceph-share
    ├── base
    │   ├── csi-cephfs-secret.yaml
    │   ├── helmchart.yaml
    │   ├── helmrelease.yaml
    │   ├── helmrepository.yaml
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    ├── kustomization.yaml
    └── overlays
        └── ral
            ├── csi-ceph-values.yaml
            ├── csi-cephfs-secret.yaml
            └── kustomization.yaml
```

Create a `csi-ceph-values.yaml` file as per below.

The values are unique to your CephFS deployment, `fsid` , `mon_host` , `mon_initial_members` , `public_network` , `clusterID` , `monitors` and should be updated accordingly.

```
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: ceph-csi
  namespace: csi-ceph-system
spec:
  values:
    cephconf: |
      [global]
        auth supported = cephx
        fsid=2660d5ed-e7ef-4950-a051-c84a0b4e78b0
        auth_cluster_required = cephx
        auth_service_required = cephx
        auth_client_required = cephx
        mon_host=deneb-dev-mon1.nubes.rl.ac.uk, deneb-dev-mon2.nubes.rl.ac.uk, deneb-dev-mon3.nubes.rl.ac.uk
        mon_initial_members=deneb-dev-mon1,deneb-dev-mon2,deneb-dev-mon3
        public_network=130.246.208.0/23,130.246.181.0/24
    keyring: ""

    csiConfig:
      - "clusterID": "2660d5ed-e7ef-4950-a051-c84a0b4e78b0"
        "monitors":
           - "deneb-dev-mon1.nubes.rl.ac.uk"
           - "deneb-dev-mon2.nubes.rl.ac.uk"
           - "deneb-dev-mon3.nubes.rl.ac.uk"
```

Create the `csi-cephfs-secret.yaml` secret in the secrets folder as above, as per the example.

```sh
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: csi-ceph-system
  annotations:
    sealedsecrets.bitnami.com/managed: "true"
type: Opaque
stringData:
  userID: username
  userKey: xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secret using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-prod/csi-cephfs-secret.yaml --sealed-secret-file infra/ceph-share/overlays/ral/csi-cephfs-secret.yaml
```

Edit the `clusters/uksrc-ral-prod/kustomization.yaml` for the cluster.

Add the new `ceph-share`resource to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`, note the `issuers.yaml` is common to all clusters but not deployed with bootstrap so added now. 

Note we point our resources to `overlays` for the apps and services that need site or cluster specific values.

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
  # - ../../infra/monitoring/overlays/ral
  # - ../../apps/soda/overlays/ral
  # - ../../apps/gatekeeper/overlays/ral
  # - ../../apps/jupyterhub/overlays/ral
  # - ../../apps/canfar/overlays/ral
```

Commit & push your changes to Gitlab. You should be able to see that the Ceph CSI driver has been deployed.


### Monitoring

Create a new overlay `ral` directory for your cluster in the `monitoring/overlays` directory.

Create a `kustomization.yaml` as per the example, updating the values to match your CName / hostname.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - target:
      kind: Ingress
      name: grafana-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: grafana.ral.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value: 
          - grafana.ral.uksrc.org     
  - target:
      kind: Ingress
      name: prometheus-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: prometheus.ral.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value: 
          - prometheus.ral.uksrc.org
```

Add the new `monitoring`resources to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
  # - ../../apps/soda/overlays/ral
  # - ../../apps/gatekeeper/overlays/ral
  # - ../../apps/jupyterhub/overlays/ral
  # - ../../apps/canfar/overlays/ral
```


Check your build using the `kustomize build` command. 

```sh
kustomize build infra/monitoring/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the monitoring has been deployed.


### SODA Service

Create a new overlay `ral` directory for your site in the `apps/soda/overlays` directory.

Now create a `kustomization.yaml` as per the example,  updating the `clusterID` value.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - ceph-volume.yaml
patches:
  - path: soda-values.yaml
    target:
      kind: HelmRelease
      name: soda-release
```

Create a `soda-values.yaml` as per the example, updating the values to match your hosts.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: soda-release
  namespace: ska-src-soda
spec:
  values:
    ingress:
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - soda.ral.uksrc.org
      tls:
        - hosts:
          - soda.ral.uksrc.org
          secretName: soda-ingress-cert
```

Add the new `SODA`resources to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
  # - ../../apps/gatekeeper/overlays/ral
  # - ../../apps/jupyterhub/overlays/ral
  # - ../../apps/canfar/overlays/ral
```

Check your build using the `kustomize build` command. 

```sh
kustomize build apps/soda/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the SODA Service has been deployed.

### GateKeeper Service

Create a new overlay `ral` directory for your site in the `apps/gatekeeper` directory.

Create a `kustomization.yaml` as per the example,  updating the `clusterID` value.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - gatekeeper-secret.yaml
patches:
  - path: gatekeeper-values.yaml
    target:
      kind: HelmRelease
      name: ska-src-service-gatekeeper
  - target:
      kind: Ingress
      name: gatekeeper-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: gatekeeper.ral-stage.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value:
          - gatekeeper.ral-stage.uksrc.org
```

Create a `gatekeeper-values.yaml` as per the example, updating the values to match your hosts.

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: gatekeeper
  namespace: ska-src-gatekeeper
spec:
  values:
    gatekeeper:
      services:
        - route: "/echo"
          namespace: ska-src-gatekeeper-echo
          prefix: "http://"
          service_name: "ska-src-gatekeeper-echo"
          ingress_host: "gatekeeper.ral-stage.uksrc.org"
          port: 8080
          uuid: "e98b8c67-7f06-47e6-9618-5cf471483fd5"
        - route: "/soda"
          namespace: ska-src-soda
          prefix: "http://"
          service_name: "ska-src-soda"
          ingress_host: "gatekeeper.ral-stage.uksrc.org"
          port: 8080
          uuid: "3b59c8b6-7717-4f58-9b63-8ee3c0e89276"
```

Create the `gatekeeper-secret.yaml` secret in the secrets folder as above, as per the example.

```sh
---
apiVersion: v1
kind: Secret
metadata:
  name: ska-src-gatekeeper-secret
  namespace: ska-src-gatekeeper
  annotations:
    sealedsecrets.bitnami.com/managed: "true"
type: Opaque
stringData:
  site_capabilities_gatekeeper_client_id: e98b8c67-7f06-47e6-9618-5cf471483fd5
  site_capabilities_gatekeeper_client_secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secret using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-prod/gatekeeper-secret.yaml --sealed-secret-file apps/gatekeeper/overlays/ral-stage/gatekeeper-secret.yaml
```

Add the new `gatekeeper`resource to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
  # - ../../apps/jupyterhub/overlays/ral
  # - ../../apps/canfar/overlays/ral
```

Check your build using the `kustomize build` command. 

```sh
kustomize build apps/jupyterhub/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that Jupyterhub has been deployed.


### Jupyterhub

Create a new overlay `ral` directory for your site in the `apps/jupyterhub/overlays` directory.

Create a `kustomization.yaml` as per the example,  updating the `clusterID` value.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - path: jupyterhub-values.yaml
    target:
      kind: HelmRelease
      name: jupyterhub
  - target:
      kind: PersistentVolume
      name: jupyterhub-cephfs-static-pv
    patch: |-
      - op: replace
        path: /spec/csi/volumeAttributes/clusterID
        value: 2660d5ed-e7ef-4950-a051-c84a0b4e78b0
  - path: secret.yaml
  ```

Create a `jupyterhub-values.yaml` as per the example, updating the values to match your CName / hostname.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: jupyterhub
  namespace: jupyterhub
spec:
  values:
    ingress:
      hosts:
        - jupyterhub.ral.uksrc.org
      tls:
        - hosts: 
          - jupyterhub.ral.uksrc.org
          secretName: jupyterhub-ingress-cert

    hub:
      config:
        GenericOAuthenticator:
          oauth_callback_url: https://jupyterhub.ral.uksrc.org/hub/oauth_callback
          authorize_url: https://ska-iam.stfc.ac.uk/authorize
          token_url: https://ska-iam.stfc.ac.uk/token
          userdata_url: https://ska-iam.stfc.ac.uk/userinfo
        baseURL: https://jupyterhub.ral.uksrc.org/
```

Create the `secrets/uksrc-ral-prod/jupyterhub-secret.yaml` secret in the secrets folder, as per the example.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: jupyterhub-secret-config
  namespace: jupyterhub
  sealedsecrets.bitnami.com/managed: "true"
type: Opaque
stringData:
  client_id: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
  client_secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secret using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-prod/jupyterhub-secret.yaml --sealed-secret-file apps/jupyterhub/overlays/ral/secret.yaml
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
  # - ../../apps/canfar/overlays/ral
```


Check your build using the `kustomize build` command. 

```sh
kustomize build apps/jupyterhub/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that Jupyterhub has been deployed.


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

Create the `secrets/uksrc-ral-stage/secret-oidc.yaml` secret in the secrets folder, as per the example.

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
  clientSecret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Seal the secret using the command below.

```sh
kubeseal --kubeconfig clusters/uksrc-ral-prod/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-stage/secret-oidc.yaml --sealed-secret-file apps/canfar/overlays/ral-stage/sealed-oidc-secret.yaml
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