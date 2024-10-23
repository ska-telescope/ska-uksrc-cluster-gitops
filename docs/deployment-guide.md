# UKSRC GitOps Deployment Guide

This guide demonstrates the methods and tools for deploying a [Kubernetes](https://kubernetes.io/)  Cluster on [OpenStack](https://www.openstack.org/)OpenStack and the required services and applications for SRCNET0.1 using a [GitOps](https://about.gitlab.com/topics/gitops/) workflow under the control of  [capi-helm-fluxcd-config](https://github.com/stackhpc/capi-helm-fluxcd-config) tooling created by StackHPC (c).

The specific tools used to accomplish this are [Cluster API](https://cluster-api.sigs.k8s.io/) for
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

## Cluster Bootstrapping Methodolgy

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

You will need to either request API Credentials or create them using your account.

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

* ukrsc-ral-cluster
* uksrc-ral-stage-cluster
* ukrsc-cam-cluster
* uksrc-cam-stage-cluster
* ukrsc-jbo-cluster
* uksrc-jbo-stage-cluster

Copy the example cluster to your new cluster.

```sh
cp -R clusters/example clusters/ukrsc-ral-cluster
```

