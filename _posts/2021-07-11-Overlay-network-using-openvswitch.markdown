---
layout: post
title:  "Create an overlay network using Open vSwitch"
date:   2021-07-11 11:24:13 +0500
categories: openvswitch networking openstack
---

## Introduction 

This tutorial demonstrate how you can setup a simple Overlay network, using just Open vSwitch. Openvswitch is a multilayer virtual switch designed to enable massive network automation through programmatic extension.

The goal of the tutorial is to demonstrate a virtual switch between two sites/hosts using Open vSwitch

![image](https://gist.githubusercontent.com/cyrenity/25e0181ce225ba98e52f6bb0dd3878f6/raw/4cb8b521f9014766dee2eb71b2613bae034dd988/l2-switch-using-openvswitch.png)


## Environment
* 2 Ubuntu hosts running Ubuntu 20.04 LTS 
  
## Setting Up OVS

This section explain how to setup Open vSwitch

To install Open vSwitch run following command

```bash
sudo apt install -y openvswitch-switch
```

## Setting up bridge (switch) and interfaces
This section explains how to configure our switch (bridge) and configure virtual/physical interfaces 

Here, we are going to 
* Create an OVS bridge named `br-int` on both nodes
* An internal interface 
* An interface of type `VxLAN` for point-to-point communication 

> It should be noted that the MTU of the internal interface (veth0) should be set to 1450. This is due to the fact that packets are encapsulated in UDP packets and the size of the payload must be less than the size of the MTU of network your traffic is going to travel 

On your first host, run following commands 

```bash
sudo ovs-vsctl add-br br-int 
sudo ovs-vsctl add-port br-int veth0 -- set interface veth0 type=internal
sudo ovs-vsctl add-port br-int vxlan0 -- set interface vxlan0 type=vxlan \
                   options:remote_ip=192.168.122.11 options:key=101
sudo ip address add 10.0.0.10/24 dev veth0 
sudo ip link set dev veth0 up mtu 1450
```

And on second host, we have the same settings, however don't forget to update the underlay and overlay networks IPs

```bash
sudo ovs-vsctl add-br br-int 
sudo ovs-vsctl add-port br-int veth0 -- set interface veth0 type=internal
sudo ovs-vsctl add-port br-int vxlan0 -- set interface vxlan0 type=vxlan \
                   options:remote_ip=192.168.122.10 options:key=101
sudo ip address add 10.0.0.11/24 dev veth0 
sudo ip link set dev veth0 up mtu 1450
```

## Verify OVS configuration

This section explains how to verify overlay network, we just configured. Run `ovs-vsctl show` on both nodes and verify if the output is similar to shown below

on host-1

```
b6f578f4-1691-4573-b3ba-1d05e0eb7b22
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
        Port veth0
            Interface veth0
                type: internal
        Port vxlan0
            Interface vxlan0
                type: vxlan
                options: {key="101", remote_ip="192.168.122.11"}
    ovs_version: "2.13.3"

```

on host-2, config should look like this

```
4649a8a9-f107-41ca-ba0c-5b24903b1aea
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
        Port vxlan0
            Interface vxlan0
                type: vxlan
                options: {key="101", remote_ip="192.168.122.10"}
        Port veth0
            Interface veth0
                type: internal
    ovs_version: "2.13.3"

```


## Test the connectivity

By now, a simple ping test between overlay network interfaces should work

on host 1 ping overlay interface ip (10.0.0.11), which is configured on host 2

```
$ ping -c 3 10.0.0.11
PING 10.0.0.11 (10.0.0.11) 56(84) bytes of data.
64 bytes from 10.0.0.11: icmp_seq=1 ttl=64 time=0.987 ms
64 bytes from 10.0.0.11: icmp_seq=2 ttl=64 time=1.15 ms
64 bytes from 10.0.0.11: icmp_seq=3 ttl=64 time=1.14 ms

--- 10.0.0.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 0.987/1.093/1.152/0.075 ms
```

on host 2 ping the overlay interface ip (10.0.0.10), which is configured on host 1

```
$ ping -c 3 10.0.0.10
PING 10.0.0.10 (10.0.0.10) 56(84) bytes of data.
64 bytes from 10.0.0.10: icmp_seq=1 ttl=64 time=1.03 ms
64 bytes from 10.0.0.10: icmp_seq=2 ttl=64 time=1.17 ms
64 bytes from 10.0.0.10: icmp_seq=3 ttl=64 time=1.34 ms

--- 10.0.0.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 1.025/1.177/1.338/0.127 ms
```
## Conclusion

Although this is a simple lab demonstration, the real power comes when you use this to connect two different (virtual or physical) networks in different data centers. You’ll be able to create a Layer-2 network over Layer-3. It’s simple, fast and awesome.