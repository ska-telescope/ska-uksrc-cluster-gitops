![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### GateKeeper Service

The SRCNet Operations Documentation for Gatekeeper defaults to deploying Gatekeeper as a LoadBalancer service. To reduce the number of floating IPs used for deployment, UKSRC has deployed the service as a Nodeport, we make this changes in our `overlays/ral/values.yaml` file.

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
    ingress-nginx:
      controller:
        replicaCount: 1
        service:
          type: NodePort
          loadBalancerIP: ""          
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
  - ../../components/cert-manager/issuers.yaml
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

Once you have confirmed the GateKeeper Service has been deployed correctly, move onto the next stage.

[Next Page](./deploy-jupyterhub.md)