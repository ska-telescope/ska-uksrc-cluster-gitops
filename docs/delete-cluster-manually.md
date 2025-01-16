# Delete a Cluster Manually

Due to resttictions at RAL the `manage teardown clustername` does not work correctly and it is necessary to delete the cluster manually.

**WARNING : You must ensure that you select the correct cluster if you are following these instructions.**

To delete a cluster manually do the following.

```sh
export OS_CLOUD=RAL

openstack server list | grep stage
| 2dcd799f-58c2-47a7-b384-b95ee19e7a2a | uksrc-ral-stage-control-plane-lkm4q         | ACTIVE | UKSRC-Network=192.168.66.109                 | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
| 68897e33-26d0-425b-989b-938630a0042e | uksrc-ral-stage-workers-mpmzh-bvqsq         | ACTIVE | UKSRC-Network=192.168.65.169                 | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
| 635f5944-7815-4cfb-80d9-19dc3452155b | uksrc-ral-stage-workers-mpmzh-qlrk4         | ACTIVE | UKSRC-Network=192.168.65.197                 | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
| 7bb8a93f-7703-4896-86aa-75c127eade9f | uksrc-ral-stage-workers-mpmzh-pvljx         | ACTIVE | UKSRC-Network=192.168.64.55                  | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
| fe1beed4-56bd-4b1a-88e7-d7836497cb83 | uksrc-ral-stage-control-plane-mpmlh         | ACTIVE | UKSRC-Network=192.168.67.74                  | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
| 9a80bbe8-98f3-4e5c-9e84-c4a294f93d4e | uksrc-ral-stage-control-plane-g7rwq         | ACTIVE | UKSRC-Network=192.168.64.95                  | capi-ubuntu-2204-kube-v1.30.6-2024-11-15 | l3.nano       |
```
Delete the control plane & workers using the servernames.

```sh
openstack server list | awk -F '|' '/-stage-/ {print $3}' | xargs -n 1 openstack server delete
```

List and delete the security ports.

```sh
openstack port list | grep stage

| 028ca136-8da7-4b69-88bb-c521cbe0ad92 | uksrc-ral-stage-workers-mpmzh-qlrk4-0                | fa:ca:ae:15:0f:41 | ip_address='192.168.65.197', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'  | ACTIVE |
| 0e520596-606d-4bf4-8d21-6a82765121da | uksrc-ral-stage-workers-mpmzh-bvqsq-0                | fa:ca:ae:04:36:b6 | ip_address='192.168.65.169', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'  | ACTIVE |
| 188a2ee0-062d-4668-a4a0-268557400eea | uksrc-ral-stage-control-plane-g7rwq-0                | fa:ca:ae:68:b1:a1 | ip_address='192.168.64.95', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'   | ACTIVE |
| 29afbeaa-341d-4efd-8776-60b86cf5c97a | uksrc-ral-stage-control-plane-mpmlh-0                | fa:ca:ae:63:bc:ac | ip_address='192.168.67.74', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'   | ACTIVE |
| 45d690a8-1a8b-4dfa-9caa-c913ada0b4b2 | uksrc-ral-stage-control-plane-lkm4q-0                | fa:ca:ae:dc:65:76 | ip_address='192.168.66.109', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'  | ACTIVE |
| f6f95204-c868-49a3-a4c2-a447ccada2b7 | uksrc-ral-stage-workers-mpmzh-pvljx-0                | fa:ca:ae:45:e9:3d | ip_address='192.168.64.55', subnet_id='84bbf44e-d646-4457-a839-9cd679ea9a14'   | ACTIVE |
```

Delete the ports by name using awk.

```sh
openstack port list | awk -F '|' '/-stage-/ {print $3}' | xargs -n 1 openstack -v port delete 
```

Delete the security groups.

```sh
openstack  security group list | awk -F '|' '/-stage-/ {print $3}'
 k8s-cluster-capi-self-uksrc-ral-stage-secgroup-worker
 k8s-cluster-capi-self-uksrc-ral-stage-secgroup-controlplane

openstack  security group list | awk -F '|' '/-stage-/ {print $3}' |  xargs -n 1 openstack security group delete
```

Delete the load-balancers.

```sh
openstack loadbalancer list | awk -F '|' '/-stage-/'
 k8s-clusterapi-cluster-capi-self-uksrc-ral-stage-kubeapi
 kube_service_uksrc-ral-stage_ingress-nginx_ingress-nginx-controller

openstack loadbalancer delete k8s-clusterapi-cluster-capi-self-uksrc-ral-stage-kubeapi --cascade
openstack loadbalancer delete kube_service_uksrc-ral-stage_ingress-nginx_ingress-nginx-controller --cascase
```