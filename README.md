# Episode #4: Chained NVAs & BGP

Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.
##

# 4.1. Test environment

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the On-Prem and the Azure environment. For the sake of this guide, this security layer is provided by another 3rd Party NVA rather than by our native Azure Firewall.

A second Cisco CSR ("*FW NVA*") is used for that purpose and deployed in a dedicated subnet in the Hub VNET (*FWsubnet*: 10.0.0.0/24). 

To facilitate the management of branches being added/deleted or updated with new subnets, dynamic route advertisement (BGP) is run between the Concentrator NVA and the FW NVA.

The Spoke *SpokeRT* route table created in Episode #3 have been dissociated from the Spoke subnets.

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215359023-0e9bc953-ba11-4f89-91dd-d078a4167f6c.png">

As represented on the diagram, the expected traffic flows are:
-	Azure to On-Prem: Spokes > FW NVA > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > FW NVA > Spokes

*Depending on your architecture the Concentrator NVA could instead be in a dedicated peered Spoke VNET. The conclusions we will reach are the same.*

## 4.2.	Step1: Connectivity between the FW NVA and the Branches

### 4.2.1. FW NVA Effective routes vs FW NVA routing table

<img width="1120" alt="image" src="https://user-images.githubusercontent.com/110976272/215361316-aaf85bde-2c80-480a-80ae-95b91719fc4b.png">

The FW NVA can ping the Concentrator NVA and is learning the branch prefixes supernet (192.168.0.0/16) advertised by the Concentrator NVA (10.0.10.4) via BGP.
The On-Prem branches know from the Concentrator NVA the the Azure VNET range (10.0.0.0/8) advertised by the FW NVA (10.0.0.5).

From an NVA OS level (control-plane) the connectivity is okay. However pings are failing from the Hub NVA to Branch1.

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals/blob/main/README.md#311-nva-effective-routes--nva-routing-table-alignement), although the branch routes exist in the FW NVA routing table, they are not reflected on the FW NVA underlying VM’s Effective routes. 

The FW NVA routing table containing the Branch prefixes (control-plane) is not enough, the FW NVA’s NIC must know about these prefixes too: data-plane connectivity is currently missing.

### Solution: Align the data-plane (Effective routes) to the control-plane (NVA routing table)

To do so, the “NvaRT” route table associated to the subnet of the NVA must be updated with a new UDR (“toBranches”) for the branches, with NH = 10.3.1.4 the IP address of the concentrator. (Remember also to enable “IP Forwarding” on the Concentrator’s NIC.)


the Concentrator NVA subnet is still associated with the *ConcentratorRT*.
