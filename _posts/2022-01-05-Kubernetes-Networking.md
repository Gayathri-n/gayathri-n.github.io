---
layout: post
author: Gayathri
tag: Kubernetes
---

This blog post talks in detail about the networking in Kubernetes. We consider 3 scenarios - intrapod communication, interpod communication and services to pod communication.

## Intrapod communication

![Intrapod_communication](/assets/images/same-pod.gif)

The communication between the containers in the same pod happens via localhost and port numbers. Every pod gets its own network namespace that's created on the node's kernel. The containers inside a pod share this namespace which enables them to talk using localhost.

```bash
# List network namespace within a container
➜ ~ kubectl exec -ti $(kgp -o name | grep "device") -c linkerd-proxy -- /bin/sh
$ lsns -u -t net
        NS TYPE NPROCS PID USER    NETNSID NSFS COMMAND
4026533435 net       3   1 2102 unassigned      /usr/lib/linkerd/linkerd2-proxy
 
# List network namespaces on node => Find the same Namespace Identifier (NS - first column)
➜ ~ lsns -u -t net
        NS TYPE NPROCS     PID USER               NETNSID NSFS                                                COMMAND
4026533435 net       4 1549873 1001                    13 /run/netns/cni-21d23d1f-a1e8-6a9c-1f88-a790631ce1b1 /pause
```

Sandbox or Pause container is the secret container that k8s spins up first whenever a Pod is created. Its role is to reserve a network namespace and acts as a parent container for Pod. Any subsequent containers created for a Pod will inherit the same network namespace. 

- [Container runtime interface in Kubernetes](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)
- [What are paused containers?](https://www.ianlewis.org/en/almighty-pause-container)

### ifconfig

Each pod has 2 interfaces - eth0 and lo. The address on the eth0 interface will match the pod's IP address. The eth0 on a pod is connected to a virtual network interface VethX on the node - i.e., every pod has its own Veth interface on the node. 

```bash
➜  ~ kubectl exec -it mbiq-system-feature-deployment-84f9df6c77-mc9dd bash
bash-5.0$ ifconfig
eth0      Link encap:Ethernet  HWaddr 22:1C:69:B1:4C:A9 
          inet addr:10.42.0.106  Bcast:10.42.0.255  Mask:255.255.255.0
          inet6 addr: fe80::201c:69ff:feb1:4ca9/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:394177 errors:0 dropped:0 overruns:0 frame:0
          TX packets:313175 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:34505718 (32.9 MiB)  TX bytes:63511413 (60.5 MiB)
lo        Link encap:Local Loopback 
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:224992 errors:0 dropped:0 overruns:0 frame:0
          TX packets:224992 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:23674787 (22.5 MiB)  TX bytes:23674787 (22.5 MiB)
```

### How to find Veth tied to a pod

```bash
# Find the process ID of the container
➜  ~ ps aux | grep "go/bin/system-feature"
root      4744  0.0  0.0  14660  1120 pts/0    S+   18:42   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox go/bin/system-feature
gayathri 19791  0.0  0.2 719740 19388 ?        Ssl  Jun02   0:42 /go/bin/system-feature
 
# Find the ip addr in the container's namespace using the pid
➜  ~ nsenter -t 19791 -n ip addr         
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if147: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 22:1c:69:b1:4c:a9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.42.0.106/24 brd 10.42.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::201c:69ff:feb1:4ca9/64 scope link
       valid_lft forever preferred_lft forever
 
# eth0@if147 => 147 is the ID of the interface that is linked with the node
# Get the list of ip addresses in the node
 
➜  ~ ip addr
.....
139: veth3b27737c@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP group default
    link/ether 52:99:58:40:04:d8 brd ff:ff:ff:ff:ff:ff link-netnsid 34
    inet6 fe80::5099:58ff:fe40:4d8/64 scope link
       valid_lft forever preferred_lft forever
140: vetheee7d783@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP group default
    link/ether 42:2a:72:bc:88:46 brd ff:ff:ff:ff:ff:ff link-netnsid 28
    inet6 fe80::402a:72ff:febc:8846/64 scope link
       valid_lft forever preferred_lft forever
147: veth4e3b601a@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP group default
    link/ether 7e:9a:e2:10:f7:12 brd ff:ff:ff:ff:ff:ff link-netnsid 40
    inet6 fe80::7c9a:e2ff:fe10:f712/64 scope link
       valid_lft forever preferred_lft forever
 
# The veth corresponding to the ID 147 is the one ties to the pod/container.
```

## Interpod communication

![interpod_communication](/assets/images/pods-on-node.gif)

Inter-pod communication happens via the network bridge (cbr0). When the request hits the bridge, it checks if the destination IP matches with any of the pod's IP and routes the packet accordingly. The request then hops on to the Veth assigned to the pod whose IP matches.

### Pod to pod communication on the same node

References: [Kube-flannel](https://mvallim.github.io/kubernetes-under-the-hood/documentation/kube-flannel.html)

![pod-pod](/assets/images/pod-pod.png)
cni0 is Linux network bridge that enables communication between Pods. 

```bash
# Running on node
$ ip link show type bridge
5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 22:a2:de:cf:af:ce brd ff:ff:ff:ff:ff:ff
 
$ ip -c route show
(redacted)
10.42.0.0/24 dev cni0 proto kernel scope link src 10.42.0.1
 
$ ip neigh show
10.42.0.182 dev cni0 lladdr f6:e7:4a:6b:2a:ae REACHABLE
10.42.0.7 dev cni0 lladdr 3a:30:57:7b:50:a6 REACHABLE
10.42.0.253 dev cni0 lladdr 66:fe:8b:e0:89:93 REACHABLE
(redacted)
```

## Services to pod communication
A pod's IP is ephemeral - to have a static IP to talk to, we use Services and Cluster IP. Kube-proxy does the translation between the cluster IP and pod IP based on load balancing, basically adds/removes NAT to the node's iptables.
