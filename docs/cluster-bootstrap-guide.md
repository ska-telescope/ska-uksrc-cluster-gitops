# UKSRC GitOps Deployment Guide

This guide demonstrates the methods and tools for deploying a [Kubernetes](https://kubernetes.io/)  Cluster on [OpenStack](https://www.openstack.org/)OpenStack and the required services and applications for SRCNET0.1 using a [GitOps](https://about.gitlab.com/topics/gitops/) workflow under the control of  [capi-helm-fluxcd-config](https://github.com/stackhpc/capi-helm-fluxcd-config) tooling created by StackHPC (c).

The specific tools used to accomplish this are [Kind](https://kind.sigs.k8s.io/) & [Cluster API](https://cluster-api.sigs.k8s.io/) for
the Kubernetes provisioning, [Flux CD](https://fluxcd.io/) for automation and
[sealed secrets](https://github.com/bitnami-labs/sealed-secrets) for managing secrets.

Cluster API is a set of
[Kubernetes operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) with
associated
[custom resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
that allow Kubernetes clusters to be provisioned and operated by creating instances of those
resources in a "management" Kubernetes cluster.

The clusters created using configurations derived from this repository are "self-managed". This
means that the Cluster API controllers and the resources defining the cluster are managed on the
cluster itself.

## Cluster Bootstrapping Methodology

The use of self-managed clusters creates a bootstrapping problem, i.e. how does the cluster get
created if it manages itself? The solution to this is to use a one-off bootstrapping process that
performs the following steps:

  1. Create an ephemeral Kubernetes cluster using [kind](https://kind.sigs.k8s.io/)
  2. Use the ephemeral cluster to create the Cluster API cluster
       * Install Flux on the ephemeral cluster
       * Use Flux to install the Cluster API controllers on the ephemeral cluster
       * Use Flux to create the Cluster API cluster using the ephemeral cluster
       * Wait for the Cluster API cluster to become ready
  3. Get the `kubeconfig` file for the Cluster API cluster
  4. Install Flux and the Cluster API controllers on the Cluster API cluster
  5. Move the Cluster API resources from the ephemeral cluster to the Cluster API cluster
       * Suspend reconciliation of the Cluster API resources on the ephemeral cluster
       * Copy the Cluster API resources to the Cluster API cluster
       * Resume reconciliation of the Cluster API resources on the Cluster API cluster

At this point, the cluster is self-managed. The remaining steps set the cluster up to be managed
using Flux going forward:

  6. Use Flux to install the sealed secrets controller
  7. Seal the cluster credentials using `kubeseal`
  8. Commit the sealed secret to git and push it
  9. Create a Flux source pointing at the git repository
       * If you are using git over SSH, this will generate an SSH keypair and prompt you
         to add the public key to the git repository as a deploy key
 10. Create a Flux kustomization pointing at the cluster configuration

At this point, the cluster is self-managed using Flux and the ephemeral cluster is deleted.

This repository includes [a Python script](./bin/manage) that performs these steps (see below).

## Prerequisites.
You will can deploy from a Linux Bastion host or locally from your Macbook or a Windows machine with WSL installed. Note while it is a nice to deploy from a local machine, we should be restricting access to our cluster to private networks and not exposing the K8s API to the internet so a Bastion is the prefered method.

### Cloud Credentials

You will need to either request OpenSTack API Credentials or create them using your OpenStack account.

    auth_url:
    application_credential_id: ""
    application_credential_secret: ""

### Tools

The following tools are required to execute the bootstrap script:

  * [Python 3](https://www.python.org/)
  * [git](https://git-scm.com/)
  * [Docker](https://docs.docker.com/)
  * [kind](https://kind.sigs.k8s.io/)
  * [kubectl](https://kubernetes.io/docs/reference/kubectl/)  
  * [kustomize](https://kustomize.io/) ** Kustomize is built in to kubectl **
  * [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal)
  * [flux CLI](https://fluxcd.io/flux/cmd/)

  Additionally you can install the following tools to for ease of use when navigating the Kubernetes clusters.

  * [kubectx](https://github.com/ahmetb/kubectx)
  * [kubeps1](https://github.com/jonmosco/kube-ps1)


### Clone the Git repository

Your Git config must be configured to use *ssh* and not *https*.

```sh
git clone git@gitlab.com:ska-telescope/src/deployments/uksrc/ska-src-uksrc-cluster-gitops.git
cd ska-src-uksrc-cluster-gitops
```

## Cluster Configuration

Before we can start the bootstrapping process we need to configure our cluster, an example is provided for ease of use.

```sh
├── clusters
│   ├── example
│   │   ├── configmap.yaml
│   │   ├── credentials.yaml
│   │   └── kustomization.yaml
```

Naming convention for our clusters are based on SRC Node naming as follows, using one of the 3 letter codes for Node : 

* ukrsc-ral
* uksrc-ral-stage
* ukrsc-cam
* uksrc-cam-stage
* ukrsc-jbo
* uksrc-jbo-stage

Copy the example cluster to your new cluster, note we add `-cluster` to the folder name.

```sh
cp -R clusters/example clusters/ukrsc-ral-cluster
```

You'll need to make changes to each of the example files and these may vary depending on the site you are deploying too. 

This documentation will focus on the RAL site specfific needs, the other sites will be covered later.

### configmap.yaml

```sh
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: uksrc-ral-cluster-config    <--- Use this naming convention
  namespace: capi-self
data:
  values.yaml: |
    # Must match the name of the (sealed) secret in credentials.yaml
    cloudCredentialsSecretName: uksrc-ral-cluster-credentials   <--- Use this naming convention

    kubernetesVersion: 1.28.7  <--- Select the release that is supported by the image
    machineImageId: 7288e8a2-976a-4b53-8b10-0be864c94af8  <--- Select the image ID
    clusterNetworking:
      externalNetworkId: 5283f642-8bd8-48b6-8608-fa3006ff4539  <--- Select the external network ID
      internalNetwork:
        networkFilter:
          id: e04ec697-e979-487c-929d-3c92893f15df <--- Select the internal image ID
          name: UKSRC-Network  <--- Select the Network name

    controlPlane:
      machineFlavor: l3.nano
      machineCount: 3

    nodeGroups:
      - name: workers
        machineFlavor: l3.nano
        autoscale: true
        machineCountMin: 3
        machineCountMax: 10    

    # Configuration for the cluster autoscaler
    autoscaler:
      # The image to use for the autoscaler component
      image:
        repository: registry.k8s.io/autoscaling/cluster-autoscaler
        pullPolicy: IfNotPresent
        tag: v1.28.6  <--- Select the autoscaler by major release, minor is not critical
      imagePullSecrets: []
      # Any extra args for the autoscaler
      extraArgs:
        # Make sure logs go to stderr
        logtostderr: true
        stderrthreshold: info
        # Output at a decent log level
        v: 4
        # Cordon nodes before terminating them so new pods are not scheduled there
        cordon-node-before-terminating: "true"
        # When scaling up, choose the node group that will result in the least idle CPU after
        expander: least-waste,random
        # Allow pods in kube-system to prevent a node from being deleted
        skip-nodes-with-system-pods: "true"
        # Allow pods with emptyDirs to be evicted
        skip-nodes-with-local-storage: "false"
        # Allow pods with custom controllers to be evicted
        skip-nodes-with-custom-controller-pods: "false"

    apiServer:
      # Indicates whether to deploy a load balancer for the API server
      enableLoadBalancer: true  <--- Enable the load-balancer
      # Indicates what loadbalancer provider to use. Default is amphora
      loadBalancerProvider:
      # Restrict loadbalancer access to select IPs
      # allowedCidrs
      #  - 192.168.0.0/16  # needed for cluster to init
      #  - 10.10.0.0/16  # IPv4 Internal Network
      #  - 123.123.123.123 # some other IPs
      # Indicates whether to associate a floating IP with the API server
      associateFloatingIP: true
      # The specific floating IP to associate with the API server
      # If not given, a new IP will be allocated if required
      floatingIP: 130.246.215.245 <--- Select an appropriate floating IP
      # The specific fixed IP to associate with the API server
      # If enableLoadBalancer is true, this will become the VIP of the load balancer
      # If enableLoadBalancer and associateFloatingIP are both false, this should be
      # the IP of a pre-allocated port to be used as the VIP
      fixedIP:
      # The port to use for the API server
      port: 6443

    addons:
      # Use the cilium CNI
      cni:
        type: cilium

      # Enable the monitoring stack
      monitoring:
        enabled: true

      # Disable NFD and the NVIDIA/Mellanox operators
      nodeFeatureDiscovery:
        enabled: false
      nvidiaGPUOperator:
        enabled: false
      mellanoxNetworkOperator:
        enabled: false

      # set availabilty zone as upstream uses nova by default
      openstack:
        csiCinder:
          defaultStorageClass:
            availabilityZone: ceph  <--- Select the correct AZ, this may vary by site

      ingress:
        enabled: true
        nginx:
          release:
            values:
              controller:
                service:
                  loadBalancerIP: "130.246.81.193"  <---Select an appropriate floating IP 

    kubeNetwork:
      # By default, use the private network range 10.0.0.0/12 for the cluster network
      # We split it into two equally-sized blocks for pods and services
      # This gives ~500,000 addresses in each block and distinguishes it from
      # internal net on 172.16.x.y
      pods:
        cidrBlocks:
          - 10.0.0.0/13
      services:
        cidrBlocks:
          - 10.8.0.0/13
      serviceDomain: cluster.local

    # Settings for registry mirrors
    registryMirrors: { docker.io: ["https://dockerhub.stfc.ac.uk"] }
```

### kustomization.yaml

```sh
resources:
  - ../../components/flux
  - ../../components/sealed-secrets
  - ../../components/cluster-api
  - ../../components/cluster
  - ../../infrastructure/ingress
  - configmap.yaml
  - credentials.yaml

  # After bootstrap add the apps and services here
  # - ../../components/nginx-ingress
  # - ../../infra/
  # - ../../sites/ral


# Patch the Helm release for the cluster to update the release name
#   This ensures we get nicely named resources in OpenStack
# We also add our configmap to the values sources for the release
patches:
  - target:
      kind: HelmRelease
      name: cluster
    patch: |-
      - op: replace
        path: /spec/releaseName
        value: uksrc-ral   <--- This value must match naming convention

      - op: add
        path: /spec/valuesFrom/-
        value:
          kind: ConfigMap
          name: uksrc-ral-cluster-config  <--- This must match the metadata name of the configmap
          valuesKey: values.yaml
```

### credentials.yaml

These credentials MUST NOT BE COMMITED to Gitlab, they will be sealed using [kubeseal]`

```sh
---
apiVersion: v1
kind: Secret
metadata:
  name: uksrc-ral-cluster-credentials  <--- Must match cloudCredentialsSecretName: in configmap
  namespace: capi-self
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true"
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: "<auth url>"
          application_credential_id: "<app cred id>"
          application_credential_secret: "<app cred secret>"
          project_id: "<project id>"
        region_name: "<region name>"
        interface: "public"
        identity_api_version: 3
        auth_type: "v3applicationcredential"
```

## Cluster Bootstrap

To start the bootstrap we need to create a Python Virtual Environment. For ease of use it is recommended to create a bash function.

Edit your `.bashrc` file and add the following.

```sh
function bootstrap() {
        cd ~/ska-uksrc-cluster-gitops/  <--- Must match the path of the cloned Git repo
        python3 -m venv ./.venv
        source .venv/bin/activate
        pip install -U pip
        pip install -r requirements.txt
}
```

To enter the *venv* run the function, note you only need the `source .bashrc` after the inital edit of the `.bashrc`.

```sh
source .bashrc
bootstrap
```

Alternatively you can run the commands manually.

```sh
        cd ~/ska-uksrc-cluster-gitops/  <--- Must match the path of the cloned Git repo
        python3 -m venv ./.venv
        source .venv/bin/activate
        pip install -U pip
        pip install -r requirements.txt
```


Now we're ready to bootstrap our cluster, run the `manage` command for the cluster that you created:

```sh
./bin/manage bootstrap uksrc-ral
```

The bootstrap process should take around 30 to 40 minutes.

To monitor the deployment, in a new Terminal window connect to the Kind cluster and monitor events.

```sh
kind export kubeconfig

kubectl get events -A --watch
```

Note that the output is very 'chatty', you're likely to see 'errors' or 'warnings' as the cluster waits for components to come online.

You can also check the `cluster-api` once it starts its deployment, use `kubens` to check for the `capi-self` namespace, if it exists switch to that namespace.

```sh
kubens capi-self
```

Check the cluster-api.

```sh
kubectl get cluster-api
```

Troubleshooting link here.