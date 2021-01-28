# Farmers network setup.

So you have become a farmer. And you have bought some shiny new nodes to host the Threefold Grid and provide for services in that grid.

Grid means network. The network is the computer. 

There are a few ways to connnect a node to a grid, one of them consisting in merely connecting a node to your house network, which will not be discussed here.

this document surmises you will connect several nodes on a network, separate from other networks in your location, solely for the purpose to add nodes to the grid in a more professional setup, where multiple nodes live on the same setup

## Necessities

What you'll need.

  - a link to the Internet with appropriate bandwith for the services you want to provide
  - a router that connects your internal network with the Internet Uplink
  - one or more switches in function of the number of nodes you need to connect
  - some knowledge about how to configure your switch and router

## What it's going to look like

### physical layout

```
+                                          +
|  Internet link           optional Internet link
|                                          |
|          +-------------------+           |
|          |  Router           +-----------+
+----------+  NAT FW           |
           |                   +-----------+
+-------+  +-+-+---------------+ mgmt      |
|DHCP   |    | |                           |
|server |    | |Vlan trunk                 |
|       |    | |                         +-+--+
+---+---+  +-+-+-----+---------+         |OOB |
    |      | Switch  |         |         |    |
    +------+  vlan   | vlan    +---------+    | OOB Internet
           |         |         | mgmt    |    +---------+
           +-+-------+---+-----+         |    | access
             |           |               |    |
             |           |               +-+--+
      NIC 1  |           |  NIC 2          |
           +-+-----------+-----+           |
           | NODES             +-+         |
           |                   | +-+       |
           |                   | | +-+     |
           +-------------------+ | | |     |
            +--------------------+ | +-----+
             +---------------------+ | IPMI
              +----------------------+

```
The setup here is drawn without Highly available routers or multi-switch link aggregation, thus fairly simple.  

The OOB (out-of-band) physical network is optional and can be handled with the logical setup too. The OOB network would be typically used to make sure the infrastructure cannot be accessed in any way except by the admins. That segment can of course also be a vlan on the router/switch too.  

The optional Internet link drawn here is for when a farmer has possibility to have nultiple uplinks. This will not be discussed here as we suppose a farmer has the necessary knowledge in house to properly configure BGP and routing in case of such a setup.

### logical layout

In order for nodes to properly boot, they need to download and start 0-OS: 
  - with a bios configured to boot from an USB stick
  - with a bios configured to boot from the network

Either way, a node needs to obtain an IP address and access the Internet to be able to charge 0-OS and boot it.

To give a node an IP address you'll need to setup a DHCP server that provides for an IP address to one of the NICs of the nodes.

Each NIC you use in the nodes (typically 2) needs to live in it's own segment. That means they live on a separate VLAN, or on a separate switch. NIC2 is then used by 0-OS as customer workload backend, while NIC1 is used by 0-OS to do management tasks and interact with the management part of th grid.

NIC2 can thus live on a different swith, e.g. an 10GBit switch, provided that there is such a need.

So: Given a small environment that can live on a single switch, and when comfortable with vlans, you configure the switch with 3 VLANs (public,management,oob), essentially mimicing separate switches.

  - The public vlan.  
  That is the network that will carry:
    - an IPv6 prefix that is global unicast (real internet)
    - an IPv4 subnet from which addresses can be allocated to node `public_config`s and to Kubernetes vms
  - The management vlan will carry:
    - an optional IPv6 prefix (can be ULA or global unicast)
    - an RFC1918 subnet containing a DHCP server that provides ip addresses for that subnet
    - a default gateway (that typically does NAT)
    - An optional DNS server (the dhscp server can be configured to provide dns entries outside of that network e.g. `8.8.8.8,1.1.1.1`)
  - The OOB network will typically carry traffic for managing infrastructure, like the managament ports of switches and router, as well as the IPMI ports of nodes if they have these. Mostly these things are configures statically (in bios), but if not, you'll need to provide DHCP in that sement as well.

At threefold, we configure the DHCP server to provide specific IP addresses to nodes in function of their MAC address on the management as wel as on the OOB network, so that we can be sure which physical node receives which IP address. That makes it easier to physically locate them. Also, we can then configure the bios IPMI the same for all nodes, not needing a static config. A small drawback is thet we have to recense all the mac addresses and confiure the DHCP server prior to booting them, but as we need to access all bios anyway, we can do that while racking and configuring the bios.

