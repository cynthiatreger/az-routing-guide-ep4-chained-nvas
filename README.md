### [< BACK TO THE MAIN MENU](https://github.com/cynthiatreger/az-routing-guide-intro)
##
# Episode #4: Chained NVAs

*Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.*
##
[4.1. Test environment description and target traffic flows ](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#41-test-environment-description-and-target-traffic-flows)

[4.2. Step1: Connectivity between the FW NVA and the branches](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#42step1-connectivity-between-the-fw-nva-and-the-branches)

&emsp;[4.2.1. Connectivity recap & NVA routing](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#421-connectivity-recap--nva-routing)

&emsp;[4.2.2. FW NVA Effective routes and FW NVA routing table misalignment](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#422-fw-nva-effective-routes-and-fw-nva-routing-table-misalignment)

&emsp;[4.2.3. Solution: Align the FW NVA Effective routes to the FW NVA routing table](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#423-solution-align-the-fw-nva-effective-routes-to-the-fw-nva-routing-table)

[4.3. Step 2: End-to-end Connectivity and FW NVA transit between Spoke1 VNET and the branches](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#43-step-2-end-to-end-connectivity-and-fw-nva-transit-between-spoke1-vnet-and-the-branches)

&emsp;[4.3.1. Spoke1 => On-Prem FW transit](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#431-spoke1--on-prem-fw-transit)

&emsp;[4.3.2. On-Prem => Spoke1 FW transit](https://github.com/cynthiatreger/az-routing-guide-ep4-chained-nvas#432-on-prem--spoke1-fw-transit)
##
# 4.1. Test environment description and target traffic flows 

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having firewall inspection between the On-Prem and Azure.

In Azure it is possible to customize routing to provide FW inspection to specific workloads only for example. Let's consider such filtering for Spoke1, while Spoke2 will keep bypassing the FW.

For this use-case, a second Cisco CSR (named "FW NVA") is deployed in a new subnet in the Hub VNET ("FWsubnet": 10.0.0.0/24). 

Targeted traffic flows:

<img width="1011" alt="image" src="https://user-images.githubusercontent.com/110976272/216576593-98efca4f-887b-4b22-b59d-dcaaa359764d.png">

We will see further in this article how to achieve FW transit for Spoke1 VNET. For now, the "SpokeRT" *Route table* (configured in Episode #3 and containing a UDR towards the On-Prem branches pointing to the Concentrator NVA for direct On-Prem connectivity) has been dissociated from all the Spoke1 subnets but remains associated to the Spoke2 subnets.

# 4.2.	Step1: Connectivity between the FW NVA and the branches

 ## 4.2.1. Connectivity recap & NVA routing

### 4.2.1.1. Episode #3 recap

- Reachability between VMs within the Hub VNET and to the peered VNETs is established by default:
    - the NVAs can reach each other and any VM of the test environment
    - unless there are any UDRs configured, by default there won't be any connectivity further than the scope of the Hub VNETs and its Spokes

- Connectivity between the Concentrator NVA and the On-Prem has been validated in Episode #3:
    - The On-Prem branches learn the overall 10/8 Azure range from the Concentrator NVA 
    - The Concentrator NVA routing table includes various subnets of the 192.168.0.0/16 range
    - successful pings

- Connectivity from Spoke2 VNET to the On-Prem branches is achieved by a UDR for the 192.168.0.0/16 supernet pointing to the Concentrator NVA (Next-Hop = 10.0.10.4) and configured on the Spoke2 subnets ("SpokeRT" *Route table* just discussed).  

### 4.2.1.2. NVA routing & connectivity diagram

From a traditional routing perspective, static routing or BGP would have been used between the FW NVA and the Concentrator NVA for On-Prem branch connectivity.

Although BGP would be more relevant in an enterprise environment for scalability considerations, for simplicity we will here consider the static routing approach and configure, **at FW NVA OS level**, a static route towards the On-Prem branches and pointing to the Concentrator NVA.

<img width="1119" alt="image" src="https://user-images.githubusercontent.com/110976272/217296494-c82b5184-caa8-47ca-bfa0-3c3d0a929259.png">

Despite the On-Prem branches being reachable from the Concentrator NVA, despite confirmed connectivity between the Concentrator NVA and the FW NVA, and despite the FW NVA having an entry in its routing table for the traffic to On-Prem pointing to the Concentrator NVA, pings are failing.

(The exact same outcome would have been observed with BGP running between the FW NVA and the Concentrator NVA.)

## 4.2.2. FW NVA *Effective routes* and FW NVA routing table misalignment

Just like in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#322azure-vm-effective-routes-and-nva-routing-table-misalignment), although the branch routes exist in the FW NVA routing table, they don't exist in the FW NVA's *Effective routes*.

:arrow_right: Based on the packet destination IP, an NVA via its routing table (eventually through recursive lookups) redirects traffic to its NIC*, where the *Effective routes* take over. (see [Episode #3's packet walk](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#312packet-walk)) 

\* *In the case of multiple NICs attached to an NVA, redirection to the appropriate NIC's *Effective routes* is determined by the NVA routing table.*

<img width="487" alt="image" src="https://user-images.githubusercontent.com/110976272/216313336-98182593-1dd5-44b5-a4f7-1a5541be5143.png">

:arrow_right: **Whatever an NVA routing configuration is (static routes, BGP etc), the NVA routing table is by default not reflected in the NVA's *Effective routes*, creating a misalignment between the NVA control plane and the NVA data-plane.**

The FW NVA routing table containing the On-Prem branch prefixes (control-plane) and a valid next-hop to reach them is not enough, the FW NVA’s NIC must know about these On-Prem prefixes too: data-plane connectivity is currently missing.

## 4.2.3. Solution: Align the FW NVA *Effective routes* to the FW NVA routing table

Following the learnings of [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals#323solution-align-the-data-plane-effective-routes-to-the-control-plane-nva-routing-table), to enable connectivity at the Azure platform level, a *Route table* must be created (here named "NvaRT") and associated to the subnet of the FW NVA with a UDR ("toBranches") pointing to the IP address of the Concentrator NVA (Next-Hop = 10.0.10.4):

<img width="817" alt="image" src="https://user-images.githubusercontent.com/110976272/215448424-747eec8d-9d69-474d-b381-f906c25af52c.png">

The result is successful connectivity between the FW NVA and the On-Prem branches:

<img width="884" alt="image" src="https://user-images.githubusercontent.com/110976272/216486557-2d65d60d-d637-4e1e-a64a-cdbe7aa41bbd.png">

# 4.3. Step 2: End-to-end Connectivity and FW NVA transit between Spoke1 VNET and the branches

Now that the connectivity between the FW NVA and the On-Prem branches via the Concentrator NVA is confirmed, in this section we will see how to extend this connectivity end-to-end and provide FW transit between the Spoke1 VNET and the branches.

## 4.3.1. Spoke1 => On-Prem FW transit

In this Episode, we want the VMs in Spoke1 to transit via the FW NVA and the Concentrator NVA for On-Prem branches reachability, as opposed to direct access to the FW Concentrator like in Episode #3.

Based on the default [VNET peering rules](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) discussed in [Episode #1](https://github.com/cynthiatreger/az-routing-guide-ep1-vnet-peering-and-virtual-network-gateways), and since Spoke1 VNET has been dissociated from any *Route table* at the beginning of this article, the Spoke1 VMs don't have visibility of any IP range outside of the Spoke1 and Hub VNETs.

A UDR is again required to bring the On-Prem prefixes knowledge to the Spoke1 subnets, but with the Next-Hop becoming the FW NVA: a new *Route table* is associated to the Spoke1 subnets, with a UDR for the On-Prem branches (192.168.0.0/16) pointing to the FW NVA (10.0.0.5):

<img width="816" alt="image" src="https://user-images.githubusercontent.com/110976272/215566526-b9afd33f-9f6e-40e3-9d2a-69b419b914e3.png">

:warning: Remember to enable “IP Forwarding” on the FW NVA’s NIC receiving the traffic, or the packets will be dropped.

Let's check using traceroutes that the FW NVA is in the path:

<img width="1151" alt="image" src="https://user-images.githubusercontent.com/110976272/217297208-f79bf523-7c03-4014-a012-71cf5e2f74da.png">

Connectivity between the On-Prem and Spoke1 VNET is achieved. However only traffic from Spoke1 VNET to OnPrem is inspected by the FW, the return traffic bypasses the FW as demonstrated by the traceroute from the Concentrator NVA to Spoke1VM.

:arrow_right: It’s not only about traffic from Azure to OnPrem, the On-Prem has to find the right way back to Azure too.

## 4.3.2. On-Prem => Spoke1 FW transit

When packets destined to Spoke1VM reach the NIC of the Concentrator NVA, the destination (10.1.1.4) will be matched against the NIC *Effective routes* and forwarded over the peering to Spoke1 VNET directly.

One last layer of UDRs is required on the Concentrator NVA to force traffic to Spoke1 VNET towards the FW NVA. The "ConcentratorRT" *Route table* is updated accordingly with the "toSpoke1" UDR:

<img width="829" alt="image" src="https://user-images.githubusercontent.com/110976272/217299758-a4d170a3-fd1e-471e-ae41-50e821cb665e.png">

This completes the return inspection of traffic from the On-Prem branches to Spoke1VNET:

<img width="1186" alt="image" src="https://user-images.githubusercontent.com/110976272/217298063-1b557006-75a9-4d61-8d14-6f6e4a90d170.png">

:arrow_right: *Default* routes overridden by UDRs become "Invalid" and a new *User* entry is added at the bottom of the *Effective routes*. 

## 4.4. Conclusion

Finally both connectivity & FW inspection between Azure resources and branches connected to a Concentrator NVA have been successfully completed!

But admittedly with quite some complexity, as UDRs are required on every subnet of every VNET, moreover in distinct *Route tables* depending on the environment (Spoke, FW, Concentrator), in order to match the OS routing level view.

And we haven't considered the scalability of this solution, which would add up to this heaviness should there be more Spokes, more subnets, more branch prefixes, less aggregation possibilities etc.

Fortunately there is room for improvement and ways to simplify both the deployment and the management, as we will find out in the next Episode!
##
### [>> EPISODE #5](https://github.com/cynthiatreger/az-routing-guide-ep4-nva-routing-2-0)
