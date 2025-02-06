![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### Cluster Issuer

The Cluster Issuer is a one fits all and does not need any modifications.

Add the `../../infra/issuers`resources to the `clusters/uksrc/ral-prod/kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
```


Check your build using the `kustomize build` command. 

```sh
kustomize build infra/monitoring/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the Cluster Issuer has been deployed.

Once you have confirmed the Cluster Issuer has been deployed correctly, move onto the next stage.

[Next Page](./deploy-csi-cephfs.md)

[Document Home](./readme.md)