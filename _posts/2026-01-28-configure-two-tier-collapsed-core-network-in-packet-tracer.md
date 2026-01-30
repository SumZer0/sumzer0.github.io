---
title: Configure Two-tier Collapsed Core Network in Cisco Packet Tracer
date: 2026-01-28 12:11:00 +0100
categories: [network]
tags: [cisco,dmz,firewall]
comments: false
layout: post
---

In this post, I'll configure a Two-Tier Campus Network (also known as _collapsed core network_) using Cisco Packet Tracer, as opposed to the Three-Tier Network Hierarchy. I will then configure internet access and a DMZ, managing access control via the Cisco ASA Firewall.

## Tiered Network Hierarchies
Cisco designed the Three-Tier Network Hierarchy model to be used in corporate networks in order to break down complex network designs into smaller sections.

The Three-Tier Network Hierarchy is composed by the following layers:
- **Access Layer**: allows end-devices to connect to the network. This layer is implemented using structured cabling and WAPs that are both ultimately connected to _access switches_. There is no direct link between switches at this layer.
- **Distribution Layer**: provides fault-tolerant interconnections between different _access blocks_ and either the _core_ or other _distribution blocks_. Each _access switch_ has full/partial mesh links to each router or layer 3 switch in its distribution layer block.
- **Core Layer**: provides an highly available network backbone by providing redundant traffic paths for data to flow around the access and distribution layers of the network. Routers or layer 3 switches in the core layer establish a full mesh topology with switches in distribution layer blocks.

On the other side, a _Collapsed Core Network Hierarchy_ is a network topology where the Core and Distribution layers are "collapsed" into a single distribution-core layer. This topology is suited for smaller businesses that only have one office, as the core layer is usually used as a backbone to connect multiple branch offices together.

## LAN Configuration
The network will have three L2 access switches at the access layer and two L3 distribution switches at the distribution layer. Each access switch will be dual-homed to both distribution switches and the two distribution switches will be connected together via a two port L3 etherchannel to provide additional fault tolerance and to avoid over-complicating the STP configuration.

To increase network redundancy and fault-tolerance, Hot Standby Routing Protocol (HSRP) will be configured on the distribution switches, allowing default gateway redundancy for each vlan.

End devices such as employees computers and internal servers will be connected to the access switches. Also, a simple Wireless Access Point will be configured and connected to an access switch to allow wireless devices to connect to the network.

The internal network will be divided in four different VLANs, to segment the network based on host roles and to simplify network management. These VLANs will be configured as follows:

|VLAN ID|Name|Subnet|
|:-|:-|:-|
|10|sales|192.168.10.0/24|
|20|finance|192.168.20.0/24|
|30|servers|192.168.30.0/24|
|40|wlan|192.168.40.0/24|

The following three internal servers will be deployed in the Servers VLAN:

|Server|IP Address|Purpose|
|:-|:-|:-|
|DHCP Server|192.168.30.4|Provides an address pool for each VLAN|
|DNS Server|192.168.30.5|Name resolution for internal hosts|
|FILE01 Server|192.168.30.6|Centrally manage business-related files|


The final LAN topology can be seen below:

![lan_final](/assets/img/post/packet_tracer_blog/lan_final.png)

### Configure Access Switches
For the Access Layer we will use three Cisco Catalyst 2960 switches, named respectively ACCESS01, ACCESS02, ACCESS03. The image below shows these switches in packet tracer:

![access_switches](/assets/img/post/packet_tracer_blog/access_switches.png)

The following configuration is shared amongst all access switches, minus the hostname command:
```bash
# Varies based on what switch you are configuring
hostname ACCESS01

# Create VLANs and assign names
vlan 10
 name sales
vlan 20
 name finance
vlan 30
 name servers
vlan 40
 name wlan

# Create the trunks that will connect to distribution switches
int range Gi0/1-2
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40
```

Both ACCESS01 and ACCESS02 switches will have dedicated ports for VLAN 10 and VLAN 20:
```bash
int range Fa0/1-5
 switchport access vlan 10
 switchport mode access
int range Fa0/6-10
 switchport access vlan 20
 switchport mode access
```

The Wireless AP mentioned earlier will be connected to ACCESS01, so VLAN 40 will be configured:
```bash
int Fa0/11
 switchport access vlan 40
 switchport mode access
```

ACCESS03 is only used to connect internal servers, so ports will be configured only for VLAN 40:
```bash
int range Fa0/1-5
 switchport access vlan 30
 switchport mode access
```

### Configure Distribution Switches
Now that all the Access Switches have been configured, we can start by setting up the Distribution Layer by using two Cisco Catalyst 3650 L3 switches, named respectively DIST01 and DIST02:

![distribution_switches](/assets/img/post/packet_tracer_blog/distribution_switches.png)

These two distribution switches will be configured as the primary and secondary STP root switches for all the custom VLANs. They will also be responsible for inter-VLAN routing.

An L3 etherchannel composed by two routed switch ports will be configured to allow Layer3 connectivity between both distribution switches, useful in case a downlink fails. Also, an IP helper address will be configured to each VLAN interface pointing to the DHCP Server IP (192.168.30.4), to allow the relay of DHCP related traffic across VLANs.

As explained earlier, HSRP will be configured on each VLAN interface and on both distribution switches to provide first-hop redundancy to network hosts. Using HSRP we can configure a shared Virtual IP (VIP) that will map to the underlying real IP of the currently HSRP active L3 switch.

VLAN interfaces and HSRP will be configured as follows:

|VLAN ID|DIST01 IP|DIST02 IP|Virtual IP|
|:-|:-|:-|:-|
|10|192.168.10.2|192.168.40.3|192.168.10.1|
|20|192.168.20.2|192.168.40.3|192.168.20.1|
|30|192.168.30.2|192.168.40.3|192.168.30.1|
|40|192.168.40.2|192.168.40.3|192.168.40.1|

The following configuration is shared amongst all distribution switches, minus the hostname command:
```bash
# Varies based on which switch you are configuring
hostname DIST01

# Enable Layer3 capabilities
ip routing

# Create VLANs and assign names
vlan 10
 name sales
vlan 20
 name finance
vlan 30
 name servers
vlan 40
 name wlan

# Configure Switch Routed Ports for L3 etherchannel
int range Gi1/0/23-24
 no switchport
 no ip address
 channel-group 1 mode on
```

The DIST01 distribution switch will be the STP root for all VLANs and will also have an higher HSRP priority value (105) than DIST02, making it the active router in the configuration:
```bash
# Lowest priority to become STP primary root
spanning-tree vlan 10,20,30,40 priority 4096

# L3 Etherchannel IP address
int Port-channel1
 ip address 192.168.99.1 255.255.255.252

interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 ip helper-address 192.168.30.4
 standby 10 ip 192.168.10.1
 standby 10 priority 105
 standby 10 preempt

interface Vlan20
 ip address 192.168.20.2 255.255.255.0
 ip helper-address 192.168.30.4
 standby 20 ip 192.168.20.1
 standby 20 priority 105
 standby 20 preempt

interface Vlan30
 ip address 192.168.30.2 255.255.255.0
 ip helper-address 192.168.30.4
 standby 30 ip 192.168.30.1
 standby 30 priority 105
 standby 30 preempt

interface Vlan40
 ip address 192.168.40.2 255.255.255.0
 ip helper-address 192.168.30.4
 standby 40 ip 192.168.40.1
 standby 40 priority 105
 standby 40 preempt
```

The DIST02 distribution switch will be the secondary STP root, ready to takeover in case DIST01 fails. It will also be configured with a lower HSRP priority (default 100) to become the standby HSRP router:
```bash
# Second-lowest priority to become STP secondary root
spanning-tree vlan 10,20,30,40 priority 8192

# L3 etherchannel IP address
interface Port-channel1
 no switchport
 ip address 192.168.99.2 255.255.255.252

interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 ip helper-address 192.168.30.4
 standby 10 ip 192.168.10.1
 standby 10 preempt

interface Vlan20
 ip address 192.168.20.3 255.255.255.0
 ip helper-address 192.168.30.4
 standby 20 ip 192.168.20.1
 standby 20 preempt

interface Vlan30
 ip address 192.168.30.3 255.255.255.0
 ip helper-address 192.168.30.4
 standby 30 ip 192.168.30.1
 standby 30 preempt

interface Vlan40
 ip address 192.168.40.3 255.255.255.0
 ip helper-address 192.168.30.4
 standby 40 ip 192.168.40.1
 standby 40 preempt
```

Now that both access and distribution layer switches are configured, we can connect them by connecting each access switch to both distribution switches using the two trunk ports and the distribution switches together using the routed ports configured in the L3 etherchannel:

![connected_layers](/assets/img/post/packet_tracer_blog/connected_layers.png)

### Configure Internal Servers
Now that the switches are configured, we can add our three internal servers, connect them to ACCESS03 access switch (on VLAN30 ports), and configure their static IP address and default gateway (192.168.30.1):

![internal_servers](/assets/img/post/packet_tracer_blog/internal_servers.png)

The FTP Server can be left unconfigured, as the default cisco:cisco user and his access rights are enough for our testing playground.

The DNS Server will be configured with A records that point to the three internal servers:

![internal_dns_1](/assets/img/post/packet_tracer_blog/internal_dns_1.png)

Finally, on the DHCP Server we will create an address pool for each VLAN, assigning the correct default gateway, DNS Server, and address range:

|Pool Name|Default Gateway|DNS Server|Start IP Address|Max User|
|:-|:-|:-|:-|:-|
|VLAN10|192.168.10.1|192.168.30.5|192.168.10.50/24|50|
|VLAN20|192.168.20.1|192.168.30.5|192.168.20.50/24|50|
|VLAN30|192.168.30.1|192.168.30.5|192.168.30.50/24|50|
|VLAN40|192.168.40.1|192.168.30.5|192.168.40.50/24|50|

An example of an address pool configuration for VLAN10 can be found below:

![dhcp_config](/assets/img/post/packet_tracer_blog/dhcp_config.png)

### Connect Clients
Now that the switches and internal servers are configured, we can start connecting client hosts to the ACCESS01 and ACCESS02 switches, making sure to connect them to the correct VLAN enabled port.

> Remember to enable DHCP on client hosts to allow them to contact the DHCP server.
{: .prompt-info }

The only restriction is the placement of the Wireless AP (AP-PT), which must be connected to port Fa0/11 on the ACCESS01 switch, since it's the port configured for VLAN40 (WLAN). The Access Point will then be configured with WPA2-PSK authentication and the passphrase of "Cisco123" as shown below:

![wap_config](/assets/img/post/packet_tracer_blog/wap_config.png)

Now that the Access Point is configured, you can connect some wireless-enabled clients to it:

![wireless_connection](/assets/img/post/packet_tracer_blog/wireless_connection.png)

### Test Connectivity and Fault Tolerance
We can finish this section by testing connectivity between the various client hosts and internal servers, as well as trying to resolve internal names and connect to the FTP Server.

Due to the redundancy implemented in the topology design, the network is able to tolerate these faults:
- If one access switch uplink fails (specifically the one pointing to the STP root distribution switch), the packets will be forwarded to the other distribution switch, no loops will happen due to STP.
- If the STP root distribution switch fails, the secondary root will take over and continue to manage traffic correctly.

### Internet Connectivity Configuration
Now that the LAN topology is implemented and fully working, we can move on to provide WAN connectivity and to create a DMZ for publicly accessible servers. Access control between the internal network, DMZ, and the WAN (internet) will be managed by the Cisco ASA 5506-X stateful firewall.

Due to Packet Tracer limitations, especially in terms of ASA configuration, the distribution switches will connect to a router that will perform NAT Overloading (PAT) for all the internal VLAN subnets. This is needed as the Packet Tracer version of the Cisco ASA 5506-X firewall is only able to apply NAT rules to directly connected subnets (for more information see [here](https://community.cisco.com/t5/routing/packet-tracer-asa-nat-problem/td-p/3936024) and [here](https://learningnetwork.cisco.com/s/question/0D56e0000DeyWTkCQM/packet-tracer-cisco-asa5506x-pat-not-working)).

Cisco ASA Firewalls, as many other firewalls appliances, allow the administrator to configure a _security level_ to every interface. By default, ASA allows traffic from higher to lower security levels, while denying traffic from a lower to an higher security level. These rules can be overridden via ACLs and Access Rules.

The three zones that will be managed by the firewall are the following:

|Zone|Subnet|Security Level|
|:-|:-|:-|
|Inside|10.0.3.0/30|100|
|DMZ|192.168.50.0/24|50|
|Outside|200.74.148.0/29|0|

Where `10.0.3.0/30` is the transit subnet between the internal router and the firewall, where the router will perform PAT on its interface (10.0.3.1) for all the internal VLANs. Also, `200.73.148.0/29` is the external subnet of public IPs used to connect to the emulated internet. For this demonstration, we have been assigned five public IPs to connect to the internet, allowing us to expose DMZ servers using Static NAT.

The final network topology will be the following:

![final_topology](/assets/img/post/packet_tracer_blog/final_topology.png)

#### Internet Emulation
To emulate internet connectivity, we will simulate an ISP connection and an external network that will be used to test firewall rules and network reachability:

![internet](/assets/img/post/packet_tracer_blog/internet.png)

The network `192.168.56.0/24` is connected to the "internet" via the `200.73.149.0/24` transit subnet. The network's edge router will also apply the following NAT rules:

|Original Address|Translated Address|NAT Type|
|:-|:-|:-|
|192.168.56.10|200.73.149.2|NAT Overload|
|192.168.56.11|200.73.149.10|Static NAT|

This allows the client host (192.168.56.10) to connect to the internet and get responses back, and will also expose the web server (192.168.56.11) to a public address via Static NAT (200.73.149.2).

To achieve this configuration, the edge Cisco 2911 router needs the following configuration:
```bash
# Configure internal interface
interface Gi0/0
 ip address 192.168.56.1 255.255.255.0
 ip nat inside
 no shutdown

# Configure internet facing interface
interface Gi0/1
 ip address 200.73.149.2 255.255.255.0
 ip nat outside
 no shutdown

# Configure static NAT to expose the Web Server
ip nat inside source static 192.168.56.11 200.73.149.10

# Configure NAT Overload for the client host
access-list 1 permit 192.168.56.10
ip nat inside source list 1 interface Gi0/1 overload

# Default route to ISP router
ip route 0.0.0.0 0.0.0.0 200.73.149.1
```

If we expand the Internet Cluster shown in the image above, we can see that the "internet" is composed by a single router that is directly connected to the external network configured before and our edge firewall. Additionally, this router is also connected to a Public DNS server (200.73.150.10):

![isp_topology](/assets/img/post/packet_tracer_blog/isp_topology.png)

The ISP router needs to be configured as follows:
```bash
#Interface connected to our network
interface Gi0/0
 ip address 200.73.148.1 255.255.255.248
 no shutdown

# Interface connected to the external network
interface Gi0/1
 ip address 200.73.149.1 255.255.255.0
 no shutdown

# Interface connected to the Public DNS Server
interface Gi0/2
 ip address 200.73.150.1 255.255.255.0
 no shutdown
```

For now, the Public DNS Server can be configured with an A record pointing to the exposed external network web server:

![public_dns_1](/assets/img/post/packet_tracer_blog/isp_topology.png)

#### Connect LAN to Internet
Now that the emulated internet is configured, we need to connect our collapsed-core network to it. To do so we first need to connect our distribution switches to a router for NAT Overload in order to avoid Packet Traces limitations as explained in the beginning of this section.

The following three transit networks will be used for these connections:

|Network|Description|
|:-|:-|
|10.0.1.0/30|Connect DIST01 to Router|
|10.0.2.0/30|Connect DIST02 to Router|
|10.0.3.0/30|Connect Router to Edge Firewall|

The final internet connection topology can be seen below:

![lan_to_wan](/assets/img/post/packet_tracer_blog/lan_to_wan.png)

We can start by configuring a routed port on DIST01 and the required routes:
```bash
# Interface connected to Router
interface Gi1/0/22
 no switchport
 ip address 10.0.1.2 255.255.255.252

# Default gateway to reach the router
ip route 0.0.0.0 0.0.0.0 10.0.1.1 

# Reach the router through L3 etherchannel if router uplink fails
ip route 0.0.0.0 0.0.0.0 10.0.99.2 130
```

Next, we can configure DIST02 in a similar way:
```bash
# Interface connected to Router
interface Gi1/0/22
 no switchport
 ip address 10.0.2.2 255.255.255.252
 
# Default gateway to reach the router
ip route 0.0.0.0 0.0.0.0 10.0.2.1

# Reach the router through L3 etherchannel if router uplink fails
ip route 0.0.0.0 0.0.0.0 10.0.99.1 130
```

We then need to configure HSRP interface tracking for the interface connected to the Router on both distribution switches and for all VLANs. Interface tracking will decrease the HSRP priority value by 10 if the tracked interface goes down. This is helpful in the scenario where the HSRP active L3 switch loses connectivity to the Router, allowing the standby L3 switch to become the active one (assuming that it has a working Router uplink).

Interface tracking can be configured on both distribution switches as follows:
```bash
interface Vlan10
 standby 10 track Gi1/0/22
interface Vlan20
 standby 20 track Gi1/0/22
interface Vlan30
 standby 30 track Gi1/0/22
interface Vlan40
 standby 40 track Gi1/0/22
```

Now that both distribution switches are configured, we need to configure the Router:
```bash
# Interface connected to the edge firewall
interface Gi0/0
 ip address 10.0.3.2 255.255.255.252
 ip nat outside
 no shutdown

# Interface connected to DIST01
interface Gi0/1
 ip address 10.0.1.1 255.255.255.252
 ip nat inside
 no shutdown

# Interface connected to DIST02
interface Gi0/2
 ip address 10.0.2.1 255.255.255.252
 ip nat inside
 no shutdown

# Configure NAT Overload for all VLAN subnets
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
access-list 1 permit 192.168.30.0 0.0.0.255
access-list 1 permit 192.168.40.0 0.0.0.255
ip nat inside source list 1 interface Gi0/0 overload

# Default route to the Edge Firewall
ip route 0.0.0.0 0.0.0.0 10.0.3.1

# Preferred routes to VLANs using DIST01
ip route 192.168.10.0 255.255.255.0 10.0.1.2
ip route 192.168.20.0 255.255.255.0 10.0.1.2
ip route 192.168.30.0 255.255.255.0 10.0.1.2
ip route 192.168.40.0 255.255.255.0 10.0.1.2

# Alternate routes to VLANs using DIST02
ip route 192.168.10.0 255.255.255.0 10.0.2.2 130
ip route 192.168.20.0 255.255.255.0 10.0.2.2 130
ip route 192.168.30.0 255.255.255.0 10.0.2.2 130
ip route 192.168.40.0 255.255.255.0 10.0.2.2 130
```

The next step is the configuration of the Edge ASA 5506-X Stateful Firewall by setting the correct security level to each zone and configuring NAT Overload: 
```bash
# Interface that connects to the internal network
interface Gi1/1
 nameif inside
 security-level 100
 ip address 10.0.3.1 255.255.255.252
 no shutdown

# Interface that connects to the ISP
interface Gi1/3
 nameif outside
 security-level 0
 ip address 200.73.148.2 255.255.255.248
 no shutdown

# Configure NAT Overload for inside-to-outside communications
object network INSIDE-NET
 subnet 10.0.3.0 255.255.255.252
 nat (inside,outside) dynamic interface

# Default route to the ISP router
route outside 0.0.0.0 0.0.0.0 200.73.148.1
```

In real world ASA appliances, the stateful inspection of common network protocols is enabled by default, allowing traffic to come back from an interface with a lower security level to one with an higher security level for already established connections.

The simulated ASA 5506-X appliance comes by default with an inspection policy for some protocols, however, due to a Packet Tracer limitation ([more info](https://community.cisco.com/t5/routing/packet-tracer-cisco-5506-asa-firewall-not-able-to-ping-icmp/m-p/4744814/highlight/true#M376555)) we need to remove the default configuration and manually configure the protcol to inspect:
```bash
# Remove default Service Policy and Policy Map
no service-policy global_policy global
no policy-map global_policy

# Configure Policy Map (global_policy) to inspect dns,http,icmp traffic
policy-map global_policy
 class inspection_default
  inspect dns
  inspect http
  inspect icmp

# Create Service Policy using the defined Policy Map (global_policy)
service-policy global_policy global
```

Everything is now configured and LAN hosts can successfully connect to internet resources using the specified protocols for inspection. However, we still need to configure DNS Forwarding to our Internal DNS Server to make it use the Public DNS Server when it doesn't find a match in its local records.

To configure DNS Forwarding in Packet Tracer, we need to create an NS record with the name `.`, in order to match all TLDs and point to a server name, which in our case will be _root_. The final step is to configure an A record for the server name specified earlier, pointing to the Public DNS Server address:

![internal_dns_2](/assets/img/post/packet_tracer_blog/internal_dns_2.png)

#### Configure DMZ
The final step of the network that is missing is the Demilitarized Zone (DMZ), or Screened Subnet as it's called nowadays. For this example we will configure a very simple DMZ using the 192.168.50.0/24 network and containing only two web servers. Additionally, Static NAT rules to expose both these web servers to the internet will be configured on the firewall as follows:

|Server|Original IP|Translated IP|
|:-|:-|:-|
|WEB01|192.168.50.10|200.73.148.3|
|WEB02|192.168.50.11|200.73.148.4|

Since those server must be accessible from the internet and thus from an interface with a lower security level, we will also need to configure specific inbound ACLs on the outside interface of the firewall to explicitly allow web (TCP port 80) and ICMP traffic.

The final DMZ topology can be seen below:

![dmz_topology](/assets/img/post/packet_tracer_blog/dmz_topology.png)

To complete the configuration we need add these settings to the Edge Firewall:
```bash
# Configure DMZ interface
interface GigabitEthernet1/2
 nameif dmz
 security-level 50
 ip address 192.168.50.1 255.255.255.0
 no shutdown

# Static NAT for WEB01
object network DMZ-WEB01
 host 192.168.50.10
 nat (dmz,outside) static 200.73.148.3

# Static NAT for WEB02
object network DMZ-WEB02
 host 192.168.50.11
 nat (dmz,outside) static 200.73.148.4

# ACLs to allow internet clients to connect to DMZ Servers via TCP port 80 and ICMP
access-list OUTSIDE-IN extended permit tcp any host 192.168.50.10 eq www
access-list OUTSIDE-IN extended permit tcp any host 192.168.50.11 eq www
access-list OUTSIDE-IN extended permit icmp any host 192.168.50.10
access-list OUTSIDE-IN extended permit icmp any host 192.168.50.11

# Apply the ACL as an inbound rule to the outside interface
access-group OUTSIDE-IN in interface outside
```

Now the entire network is configured, but we still need to make some minor updates to both the Internal and Public DNS Servers. We can start by adding internal names to the DMZ Servers on the Internal DNS:

![internal_dns_3](/assets/img/post/packet_tracer_blog/internal_dns_3.png)

Then, we need to add entries on the Public DNS Servers pointing to the public NAT-ed addresses:

![public_dns_2](/assets/img/post/packet_tracer_blog/public_dns_2.png)

### Final Thoughts
I hope that this article has been helpful to you and in the future I will probably improve this network by adding router and firewall redundancy as well as improving the DMZ topology. However I will not use Packet Tracer as it has, as you have seen, very strict limitations on what you can do, especially in terms of firewalls. For this reason, I will use [GNS3](https://www.gns3.com/) for future network configurations.