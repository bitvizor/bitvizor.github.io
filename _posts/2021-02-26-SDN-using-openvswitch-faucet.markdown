---
layout: post
title:  "SDN using Open vSwitch and Faucet OpenFlow Controller"
date:   2020-02-26 11:24:13 +0500
categories: k8s kubernetes talos
---

## Introduction 

This tutorial demonstrate how you can setup a simple SDN network, using the Faucet controller and Open vSwitch

The goal of the tutorial is to demonstrate Open vSwitch and Faucet in an end-to-end way, that is, to show how it works from the Faucet controller configuration at the top, through the OpenFlow flow table, to the datapath processing. Along the way

## Environment 
* Ubuntu 20.04
  
## Setting Up OVS

This section explain how to setup Open vSwitch for the purpose of using it with Faucet controller

To install Open vSwitch run following command

```bash
sudo apt install -y openvswitch-switch
```

## Setting up Faucet
This section explains how to install Faucet inside a docker container 

```bash
sudo mkdir -p /opt/faucet/{logs,config,scripts,images}
sudo touch /opt/faucet/config/faucet.yaml

sudo docker run -d \
    --name faucet \
    --restart=always \
    -v /opt/faucet/config/:/etc/faucet/ \
    -v /opt/faucet/logs/:/var/log/faucet/ \
    -p 6653:6653 \
    -p 9302:9302 \
    faucet/faucet
```


## Overview

Now that Open vSwitch and Faucet are ready, here’s an overview of what we’re going to do for the remainder of the tutorial:

* Switching: Set up an L2 network with Faucet.
* Routing: Route between multiple L3 networks with Faucet.
* ACLs: Add and modify access control rules.

At each step, we will take a look at how the features in question work from Faucet at the top to the data plane layer at the bottom. From the highest to lowest level, these layers and the software components that connect them are:

![image](https://gist.githubusercontent.com/cyrenity/397c6baebdc20d9a9e377523f256620e/raw/3f9b3d14bd21affd6c7e4c44e9dac55b02ce384e/OpenvSwitch-Faucet-Lab1.png)

### Faucet.
As the top level in the system, this is the authoritative source of the network configuration.

> Faucet connects to a variety of monitoring and performance tools, but we won’t use them in this tutorial. Our main insights into the system will be through faucet.yaml for configuration and faucet.log to observe state, such as MAC learning and ARP resolution, and to tell when we’ve screwed up configuration syntax or semantics.

### The OpenFlow subsystem in Open vSwitch.
OpenFlow is the protocol, standardized by the Open Networking Foundation, that controllers like Faucet use to control how Open vSwitch and other switches treat packets in the network.

We will use `ovs-ofctl`, a utility that comes with Open vSwitch, to observe and occasionally modify Open vSwitch’s OpenFlow behavior. We will also use `ovs-appctl`, a utility for communicating with `ovs-vswitchd` and other Open vSwitch daemons, to ask “what-if?” type questions.


## Setup a switch

Layer-2 (L2) switching is the basis of modern networking. It’s also very simple and a good place to start, so let’s set up a switch with some VLANs in Faucet and see how it works at each layer. Begin by putting the following into `faucet/config/faucet.yaml`:


```yaml
dps:
    switch0:
        dp_id: 0x1
        timeout: 7201
        arp_neighbor_timeout: 3600
        stack:
            priority: 1
        interfaces:
            1:
                native_vlan: office
            2:
                native_vlan: office
            3:
                native_vlan: office
            4:
                native_vlan: guest
            5:
                native_vlan: guest
vlans:
    office:
        vid: 100
        description: "Office network 100 Vlan"
        faucet_mac: "0e:00:00:00:00:01"
        faucet_vips: ['10.0.100.254/24'] 
    guest:
        vid: 200
        description: "Guest network"
        faucet_mac: "0e:00:00:00:00:02"
        faucet_vips: ['10.0.200.254/24']

routers:
    router-datacenter-networks:
        vlans: [office, guest]  
        
```
This configuration file defines a single switch (“datapath” or “dp”) named `switch0`. The switch has five ports, numbered 1 through 5. Ports 1, 2, and 3 are in VLAN 100, and ports 4 and 5 are in VLAN 200. Faucet can identify the switch from its datapath ID, which is defined to be `0x1`.

Now restart Faucet so that the configuration takes effect, e.g.:

```bash
$ sudo docker restart faucet
```

Assuming that the configuration update is successful, you should now see a new line at the end of `/opt/faucet/logs/faucet.log`:

```bash
faucet INFO     Add new datapath DPID 1 (0x1)
faucet.valve INFO     DPID 1 (0x1) switch0 IPv4 routing is active on VLAN office vid:100 untagged: Port 1,Port 2,Port 3 with VIPs ['10.0.100.254/24']
faucet.valve INFO     DPID 1 (0x1) switch0 IPv4 routing is active on VLAN guest vid:200 untagged: Port 4,Port 5 with VIPs ['10.0.200.254/24']

```

Faucet is now waiting for a switch with datapath ID `0x1` to connect to it over OpenFlow, so our next step is to create a switch with OVS and make it connect to Faucet. Let's create a switch (“bridge”) named switch0, set its datapath ID to `0x1` and tell it to connect to the Faucet controller

To do that, switch to the terminal and issue following command

```bash
sudo ovs-vsctl add-br switch0 \
         -- set bridge switch0 other-config:datapath-id=0000000000000001 \
         -- set-controller switch0 tcp:127.0.0.1:6653 \
         -- set controller switch0 connection-mode=out-of-band
```


We are now ready to launch some VMS with and connect ports into the switch

## Setting up KVM instances
This section describes how to use Open vSwitch with the Kernel-based Virtual Machine (KVM).

You will need to modify or create custom versions of the qemu-ifup and qemu-ifdown scripts. In this guide, we’ll create custom versions that make use of example Open vSwitch bridges that we’ll describe in this guide.

Create the following two files and store them in known locations. For example:

```bash
$ cat << 'EOF' > /opt/faucet/scripts/ovs-ifup
#!/bin/bash

netdev=$1
switch="switch${netdev:2:1}"
ip link set $netdev up
ovs-vsctl add-port ${switch} $netdev -- set Interface $netdev ofport=${netdev: -1}
EOF
```
```bash
$ cat << 'EOF' > /opt/faucet/scripts/ovs-ifdown
#!/bin/bash

netdev=$1
switch="br${netdev:2:1}"
ip addr flush dev $netdev
ip link set $netdev down
ovs-vsctl del-port ${switch} $netdev
EOF
```

Make these script executable by issuing:
```bash
chmod +x /opt/faucet/scripts/ovs-*
```

###
CirrOS is a minimal Linux distribution that was designed for use as a test image on clouds such as OpenStack Compute.

Download CirrOS 

```bash
sudo wget -P /opt/faucet/images https://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
```

## Launch Instances 
We will launch 4 instances of CirrOS and connect them with `switch0` 

Let's start by booting up and connecting our first instance, open a new terminal windows/tab and issuing following commands in an interactive way

```bash
IFACE=switch0p1
HOST=host1
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))

cp /opt/faucet/images/cirros-0.5.2-x86_64-disk.img /opt/faucet/images/cirros-${HOST}.img

kvm -m 512 \
	-device e1000,netdev=${IFACE},mac=$MAC_ADDR  \
	-drive file=/opt/faucet/images/${HOST}.img,boot=on -nographic \
	-netdev tap,id=${IFACE},ifname=${IFACE},script=/opt/faucet/scripts/ovs-ifup,downscript=/opt/faucet/scripts/ovs-ifdown
```

Verify if port has been created, check for following lines at the end of `logs/faucet.log`

```
DPID 1 (0x1) switch-1 status change: Port 1 up status True reason ADD state 0
DPID 1 (0x1) switch-1 Port 1 (1) up
DPID 1 (0x1) switch-1 Configuring VLAN office vid:100 untagged: Port 1,Port 2,Port 3
DPID 1 (0x1) switch-1 status did not change: Port 1 up status True reason MODIFY state 4
DPID 1 (0x1) switch-1 L2 learned on Port 1 52:54:00:b1:6c:84 (L2 type 0x0800, L2 dst ff:ff:ff:ff:ff:ff, L3 src 0.0.0.0, L3 dst 255.255.255.255) Port 1 VLAN 100 (1 hos
```
> take a look at line at the end of `faucet/faucet.log`. It says that it learned about our instance's MAC 52:54:00:b1:6c:84

Take a look at current state of Open vSwitch using `ovs-vsctl show` command.

```bash
ovs-vsctl show 
```

```
328b079b-4efb-4b3e-8e49-2d1626fe73fc
    Bridge switch0
        Controller "tcp:127.0.0.1:6653"
            is_connected: true
        Port switch0p1
            Interface switch0p1
        Port switch0
            Interface switch0
                type: internal
    ovs_version: "2.13.3"
```

You can see a new port `switch0p1` created using `ovs-ifup` script on instance launch, is added into `switch0` 

`ovs-ifup` and `ovs-ifdown` take `switch0p1` as input argument and create the tap device and add it into port 1 of the `switch0` using `ovs-vsctl`, this naming scheme helps identify data paths and ports at every layer (OVS, Faucet, Host system) 

Let's launch more instances and assign ports in Open vSwitch

### host2, port = 2 


```bash
IFACE=switch0p2
HOST=host2
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))

cp /opt/faucet/images/cirros-0.5.2-x86_64-disk.img /opt/faucet/images/cirros-${HOST}.img

kvm -m 512 \
	-device e1000,netdev=${IFACE},mac=$MAC_ADDR  \
	-drive file=/opt/faucet/images/${HOST}.img,boot=on -nographic \
	-netdev tap,id=${IFACE},ifname=${IFACE},script=/opt/faucet/scripts/ovs-ifup,downscript=/opt/faucet/scripts/ovs-ifdown
```


### host3, port = 3


```bash
IFACE=switch0p3
HOST=host3
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))

cp /opt/faucet/images/cirros-0.5.2-x86_64-disk.img /opt/faucet/images/cirros-${HOST}.img

kvm -m 512 \
	-device e1000,netdev=${IFACE},mac=$MAC_ADDR  \
	-drive file=/opt/faucet/images/${HOST}.img,boot=on -nographic \
	-netdev tap,id=${IFACE},ifname=${IFACE},script=/opt/faucet/scripts/ovs-ifup,downscript=/opt/faucet/scripts/ovs-ifdown
```


### host4, port = 4 


```bash
IFACE=switch0p4
HOST=host4
MAC_ADDR=$(printf '52:54:00:%02x:%02x:%02x' $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)))

cp /opt/faucet/images/cirros-0.5.2-x86_64-disk.img /opt/faucet/images/cirros-${HOST}.img

kvm -m 512 \
	-device e1000,netdev=${IFACE},mac=$MAC_ADDR  \
	-drive file=/opt/faucet/images/${HOST}.img,boot=on -nographic \
	-netdev tap,id=${IFACE},ifname=${IFACE},script=/opt/faucet/scripts/ovs-ifup,downscript=/opt/faucet/scripts/ovs-ifdown
```

## Configure Network on all nodes


### host1: 
```
sudo ip a a 10.0.100.1/24 dev eth0
sudo ip r a default via 10.0.100.254 dev eth0
```

### host2: 
```
sudo ip a a 10.0.100.2/24 dev eth0
sudo ip r a default via 10.0.100.254 dev eth0
```

### host3:
```
sudo ip a a 10.0.100.3/24 dev eth0
sudo ip r a default via 10.0.100.254 dev eth0
```

### host4:
``` 
sudo ip a a 10.0.200.1/24 dev eth0
sudo ip r a default via 10.0.200.254 dev eth0
```

Verify by ping-ing nodes from each other, in this scnario all nodes shoule be able to ping all nodes