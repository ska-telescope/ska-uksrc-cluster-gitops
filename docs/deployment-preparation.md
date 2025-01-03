![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
# UKSRC Apps and Services Deployment

These guides will demonstrate how to deploy each of the applications & services required for srcNet0.1 and subsequent versions.

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

## IAM Clients & Secrets

As part of the prepation for deployment you or one of your team must create the IAM Clients for each App or Service that requires them. Follow the instructions at [SRCNet Operations Documentation](https://ska-telescope.gitlab.io/src/kb/ska-src-docs-operator/index.html#) for each Application or Service that requires an IAM Client, currently that list is as follows.

- GateKeeper
- Jupyterhub
- CANFAR Science Portal
- CANFAR Science Gateway

During deployment you should be logged into [SKA-IAM](https://ska-iam.stfc.ac.uk/dashboard#!/home) to be retrieve the necessary Client ID & Client Secrets created to enable you to create the Sealed K8s Secrets.

If you did not create create the IAM credentials, you must ensure that you have access to them through a SECURE method, such as using an applications like 1Password or similar Vault based tools, DO NOT SHARE them via email or instant messaging.

[Next page](./deploy-csi-cephfs.md)
