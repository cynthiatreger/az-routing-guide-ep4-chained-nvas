### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #4: Chained NVAs & BGP

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##

# 4.1. Test environment description

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the On-Prem and Azure, either for the entire Cloud environment or for dedicated Spoke VNETs only. Here we will consider this requirement for Spoke1 VNET only. 

This security layer is here provided by another 3rd Party NVA rather than by the native Azure Firewall. A second Cisco CSR ("*FW NVA*") is used for that purpose and deployed in a dedicated subnet in the Hub VNET (*FWsubnet*: 10.0.0.0/24). 

To facilitate the management of branches being added/deleted or updated with new subnets, dynamic route advertisement (BGP) is run between the Concentrator NVA and the FW NVA.

The Spoke *SpokeRT* route table created in Episode #3 for direct connectivity between the Spoke VMs and the Concentrator NVA has been dissociated from the Spoke1 VNET subnets.

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215558452-b22e0f2d-6bfd-4372-8881-6610e41dcd23.png">

As represented on the diagram, the expected traffic flows between Spoke1 VNET and the On-Prem are:
-	Azure to On-Prem: Spoke1 VMs > FW NVA > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > FW NVA > Spoke1 VMs

The traffic flows between Spoke2 VNET and the On-Prem remain as in Episode #3:
-	Azure to On-Prem: Spoke2 VMs > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > Spoke2 VMs

## 4.2.	Step1: Connectivity between the FW NVA and the Branches

### 4.2.1. FW NVA Effective routes vs FW NVA routing table

<img width="1122" alt="image" src="https://user-images.githubusercontent.com/110976272/215564791-ada61c2f-a144-4f8d-8f9f-34199d2f26ff.png">

The FW NVA can ping the Concentrator NVA and is learning the branch prefixes supernet (192.168.0.0/16) advertised by the Concentrator NVA (10.0.10.4) via BGP.
The On-Prem branches know from the Concentrator NVA the the Spoke1 Azure VNET range (10.1.0.0/16) advertised via BGP by the FW NVA (10.0.0.5).

From an NVA OS level (control-plane) the connectivity is okay. However pings are failing from the Hub NVA to Branch1.

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals/blob/main/README.md#311-nva-effective-routes--nva-routing-table-alignement), although the branch routes exist in the FW NVA routing table, they are not reflected on the FW NVA underlying VM’s Effective routes. 

<img width="558" alt="image" src="https://user-images.githubusercontent.com/110976272/215434704-c4f2b2b2-6de0-4cd3-905e-0e92b71f79a1.png">

:arrow_right: **Whatever an NVA custom routing configuration is (static routes, BGP etc), the NVA routing table is by default not reflected in the NVA's Effective routes, creating a misalignment between the NVA control plane and the NVA data-plane.**

The FW NVA routing table containing the Branch prefixes (control-plane) is not enough, the FW NVA’s NIC must know about these prefixes too: data-plane connectivity is currently missing.

### 4.2.2. (Samoe old) solution: Align the data-plane (Effective routes) to the control-plane (NVA routing table)

Following the learnings of [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), to enable connectivity at the Azure platform level, a route table must be created and associated to the subnet of the FW NVA with a UDR ("*toBranches*") to the IP address of the Concentrator NVA (Next-Hop = 10.0.10.4). 

<img width="817" alt="image" src="https://user-images.githubusercontent.com/110976272/215448424-747eec8d-9d69-474d-b381-f906c25af52c.png">

(Note: the Concentrator NVA subnet is still associated with the *ConcentratorRT* route table containing a UDR for its own branches.)

<img width="900" alt="image" src="https://user-images.githubusercontent.com/110976272/215562521-9ea33195-9c46-4de0-b2e8-e7842c711a6f.png">

## 4.3. Step 2: Branch Connectivity & FW NVA transit for Spoke1 VNET

Now that the connectivity between the FW NVA and the On-Prem Branches via the Concentrator NVA is confirmed, in this section we will see how to extend this connectivity end-to-end and provide FW transit between the Spoke1 VNET and the Branches.

### 4.3.1. UDRs on the Spoke1 VMs
A new route table is associated to the the Spoke1 subnets, with a UDR for the the branches (92.168.0.0/16) pointing at th FW NVA (Next-Hop = 10.0.0.5), that will create a "*User*" entry in the Effective routes of Spoke1VM for that prefix. Let's control with traceroute that the FW NVA is in the path:

<img width="816" alt="image" src="https://user-images.githubusercontent.com/110976272/215566526-b9afd33f-9f6e-40e3-9d2a-69b419b914e3.png">

:warning: Remember to enable “IP Forwarding” on the FW NVA’s NIC receiving the traffic, or the packets will be dropped.

<img width="1149" alt="image" src="https://user-images.githubusercontent.com/110976272/215594263-b0e5a82c-7b72-4ec4-ba5c-5606cc2ab001.png">

Connectivity between On-Prem and Spoke1 is achieved, however only traffic from Spoke1 to OnPrem is inspected by the FW, the return traffic bypasses the FW as demonstrated by the traceroute to Spoke1VM.

:arrow_right: It’s not only about traffic from Azure to OnPrem, the On-Prem has to find the right way back to Azure too.

### 4.3.2. Forcing traffic to Spoke1 from the Concentrator NVA into the FW NVA
Although the Concentrator NVA learns the Spoke1 range (10.1.0.0/16) from the FW NVA via BGP, when the packet destined to Spoke1VM reaches the NIC of the Concentrator NVA, the destination (10.1.1.4) will be matched against the NIC Effective routes. Due to the transitivity of the direct peering between the Hub VNET and Spoke1 VNET (see [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways)) traffic will be forwarded over the peering to Spoke1 VNET directly.

One last layer of UDRs is required on the Concentrator NVA to force traffic to Spoke1 VNET towards the FW NVA, as per the ConcentratorRT route table:

<img width="705" alt="image" src="https://user-images.githubusercontent.com/110976272/215595108-779f79c2-02ad-4fa5-ae25-e03860765190.png">

##
### [>> EPISODE #5](https://github.com/cynthiatreger/az-routing-guide-ep4-nva-routing-2-0) (out 02/02)
