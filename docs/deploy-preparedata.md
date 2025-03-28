![Local Image](images/SKAO_ukSRC_logo_nostrapline_colour_rgb.png)
### Deploy PrepareData

DISCLAIMER : We're not using Helm at the moment for this at the moment, we need to switch the deployment to Helm but for now this method will do.

Create a new overlay `site` directory, e.g. `ral/ for your site in the `apps/prepare/overlays` directory.

The following values are unique to your deployment, `shareAccessID`, `shareID` , `clusterID` , `rootpath` and should be updated accordingly in the `kustomization.yaml` overlays below.

The value for `clusterID` can be obtained from the [Ceph CSI Deployment](docs/deploy-csi-cephfs.md) and `rootPath` can be obtained from your RSE/xrootd administrator.

For the `ShareID`value run the following command, in this example we're deploying to production so will use the `cavern` ID.

  ```sh
  $ openstack share list

+--------------------------------------+--------------+------+-------------+-----------+-----------+-----------------+------+-------------------+
| ID                                   | Name         | Size | Share Proto | Status    | Is Public | Share Type Name | Host | Availability Zone |
+--------------------------------------+--------------+------+-------------+-----------+-----------+-----------------+------+-------------------+
| d8395514-8896-4817-9193-4e550e9ac483 | cavern       |  100 | CEPHFS      | available | False     | cephfs          |      | nova              |
| 94ca176b-a7c2-4750-8051-5c1fa357c005 | cavern-stage |  100 | CEPHFS      | available | False     | cephfs          |      | nova              |
| b73f8f41-8287-4e8b-b057-4bebcde73701 | rse-share    |   10 | CEPHFS      | available | False     | cephfs          |      | nova              |
+--------------------------------------+--------------+------+-------------+-----------+-----------+-----------------+------+-------------------+
```

For the `shareAccessID` value run the following command using the `cavern` ID.

```sh
$ openstack share access list d8395514-8896-4817-9193-4e550e9ac483

+--------------------------------------+-------------+-----------+--------------+--------+-------------+------------------+----------------------------+
| ID                                   | Access Type | Access To | Access Level | State  | Access Key  | Created At       | Updated At                 |
+--------------------------------------+-------------+-----------+--------------+--------+-------------+------------------+----------------------------+
| e58edeab-8b81-41ae-8946-3da3e471cdf9 | cephx       | cavern    | rw           | active | xxxxxxxxxxx | 2024-11-21T14:59 | 2024-11-21T14:59:49.943195 |
+--------------------------------------+-------------+-----------+--------------+--------+-------------+------------------+----------------------------+
```


Create a `kustomization.yaml` as per the example, in the `apps/prepare/overlays/site` directory, updating the `clusterID` , `rootpath`,  `shareAccessID`, `shareID` values accordingly.

```yaml
  - target:
      kind: PersistentVolume
      name: preparedata-rse-pv
    patch: |-
      - op: replace
        path: /spec/csi/volumeAttributes/clusterID
        value: 2660d5ed-e7ef-4950-a051-c84a0b4e78b0
      - op: replace
        path: /spec/csi/volumeAttributes/rootPath
        value: /ska/testbed
  - target:
      kind: PersistentVolume
      name: preparedata-userspace-pv
    patch: |-
      - op: replace
        path: /spec/csi/volumeAttributes/shareAccessID
        value: e58edeab-8b81-41ae-8946-3da3e471cdf9
      - op: replace
        path: /spec/csi/volumeAttributes/shareID
        value: d8395514-8896-4817-9193-4e550e9ac483
```

Now create the `rmq-values.yaml` to update the RabbitMQ Helm values.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: rabbitmq
  namespace: rabbitmq
spec:
  values:
    auth:
      username: user
      password: "user"
```

Edit the `clusters/uksrc-ral-prod/kustomization.yaml` for the cluster.

Add the new `preparedata`resource to the `kustomization.yaml` after the comment `# After bootstrap add the apps and services here`.

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
  - ../../apps/soda/overlays/ral
  - ../../apps/gatekeeper/overlays/ral
  - ../../apps/jupyterhub/overlays/ral
  - ../../apps/canfar/overlays/ral
  - ../../apps/preparedata/overlays/ral  
```


Check your build using the `kustomize build` command. 

```sh
kustomize build apps/jupyterhub/overlays/ral | less
```

Commit & push your changes to Gitlab. You should be able to see that prepareData has been deployed.

[Document Home](./readme.md)