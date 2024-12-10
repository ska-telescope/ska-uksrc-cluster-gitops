# UKSRC Apps and Services Deployment

This guide will demonstrate how to deploy each of the applications & services required for srcNet0.1 and subsequent versions.

Once you have deployed the Kubernetes cluster and are satisfied it is ready, you will need to make the following changes to deploy the necessary apps and services for srcNet0.1

It is worth understanding the structure of the GitOps repository before continuing.

```sh
├── apps
│   ├── jupyterhub
│   │   ├── base
│   │   └── overlays
│   │       ├── cam
│   │       ├── ral
│   │       └── ral-stage
│   └── soda
│       └── templates
├── bin
├── clusters
│   ├── example
│   ├── uksrc-ral-cluster
│   └── uksrc-ral-stage-cluster
├── components
│   ├── cert-manager
│   ├── cluster
│   ├── cluster-api
│   ├── cluster-api-addon-provider
│   ├── cluster-api-janitor-openstack
│   ├── cluster-api-operator
│   ├── flux
│   ├── nginx-ingress
│   └── sealed-secrets
├── infra
│   ├── base
│   └── overlays
│       ├── ral
│       └── ral-stage
└── secrets
    └── uksrc-ral-cluster
```

Cluster configuration is managed in the clusters directory and the components directory contains all the components for managing the cluster. We therefore do not add any of our apps or services to these diretories.

Applications are managed under the apps directory and services are managed under the infra directory. If you are adding new applications or services to the srcNet0.1 deployment they should be placed in the applicable directory.

Each application or service may also have a base and overlays folder. The base directory contains the base application and the overlays folders will contain site or cluster specific values or secrets for that site.

Each Cluster will require Kustomization to deploy the applications and services and some applications and services will un turn require Kustomization for site specific values and secrets.

All secrets are sealed using kubeseal which uses the cluster certificate to encrypt & decrypt the secret, allowing it to be stored in the Gitlab repository. Please note the `secrets` folder will not be part of this directoy structure and must be added manually, this is because it is purposefully ignore using `.gitignore` to prevent unencrypted secrets being push to the repository. This will be explained in further detail later in the document.

## Preparing for the Deployment

In this example we will be deploying the following Apps and Services.
    * All Infra Services
    * SODA
    * Jupyterhub

Create a branch as per the SKAO guidelines, this example uses a ficticious ticket DAAC-100.

```sh
git checkout main
git pull
git checkout -b daac-100-ral-apps-svcs-deployment
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
  - ../../infra/overlays/ral
  - ../../apps/soda
  - ../../apps/jupyterhub/overlays/ral


## Truncated for brevity ##
```

Note we point our resources to `overlays` for the apps and services that need site or cluster specific values, eg Jupyterhub & Infra but SODA does not require any site or cluster specific changes.


