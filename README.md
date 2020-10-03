# Rancher on KVM

This document serves as a guide for setting up a [Rancher](https://rancher.com/) [Kubernetes](https://kubernetes.io/) cluster on a local network [KVM](https://www.linux-kvm.org/) cluster. Please note, I would likely explore using [Proxmox](https://www.proxmox.com/) to manage the cluster nodes if I were to implement a new local network Kubernetes cluster instead of using the Terraform KVM Libvirt provider.

Additional routing configuration, not covered in this document, can be implemented to expose services to the public internet. This will require configuration specific to your network architecture.

The steps outlined here cover my experience setting up a KVM cluster ontop of a bare metal server. The host base image was [Ubuntu 18.04 LTS (Bionic Beaver)](https://releases.ubuntu.com/18.04/).

> :warning: **Warning**
>
> This document has been compiled with notes taken during this setup. This actual document was writen months after the setup, so please follow the instructions at your own risk.
