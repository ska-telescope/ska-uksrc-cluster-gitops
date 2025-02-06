![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)

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

```yaml
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
```

Check your build using the `kustomize build` command. 

```sh
kustomize build infra/ceph-share/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the Ceph CSI driver has been deployed.

Once you have confirmed the CEPH CSI driver has been deployed correctly, move onto the next stage.

[Next Page](./deploy-monitoring.md)

[Document Home](./readme.md)