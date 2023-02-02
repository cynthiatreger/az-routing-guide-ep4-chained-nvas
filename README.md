### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #4: Chained NVAs

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[4.1. Test environment description and target traffic flows ](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#41-test-environment-description-and-target-traffic-flows)

[4.2. Step1: Connectivity between the FW NVA and the branches](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#42step1-connectivity-between-the-fw-nva-and-the-branches)

&emsp;[4.2.1. Current connectivity recap](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#421-current-connectivity-recap)

&emsp;[4.2.2. FW NVA Effective routes and FW NVA routing table misalignment](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#422-fw-nva-effective-routes-and-fw-nva-routing-table-misalignment)

&emsp;[4.2.3. Solution: Align the FW NVA Effective routes to the FW NVA routing table](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#423-solution-align-the-fw-nva-effective-routes-to-the-fw-nva-routing-table)

[4.3. Step 2: End-to-end Connectivity and FW NVA transit between Spoke1 VNET and the branches](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#43-step-2-end-to-end-connectivity-and-fw-nva-transit-between-spoke1-vnet-and-the-branches)

&emsp;[4.3.1. Spoke1 => On-Prem FW transit](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#431-spoke1--on-prem-fw-transit)

&emsp;[4.3.2. On-Prem => Spoke1 FW transit](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#432-on-prem--spoke1-fw-transit)
##
# 4.1. Test environment description and target traffic flows 

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having firewall inspection between the On-Prem and Azure.

It is possible in Azure to customize routing to for example provide FW inspection to specific workloads only. Let's consider such filtering for Spoke1 while Spoke2 will keep bypassing the FW.

For this use-case, a second Cisco CSR (named "FW NVA") is deployed in a new subnet in the Hub VNET ("FWsubnet": 10.0.0.0/24). 

Targeted traffic flows:

<img width="1022" alt="image" src="https://user-images.githubusercontent.com/110976272/216075010-e5e03d84-9ec8-4fb5-8c87-bd497f11f099.png">

The UDR towards the On-Prem branches (in the "SpokeRT" *Route table*) configured in Episode #3 for direct connectivity between the Azure VMs and the On-Prem via the Concentrator NVA has been dissociated from all the Spoke1 subnets as FW transit is now required for this VNET. We will see further in this article how to achieve this. The "SpokeRT" *Route table* remains associated to the Spoke2 subnets.

# 4.2.	Step1: Connectivity between the FW NVA and the branches

 ## 4.2.1. Current connectivity recap

- Reachability between VMs within the Hub VNET and to the peered VNETs is established by default
    - the NVAs can reach each other and any VM of our test environment
    - unless there are any UDRs configured, by default there won't be any connectivity further than the scope of the Hub VNETs and its Spokes

- Connectivity between the Concentrator NVA and the On-Prem has been validated in Episode #3:
    - The On-Prem branches learn the overall 10/8 Azure range from the Concentrator NVA 
    - The Concentrator NVA routing table includes various subnets of the 192.168.0.0/16 range
    - successful pings

- Connectivity from Spoke2 VNET to the On-Prem branches is achieved by a UDR for the 192.168.0.0/16 range pointing to the Concentrator NVA (Next-Hop = 10.0.10.4) and configured on the Spoke2 subnets as well as on the Concentrator NVA subnet.

## 4.2.2. FW NVA *Effective routes* and FW NVA routing table misalignment

From a traditional routing perspective, static routing or BGP would have been used between the FW NVA and the Concentrator NVA for On-Prem branch connectivity.

Although BGP would be more relevant in an enterprise environment for scalability consierations, for simplicity we will here consider the static routing approach and configure the FW NVA with a static route towards the On-Prem branches and pointing to the Concentrator NVA.

<img width="1128" alt="image" src="https://user-images.githubusercontent.com/110976272/216169050-7db9cb26-69ed-4230-8ca2-34898557358d.png">

Despite the On-Prem branches being reachable from the Concentrator NVA, despite confirmed connectivity between the Concentrator NVA and the FW NVA and despite the FW NVA having an entry in its routing table for the traffic to On-Prem pointing to the Concentrator NVA, pings are failing.

The exact same outcome would have happened with BGP running between the FW NVA and the Concentrator NVA.

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#322azure-vm-effective-routes-and-nva-routing-table-misalignment), although the branch routes exist in the FW NVA routing table, they are not reflected on the FW NVA's *Effective routes*. 

<img width="487" alt="image" src="https://user-images.githubusercontent.com/110976272/216169544-ef166f6d-e998-4b74-b741-d658dbd24741.png">

:arrow_right: **Whatever an NVA routing configuration is (static routes, BGP etc), the NVA routing table is by default not reflected in the NVA's *Effective routes*, creating a misalignment between the NVA control plane and the NVA data-plane.**

The FW NVA routing table containing the On-Prem branch prefixes (control-plane) and a valid next-hop to reach them is not enough, the FW NVA’s NIC must know about these On-Prem prefixes too: data-plane connectivity is currently missing.

## 4.2.3. Solution: Align the FW NVA *Effective routes* to the FW NVA routing table

Following the learnings of [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), to enable connectivity at the Azure platform level, a *Route table* must be created (here named "NvaRT") and associated to the subnet of the FW NVA with a UDR ("toBranches") pointing to the IP address of the Concentrator NVA (Next-Hop = 10.0.10.4):

<img width="817" alt="image" src="https://user-images.githubusercontent.com/110976272/215448424-747eec8d-9d69-474d-b381-f906c25af52c.png">

The result is successful connectivity between the FW NVA and the On-Prem branches:

<img width="924" alt="image" src="https://user-images.githubusercontent.com/110976272/216075824-ddd5377c-b2b2-4213-8ab9-802b9f3afb9b.png">



# 4.3. Step 2: End-to-end Connectivity and FW NVA transit between Spoke1 VNET and the branches

Now that the connectivity between the FW NVA and the On-Prem branches via the Concentrator NVA is confirmed, in this section we will see how to extend this connectivity end-to-end and provide FW transit between the Spoke1 VNET and the branches.

## 4.3.1. Spoke1 => On-Prem FW transit

Based on the default [VNET peering rules](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) discussed in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways), and since Spoke1 VNET has been dissociated from any *Route table* at the beginning of this article, the Spoke1 VMs don't have visibility of any IP range outside of the Spoke1 and Hub VNETs.

In this Episode, we want the VMs in Spoke1 to transit via the FW NVA and the Concentrator NVA for On-Prem branches reachability, as opposed to direct access to the FW Concentrator like in Episode #3.

A UDR is again required to bring the On-Prem prefixes knowledge to the Spoke1 subnets, but with the Next-Hop becoming the FW NVA.

This is achieved by associating a new *Route table* to the Spoke1 subnets, with a UDR for the On-Prem branches (192.168.0.0/16) pointing to the FW NVA (10.0.0.5):

<img width="816" alt="image" src="https://user-images.githubusercontent.com/110976272/215566526-b9afd33f-9f6e-40e3-9d2a-69b419b914e3.png">

:warning: Remember to enable “IP Forwarding” on the FW NVA’s NIC receiving the traffic, or the packets will be dropped.

Let's check using traceroutes that the FW NVA is in the path:

<img width="1190" alt="image" src="https://user-images.githubusercontent.com/110976272/216076362-fef6da09-1345-43c0-996e-e34a0fb792bd.png">

Connectivity between the On-Prem and Spoke1 VNET is achieved. However only traffic from Spoke1 VNET to OnPrem is inspected by the FW, the return traffic bypasses the FW as demonstrated by the traceroute from the Concentrator NVA to Spoke1VM.

:arrow_right: It’s not only about traffic from Azure to OnPrem, the On-Prem has to find the right way back to Azure too.

## 4.3.2. On-Prem => Spoke1 FW transit

When packets destined to Spoke1VM reach the NIC of the Concentrator NVA, the destination (10.1.1.4) will be matched against the NIC *Effective routes* and forwarded over the peering to Spoke1 VNET directly.

One last layer of UDRs is required on the Concentrator NVA to force traffic to Spoke1 VNET towards the FW NVA. The "ConcentratorRT" *Route table* is updated accordingly with the "toSpoke1" UDR:

<img width="705" alt="image" src="https://user-images.githubusercontent.com/110976272/215595108-779f79c2-02ad-4fa5-ae25-e03860765190.png">

This completes the return traffic inspection of traffic from the On-Prem branches to Spoke1VNET:

<img width="1196" alt="image" src="https://user-images.githubusercontent.com/110976272/216076566-5242f367-6540-47fd-a89f-f8797f562a25.png">

:arrow_right: *Default* routes overridden by UDRs become "Invalid" and a new *User* entry is added at the bottom of the *Effective routes*. 

## 4.4. Conclusion

Finally both connectivity & FW inspection between Azure resources and branches connected to a Concentrator NVA have been achieved!

But admittedly with quite some complexity, with UDRs required on every subnet of every VNET in our design to match the OS routing level view.

Fortunately there is room for improvement, as we will find out in the next Episode.
##
### [>> EPISODE #5](https://github.com/cynthiatreger/az-routing-guide-ep4-nva-routing-2-0) (out soon)
