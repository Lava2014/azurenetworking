---
Title: "Designing Juniper SD-WAN Integration in Azure using Hub-Spoke Architecture"
Date: 2026-05-11
Categories: [Azure, Networking, SD-WAN]
Tags: [Azure, SDWAN, Juniper, HubSpoke, AzureFirewall, ExpressRoute]
---

![Architecture Diagram](/assets/sdwan3.png)

# Designing Juniper SD-WAN Integration in Azure using a Traditional Hub-Spoke Architecture

**Introduction**

There are multiple networking patterns available for integrating SD-WAN with Microsoft Azure. Most enterprise deployments today leverage integrated Network Virtual Appliances (NVAs) deployed directly into Azure Virtual WAN hubs through jointly managed solutions provided by Microsoft Azure and third-party SD-WAN vendors. In these architectures, routing and connectivity are largely abstracted and operational complexity is significantly reduced.

However, some enterprise environments require a more traditional self-managed Hub-Spoke architecture to retain full control over routing decisions, firewall inspection, traffic engineering, and hybrid connectivity.

This article explores a real-world Azure networking design for deploying Juniper SD-WAN in a traditional Hub-Spoke architecture while maintaining:

•	Full routing control

•	Centralized Azure Firewall inspection

•	High availability across Availability Zones

•	Hybrid datacenter connectivity through ExpressRoute and VPN gateways

•	Symmetric routing across SD-WAN and Azure workloads

The design also addresses several Azure and SD-WAN integration constraints commonly encountered in enterprise environments.

________________________________________
**Architecture Requirements**

**Requirement 1 — Highly Available SD-WAN Infrastructure**

The solution required a highly available SD-WAN deployment capable of tolerating:

•	Single VM failures

•	Availability Zone failures

•	Network appliance failover scenarios

The objective was to increase resiliency of the SD-WAN hubs while ensuring uninterrupted branch connectivity.

Constraint 1 — Azure Load Balancer Probe Limitation

A key challenge involved Juniper SD-WAN NVAs not responding to Azure Load Balancer health probes. This behavior is expected because Juniper SD-WAN overlay communication is authenticated and secured by design. The appliances do not expose generic probe listeners required by Azure Standard Load Balancer health checks.

As a result:

•	Azure Load Balancer could not reliably determine backend health

•	Traditional Active/Active or Active/Standby load balancer patterns were not feasible
________________________________________
**Requirement 2 — Mandatory Azure Firewall Inspection**

Another key requirement was ensuring that all traffic traversing between:

•	Azure workloads

•	On-premises datacenters

•	SD-WAN branches must be inspected through Azure Firewall.

This introduced additional routing complexity because traffic needed to remain symmetric while still traversing centralized security inspection points.

Constraint 2 — Asymmetric Routing Challenges

Asymmetric routing is a common issue in SD-WAN and firewall-integrated architectures.

Challenges included:

•	Return traffic bypassing Azure Firewall

•	Session state inconsistencies

•	Active/Active routing conflicts

•	Firewall session drops

Maintaining symmetric traffic flow across SD-WAN hubs, Azure Firewall, spokes, and hybrid connectivity became a critical design consideration.
________________________________________

**Requirement 3 — Hybrid Datacenter Connectivity**

The environment also included existing datacenter connectivity through:

•	ExpressRoute Gateways

•	VPN Gateways

Traffic between datacenters and SD-WAN-connected branches also required mandatory Azure Firewall inspection.

Constraint 3 — Symmetric Routing across ExpressRoute and VPN

Traffic entering Azure from datacenters through ExpressRoute or VPN gateways needed to:

•	traverse Azure Firewall

•	maintain symmetric return paths

•	avoid bypass routing scenarios

This significantly increased routing complexity across the hub network.
________________________________________

**Solution Design**

**1.High Availability Architecture for SD-WAN Hubs**

To address the Azure Load Balancer limitation, the design deployed:

•	Two Juniper SD-WAN NVAs per hub

•	Each appliance deployed in separate Availability Zones

•	Dedicated static Public IP addresses assigned per appliance

This architecture allowed SD-WAN spokes to establish direct overlay connectivity to each SD-WAN hub independently.

Failover between SD-WAN hubs was handled natively through the SD-WAN overlay using:

•	BGP communities

•	Dynamic path selection

•	Overlay route preference

This eliminated dependency on Azure Load Balancer for inbound SD-WAN connectivity.
________________________________________
**2.Azure Route Server Integration**

Azure Route Server (ARS) was introduced to enable dynamic route exchange between:

•	Juniper SD-WAN hubs

•	Azure Firewall

•	ExpressRoute Gateways

•	VPN Gateways

BGP peering was established between Azure Route Server and the SD-WAN hubs.
This allowed dynamically learned SD-WAN prefixes to propagate automatically across the Azure network fabric.
________________________________________
**3.Routing Design**

The architecture used a hybrid routing model combining:

•	Dynamic route propagation using Azure Route Server

•	Static route enforcement using User Defined Routes (UDRs)

This semi-dynamic design provided greater routing control while maintaining scalability.

**3.1.Routing Azure Firewall to SD-WAN (Dynamic via Route Server)**

Specific branch prefixes needed to be routed from Azure Firewall toward dedicated SD-WAN hubs.
The SD-WAN appliances advertised branch routes through BGP to Azure Route Server.
Azure Route Server then automatically programmed these routes onto the Azure Firewall subnet NICs.

Benefits included:

•	Dynamic route learning

•	Reduced administrative overhead

•	Simplified SD-WAN branch onboarding

**3.2.Routing SD-WAN to Azure Firewall (Static via UDRs)**

Traffic originating from the SD-WAN private subnet toward:

•	Azure spoke VNets

•	On-premises environments was statically routed toward Azure Firewall using User Defined Routes.

This ensured all east-west and north-south traffic remained inspected. One design limitation encountered was Azure’s UDR limit of 400 routes per route table.Proper route summarization and segmentation became important scalability considerations.

**3.3.Spoke to Firewall Routing**

Spoke VNets used UDRs configured with:

•	Azure Firewall as next hop

•	Gateway propagation disabled

This ensured spoke traffic could not bypass centralized firewall inspection.

**3.4.Route Redistribution to ExpressRoute and VPN Gateways**

Many hybrid environments continue using ExpressRoute and VPN connectivity while gradually migrating branches toward SD-WAN.
To support this coexistence model, Azure Route Server branch-to-branch capability was enabled.

This allowed SD-WAN branch prefixes to be dynamically redistributed toward:

•	ExpressRoute Gateways

•	VPN Gateways

As a result:

•	Datacenters gained visibility of SD-WAN-connected branches

•	Existing hybrid routing models remained operational

•	Migration complexity was reduced

**3.5.Route Advertisement toward SD-WAN**

The SD-WAN hubs dynamically learned:

•	Azure VNET prefixes

•	On-premises datacenter prefixes through Azure Route Server.

These prefixes were then advertised across the SD-WAN overlay toward remote branches.

This allowed seamless connectivity between:

•	Azure workloads

•	Datacenters

•	SD-WAN-connected sites while maintaining centralized firewall inspection.
________________________________________
**Key Design Benefits**

The final architecture provided:

•	High availability across Availability Zones

•	Elimination of Azure Load Balancer dependency

•	Dynamic route exchange using BGP

•	Centralized Azure Firewall inspection

•	Symmetric routing across hybrid environments

•	Seamless integration with ExpressRoute and VPN gateways

•	Controlled and scalable Hub-Spoke routing
________________________________________
**Lessons Learned**

Several important considerations emerged during implementation:

•	Azure Load Balancer is not always suitable for overlay-based SD-WAN appliances

•	Symmetric routing becomes critical when integrating Azure Firewall with SD-WAN

•	Azure Route Server significantly simplifies hybrid route propagation

•	Combining dynamic routing with selective static UDRs offers greater operational flexibility

•	Route scale limitations must be considered early in design phases
________________________________________
**Conclusion**

While Azure Virtual WAN simplifies many SD-WAN deployments, traditional self-managed Hub-Spoke architectures continue to play an important role in enterprise environments requiring deeper routing control, advanced security inspection, and customized traffic engineering.

By combining:

•	Juniper SD-WAN

•	Azure Route Server

•	Azure Firewall

•	ExpressRoute/VPN integration

it is possible to design a scalable, highly available, and operationally controlled SD-WAN architecture in Azure while overcoming common routing and failover challenges.

