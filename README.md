# Deploying a Kubernetes cluster on OCI with RDMA support

This guide uses Nvidia's DeepOps project to deploy a Kubernetes cluster on existing nodes using Ansible. Detailed info about DeepOps can be found in its repository at https://github.com/NVIDIA/deepops.

This guide is NOT using OKE.

## IMPORTANT NOTES

### Images
The images you use for management and worker nodes matter. The Kubernetes cluster in this guide will use [Node Feature Discovery (NFD)](https://github.com/kubernetes-sigs/node-feature-discovery) for labeling the nodes automatically. If you use an image with OFED drivers installed for the management nodes, NFD will incorrectly label them as RDMA capable.

This guide assumes that you use the following images:

**Management:** Canonical-Ubuntu-20.04-2023.02.15-0 (OCI platform image - any Ubuntu 20.04 image without the OFED drivers should work)

**Worker:** Ubuntu-20-OFED-5.4-3.6.8.1-GPU-515-2023.01.10-0

If you want to use an image with OFED drivers for the management nodes or don't want to use NFD, you'll need to label your nodes and then use the correct `nodeSelector` when deploying the deamonset.

## Step-by-step instructions for deploying the cluster and enabling RDMA

1 - Deploy provisioning node, Kubernetes management and worker nodes.

Deploy the necessary nodes prior to following the steps in this guide. This guide is based on a single Kubernetes management node and 2 Kubernetes GPU worker nodes in a cluster network.

The easiest way to deploy and configure the nodes is using the HPC stack. v2.10.1 of the stack has an option to deploy a login node. You can find the copy of the stack in this repo (`oci-hpc-clusternetwork-2.10.1.zip`).

You can use the bastion as the provisioner node, and the login node as the management node.

As a minimum, you will need:

- 1 provisioning node (use the bastion node created by the HPC stack)
- 1 management node (use the login node created by the HPC stack)
- 1 GPU worker node with RDMA NICs


2 - SSH into the provisioning (bastion) node, and clone the DeepOps repository.

```sh
git clone https://github.com/NVIDIA/deepops.git
```

3 - Set up your provisioning node.

   This will install Ansible and other software on the provisioning machine which will be used to deploy all other software to the cluster. For more information on Ansible and why we use it, consult the [Ansible Guide](../deepops/ansible.md).

```sh
cd deepops
./scripts/setup.sh
```
4 -  Create and edit the Ansible inventory.

Ansible uses an inventory which outlines the servers in your cluster. The setup script from the previous step will copy an example inventory configuration to the `config` directory.

Edit the inventory file in `config/inventory`. An example inventory file with the Kubernetes related parts:

```sh
#
# Server Inventory File
#
# Uncomment and change the IP addresses in this file to match your environment
# Define per-group or per-host configuration in group_vars/*.yml

######
# ALL NODES
# NOTE: Use existing hostnames here, DeepOps will configure server hostnames to match these values
######
[all]
mgmt01     ansible_host=10.0.0.76
#mgmt02     ansible_host=10.0.0.2
#mgmt03     ansible_host=10.0.0.3
#login01    ansible_host=10.0.1.1
gpu01      ansible_host=10.0.0.201
gpu02      ansible_host=10.0.0.189

######
# KUBERNETES
######
[kube-master]
mgmt01
#mgmt02
#mgmt03

# Odd number of nodes required
[etcd]
mgmt01
#mgmt02
#mgmt03

# Also add mgmt/master nodes here if they will run non-control plane jobs
[kube-node]
gpu01
gpu02

[k8s-cluster:children]
kube-master
kube-node
```

5 - Change `gpu_operator_preinstalled_nvidia_software` to `false` in `deepops/config/group_vars/k8s-cluster.yml`. This option will use the existing GPU drivers on the node instead of reinstalling it.


```sh
# Install NVIDIA Driver and nvidia-docker on node (true), not as part of GPU Operator (driver container, nvidia-toolkit) (false)
gpu_operator_preinstalled_nvidia_software: false
```

7 -  Verify the configuration.

```sh
ansible all -m raw -a "hostname"
```

8 - Install Kubernetes using Ansible and Kubespray.

```sh
ansible-playbook -l k8s-cluster playbooks/k8s-cluster.yml
```

9 - Verify that the Kubernetes cluster is running.

```sh
kubectl get nodes

NAME     STATUS   ROLES                  AGE    VERSION
gpu01    Ready    <none>                 122m   v1.23.7
gpu02    Ready    <none>                 122m   v1.23.7
mgmt01   Ready    control-plane,master   132m   v1.23.7
```

10 - Deploy the config map for Mellanox RDMA Shared Device Plugin.

Save the following file as `configmap.yaml` and deploy it using `kubectl apply -f configmap.yaml`.

```yaml
apiVersion: v1
data:
  config.json: |
    {
      "periodicUpdateInterval": 300,
      "configList": [
        {
          "resourceName": "roce",
          "rdmaHcaMax": 16,
          "selectors": {
            "drivers": [
              "mlx5_core"
            ]
          }
        }
      ]
    }
kind: ConfigMap
metadata:
  name: rdma-devices
  namespace: kube-system
```

Check that the config map is there:

```sh
kubectl get configmap -n kube-system | grep rdma-devices

rdma-devices                         1      98m
```

11 - Deploy the Mellanox RDMA Shared Device Plugin daemonset.

Save the following file as `rdma-ds.yaml` and deploy it using `kubectl apply -f rdma-ds.yaml`.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rdma-shared-dp-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: rdma-shared-dp-ds
  template:
    metadata:
      labels:
        name: rdma-shared-dp-ds
    spec:
      hostNetwork: true
      priorityClassName: system-node-critical
      nodeSelector:
        feature.node.kubernetes.io/custom-rdma.capable: "true"
      containers:
      - image: mellanox/k8s-rdma-shared-dev-plugin
        name: k8s-rdma-shared-dp-ds
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/
          - name: config
            mountPath: /k8s-rdma-shared-dev-plugin
          - name: devs
            mountPath: /dev/
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/
        - name: config
          configMap:
            name: rdma-devices
            items:
            - key: config.json
              path: config.json
        - name: devs
          hostPath:
            path: /dev/
```

Check that the Daemonset is deployed correctly only on the nodes that has RDMA NICs (in our example, GPU nodes). You should see a pod running in each GPU node.

```sh
kubectl get pods -n kube-system -o wide | grep rdma-shared-dp-ds

rdma-shared-dp-ds-5sk7t                      1/1     Running   0              94m    10.0.0.189     gpu02    <none>           <none>
rdma-shared-dp-ds-lzjgc                      1/1     Running   0              94m    10.0.0.201     gpu01    <none>           <none>
```

12 - Now let's test running `ib_write_bw` between two pods. Save the following file as `rdma-test.yaml` and deploy it.

**NOTE:** When creating a pod spec, make sure that you're setting `hostNetwork: true` and request `rdma/roce: 1` as a resource. The following example spec has both options set.

```sh
kubectl apply -f rdma-test.yaml
```


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-test-pod-1
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  restartPolicy: OnFailure
  containers:
  - image: oguzpastirmaci/mofed-perftest:5.4-3.6.8.1-ubuntu20.04-amd64
    name: mofed-test-ctr
    securityContext:
      capabilities:
        add: [ "IPC_LOCK" ]
    resources:
      limits:
        rdma/roce: 1
    command:
    - sh
    - -c
    - |
      ls -l /dev/infiniband /sys/class/net
      sleep 1000000
---
apiVersion: v1
kind: Pod
metadata:
  name: rdma-test-pod-2
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  restartPolicy: OnFailure
  containers:
  - image: oguzpastirmaci/mofed-perftest:5.4-3.6.8.1-ubuntu20.04-amd64
    name: mofed-test-ctr
    securityContext:
      capabilities:
        add: [ "IPC_LOCK" ]
    resources:
      limits:
        rdma/roce: 1
    command:
    - sh
    - -c
    - |
      ls -l /dev/infiniband /sys/class/net
      sleep 1000000
```

13 - Wait until both pods are in Running state.

```sh
kubectl get pods -o wide

NAME              READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
rdma-test-pod-1   1/1     Running   0          49m   10.0.0.125   gpu02   <none>           <none>
rdma-test-pod-2   1/1     Running   0          49m   10.0.0.70    gpu01   <none>           <none>
```

14 - After the pods are running, open two terminals and exec into the pods.

```sh
kubectl exec -it rdma-test-pod-1 -- bash

kubectl exec -it rdma-test-pod-2 -- bash
```

15 - Confirm that you see the RDMA interfaces inside the pods.

```sh
ip ad | grep rdma

2: rdma4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 4220 qdisc mq state UP group default qlen 20000
    inet 192.168.4.201/16 brd 192.168.255.255 scope global rdma4
3: rdma5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 4220 qdisc mq state UP group default qlen 20000
    inet 192.168.5.201/16 brd 192.168.255.255 scope global rdma5
.....    
```

17 - Run a basic `ib_write_bw` test between pods. Make sure the RDMA interface you use matches the RDMA IP (192.168.x.x). You can get the device/interface matching with the `ibdev2netdev` command.

In `rdma-test-pod-1` run:

```
ib_write_bw -d mlx5_3 -a -F
```

In `rdma-test-pod-2` run:

```
ib_write_bw -F -d mlx5_3 <IP of mlx5_3 in rdma-test-pod-1> -D 10 --cpu_util --report_gbits
```

You should see a result similar to below:

```
************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_3
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 2
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x10107 PSN 0xb5554 RKey 0x04ad6e VAddr 0x007f8389beb000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:07:201
 remote address: LID 0000 QPN 0x10107 PSN 0x92e7fa RKey 0x047c3b VAddr 0x007fd6f366c000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:07:189
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      1101500          0.00               96.25  		   0.183585
---------------------------------------------------------------------------------------
```

### Adding Nodes

To add K8s nodes, modify the `config/inventory` file to include the new nodes under `[all]`. Then list the nodes as relevant under the `[kube-master]`, `[etcd]`, and `[kube-node]` sections. For example, if adding a new master node, list it under kube-master and etcd. A new worker node would go under kube-node.

Then run the Kubespray `scale.yml` playbook...

```bash
ansible-playbook -l k8s-cluster submodules/kubespray/scale.yml
```

More information on this topic may be found in the [Kubespray docs](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md#adding-nodes).

### Removing Nodes

Removing nodes can be performed with Kubespray's `remove-node.yml` playbook and supplying the node names as extra vars...

```bash
ansible-playbook submodules/kubespray/remove-node.yml --extra-vars "node=nodename0,nodename1"
```

This will drain `nodename0` & `nodename1`, stop Kubernetes services, delete certificates, and finally execute the kubectl command to delete the nodes.

More information on this topic may be found in the [Kubespray docs](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md#remove-nodes).

### Reset the Cluster

DeepOps is largely idempotent, but in some cases, it is helpful to completely reset a cluster. KubeSpray provides a best-effort attempt at this through a playbook. The script below is recommended to be run twice as some components may not completely uninstall due to time-outs/failed dependent conditions.

```bash
ansible-playbook submodules/kubespray/reset.yml
```

### Upgrading the Cluster

Refer to the [Kubespray Upgrade docs](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/upgrades.md) for instructions on how to upgrade the cluster.

### Monitoring

Deploy Prometheus and Grafana to monitor Kubernetes and cluster nodes:

```bash
./scripts/k8s/deploy_monitoring.sh
```

Available Flags:

```bash
-h      This message.
-p      Print monitoring URLs.
-d      Delete monitoring namespace and crds. Note, this may delete PVs storing prometheus metrics.
-x      Disable persistent data, this deploys Prometheus with no PV backing resulting in a loss of data across reboots.
-w      Wait and poll the grafana/prometheus/alertmanager URLs until they properly return.
delete  Legacy positional argument for delete. Same as -d flag.
```

The services can be reached from the following addresses:

- Grafana: http://\<kube-master\>:30200
- Prometheus: http://\<kube-master\>:30500
- Alertmanager: http://\<kube-master\>:30400

DeepOps deploys monitoring services using the [prometheus-operator](https://github.com/prometheus-operator/prometheus-operator) project.
For documentation on configuring and managing the monitoring services, please see the [prometheus-operator user guides](https://github.com/prometheus-operator/prometheus-operator/tree/master/Documentation/user-guides).

![](https://github.com/OguzPastirmaci/kubernetes-oci-azure-interconnect/blob/master/images/grafana.png)
