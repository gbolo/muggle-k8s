# Muggle K8s
A deployment for Kubernetes without the use of any "magic" :)

## Purpose
This repository is mainly for educational purposes. The goal being to fully
understand how to deploy each component of the kubernetes infrastructure
without the use of any magical tools like `kubeadm`.

From a security perspective, I also want to explore the use of my own PKI
as well as to harden the system and tightly control network and ACLs.

Finally, I would like to understand how each component works and behaves under
various conditions. Only components that I care about will be supported in
this deployment, so options for extended components/plugins will be limited.

## Getting Started

### Ansible Requirements
**Ensure that you have the required python libraries installed before attempting
to run the ansible playbooks in this repo.**
To confirm your ansible system meets these requirements you can run:
```
$ ansible-playbook local_python_requirements.yaml
```

## Deploying Kubernetes

### Generate PKI
In order to begin our deployment, we will need a PKI to secure all our
components. The PKI will be used for ALL of our kubernetes components as well
as our etcd backend. At the moment, the role will generate three CAs
(two for etcd and one for kubernetes). The role will dynamically generate
the required PKI for your specific inventory objects. So if you have less hosts
then less TLS certs and keys will be generated. This role is also safe to run
multiple times, as it won't delete existing PKI. To generate the pki run the
following playbook:

```
$ ansible-playbook -i env/gbolo1 playbooks_k8s/01-generate-pki.yaml
```

When it successfully completes, you should be able to see the artifacts:
```
$ tree artifacts/pki
# output is truncated

artifacts/pki
├── etcd
│   ├── etcd-peer-ca.crt
│   ├── etcd-peer-ca.csr
│   ├── etcd-peer-ca.key
│   ├── etcd-server-ca.crt
│   ├── etcd-server-ca.csr
│   ├── etcd-server-ca.key
...
├── k8s
│   ├── apiserver-kubelet-client.crt
│   ├── apiserver-kubelet-client.csr
│   ├── apiserver-kubelet-client.key
│   ├── k8s-ca.crt
│   ├── k8s-ca.csr
│   ├── k8s-ca.key
...
└── users
    ├── admin.crt
    ├── admin.csr
    └── admin.key
```

### Deploy the ETCD cluster
Kubernetes requires an etcd cluster as a storage backend. To deploy one run the
following playbook:

```
$ ansible-playbook -i env/gbolo1 playbooks_etcd/deploy.yml
```

When it successfully completes, ensure that it's operational:
```
# ssh to one of the etcd nodes
$ ssh skdeploy@<etcd_node_in_your_inventory>

# check that all members show up
$ sudo /opt/etcd/etcdctl.sh -w table member list
+------------------+---------+--------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |     NAME     |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+--------------+---------------------------+---------------------------+------------+
| 16f286079fdb2bc5 | started | <etcd1_name> | https://<etcd1_fqdn>:2380 | https://<etcd1_fqdn>:2379 |      false |
| 4dfe47aab47228ab | started | <etcd2_name> | https://<etcd2_fqdn>:2380 | https://<etcd2_fqdn>:2379 |      false |
| c4770f7ce1f71606 | started | <etcd3_name> | https://<etcd3_fqdn>:2380 | https://<etcd3_fqdn>:2379 |      false |
+------------------+---------+--------------+---------------------------+---------------------------+------------+

# check that all endpoints are healthy
$ sudo /opt/etcd/etcdctl.sh -w table endpoint health
+-------------------+--------+-------------+-------+
|     ENDPOINT      | HEALTH |    TOOK     | ERROR |
+-------------------+--------+-------------+-------+
| <etcd1_fqdn>:2379 |   true |  73.07161ms |       |
| <etcd2_fqdn>:2379 |   true | 77.715403ms |       |
| <etcd3_fqdn>:2379 |   true | 92.452173ms |       |
+-------------------+--------+-------------+-------+

# check that we have a leader
$ sudo /opt/etcd/etcdctl.sh -w table endpoint status
```

### Generate the kubeconfig files
We are going to need various kubeconfig files (some for systems and some for our
local use). The role will generate these files dynamically based on your
inventory. To generate these files run the following playbook:

```
$ ansible-playbook -i env/gbolo1 playbooks_k8s/02-generate-kubeconfigs.yaml
```

When it successfully completes, you should be able to see the artifacts:
```
$ tree artifacts/kubeconfig/
artifacts/kubeconfig/
├── admin.kubeconfig
├── <wroker_node1_fqdn>.kubeconfig
├── <wroker_node2_fqdn>.kubeconfig
├── kube-controller-manager.kubeconfig
├── kube-proxy.kubeconfig
└── kube-scheduler.kubeconfig
```

### Download Kubernetes binaries
Our kubernetes infrastructure will be deployed as binaries running on the system
natively through the systemd init system. The role will download all k8s
binaries for the specified `k8s_version` release. The role is smart enough to
detect if you have already downloaded the correct version, so it is safe to run
multiple times without having to worry about wasted bandwidth. To download these
binaries run the following playbook:

```
$ ansible-playbook -i env/gbolo1 playbooks_k8s/03-download-k8s-bins.yaml
```

When it successfully completes, you should be able to see the artifacts:
```
$ tree artifacts/bin/
artifacts/bin/
├── kube-apiserver
├── kube-controller-manager
├── kube-proxy
├── kube-scheduler
├── kubectl
└── kubelet
```

## Deploy the Kubernetes masters
The control plane nodes (masters) are the essential piece to our kubernetes
infrastructure. They provide the main API which all components talk to (via
`kube-apiserver`). Our role will configure and deploy the following master
components to the servers in the group `master`.:

- `kube-apiserver` The Kubernetes API REST service, which handles requests and
data for all Kubernetes objects and provide shared state for all the other
Kubernetes components.
- `kube-controller-manager` Monitors the cluster desired state through the
Kubernetes API server and makes the necessary changes to the current state to
reach the desired state.
- `kube-scheduler` Responsible for scheduling cluster workloads based on various
configurations, metrics, resource requirements and workload-specific requirements.

To deploy the masters, run the following playbook:
```
$ ansible-playbook -i env/gbolo1/ playbooks_k8s/04-deploy-masters.yaml
```

When it successfully completes, you should be able to use `kubectl` to interact
with your newly deployed kubernetes control plane:
```
# you may use your own kubectl binary if your not on linux
$ ./artifacts/bin/kubectl --kubeconfig artifacts/kubeconfig/admin.kubeconfig get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

## Deploy the Kubernetes worker nodes
The worker nodes are responsible for running all the work loads.
Our role will configure and deploy the following worker
components to the servers in the group `worker`.:

- `kubelet` Acts as the "agent node". Watches the Kubernetes API (master nodes)
for any changes to apply.
- `kube-proxy` Responsible for getting traffic to the correct pods.
Does things like load balancing, SNAT, manage iptables, exc.

To deploy the worker nodes, run the following playbook:
```
$ ansible-playbook -i env/gbolo1/ playbooks_k8s/05-deploy-workers.yaml
```

When it successfully completes, you should be able to use `kubectl` to verify
that the nodes have been admitted to the cluster:
```
# NOTE: the nodes won't be ready yet because we have not yet setup the network plugin cni
$ ./artifacts/bin/kubectl --kubeconfig artifacts/kubeconfig/admin.kubeconfig get nodes
NAME                        STATUS     ROLES    AGE   VERSION
<worker_node_1>             NotReady   <none>   11s   v1.19.6
<worker_node_2>             NotReady   <none>   11s   v1.19.6

# describe the nodes to see that they are not ready because of the cni plugin
$ ./artifacts/bin/kubectl --kubeconfig artifacts/kubeconfig/admin.kubeconfig describe nodes
...
runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
...
```

## Deploy the Kubernetes cluster network
There are [many options](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
for cluster networking available in Kubernetes. I have selected to use the
[CNI](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
network plugin with a third-party provider [weave net](https://www.weave.works/oss/net/).

Weave net is being deployed as a CNI plugin. It provides many advanced features,
including support for [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

### Deploy the Network Plugin CNI Requirements
First we install some CNI requirements so that our plugin can work properly.
Basically it just makes a bunch of binaries available to the CNI plugin:

```
$ ansible-playbook -i env/gbolo1/ playbooks_k8s/06-prepare-network-plugin-cni.yaml
```

### Deploy CNI plugin Weave Net
We can now proceed to deploying weave-net as a CNI plugin. This is as simple as
applying a resource to the Kubernetes API. Run the following playbook to do just
that:
```
$ ansible-playbook -i env/gbolo1/ playbooks_k8s/07-deploy-cni-plugin-weave-net.yaml
```

When it successfully completes, you should be able to use `kubectl` to verify
that the nodes are not marked as Ready:

```
$ ./artifacts/bin/kubectl --kubeconfig artifacts/kubeconfig/admin.kubeconfig get nodes
NAME                        STATUS     ROLES    AGE   VERSION
<worker_node_1>             Ready      <none>   10h   v1.19.6
<worker_node_2>             Ready      <none>   10h   v1.19.6

# you can also verify if the weave-net pods are running if your nodes are not ready
./artifacts/bin/kubectl --kubeconfig artifacts/kubeconfig/admin.kubeconfig get pods -n kube-system -l name=weave-net
NAME              READY   STATUS    RESTARTS   AGE
weave-net-c7h59   2/2     Running   0          5m
weave-net-l9mbb   2/2     Running   0          5m
```
