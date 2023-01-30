## [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
# Episode #4: Chained NVAs & BGP

Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.
##

# 4.1. Test environment

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the On-Prem and the Azure environment. For the sake of this guide, this security layer is provided by another 3rd Party NVA rather than by our native Azure Firewall.

A second Cisco CSR ("*FW NVA*") is used for that purpose and deployed in a dedicated subnet in the Hub VNET (*FWsubnet*: 10.0.0.0/24). 

To facilitate the management of branches being added/deleted or updated with new subnets, dynamic route advertisement (BGP) is run between the Concentrator NVA and the FW NVA.

The Spoke *SpokeRT* route table created in Episode #3 for direct connectivity between the Spoke VMs and the Concentrator NVA have been dissociated from the Spoke subnets.

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215452447-c7a73b77-28d2-4281-9c00-dd3345e75f21.png">

As represented on the diagram, the expected traffic flows are:
-	Azure to On-Prem: Spokes > FW NVA > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > FW NVA > Spokes

*Depending on your architecture, with little adjustments the Concentrator NVA could instead be in a dedicated peered Spoke VNET. The conclusions we will reach are the same.*

## 4.2.	Step1: Connectivity between the FW NVA and the Branches

### 4.2.1. FW NVA Effective routes vs FW NVA routing table

<img width="1120" alt="image" src="https://user-images.githubusercontent.com/110976272/215453075-dfa5ad19-8335-4eb4-acec-b74ab46878d2.png">

The FW NVA can ping the Concentrator NVA and is learning the branch prefixes supernet (192.168.0.0/16) advertised by the Concentrator NVA (10.0.10.4) via BGP.
The On-Prem branches know from the Concentrator NVA the the Azure VNET range (10.0.0.0/8) advertised by the FW NVA (10.0.0.5).

From an NVA OS level (control-plane) the connectivity is okay. However pings are failing from the Hub NVA to Branch1.

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals/blob/main/README.md#311-nva-effective-routes--nva-routing-table-alignement), although the branch routes exist in the FW NVA routing table, they are not reflected on the FW NVA underlying VM’s Effective routes. 

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215434704-c4f2b2b2-6de0-4cd3-905e-0e92b71f79a1.png">

:arrow_right: **Whatever an NVA custom routing configuration is (static routes, BGP etc), the NVA routing table is by default not reflected in the NVA's Effective routes, creating a misalignment between the NVA control plane and the NVA data-plane.**

The FW NVA routing table containing the Branch prefixes (control-plane) is not enough, the FW NVA’s NIC must know about these prefixes too: data-plane connectivity is currently missing.

### 4.2.2. Solution: Align the data-plane (Effective routes) to the control-plane (NVA routing table)

Following the learnings of [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), to enable connectivity at the Azure platform level, a route table must be created and associated to the subnet of the FW NVA with a UDR ("toBranches") to the IP address of the Concentrator NVA (Next-Hop = 10.0.10.4). 

<img width="817" alt="image" src="https://user-images.githubusercontent.com/110976272/215448424-747eec8d-9d69-474d-b381-f906c25af52c.png">

(Note: the Concentrator NVA subnet is still associated with the *ConcentratorRT* route table containing an entry for its own branches.)

<img width="896" alt="image" src="https://user-images.githubusercontent.com/110976272/215454939-c3f9a28f-d652-4fe1-999e-b7f54b451a0a.png">

Remember to also to enable “IP Forwarding” on the FW NVA’s NIC.
