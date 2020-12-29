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
ansible-playbook local_python_requirements.yaml
```
