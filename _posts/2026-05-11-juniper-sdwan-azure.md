---
Title: "Designing Juniper SD-WAN Integration in Azure using Hub-Spoke Architecture"
Date: 2026-05-11
Categories: [Azure, Networking, SD-WAN]
Tags: [Azure, SDWAN, Juniper, HubSpoke, AzureFirewall, ExpressRoute]
---

# Designing Juniper SD-WAN Integration in Azure using a Traditional Hub-Spoke Architecture

**Introduction**

There are multiple networking patterns available for integrating SD-WAN with Microsoft Azure. Most enterprise deployments today leverage integrated Network Virtual Appliances (NVAs) deployed directly into Azure Virtual WAN hubs through jointly managed solutions provided by Microsoft Azure and third-party SD-WAN vendors. In these architectures, routing and connectivity are largely abstracted and operational complexity is significantly reduced.

However, some enterprise environments require a more traditional self-managed Hub-Spoke architecture to retain full control over routing decisions, firewall inspection, traffic engineering, and hybrid connectivity.

This article explores a real-world Azure networking design for deploying Juniper SD-WAN(mist managed ssrs) in a traditional Hub-Spoke architecture while maintaining:

•	Full routing control

•	Centralized Azure Firewall inspection

•	High availability across Availability Zones

•	Hybrid datacenter connectivity through ExpressRoute and VPN gateways

•	Symmetric routing across SD-WAN and Azure workloads

The design also addresses several Azure and SD-WAN integration constraints commonly encountered in enterprise environments.

________________________________________
**Architecture Requirements**

**Requirement 1 — Highly Available SD-WAN Infrastructure**

The architecture requires a highly available SD-WAN deployment capable of tolerating:

•	Single VM failures

•	Availability Zone failures

•	Network appliance failover scenarios

The primary objective is to increase resiliency of the SD-WAN hubs while ensuring uninterrupted branch connectivity.

Constraint 1 — Azure Load Balancer Probe Limitation

A key challenge with Juniper SD-WAN NVAs is that they do not respond to Azure Load Balancer health probes. This behavior is expected because Juniper SD-WAN overlay communication is authenticated and secured by design. The appliances do not expose generic probe listeners required by Azure Standard Load Balancer health checks.

As a result:

•	Azure Load Balancer cannot reliably determine backend health

•	Traditional Active/Active or Active/Standby load balancer patterns are not feasible
________________________________________
**Requirement 2 — Mandatory Azure Firewall Inspection**

The architecture requires all traffic traversing between:

•	Azure workloads

•	On-premises datacenters

•	SD-WAN branches to be inspected through Azure Firewall.

This introduces additional routing complexity because traffic must remain symmetric while traversing centralized security inspection points.

Constraint 2 — Asymmetric Routing Challenges

Asymmetric routing is a common challenge in SD-WAN and firewall-integrated architectures.

Challenges included:

•	Return traffic bypassing Azure Firewall

•	Session state inconsistencies

•	Active/Active routing conflicts

•	Firewall session drops

Maintaining symmetric traffic flow across SD-WAN hubs, Azure Firewall, spokes, and hybrid connectivity becomes a critical design consideration.
________________________________________

**Requirement 3 — Hybrid Datacenter Connectivity**

The architecture must integrate existing hybrid connectivity through:

•	ExpressRoute Gateways

•	VPN Gateways

Traffic between datacenters and SD-WAN-connected branches also requires mandatory Azure Firewall inspection.

Constraint 3 — Symmetric Routing across ExpressRoute and VPN

Traffic entering Azure from datacenters through ExpressRoute or VPN gateways must:

•	traverse Azure Firewall

•	maintain symmetric return paths

•	avoid bypass routing scenarios

This significantly increases routing complexity across the hub network.
________________________________________

**Solution Design**

**1.High Availability Architecture for SD-WAN Hubs**

To overcome Azure Load Balancer probe limitations while maintaining high availability for SD-WAN hubs, the architecture can use:

•	Two Juniper SD-WAN NVAs per hub

•	Deployment of each appliance across separate Availability Zones

•	Dedicated static Public IP addresses assigned per appliance

This approach allows SD-WAN spokes to establish direct overlay connectivity independently to each SD-WAN hub appliance.

High availability and failover can be achieved natively through the SD-WAN overlay using:

•	BGP communities

•	Dynamic path selection

•	Overlay route preference

This architecture removes dependency on Azure Load Balancer for inbound SD-WAN connectivity while providing resiliency against VM and zone-level failures.
________________________________________
**2.Azure Route Server Integration**

Azure Route Server (ARS) can be integrated to enable dynamic route exchange between:

•	Juniper SD-WAN hubs

•	Azure Firewall

•	ExpressRoute Gateways

•	VPN Gateways

BGP peering between Azure Route Server and SD-WAN hubs enables automatic propagation of dynamically learned SD-WAN prefixes across the Azure network fabric.

This simplifies route distribution and reduces manual route management complexity across hybrid environments.
________________________________________
**3.Routing Design**

![Architecture Diagram](/assets/SDWAN3.png)

The architecture can use a hybrid routing model combining:

•	Dynamic route propagation using Azure Route Server

•	Static route enforcement using User Defined Routes (UDRs)

This semi-dynamic routing model provides greater operational control while maintaining scalability and predictable traffic flow.

**3.1.Routing Azure Firewall to SD-WAN (Dynamic via Route Server)**

Traffic from Azure Firewall toward SD-WAN-connected branches can be dynamically routed through Azure Route Server.

In this design:

SD-WAN appliances advertise branch prefixes through BGP to Azure Route Server

Azure Route Server propagates learned routes toward Azure Firewall subnet NICs

Azure Firewall dynamically learns branch routes without requiring manual route updates

Benefits of this approach include:

•	Dynamic route learning

•	Reduced administrative overhead

•	Simplified SD-WAN branch onboarding

**3.2.Routing SD-WAN to Azure Firewall (Static via UDRs)**

Traffic originating from the SD-WAN private subnet toward:

•	Azure spoke VNets

•	On-premises environments can be routed toward Azure Firewall using User Defined Routes.

This ensures all east-west and north-south traffic remains centrally inspected through Azure Firewall.

One important design consideration is Azure’s UDR limitation of 400 routes per route table. Proper route summarization and segmentation become important for maintaining scalability in large environments.

**3.3.Spoke to Firewall Routing**

Spoke VNets can use User Defined Routes configured with:

•	Azure Firewall as next hop

•	Gateway propagation disabled

This prevents spoke traffic from bypassing centralized firewall inspection and ensures consistent security enforcement across all spoke workloads.

**3.4.Route Redistribution to ExpressRoute and VPN Gateways**

In hybrid environments where ExpressRoute and VPN connectivity coexist alongside SD-WAN, Azure Route Server branch-to-branch capability can be enabled to redistribute SD-WAN prefixes dynamically toward:

•	ExpressRoute Gateways

•	VPN Gateways

As a result:

•	Datacenter visibility of SD-WAN-connected branches

•	Seamless coexistence between traditional hybrid connectivity and SD-WAN

•	Simplified migration between network architectures and Dynamic route exchange across hybrid environments

**3.5.Route Advertisement toward SD-WAN**

The SD-WAN hubs can dynamically learn:

•	Azure VNET prefixes

•	On-premises datacenter prefixes through Azure Route Server.

These prefixes can then be advertised across the SD-WAN overlay toward remote branches.

This allows seamless connectivity between:

•	Azure workloads

•	Datacenters

•	SD-WAN-connected sites while maintaining centralized firewall inspection.
________________________________________
**Key Design Benefits**

The final architecture provides:

•	High availability across Availability Zones

•	Elimination of Azure Load Balancer dependency

•	Dynamic route exchange using BGP

•	Centralized Azure Firewall inspection

•	Symmetric routing across hybrid environments

•	Seamless integration with ExpressRoute and VPN gateways

•	Controlled and scalable Hub-Spoke routing
________________________________________
**Points to remember**

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

