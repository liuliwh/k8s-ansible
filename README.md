# Description
This repository contains a sample ansible playbook (i.e. installation instructions) for managing Kubernetes masters and minions using ansible. 

The keepalived and haproxy is used to setup HA k8s cluster, and it's setup as the static pods. The keeaplived and haproxy are only installed on master-1 and master-2, the control plane is with VIP $ip_base.100:8443.

Currently, this cookbook is designed to run only once.
# Requirements
Vagrant and ansible setup is needed.

# Download and installation
```shell
vagrant up
```

# Configuration
The configuration file is located at ./config.yaml, you can modify the masters/minions/ip_base etc. 


