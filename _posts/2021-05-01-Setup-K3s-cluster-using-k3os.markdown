---
layout: post
title:  "Setup K3s (Lightweight Kubernetes) cluster in seconds using k3os "
date:   2020-05-01 06:16:30 +0500
categories: k3s kubernetes 
---

## Setup K3s (Lightweight Kubernetes) cluster in seconds using k3os 

We are going to setup one master and two worker nodes 

###### Prepare nodes/hosts

Create the master node k3s-master with following script: 

```bash
VM="k3s-master"
IP_ADDRESS="192.168.122.20"
MAC_ADDRESS="52:54:00:f2:d3:20" 


KERNEL_ARGS="\
console=ttyS0 \
k3os.mode=install \
k3os.install.silent=true \
k3os.install.device=/dev/vda \
hostname=k3s-master \
k3os.password=emergen \
k3os.token=A_LONG_SECRET_STRING \
k3os.dns_nameserver=8.8.4.4 \
k3os.dns_nameserver=8.8.8.8 \
k3os.ntp_server=0.us.pool.ntp.org \
k3os.ntp_server=1.us.pool.ntp.org \
k3os.k3s_args=server \
k3os.k3s_args='--disable-agent' \
k3os.install.tty=ttyS0 
"  

virsh net-update --network default \
     --command add-last --section ip-dhcp-host \
     --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
     --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 2 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk \
     --disk path=k3os-amd64.iso,device=cdrom \
     --install kernel=k3os-vmlinuz-amd64,initrd=k3os-initrd-amd64 \
     --extra-args "${KERNEL_ARGS}" \
     --console=pty,target.type=serial --noautoconsole
     

```

###### Create worker nodes 

Run following script to create k3s-node1:

```bash
VM="k3s-node1"
IP_ADDRESS="192.168.122.21"
MAC_ADDRESS="52:54:00:f2:d3:21" 

KERNEL_ARGS="\
console=ttyS0 \
k3os.mode=install \
k3os.install.silent=true \
k3os.install.device=/dev/vda \
hostname=k3s-master \
k3os.password=emergen \
k3os.token=A_LONG_SECRET_STRING \
k3os.dns_nameserver=8.8.4.4 \
k3os.dns_nameserver=8.8.8.8 \
k3os.ntp_server=0.us.pool.ntp.org \
k3os.ntp_server=1.us.pool.ntp.org \
k3os.server_url=https://192.168.122.20:6443 \
k3os.k3s_args=agent \
k3os.install.tty=ttyS0 
"

virsh net-update --network default \
     --command add-last --section ip-dhcp-host \
     --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
     --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 1 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk \
     --disk path=k3os-amd64.iso,device=cdrom \
     --install kernel=k3os-vmlinuz-amd64,initrd=k3os-initrd-amd64 \
     --extra-args "${KERNEL_ARGS}" \
     --console=pty,target.type=serial --noautoconsole
     
```

Run following script to create k3s-node2:

```bash
VM="k3s-node2"
IP_ADDRESS="192.168.122.22"
MAC_ADDRESS="52:54:00:f2:d3:22" 

KERNEL_ARGS="\
console=ttyS0 \
k3os.mode=install \
k3os.install.silent=true \
k3os.install.device=/dev/vda \
hostname=k3s-master \
k3os.password=emergen \
k3os.token=A_LONG_SECRET_STRING \
k3os.dns_nameserver=8.8.4.4 \
k3os.dns_nameserver=8.8.8.8 \
k3os.ntp_server=0.us.pool.ntp.org \
k3os.ntp_server=1.us.pool.ntp.org \
k3os.server_url=https://192.168.122.20:6443 \
k3os.k3s_args=agent \
k3os.install.tty=ttyS0 
"
virsh net-update --network default \
     --command add-last --section ip-dhcp-host \
     --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
     --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 1 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk \
     --disk path=k3os-amd64.iso,device=cdrom \
     --install kernel=k3os-vmlinuz-amd64,initrd=k3os-initrd-amd64 \
     --extra-args "${KERNEL_ARGS}" \
     --console=pty,target.type=serial --noautoconsole
     
```




###### Start Nodes

```bash
virsh start k3s-master
virsh start k3s-node1
virsh start k3s-node2
```



Login to k3s-master and issue `kubectl get nodes`, you should see all three nodes 




##### Destroying the cluster

```bash
virsh shutdown k3s-master
virsh shutdown k3s-node1
virsh shutdown k3s-node2

virsh undefine k3s-master
virsh undefine k3s-node1
virsh undefine k3s-node2

virsh net-update default delete ip-dhcp-host "<host name='k3s-master'/>" --live --config
virsh net-update default delete ip-dhcp-host "<host name='k3s-node1'/>" --live --config
virsh net-update default delete ip-dhcp-host "<host name='k3s-node2'/>" --live --config

rm -rf ~/libvirt/images/* 

```