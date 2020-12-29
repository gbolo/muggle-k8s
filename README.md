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
