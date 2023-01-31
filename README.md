### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #4: Chained NVAs & BGP

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
# 4.1. Expected traffic flows and test environment description

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the On-Prem and Azure, either for the entire Cloud environment or for a limited set Spoke VNETs only. Here we will consider this requirement for Spoke1 VNET. 

Expected traffic flows:

<img width="1022" alt="image" src="https://user-images.githubusercontent.com/110976272/215856350-b3ceb0f9-e0b0-425c-a29d-afff4484c8ce.png">
To illustrate the usage of BGP, this security layer is provided by another 3rd Party NVA rather than by the native Azure Firewall. For that purpose, a second Cisco CSR (named "FW NVA") is deployed in a new subnet in the Hub VNET ("FWsubnet": 10.0.0.0/24). 

To facilitate the management of branches being added/deleted or updated with new subnets, dynamic route advertisement (BGP) is run between the Concentrator NVA and the FW NVA.

The "SpokeRT" *route table* created in Episode #3 and applied to the Spoke VNETs for direct connectivity between the Spoke VMs and the Concentrator NVA has been dissociated from all the Spoke1 VNET subnets.

# 4.2.	Step1: Connectivity segment between the FW NVA and the branches

## 4.2.1. FW NVA *Effective routes* vs FW NVA routing table

 ### 4.2.1.1. Episode #3 recap

 As seen in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals):
 - Data plane reachability between VMs within the Hub VNET and to the peered VNETs is established by default (successful pings between the FW NVA and the Concentrator NVA).

-  The On-Prem branches learn the overall 10/8 Azure range from the Concentrator NVA. Connectivity to the On-Prem branches from the Concentrator NVA is confirmed by its routing table and the successful pings.

- the Concentrator NVA didn’t have the On-Prem prefixes in its Effective routes, requiring its subnet to be associated to a *Route table* ("ConcentratorRT") with the 192.16.0.0/16 UDR:

<img width="688" alt="image" src="https://user-images.githubusercontent.com/110976272/215297079-9faf1ef6-b1d3-477c-b612-0d49563af5e1.png">

 ### 4.2.1.2. NVA routing table analysis

 Let's now focus on the BGP route exchanges between the FW NVA and the Concentrator NVA:

<img width="1122" alt="image" src="https://user-images.githubusercontent.com/110976272/215866207-ea917e9c-d35c-422c-b96d-b50693adbef2.png">


FW NVA:
- advertises the specific Spoke1 VNET range (10.1.0.0/16) to the Concentrator NVA
- learns the On-Prem branch prefixes supernet (192.168.0.0/16) from the Concentrator NVA (10.0.10.4)

Concentrator NVA:
- advertises via BGP the OnPrem branch prefixes supernet (192.168.0.0/16) to the FW NVA
- learns the Spoke1 VNET range via BGP from the FW NVA (10.0.0.5)

From an NVA OS level (control-plane/routing table) the connectivity is okay. However pings are failing from the Hub NVA to Branch1.

 ### 4.2.1.3. (Same old) *Effective routes* and routing table misalignment

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#322azure-vm-effective-routes-and-nva-routing-table-misalignment), although the branch routes exist in the FW NVA routing table, they are not reflected on the FW NVA's *Effective routes*. 

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215434704-c4f2b2b2-6de0-4cd3-905e-0e92b71f79a1.png">

:arrow_right: **Whatever an NVA routing configuration is (static routes, BGP etc), the NVA routing table is by default not reflected in the NVA's *Effective routes*, creating a misalignment between the NVA control plane and the NVA data-plane.**

The FW NVA routing table containing the On-Prem branch prefixes (control-plane) is not enough, the FW NVA’s NIC must know about these On-Prem prefixes too: data-plane connectivity is currently missing.

This also reminds on the reason why, in [Epsiode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), the Concentrator NVA didn’t have by default its reachable On-Prem prefixes in its own *Effective routes*, requiring its subnet to be associated to a *Route table* ("ConcentratorRT") with the 192.16.0.0/16 UDR:

<img width="688" alt="image" src="https://user-images.githubusercontent.com/110976272/215297079-9faf1ef6-b1d3-477c-b612-0d49563af5e1.png">

## 4.2.2. (Same old) solution: Align the data-plane (*Effective routes*) to the control-plane (NVA routing table)

Following the learnings of [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), to enable connectivity at the Azure platform level, a *Route table* must be created (here named "NvaRT") and associated to the subnet of the FW NVA with a UDR ("toBranches") pointing to the IP address of the Concentrator NVA (Next-Hop = 10.0.10.4):

<img width="817" alt="image" src="https://user-images.githubusercontent.com/110976272/215448424-747eec8d-9d69-474d-b381-f906c25af52c.png">

... resulting in successful connectivity between the FW NVA and the On-Prem branches.

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/215562521-9ea33195-9c46-4de0-b2e8-e7842c711a6f.png">

# 4.3. Step 2: End-to-end Connectivity & FW NVA transit between Spoke1 VNET and the branches

Now that the connectivity between the FW NVA and the On-Prem branches via the Concentrator NVA is confirmed, in this section we will see how to extend this connectivity end-to-end and provide FW transit between the Spoke1 VNET and the branches.

## 4.3.1. UDRs on the Spoke1 VMs

Based on the default [VNET peering rules](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) discussed in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways), and since Spoke1 VNET has been dissociated from any *Route table* at the beginning of this article, the Spoke1 VMs don't have visibility of any IP range outside of the Spoke1 and Hub VNETs.

A UDR is hence required to bring the On-Prem prefixes knowledge to the Spoke1 subnets.

This is achieved by associating a new *Route table* to all or a selected set of subnets in the Spoke1 VNET, with a UDR for the On-Prem branches (192.168.0.0/16) pointing to the FW NVA (Next-Hop = 10.0.0.5).

<img width="816" alt="image" src="https://user-images.githubusercontent.com/110976272/215566526-b9afd33f-9f6e-40e3-9d2a-69b419b914e3.png">

:warning: Remember to enable “IP Forwarding” on the FW NVA’s NIC receiving the traffic, or the packets will be dropped.

Let's check with traceroute that the FW NVA is in the path:

<img width="1149" alt="image" src="https://user-images.githubusercontent.com/110976272/215594263-b0e5a82c-7b72-4ec4-ba5c-5606cc2ab001.png">

Connectivity between the On-Prem and Spoke1 VNET is achieved, however only traffic from Spoke1 VNET to OnPrem is inspected by the FW, the return traffic bypasses the FW as demonstrated by the traceroute to Spoke1VM.

:arrow_right: It’s not only about traffic from Azure to OnPrem, the On-Prem has to find the right way back to Azure too.

## 4.3.2. Forcing traffic to Spoke1 from the Concentrator NVA into the FW NVA

Although the Concentrator NVA learns the Spoke1 range (10.1.0.0/16) from the FW NVA via BGP, when packets destined to Spoke1VM reach the NIC of the Concentrator NVA, the destination (10.1.1.4) will be matched against the NIC *Effective routes*. Due to the transitivity of the direct peering between the Hub VNET and Spoke1 VNET (see [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways)) traffic will be forwarded over the peering to Spoke1 VNET directly.

One last layer of UDRs is required on the Concentrator NVA to force traffic to Spoke1 VNET towards the FW NVA:

<img width="705" alt="image" src="https://user-images.githubusercontent.com/110976272/215595108-779f79c2-02ad-4fa5-ae25-e03860765190.png">

Finally both connectivity & FW insection between Azure resources and branches connected to a Concentrator NVA have been achieved!

But admittedly with quite some complexity. Fortunately there is room for improvement, as we will find out in the next Episode.
##
### [>> EPISODE #5](https://github.com/cynthiatreger/az-routing-guide-ep4-nva-routing-2-0) (out 02/02)
