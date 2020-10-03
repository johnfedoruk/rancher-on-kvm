# Rancher on KVM

This document serves as a guide for setting up a [Rancher](https://rancher.com/) [Kubernetes](https://kubernetes.io/) cluster on a local network [KVM](https://www.linux-kvm.org/) cluster. Please note, I would likely explore using [Proxmox](https://www.proxmox.com/) to manage the cluster nodes if I were to implement a new local network Kubernetes cluster instead of using the Terraform KVM Libvirt provider.

Additional routing configuration, not covered in this document, can be implemented to expose services to the public internet. This will require configuration specific to your network architecture.

The steps outlined here cover my experience setting up a KVM cluster ontop of a bare metal server. The host base image was [Ubuntu 18.04 LTS (Bionic Beaver)](https://releases.ubuntu.com/18.04/).

> :warning: **Warning**
>
> This document has been compiled with notes taken during this setup. This actual document was writen months after the setup, so please follow the instructions at your own risk.

## Host Setup

### Download and Install Terraform 0.12

Download the ZIP file. It must be unzipped, and the executable `terraform` must be put on your path.

```shell
$ wget https://releases.hashicorp.com/terraform/0.12.24/terraform_0.12.24_linux_amd64.zip
$ unzip terraform_0.12.24_linux_amd64.zip
$ sudo mv ./terraform /usr/bin/terraform
```

Congratulations! You now have Terraform installed!

### Installing KVM

Begin by installing the dependency packages listed below.

```bash
sudo apt-get install qemu qemu-kvm libvirt-clients libvirt-dev libvirt-daemon-system bridge-utils qemu-guest-agent virt-top virt-viewer
sudo apt-get install virt-manager
```

You may experience issues related to SELinux. This has been [referenced here](https://github.com/dmacvicar/terraform-provider-libvirt/commit/22f096d9) as a known issue. The solution is to set the security_drivers to none in `/etc/libvirt/qemu.conf`.

```bash
sudo sed -i s/#security_driver\ =\ \"selinux\"/security_driver\ =\ \"none\"/g /etc/libvirt/qemu.conf
sudo sed -i s/#user/user/g /etc/libvirt/qemu.conf
sudo sed -i s/#group/group/g /etc/libvirt/qemu.conf
sudo systemctl restart libvirtd.service
sudo systemctl status libvirtd.service
```

### Creating Bridge Network

You will need to add a br0 interface. In this case, it bridges eno1. Please note that your physical nics may be named differently. After creating the bridge, restart your computer.

```bash
sudo brctl addbr br0
sudo brctl addif br0 eno1
sudo reboot
```

### Installing Terraform Libvirt Provider

Fist, you will need to install the system dependencies for this provider.

```bash
sudo apt-get install genisoimage
sudo apt install golang-go
```

The [original library repository](https://github.com/dmacvicar/terraform-provider-libvirt.git) does not support [cloud_init](https://cloudinit.readthedocs.io/en/latest/) running on forks of [CoreOS](https://coreos.com/). Unfortunatly, [RancherOS](https://rancher.com/docs/os/v1.x/en/) is a fork of CoreOS, so if you are interested in experimenting with a RancherOS image for the nodes, follow [Terraform Libvirt Provider for RancherOS](
#Terraform-Libvirt-Provider-for-RancherOS), otherwise follow [Terraform Libvirt Provider for Ubuntu Cloud](#Terraform-Libvirt-Provider-for-Ubuntu-Cloud) (recommended).

#### Terraform Libvirt Provider for Ubuntu Cloud

```bash
source <( go env )
mkdir -p $GOPATH/src/github.com/dmacvicar; cd $GOPATH/src/github.com/dmacvicar
git clone https://github.com/dmacvicar/terraform-provider-libvirt.git
cd $GOPATH/src/github.com/dmacvicar/terraform-provider-libvirt
make install
ls $GOPATH/bin/terraform-provider-libvirt >/dev/null 2>/dev/null && echo "Install ok" || echo "Install failed"
mkdir -p "$HOME/.terraform.d/plugins"
cp "$GOPATH/bin/terraform-provider-libvirt" "$HOME/.terraform.d/plugins"
```

The [dmacvicar repository](https://github.com/dmacvicar/terraform-provider-libvirt.git) is the only community libvirt provider library listed on [Terraform's website](https://www.terraform.io/docs/providers/type/community-index.html). Terraform does not have an official library for this provider.

#### Terraform Libvirt Provider for RancherOS

```bash
$ source <( go env )
$ mkdir -p $GOPATH/src/github.com/johnfedoruk; cd $GOPATH/src/github.com/johnfedoruk
$ git clone https://github.com/johnfedoruk/terraform-provider-libvirt.git
$ cd $GOPATH/src/github.com/johnfedoruk/terraform-provider-libvirt
$ git checkout feature/ci-datasourcetypes
$ git remote add upstream https://github.com/dmacvicar/terraform-provider-libvirt.git
$ git pull upstream master
$ git merge upstream/master
$ grep "dmacvicar" -rl . | xargs sed -i s/dmacvicar/johnfedoruk/g
$ make install
$ ls $GOPATH/bin/terraform-provider-libvirt >/dev/null 2>/dev/null && echo "Install ok" || echo "Install failed"
$ mkdir -p "$HOME/.terraform.d/plugins"
$ cp "$GOPATH/bin/terraform-provider-libvirt" "$HOME/.terraform.d/plugins"
```

For more information on the issues with running RancherOS on Terraform, please see:
- https://github.com/dmacvicar/terraform-provider-libvirt/pull/476
- https://github.com/rancher/os/issues/2559

## Setting Up the VMs

At this point, we will want to install our target image. We are moving forward with Ubuntu as our node image.

```bash
$ wget https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-amd64.img
```

## Resources

- [Terraform Download Page](https://www.terraform.io/downloads.html)
- [Experimenting with Terraform Thread](https://stackoverflow.com/questions/39211000/experimenting-locally-with-terraform)