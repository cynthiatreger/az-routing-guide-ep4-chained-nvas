# Episode #4: Chained NVAs & BGP

Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.
##

# 4.1. Test environment

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the branch Concentrator and the Azure environment. For the sake of this guide, this security layer is provided by another 3rd Party NVA rather than by our native Azure Firewall.

A second Cisco CSR is used for that purpose and deployed in a new FWsubnet (10.0.0.0/24) in the Hub VNET.





Let’s take the example of remote sites (br 1 to 5) connected to a new “Concentrator” NVA in a Spoke VNET. The Concentrator could be an SDWAN hub, a FW or IPSec GW or a FW etc.
Depending on your architecture this Concentrator NVA could instead be in a dedicated subnet in the Hub VNET, the conclusions we will reach are the same.
A common scenario is to have traffic between Azure resources and these branches transiting via our existing NVA in the hub, for the following 2 reasons:
1.	To provide inspection between the Azure resources and the branches.
2.	(specific to the Concentrator being in a Spoke VNET) as there is no recursive routing nor VNET peering transitivity, there is no other way for the Spoke VMs to reach the Concentrator in a 3rd spoke other than by hoping via their common Hub VNET.
Expected traffic flow:
-	Azure to On-Prem: Spokes > HubNVA > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > HubNVA > Spokes
To facilitate the management of branches being added/deleted/updated with new subnets, dynamic route advertisement is running between the branches and their Concentrator and between the Concentrator and our Hub NVA (BGP).
Assumption: connectivity between the Concentrator and the branches is confirmed.
