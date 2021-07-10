---
layout: post
title:  "Setup Kubernetes in minutes using Talos"
date:   2020-03-05 06:16:30 +0500
categories: k3s kubernetes 
---

# Setup Kubernetes in minutes using Talos

### 1.  Get talosctl 

Download and install `talosctl` binary

```bash
wget https://github.com/talos-systems/talos/releases/download/v0.8.1/talosctl-linux-amd64

sudo install talosctl-linux-amd64 /usr/local/bin/talosctl 
```

### 2. Setup Docker Registry (Optional)

Create a local registry mirror for images required by talos for offline/quick installation


```
docker run -d -p 6000:5000 --restart always --name registry-airgapped registry:2
```

Identify the images required by talos

`talosctl images`


First, we pull all the images to our local Docker daemon

```bash
for image in `talosctl images`; do docker pull $image; done
```

Verify by running `docker images` command

Now we need to re-tag them so that we can push them to our local registry. We are going to replace the first component of the image name (before the first slash) with our registry endpoint 127.0.0.1:6000

```bash
for image in `talosctl images`; do \
    docker tag $image `echo $image | sed -E 's#^[^/]+/#127.0.0.1:6000/#'` \
done
```

As the next step, we push images to the internal registry

```bash
for image in `talosctl images`; do \
    docker push `echo $image | sed -E 's#^[^/]+/#127.0.0.1:6000/#'` \
done
```

We can now verify that the images are pushed to the registry

`curl  http://127.0.0.1:6000/v2/_catalog
`

The output should be similar 

```json
{"repositories":["autonomy/kubelet","coredns","coreos/flannel","etcd-development/etcd",
"kube-apiserver-amd64","kube-controller-manager-amd64","kube-proxy-amd64",
"kube-scheduler-amd64","talos-systems/install-cni","talos-systems/installer"]}
```


### 3. Create cluster nodes (VMs or Baremetal)

For the demonstration purpose we will use libvirtd/kvm for creating nodes in our lab environmnet 

##### Download the ISO Image
In order to install Talos in Virtual Machines, you will need the ISO image from the Talos release page. You can download `talos-amd64.iso` from github.com/talos-systems/talos/releases



##### Create the master node 

```bash
VM="k8s-master"
IP_ADDRESS="192.168.122.20"
MAC_ADDRESS="52:54:00:f2:d3:20" 

virsh net-update --network default \
     --command add-last --section ip-dhcp-host \
     --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
     --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 2 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk,bus=scsi \
     --cdrom ~/Downloads/talos-amd64.iso \
     --console=pty,target.type=serial --graphics none --noautoconsole   

```
##### Create worker nodes 

k8s-node1:

```bash
VM="k8s-node1"
IP_ADDRESS="192.168.122.21"
MAC_ADDRESS="52:54:00:f2:d3:21" 

virsh net-update --network default \
     --command add-last --section ip-dhcp-host \
     --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
     --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 2 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk,bus=scsi \
     --cdrom ~/Downloads/talos-amd64.iso \
     --console=pty,target.type=serial --graphics none --noautoconsole   


```
k8s-node2:

```bash
VM="k8s-node2"
IP_ADDRESS="192.168.122.22"
MAC_ADDRESS="52:54:00:f2:d3:22" 

virsh net-update --network default \
    --command add-last --section ip-dhcp-host \
    --xml "<host mac='${MAC_ADDRESS}' name='${VM}' ip='${IP_ADDRESS}'/>" \
    --live --config

virt-install --name ${VM} \
     --ram 2048 --vcpus 2 --os-type linux --os-variant ubuntu20.04 \
     --network bridge=virbr0,mac=${MAC_ADDRESS} \
     --disk path=~/libvirt/images/${VM}.qcow2,size=20,device=disk,bus=scsi \
     --cdrom ~/Downloads/talos-amd64.iso \
     --console=pty,target.type=serial --graphics none --noautoconsole   
```



### 4. Form the cluster using `talosctl`

Generate the machine configurations to use for installing Talos and Kubernete.
Using the DNS name of the load balancer, generate the base configuration files for the Talos machines

> **_NOTE:_** We are using IP (192.168.122.20) of the master node in our example, since this is just a proof of concept type lab work
> In production environment this will be a DNS name of the load balancer

```bash
talosctl gen config talos-virsh-cluster https://192.168.122.20:6443 \
    --output-dir _out \
    --registry-mirror '*'=http://192.168.122.1:6000
```
This will create several files in the `_out` directory: `init.yaml`, `controlplane.yaml`, `join.yaml`, and `talosconfig`.

> **_NOTE:_**  If you have setup offline registry mirror, you can provide the local registry mirror using --registry-mirror option, by default talos will pull images from docker hub





Apply init.yaml to control-plane/master node 

```bash
talosctl apply-config --insecure --nodes 192.168.122.20 --file _out/init.yaml
```


> **_Note:_** This process can be repeated multiple times to create an HA control plane. Simply apply controlplane.yaml instead of init.yaml for subsequent nodes.


Create worker nodes using a process similar to the control plane creation above. Start the worker node VMs and wait for them to enter "maintenance mode".

```bash
talosctl apply-config --insecure --nodes 192.168.122.21 --file _out/join.yaml
talosctl apply-config --insecure --nodes 192.168.122.22 --file _out/join.yaml
```

### 5. Using the cluster

Once the cluster is available, you can make use of `talosctl` and `kubectl` to interact with the cluster. 

First, configure `talosctl` to talk to your control plane node by issuing the following, updating paths and IPs as necessary:

```bash
export CONTROL_PLANE_IP=192.168.122.20
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
```

Fetch the `kubeconfig` file from the control plane node by issuing:

```bash
talosctl kubeconfig
```

You can then use kubectl

```bash
kubectl get nodes
```

### 6. Deploy first app

Create a deployment

```bash
kubectl create deployment hello-world --image=k8s.gcr.io/echoserver:1.4
```

Expose the service to the world 

```bash
kubectl expose deployment hello-world --type=NodePort --port=8080
```

### 7. Destroy cluster

Copy and pase following in your terminal 

```bash
virsh shutdown k8s-master
virsh shutdown k8s-node1
virsh shutdown k8s-node2

virsh destroy k8s-master
virsh destroy k8s-node1
virsh destroy k8s-node2

virsh undefine k8s-master
virsh undefine k8s-node1
virsh undefine k8s-node2

virsh net-update default delete ip-dhcp-host "<host name='k8s-master'/>" --live --config
virsh net-update default delete ip-dhcp-host "<host name='k8s-node1'/>" --live --config
virsh net-update default delete ip-dhcp-host "<host name='k8s-node2'/>" --live --config


rm -rf ~/libvirt/images/* 
