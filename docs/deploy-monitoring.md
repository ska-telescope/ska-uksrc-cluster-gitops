![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### Monitoring



Create a new overlay `ral` directory for your cluster in the `monitoring/overlays` directory.

Create a `kustomization.yaml` as per the example, updating the values to match your CName / hostname.

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - target:
      kind: Ingress
      name: grafana-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: grafana.ral.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value: 
          - grafana.ral.uksrc.org     
  - target:
      kind: Ingress
      name: prometheus-ingress
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: prometheus.ral.uksrc.org
      - op: replace
        path: /spec/tls/0/hosts
        value: 
          - prometheus.ral.uksrc.org
```

Add the new `monitoring`resources to the `clusters/uksrc/ral-prod/kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
  - ../../infra/ceph-share/overlays/ral
  - ../../infra/monitoring/overlays/ral
```


Check your build using the `kustomize build` command. 

```sh
kustomize build infra/monitoring/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that the monitoring has been deployed.

Once you have confirmed the Monitoring has been deployed correctly, move onto the next stage.

[Next Page](./deploy-soda.md)

[Document Home](./readme.md)
