![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### SODA Service

Create a new overlay `ral` directory for your site in the `apps/soda/overlays` directory.

Now create a `kustomization.yaml` as per the example,  updating the `clusterID` value.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - ceph-volume.yaml
patches:
  - path: soda-values.yaml
    target:
      kind: HelmRelease
      name: soda-release
```

Create a `soda-values.yaml` as per the example, updating the values to match your hosts.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: soda-release
  namespace: ska-src-soda
spec:
  values:
    ingress:
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
        - soda.ral.uksrc.org
      tls:
        - hosts:
          - soda.ral.uksrc.org
          secretName: soda-ingress-cert
```

Add the new `SODA`resources to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
```

Check your build using the `kustomize build` command. 

```sh
kustomize build apps/soda/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the SODA Service has been deployed.

Once you have confirmed the Soda Service has been deployed correctly, move onto the next stage.

[Next Page](./deploy-gatekeeper.md)
