# UKSRC Apps and Services Deployment

This guide will demonstrate how to deploy each of the applications & services required for srcNet0.1 and subsequent versions.

Once you have deployed the Kubernetes cluster and are satisfied it is ready, you will need to make the following changes to deploy the necessary apps and services for srcNet0.1

Cluster configuration is managed in the clusters directory and the components directory contains all the components for managing the cluster. We therefore do not add any of our apps or services to these diretories.

Applications are managed under the apps directory and services are managed under the infra directory. If you are adding new applications or services to the srcNet0.1 deployment they should be placed in the applicable directory.

Each application or service may also have a base and overlays folder. The base directory contains the base application and the overlays folders will contain site / cluster specific values or secrets for that site.

Each Cluster will require Kustomization to deploy the applications and services and some applications and services will in turn require Kustomization for site specific values and secrets.

All secrets are sealed using kubeseal which uses the cluster certificate to encrypt & decrypt the secret, allowing it to be stored in the Gitlab repository. Please note the `secrets` folder will not be part of this directoy structure and must be added manually, this is because it is purposefully ignore using `.gitignore` to prevent unencrypted secrets being push to the repository. This will be explained in further detail later in the document.

## Preparing for the Deployment

In this example we will be deploying the following Apps and Services.
    * All Infra Services
        * Ceph CSI Driver
        * Monitoring
    * Apps
        * GateKeeper
        * SODA
        * Jupyterhub
        * CANFAR

Create a branch as per the SKAO guidelines, this example uses a ficticious ticket DAAC-100.

```sh
git checkout main
git pull
git checkout -b daac-100-ral-apps-svcs-deployment
```

For the purposes of this deployment we've deployed the cluster at RAL, so references and file structure will reflect that. Make the necessary changes for your site / cluster. 

### Configure the ceph-csi-cephfs Driver

Create a new overlay directory for your site in the `infra/ceph-share` directory.

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

The values fare unique to your CephFS deployment, `fsid` , `mon_host` , `mon_initial_members` , `public_network` , `clusterID` , `monitors` and should be updated accordingly. 
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

Create the `csi-cephfs-secret.yaml` sealed secret.

In order to keep our unsealed secrets secure and not have them accidentally commited to the Git repo, create the secret file under the secrets directory, this is protected by `.gitignore` rules.

Use the structure as below.

```sh
secrets/
│
└── uksrc-ral-cluster
    ├── cloud-credentials.yaml
    ├── csi-cephfs-secret.yaml
    ├── gatekeeper-secret.yaml
    └── jupyterhub-secret-config.yaml
```

Create your `csi-cephfs-secret.yaml` as per the example.

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
kubeseal --kubeconfig clusters/uksrc-ral-cluster/kubeconfig --format yaml --controller-name sealed-secrets --controller-namespace sealed-secrets-system --secret-file secrets/uksrc-ral-cluster/csi-cephfs-secret.yaml --sealed-secret-file infra/ceph-share/overlays/ral/csi-cephfs-secret.yaml
```





### Jupyterhub Configuration

For Jupyterhub we need to make changes to the secret and 



In your editor, open the kustomization.yaml for the cluster.

```sh
clusters/
|
└── uksrc-ral-cluster
    ├── configmap.yaml
    ├── credentials.yaml
    └── kustomization.yaml
```

Add the new resources to the kustomization.yaml after the comment `# After bootstrap add the apps and services here`.

```sh
resources:
  - ../../components/flux
  - ../../components/sealed-secrets
  - ../../components/cluster-api
  - ../../components/cluster
  - configmap.yaml
  - credentials.yaml

  # After bootstrap add the apps and services here
  - ../../components/cert-manager/issuers.yaml
  - ../../infra/ceph-share/overlays/ral
  - ../../infra/monitoring/overlays/ral
  - ../../apps/soda/overlays/ral
  - ../../apps/gatekeeper/overlays/ral
  - ../../apps/jupyterhub/overlays/ral
  - ../../apps/canfar/overlays/ral
```

Note we point our resources to `overlays` for the apps and services that need site or cluster specific values.

Configuring 


