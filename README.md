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
