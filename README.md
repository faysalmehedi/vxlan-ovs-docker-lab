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

![Project Diagram](https://github.com/faayam/vxlan-ovs-docker-lab/blob/main/vxlan_PacketWalk.png)

_the diagrom is taken from networkdirection blog_

- A frame arrives on a switch port from a host. This port is a regular untagged (access) port, which assigns a VLAN to the traffic - The switch determines that the frame needs to be forwarded to another location. The remote switch is connected by an IP network It may be close or many hops away.
- The VLAN is associated with a VNI, so a VxLAN header is applied. The VTEP encapsulates the traffic in UDP and IP headers. UDP port 4789 is used as the destination port. The traffic is sent over the IP network 
- The remote switch receives the packet and decapsulates it. A regular layer-2 frame with a VLAN ID is left 
- The switch selects an egress port to send the frame out. This is based on normal MAC lookups. The rest of the process is as normal.

### Get an overview of the hands-on from the diagram below
![Project Diagram](https://github.com/faayam/vxlan-ovs-docker-lab/blob/main/vxlan-demo.png)