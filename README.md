# Rancher on KVM

> :warning: **Warning**
>
> This document has been compiled with notes taken during this setup. These steps were finalized into a document was months after the setup was completed. Please follow the instructions at your own risk.

This document serves as a guide for setting up a [Rancher](https://rancher.com/) [Kubernetes](https://kubernetes.io/) cluster on a local network [KVM](https://www.linux-kvm.org/) cluster. Please note, I would likely explore using [Proxmox](https://www.proxmox.com/) to manage the cluster nodes if I were to implement a new local network Kubernetes cluster instead of using the Terraform KVM Libvirt provider.

Additional routing configuration, not covered in this document, can be implemented to expose services to the public internet. This will require configuration specific to your network architecture.

The steps outlined here cover my experience setting up a KVM cluster ontop of a bare metal server. The host base image was [Ubuntu 18.04 LTS (Bionic Beaver)](https://releases.ubuntu.com/18.04/). This is an 11 node production Rancher cluster ontop of 24 cores and 100GB ECC memory. The nodes roles are:


| Count | Role                  | vCPUs | Memory |
| ----: | :-------------------- | :---- | :----- |
| 1     | Rancher Server Master | 4     | 4GB    |
| 2     | Rancher Server Slave  | 1     | 2GB    |
| 2     | Control Plane         | 1     | 2GB    |
| 3     | etcd                  | 2     | 6GB    |
| 3     | Worker                | 4     | 24GB   |
> **Note**
>
> The allocation to the *Rancher Server Master* node was changed with KVM directly, and it's listed values are not relected in the Terraform script.

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
source <( go env )
mkdir -p $GOPATH/src/github.com/johnfedoruk; cd $GOPATH/src/github.com/johnfedoruk
git clone https://github.com/johnfedoruk/terraform-provider-libvirt.git
cd $GOPATH/src/github.com/johnfedoruk/terraform-provider-libvirt
git checkout feature/ci-datasourcetypes
git remote add upstream https://github.com/dmacvicar/terraform-provider-libvirt.git
git pull upstream master
git merge upstream/master
grep "dmacvicar" -rl . | xargs sed -i s/dmacvicar/johnfedoruk/g
make install
ls $GOPATH/bin/terraform-provider-libvirt >/dev/null 2>/dev/null && echo "Install ok" || echo "Install failed"
mkdir -p "$HOME/.terraform.d/plugins"
cp "$GOPATH/bin/terraform-provider-libvirt" "$HOME/.terraform.d/plugins"
```

For more information on the issues with running RancherOS on Terraform, please see:
- https://github.com/dmacvicar/terraform-provider-libvirt/pull/476
- https://github.com/rancher/os/issues/2559

## Setting Up the VMs

### Get Node OS

At this point, we will want to install our target image. We are moving forward with Ubuntu as our node image.

```bash
wget https://cloud-images.ubuntu.com/releases/bionic/release/ubuntu-18.04-server-cloudimg-amd64.img
```

### Disabling AppArmor for KVM

The following error may become familiar to you if you do not disable AppArmor for KVM:

```
Error: Error creating libvirt domain: virError(Code=1, Domain=10, Message='internal error: qemu unexpectedly closed the monitor: 2020-04-13T00:46:14.839704Z qemu-system-x86_64: -drive file=/var/lib/libvirt/images/rancher-work_node-0_volume.qcow2,format=qcow2,if=none,id=drive-virtio-disk0: Could not reopen file: Permission denied')
```

Make the folloing changes in `/etc/apparmor.d/libvirt/TEMPLATE.qemu`

```
#
# This profile is for the domain whose UUID matches this file.
#

#include <tunables/global>

profile LIBVIRT_TEMPLATE flags=(attach_disconnected) {
  #include <abstractions/libvirt-qemu>
  /var/lib/libvirt/images**.qcow2 rwk,
}
```

For more information, see
- https://askubuntu.com/questions/741035/disabling-apparmor-for-kvm


### TerraForm Files

Now we get into it! We are going to create four files: `bridge.xml`, `cloud_ini.cfg`, `network_config.cfg`, and `main.tf`.

```plaintext
<network>
    <name>host-bridge</name>
    <forward mode="bridge"/>
    <bridge name="br0"/>
</network>
```
> **bridge.xm**
>
> No changes need to be made to this file unless your bridge interface name is different.

```plaintext
#cloud-config
users:
  - name: ${VM_USER}
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: users, admin
    home: /home/${VM_USER}
    shell: /bin/bash
    # lock-passwd: false
    ssh-authorized-keys:
      - ssh-rsa ...
ssh_pwauth: false
disable_root: false
chpasswd:
  list: |
     ${VM_USER}:rancher
  expire: False
package_update: true
package_upgrade: true
packages:
    - qemu-guest-agent
    - apt-transport-https
    - ca-certificates
    - curl
    - gnupg-agent
    - software-properties-common
    - zsh
growpart:
  mode: auto
  devices: ['/']
runcmd:
  - [ sh, -c, 'curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -' ]
  - [ sh, -c, 'sudo apt-key fingerprint 0EBFCD88']
  - [ sh, -c, 'sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"' ]
  - [ sh, -c, 'sudo apt update' ]
  - [ sh, -c, 'sudo apt install -y docker-ce docker-ce-cli containerd.io' ]
  - [ sh, -c, 'printf "\nalias dk=\"sudo docker\"\n" >> /home/${VM_USER}/.bashrc' ]
  - [ sh, -c, 'printf "\nalias dkc=\"sudo docker container\"\n" >> /home/${VM_USER}/.bashrc' ]
  - [ sh, -c, 'printf "\nalias dki=\"sudo docker image\"\n" >> /home/${VM_USER}/.bashrc' ]
  - [ sh, -c, 'printf "\nalias dks=\"sudo docker service\"\n" >> /home/${VM_USER}/.bashrc' ]
  - [ sh, -c, 'printf "\nalias dkn=\"sudo docker node\"\n" >> /home/${VM_USER}/.bashrc' ]
```
> **cloud_init.cfg**
>
> Be sure to add your own ssh key in line 10

```plaintext
version: 2
ethernets:
  ens3:
    dhcp4: true
```
> **network_config.cfg**
>
> Note that we need to set the servers to DHCP. They will be manually set to static IPs after the servers are created.

```plaintext
################################################################################
# ENV VARS
################################################################################

## Gloabl

variable "VM_USER" {
  default = "user"
  type    = string
}

variable "VM_IMG_PATH" {
  default = "./ubuntu-18.04-server-cloudimg-amd64.img"
  type    = string
}

variable "VM_IMG_DIR" {
  default = "/var/lib/libvirt/images"
  type    = string
}

variable "VM_IMG_FORMAT" {
  default = "qcow2"
  type    = string
}

variable "VM_BRIDGE" {
  default = "br0"
  type    = string
}

variable "VM_CIDR_RANGE" {
  default = "192.168.10.1/24"
  type    = string
}

variable "VM_CLUSTER" {
  default = "rancher"
  type    = string
}

## Server Nodes

variable "SRVR_NODE_HOSTNAME" {
  default = "srvr-node"
  type    = string
}

variable "SRVR_NODE_COUNT" {
  default = 3
  type    = number
}

variable "SRVR_NODE_VCPU" {
  default = 1
  type    = number
}

variable "SRVR_NODE_MEMORY" {
  default = "2048"
  type    = string
}

## ETCD Nodes

variable "ETCD_NODE_HOSTNAME" {
  default = "etcd-node"
  type    = string
}

variable "ETCD_NODE_COUNT" {
  default = 3
  type    = number
}

variable "ETCD_NODE_VCPU" {
  default = 2
  type    = number
}

variable "ETCD_NODE_MEMORY" {
  default = "6144"
  type    = string
}

## Controlplane Nodes

variable "CTRL_NODE_HOSTNAME" {
  default = "ctrl-node"
  type    = string
}

variable "CTRL_NODE_COUNT" {
  default = 2
  type    = number
}

variable "CTRL_NODE_VCPU" {
  default = 1
  type    = number
}

variable "CTRL_NODE_MEMORY" {
  default = "2048"
  type    = string
}

## Worker Nodes

variable "WORK_NODE_HOSTNAME" {
  default = "work-node"
  type    = string
}

variable "WORK_NODE_COUNT" {
  default = 3
  type    = number
}

variable "WORK_NODE_VCPU" {
  default = 4
  type    = number
}

variable "WORK_NODE_MEMORY" {
  default = "24576"
  type    = string
}

################################################################################
# PROVIDERS
################################################################################

# instance the provider
provider "libvirt" {
  uri = "qemu:///system"
}

################################################################################
# DATA TEMPLATES
################################################################################

# https://www.terraform.io/docs/providers/template/d/file.html

# https://www.terraform.io/docs/providers/template/d/cloudinit_config.html
data "template_file" "user_data" {
  template = file("${path.module}/cloud_init.cfg")
  vars = {
    VM_USER = var.VM_USER
  }
}

data "template_file" "network_config" {
  template = file("${path.module}/network_config.cfg")
}

################################################################################
# RESOURCES
################################################################################

resource "libvirt_pool" "vm" {
  name = "${var.VM_CLUSTER}_pool"
  type = "dir"
  path = var.VM_IMG_DIR
}

resource "libvirt_network" "vm_public_network" {
  name   = "${var.VM_CLUSTER}_network"
  mode   = "bridge"
  bridge = var.VM_BRIDGE

  addresses = ["${var.VM_CIDR_RANGE}"]
  autostart = true
  dhcp {
    enabled = true
  }
  dns {
    enabled = true
  }
}

resource "libvirt_cloudinit_disk" "cloudinit" {
  name             = "${var.VM_CLUSTER}_cloudinit.iso"
  user_data        = data.template_file.user_data.rendered
  network_config   = data.template_file.network_config.rendered
  data_source_type = "ec2"
  pool             = libvirt_pool.vm.name
}

## SRVR Node

resource "libvirt_volume" "srvr_node" {
  count  = var.SRVR_NODE_COUNT
  name   = format("${var.VM_CLUSTER}-${var.SRVR_NODE_HOSTNAME}-%02s_volume.${var.VM_IMG_FORMAT}", count.index)
  pool   = libvirt_pool.vm.name
  source = var.VM_IMG_PATH
  format = var.VM_IMG_FORMAT
}

resource "libvirt_domain" "srvr_node" {
  count      = var.SRVR_NODE_COUNT
  name       = format("${var.SRVR_NODE_HOSTNAME}-%02s", count.index)
  memory     = var.SRVR_NODE_MEMORY
  vcpu       = var.SRVR_NODE_VCPU
  autostart  = true
  qemu_agent = true

  cloudinit = libvirt_cloudinit_disk.cloudinit.id

  network_interface {
    network_id = libvirt_network.vm_public_network.id
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  console {
    type        = "pty"
    target_type = "virtio"
    target_port = "1"
  }

  disk {
    volume_id = libvirt_volume.srvr_node[count.index].id
  }

  graphics {
    type        = "vnc"
    listen_type = "address"
    autoport    = true
  }
}

## ETCD Node

resource "libvirt_volume" "etcd_node" {
  count  = var.ETCD_NODE_COUNT
  name   = format("${var.VM_CLUSTER}-${var.ETCD_NODE_HOSTNAME}-%02s_volume.${var.VM_IMG_FORMAT}", count.index)
  pool   = libvirt_pool.vm.name
  source = var.VM_IMG_PATH
  format = var.VM_IMG_FORMAT
}

resource "libvirt_domain" "etcd_node" {
  count      = var.ETCD_NODE_COUNT
  name       = format("${var.ETCD_NODE_HOSTNAME}-%02s", count.index)
  memory     = var.ETCD_NODE_MEMORY
  vcpu       = var.ETCD_NODE_VCPU
  autostart  = true
  qemu_agent = true

  cloudinit = libvirt_cloudinit_disk.cloudinit.id

  network_interface {
    network_id = libvirt_network.vm_public_network.id
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  console {
    type        = "pty"
    target_type = "virtio"
    target_port = "1"
  }

  disk {
    volume_id = libvirt_volume.etcd_node[count.index].id
  }

  graphics {
    type        = "vnc"
    listen_type = "address"
    autoport    = true
  }
}

## CTRL Node

resource "libvirt_volume" "ctrl_node" {
  count  = var.CTRL_NODE_COUNT
  name   = format("${var.VM_CLUSTER}-${var.CTRL_NODE_HOSTNAME}-%02s_volume.${var.VM_IMG_FORMAT}", count.index)
  pool   = libvirt_pool.vm.name
  source = var.VM_IMG_PATH
  format = var.VM_IMG_FORMAT
}

resource "libvirt_domain" "ctrl_node" {
  count      = var.CTRL_NODE_COUNT
  name       = format("${var.CTRL_NODE_HOSTNAME}-%02s", count.index)
  memory     = var.CTRL_NODE_MEMORY
  vcpu       = var.CTRL_NODE_VCPU
  autostart  = true
  qemu_agent = true

  cloudinit = libvirt_cloudinit_disk.cloudinit.id

  network_interface {
    network_id = libvirt_network.vm_public_network.id
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  console {
    type        = "pty"
    target_type = "virtio"
    target_port = "1"
  }

  disk {
    volume_id = libvirt_volume.ctrl_node[count.index].id
  }

  graphics {
    type        = "vnc"
    listen_type = "address"
    autoport    = true
  }
}

## WORK Node

resource "libvirt_volume" "work_node" {
  count  = var.WORK_NODE_COUNT
  name   = format("${var.VM_CLUSTER}-${var.WORK_NODE_HOSTNAME}-%02s_volume.${var.VM_IMG_FORMAT}", count.index)
  pool   = libvirt_pool.vm.name
  source = var.VM_IMG_PATH
  format = var.VM_IMG_FORMAT
}

resource "libvirt_domain" "work_node" {
  count      = var.WORK_NODE_COUNT
  name       = format("${var.WORK_NODE_HOSTNAME}-%02s", count.index)
  memory     = var.WORK_NODE_MEMORY
  vcpu       = var.WORK_NODE_VCPU
  autostart  = true
  qemu_agent = true

  cloudinit = libvirt_cloudinit_disk.cloudinit.id

  network_interface {
    network_id = libvirt_network.vm_public_network.id
  }

  console {
    type        = "pty"
    target_port = "0"
    target_type = "serial"
  }

  console {
    type        = "pty"
    target_type = "virtio"
    target_port = "1"
  }

  disk {
    volume_id = libvirt_volume.work_node[count.index].id
  }

  graphics {
    type        = "vnc"
    listen_type = "address"
    autoport    = true
  }
}

################################################################################
# TERRAFORM CONFIG
################################################################################

terraform {
  required_version = ">= 0.12"
}
```
> **main.tf**
>
> You may want to set a better username on line 8.

### Bringing Up the Nodes

Simply run `terraform apply` to bring the nodes up.

If you need to destroy your terraform cluster, run

```
terraform destroy
virsh list --all | grep " - " | cut -d\  -f6 | xargs -I {} virsh undefine {}
virsh net-undefine rancher_network
virsh net-undefine br0
```

## Installing Rancher

### Network Configuration

You should be able to SSH into the nodes using the key you set in `cloud_init.cfg`. Alternatively, you may opt to access the nodes using `virt-manager`.

Access each node and use netplan to set the network configuration.

```
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: false
      addresses: [ 192.168.10.181/24 ]
      gateway4: 192.168.10.1
      nameservers:
        addresses: [ 192.168.10.1,8.8.8.8 ]
        search: [ work-node-01 ]
```
> **/etc/netplan/50-cloud-init.yaml**
>
> Each node will require modification of this line to reflect planned network architecture.

Then run netplan apply and reboot the system:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml 
sudo netplan apply
```

Now, enable IP forward. Without this, you may have issues with the docker host routing traffic across the overlay netowrk.

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Now reboot the system

```
sudo reboot
```

### Installing the Rancher Server

We will install Rancher on the server nodes. Choose a node to be the master, and run the following commands.

```bash
sudo curl -sfL https://get.k3s.io | sh -
sudo snap install helm --classic
sudo helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
export RANCHER_HOSTNAME="admin.rancher.local"
sudo helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname="$RANCHER_HOSTNAME" --set ingress.tls.source=rancher --kubeconfig /etc/rancher/k3s/k3s.yaml
```

Before existing, read `/var/lib/rancher/k3s/server/node-token`. You will need privleges to read this file. You will need to set an env variable to this value for each of the Rancher server slaves.

We assume the master's address is `192.168.10.150`. Please make any necessary adjustments.

Login to each of the Rancher server slaves and run the following:

```bash
export K3S_TOKEN="..."
sudo curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.150:6443 K3S_TOKEN=$K3S_TOKEN sh -
```

From the master node, run the following and wait until all the slaves are online:

```bash
watch -n 1 "sudo k3s kubectl get nodes"
```

Once all slaves are online, you should setup DNS to resolve `admin.rancher.local` (hostname set in the Rancher server master node) to `192.168.10.150` (address of the rancher admin master node). The easiest way is to complete this is to simply edit the hosts file on a client computer.

Navigating to [admin.rancher.local](admin.rancher.local) should bring up the Rancher admin screen. Please follow the instructions to complete your setup. Once complete, you should be able to utilize your remaining VMs to build your local cluster! 

## Resources

- https://www.terraform.io/downloads.html
- https://stackoverflow.com/questions/39211000/experimenting-locally-with-terraform
- https://www.pimwiddershoven.nl/entry/deploy-kubernetes-cluster-with-rancher-kubernetes-engine-rke
- https://www.pimwiddershoven.nl/entry/deploy-kubernetes-cluster-with-rancher-kubernetes-engine-rke
- https://github.com/dmacvicar/terraform-provider-libvirt/blob/master/website/docs/r/network.markdown
- https://github.com/dmacvicar/terraform-provider-libvirt/blob/master/libvirt/resource_libvirt_network.go
- https://www.redhat.com/archives/libvir-list/2011-November/msg00876.html
- https://forum.level1techs.com/t/- cant-seem-to-configure-a-bridge-for-windows-guest-in-virt-manager-update-trying-to-use-virtio/111719/13
- https://www.reddit.com/r/linuxquestions/comments/97ksky/how_to_make_a_network_bridge_for_kvm_with_enp4s0/
- https://askubuntu.com/questions/179508/kvm-bridged-network-not-working
- https://www.redhat.com/archives/libvirt-users/2014-March/msg00091.html
- https://blog.wikichoon.com/2019/04/host-network-interfaces-panel-removed.html
- https://www.cyberciti.biz/faq/how-to-add-network-bridge-with-nmcli-networkmanager-on-linux/
- https://ubuntuforums.org/showthread.php?t=1789126
- https://github.com/dmacvicar/terraform-provider-libvirt/issues/272
- https://serverfault.com/questions/672253/- how-to-configure-and-use-qemu-guest-agent-in-ubuntu-12-04-my-main-aim-is-to-get
- https://askubuntu.com/questions/741035/disabling-apparmor-for-kvm
