![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)

### K8s Cluster Bootstrp & Configuration

Before we can start the bootstrapping process we need to configure our cluster, an example is provided for ease of use.

```sh
├── clusters
│   ├── example
│   │   ├── configmap.yaml
│   │   ├── credentials.yaml
│   │   └── kustomization.yaml
```

Using the naming convention for our clusters, use one of the following for your cluster. 

* ukrsc-ral-prod
* uksrc-ral-stage
* ukrsc-cam-prod
* uksrc-cam-stage
* ukrsc-jbo-prod
* uksrc-jbo-stage

Copy the example cluster to your new cluster.

```sh
cp -R clusters/example clusters/ukrsc-ral-prod
```

You'll need to make changes to each of the example files and these may vary depending on the site you are deploying too. 

This documentation will focus on the RAL site specfific needs, make changes as necessaary for your node.

### configmap.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: uksrc-ral-cluster-config   # <--- Use this naming convention
  namespace: capi-self
data:
  values.yaml: |
    # Must match the name of the (sealed) secret in credentials.yaml
    cloudCredentialsSecretName: uksrc-ral-cluster-credentials  # <--- Use this naming convention

    kubernetesVersion: 1.28.7 # <--- Select the release that is supported by the image
    machineImageId: 7288e8a2-976a-4b53-8b10-0be864c94af8 # <--- Select the image ID
    clusterNetworking:
      externalNetworkId: 5283f642-8bd8-48b6-8608-fa3006ff4539 # <--- Select the external network ID
      internalNetwork:
        networkFilter:
          id: e04ec697-e979-487c-929d-3c92893f15df# <--- Select the internal image ID
          name: UKSRC-Network # <--- Select the Network name

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
        tag: v1.28.6 # <--- Select the autoscaler by major release, minor is not critical
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
      enableLoadBalancer: true # <--- Enable the load-balancer
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
      floatingIP: 130.246.215.245# <--- Select an appropriate floating IP
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
            availabilityZone: ceph # <--- Select the correct AZ, this may vary by site

      ingress:
        enabled: true
        nginx:
          release:
            values:
              controller:
                service:
                  loadBalancerIP: "130.246.81.193"  # <--- Select an appropriate floating IP 

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
    registryMirrors: { docker.io: ["https://dockerhub.stfc.ac.uk"] } # <--- Update to match your registry mirror.
```

### kustomization.yaml

```yaml
resources:
  - ../../components/flux
  - ../../components/sealed-secrets
  - ../../components/cluster-api
  - ../../components/cluster
  - ../../infrastructure/ingress
  - configmap.yaml
  - credentials.yaml

  # After bootstrap add the apps and services here



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
        value: uksrc-ral  # <--- This value must match naming convention

      - op: add
        path: /spec/valuesFrom/-
        value:
          kind: ConfigMap
          name: uksrc-ral-cluster-config # <--- This must match the metadata name of the configmap
          valuesKey: values.yaml
```

### credentials.yaml

These credentials MUST NOT BE COMMITED to Gitlab, they will be sealed using [kubeseal]`

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: uksrc-ral-cluster-credentials # <--- Must match cloudCredentialsSecretName: in configmap
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
        cd ~/ska-uksrc-cluster-gitops/ # <--- Must match the path of the cloned Git repo
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
        cd ~/ska-uksrc-cluster-gitops/ # <--- Must match the path of the cloned Git repo
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

[Next Page](./deployment-preparation.md)

Troubleshooting link here.