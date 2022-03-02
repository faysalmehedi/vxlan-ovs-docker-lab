# VXLAN OPENVSWITCH DOCKER LAB

## This hands-on demo will provide an overview of container communication between `multi-node or multi container daemon` under the hood using Open vSwitch, docker and VXLAN.

#### What is Underlay and Overlay Network?

`Underlay Network` is physical infrastructure above which overlay network is built. It is the underlying network responsible for delivery of packets across networks. Underlay networks can be Layer 2 or Layer 3 networks. Layer 2 underlay networks today are typically based on Ethernet, with segmentation accomplished via VLANs. The Internet is an example of a Layer 3 underlay network.

`An Overlay Network` is a virtual network that is built on top of underlying network infrastructure (Underlay Network). Actually, “Underlay” provides a “service” to the overlay. Overlay networks implement network virtualization concepts. A virtualized network consists of overlay nodes (e.g., routers), where Layer 2 and Layer 3 tunneling encapsulation (VXLAN, GRE, and IPSec) serves as the transport overlay protocol.

#### What is VxLAN? 

`VxLAN — or Virtual Extensible LAN` addresses the requirements of the Layer 2 and Layer 3 data center network infrastructure in the presence of VMs in a multi-tenant environment. It runs over the existing networking infrastructure and provides a means to "stretch" a Layer 2 network.  In short, VXLAN is a Layer 2 overlay scheme on a Layer 3 network.  Each overlay is termed a VXLAN segment.  Only VMs within the same VXLAN segment can communicate with each other.  Each VXLAN segment is identified through a 24-bit segment ID, termed the "VNI".  This allows up to 16 M VXLAN segments to coexist within the same administrative domain.

#### What is VNI?

Unlike VLAN, VxLAN does not have ID limitation. It uses a 24-bit header, which gives us about 16 million VNI’s to use. A VNI `VXLAN Network Identifier (VNI)` is the identifier for the LAN segment, similar to a VLAN ID. With an address space this large, an ID can be assigned to a customer, and it can remain unique across the entire network.

#### What is VTEP?

VxLAN traffic is encapsulated before it is sent over the network. This creates stateless tunnels across the network, from the source switch to the destination switch. The encapsulation and decapsulation are handled by a component called a `VTEP (VxLAN Tunnel End Point)`. A VTEP has an IP address in the underlay network. It also has one or more VNI’s associated with it. When frames from one of these VNI’s arrives at the Ingress VTEP, the VTEP encapsulates it with UDP and IP headers.  The encapsulated packet is sent over the IP network to the Egress VTEP. When it arrives, the VTEP removes the IP and UDP headers, and delivers the frame as normal.


### Packet Walk

#### How traffic passes through a simple VxLAN network.

![Project Diagram](https://github.com/faysalmehedi/vxlan-ovs-docker-lab/blob/main/vxlan_PacketWalk.png)

_the diagrom is taken from networkdirection blog_

- A frame arrives on a switch port from a host. This port is a regular untagged (access) port, which assigns a VLAN to the traffic - The switch determines that the frame needs to be forwarded to another location. The remote switch is connected by an IP network It may be close or many hops away.
- The VLAN is associated with a VNI, so a VxLAN header is applied. The VTEP encapsulates the traffic in UDP and IP headers. UDP port 4789 is used as the destination port. The traffic is sent over the IP network 
- The remote switch receives the packet and decapsulates it. A regular layer-2 frame with a VLAN ID is left 
- The switch selects an egress port to send the frame out. This is based on normal MAC lookups. The rest of the process is as normal.

### Get an overview of the hands-on from the diagram below
![Project Diagram](https://github.com/faysalmehedi/vxlan-ovs-docker-lab/blob/main/vxlan-demo.png)


***For this demo, as I am going to keep everything simple and only focus on vxlan feature, anyone can deploy two VM on any hypervisor or virtualization technology. Make sure they are on the same network thus hosts can communicate each other. I launched two ec2 instance(ubuntu) which is on same VPC from AWS to simulate this hands-on. In case of AWS, please allow all traffic in security group to avoid connectivity issues.***

#### What are we going to cover in this hands-on demo?

-     We will use two VM for this, will install OpenVSwitch, docker on them
-     Then we will create two bridges via OpenVSwitch and configure them
-     Then we will create docker container with none network and will connect them to the previously created bridges
-     After that the main part of this demo, we will create VXLAN Tunneling between VM's and make the overlay network
-     We will how we can ping one host to each other
-    Last not least will configure iptables for communicating with the outer world.

### Let's start...

**Step 0: As we already have our VM installed or launch from AWS, please make sure they can communicate each other. It can be done by ping utility. It's important because it means that our UNDERLAY network is working properly.  Then update packeages and install essential packeges for this demo on both VM.**
```bash
# update the repository
sudo apt update
# Install essential tools
sudo apt -y install net-tools docker.io openvswitch-switch
```

**Step 1: Now create two bridges per VM using OpenVSwitch `ovs-vsctl` cli utility.**

```bash
# For VM1 & VM2:
# Create two bridge using ovs
sudo ovs-vsctl add-br ovs-br0
sudo ovs-vsctl add-br ovs-br1
```

**Then create the internal port/interfaces to the ovs-bridge:**
```bash
# For VM1 & VM2
# add port/interfaces to bridges
sudo ovs-vsctl add-port ovs-br0 veth0 -- set interface veth0 type=internal
sudo ovs-vsctl add-port ovs-br1 veth1 -- set interface veth1 type=internal
# ovs-br0 is the bridge name
# veth0 is the interface/port name where type is 'internal'

# check the status of bridges
sudo ovs-vsctl show
```

**Now it's time to set the IP of the bridges and up the inteface:**

```bash
# For VM1 & VM2

# set the ip to the created port/interfaces
sudo ip address add 192.168.1.1/24 dev veth0 
sudo ip address add 192.168.2.1/24 dev veth1

# Check the status, link should be down
ip a

# up the interfaces and check status
sudo ip link set dev veth0 up mtu 1450
sudo ip link set dev veth1 up mtu 1450

# Check the status, link should be UP/UNKNOWN 
ip a
```
**
Step 2: It's time to set docker container with None network. Also as container will not get any internet connection for now, we will need some tools to analysis so I have wriiten a Dockerfile for this. Build the image first then run the container.**

```bash
# For VM1
# create a docker image from the docker file 
# find the Dockerfile in the repo
sudo docker build . -t ubuntu-docker

# create containers from the created image; Containers not connected to any network
sudo docker run -d --net=none --name docker1 ubuntu-docker
sudo docker run -d --net=none --name docker2 ubuntu-docker

# check container status and ip 
sudo docker ps
sudo docker exec docker1 ip a
sudo docker exec docker2 ip a
```

```bash
# For VM2
# create a docker image from the docker file 
sudo docker build . -t ubuntu-docker

# create containers from the created image; Containers not connected to any network
sudo docker run -d --net=none --name docker3 ubuntu-docker
sudo docker run -d --net=none --name docker4 ubuntu-docker

# check container status and ip 
sudo docker ps
sudo docker exec docker3 ip a
sudo docker exec docker4 ip a
```

**Now assign the static IP address to the containers using `ovs-docker` utility. also ping the GW to test the connectivity.**

```bash
# For VM1
# add ip address to the container using ovs-docker utility 
sudo ovs-docker add-port ovs-br0 eth0 docker1 --ipaddress=192.168.1.11/24 --gateway=192.168.1.1
sudo docker exec docker1 ip a

sudo ovs-docker add-port ovs-br1 eth0 docker2 --ipaddress=192.168.2.11/24 --gateway=192.168.2.1
sudo docker exec docker2 ip a

# ping the gateway to check if container connected to ovs-bridges
sudo docker exec docker1 ping 192.168.1.1
sudo docker exec docker2 ping 192.168.2.1
```

```bash
# For VM2
# add ip address to the container using ovs-docker utility 
sudo ovs-docker add-port ovs-br0 eth0 docker3 --ipaddress=192.168.1.11/24 --gateway=192.168.1.1
sudo docker exec docker3 ip a

sudo ovs-docker add-port ovs-br1 eth0 docker4 --ipaddress=192.168.2.11/24 --gateway=192.168.2.1
sudo docker exec docker4 ip a

# ping the gateway to check if container connected to ovs-bridges
sudo docker exec docker3 ping 192.168.1.1
sudo docker exec docker4 ping 192.168.2.1
```

**Step 3: Now we are going to establish the VXLAN TUNNELING between the two VM. Most importantly the vxlan ID or VNI and udp port 4789 is important. Also we have to configure the remote IP which is opposite VM IP.**
```bash
# For VM1
# one thing to check; as vxlan communicate using udp port 4789, check the current status
netstat -ntulp

# Create the vxlan tunnel using ovs vxlan feature for both bridges to another hosts bridges
# make sure remote IP and key options; they are important
sudo ovs-vsctl add-port ovs-br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.0.1.169 options:key=1000
# key is VNI
# vxlan0 is the interface/port name
# type is vxlan which also configures udp port 4789 default
sudo ovs-vsctl add-port ovs-br1 vxlan1 -- set interface vxlan1 type=vxlan options:remote_ip=10.0.1.169 options:key=2000

# check the port again; it should be listening
netstat -ntulp | grep 4789

sudo ovs-vsctl show

ip a
```
```bash
# For VM 2
# one thing to check; as vxlan communicate using udp port 4789, check the current status
netstat -ntulp

# Create the vxlan tunnel using ovs vxlan feature for both bridges to another hosts bridges
# make sure remote IP and key options; they are important
sudo ovs-vsctl add-port ovs-br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=10.0.1.43 options:key=1000
sudo ovs-vsctl add-port ovs-br1 vxlan1 -- set interface vxlan1 type=vxlan options:remote_ip=10.0.1.43 options:key=2000

# check the port again; it should be listening
netstat -ntulp | grep 4789

sudo ovs-vsctl show

ip a

```

**Now test the connectivity and see the magic!**

```bash
# FROM docker1
# will get ping 
sudo docker exec docker1 ping 192.168.1.12
sudo docker exec docker1 ping 192.168.1.11

# will be failed
sudo docker exec docker1 ping 192.168.2.11
sudo docker exec docker1 ping 192.168.2.12

# FROM docker2
# will get ping 
sudo docker exec docker2 ping 192.168.2.11
sudo docker exec docker2 ping 192.168.2.12

# will be failed
sudo docker exec docker2 ping 192.168.1.11
sudo docker exec docker2 ping 192.168.1.12
```

#### The VXLAN TUNNELING is working. We can reach the other host's docker conatiner with same VNI. That's awesome!!!

**Step 4: But we can communicate between containers with same VNI, but can't reach the outer world. Let's fix that by adding some iptables rules for NATing.**

_Note: Not going to describe all the commands here cause it's not our main focus. Go to this [link](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/4/html/security_guide/s1-firewall-ipt-fwd) for details: _

```bash
# NAT Conncetivity for recahing the internet

sudo cat /proc/sys/net/ipv4/ip_forward

# enabling ip forwarding by change value 0 to 1
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
sudo cat /proc/sys/net/ipv4/ip_forward

sudo iptables -t nat -L -n -v

sudo iptables --append FORWARD --in-interface veth0 --jump ACCEPT
sudo iptables --append FORWARD --out-interface veth0 --jump ACCEPT
sudo iptables --table nat --append POSTROUTING --source 192.168.1.0/24 --jump MASQUERADE
```

```bash
# ping the outer world, should not reach the internet
ping 1.1.1.1 -c 2

# Now let's make NAT Conncetivity for recahing the internet

sudo cat /proc/sys/net/ipv4/ip_forward

# enabling ip forwarding by change value 0 to 1
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
sudo cat /proc/sys/net/ipv4/ip_forward

# see the rules
sudo iptables -t nat -L -n -v

sudo iptables --append FORWARD --in-interface veth1 --jump ACCEPT
sudo iptables --append FORWARD --out-interface veth1 --jump ACCEPT
sudo iptables --table nat --append POSTROUTING --source 192.168.2.0/24 --jump MASQUERADE

# ping the outer world now, should be working now
ping 1.1.1.1 -c 2
```

#### Now the hands on completed;
