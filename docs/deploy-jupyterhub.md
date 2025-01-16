![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
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

Create the `secrets/uksrc-ral-prod/jupyterhub-secret.yaml` secret in the secrets folder, as per the example using the Client ID & Client Secret from the SKA-IAM client.

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
  - ../../components/cert-manager/issuers.yaml
  - ../../infra/ceph-share/overlays/ral
  - ../../infra/monitoring/overlays/ral
  - ../../apps/soda/overlays/ral
  - ../../apps/gatekeeper/overlays/ral
  - ../../apps/jupyterhub/overlays/ral
  - ../../apps/canfar/overlays/ral
```


Check your build using the `kustomize build` command. 

```sh
kustomize build apps/jupyterhub/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that Jupyterhub has been deployed.

Once you have confirmed the Jupyterhub has been deployed correctly, move onto the next stage.

[Next Page](./deploy-canfar.md)