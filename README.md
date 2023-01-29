# Episode #4: Chained NVAs & BGP

Introduction note: This guide aims at providing a better understanding of the Azure routing mechanisms and how they translate from On-Prem networking. The focus will be on private routing in Hub & Spoke topologies. For clarity, network security and resiliency best practices as well as internet breakout considerations have been left out of this guide.
##

# 4.1. Test environment

The scenario used in [Episode #3](https://github.com/cynthiatreger/az-routing-guide-ep3-nva-routing-fundamentals) is now updated to reflect a common requirement of having a firewall between the On-Prem and the Azure environment. For the sake of this guide, this security layer is provided by another 3rd Party NVA rather than by our native Azure Firewall.

A second Cisco CSR ("*FW NVA*") is used for that purpose and deployed in a dedicated subnet in the Hub VNET (*FWsubnet*: 10.0.0.0/24).

*Depending on your architecture the Concentrator NVA could instead be in a dedicated peered Spoke VNET. The conclusions we will reach are the same.*

<img width="512" alt="image" src="https://user-images.githubusercontent.com/110976272/215352582-aafdad78-923b-496b-af52-27cd82ce5010.png">

As represented on the diagram, the expected traffic flows are:
-	Azure to On-Prem: Spokes > FW NVA > Concentrator NVA > Branches
-	On-Prem to Azure: Branches > Concentrator NVA > FW NVA > Spokes

## 4.2.	Step1: Connectivity between the FW NVA and the Branches
Since the branches are reachable from the Concentrator, this step consists in extending it to the Hub NVA.

