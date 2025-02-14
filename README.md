# vWAN-Routing-Intent-with-Forced-Tunneling

# Intro
In this article we are going to discuss about Azure Virtual WAN with Routing Intent enabled For Internet inspection (Internet Breakout) while also force tunneling from on-premise. There are still requirements to force tunnel onprem for inspection or regulatory requirements. FOr this reason, folks sometimes want to do both. They want to inspect inside the vhub, but also still send that traffic on-premise to be inspected! This force tunneling can be over IPSEC, SDWan, or Express-Route. For demonstration purposes, I am going to advertise a 0.0.0.0/0 from on-premise over express-route via BGP while also enabling routing intent on the virtual hub and checking that behavior. We will go over various work-arounds to this behavior and also other gotchas that it affects as well! Information on what routing intent is, can be found [here](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-routing-policies)

> [!NOTE]
> Officially advertising a 0.0.0.0/0 from On-Premise while also enabling routing-intent for internet breakout is NOT supported at the time of this article, and we will demonstrate below why! This is mentioned via this [link](https://learn.microsoft.com/en-us/azure/firewall/forced-tunneling)
> ![image](https://github.com/user-attachments/assets/8d1be4b9-3d90-4524-9987-f0e4d135c011)

# Simple Base Topology
![image](https://github.com/user-attachments/assets/4cb363ae-a85a-499a-8ad7-46405b435035)

# What happens when I advertise 0/0 from OnPrem and RI injects 0/0 down Express-Route and the Spoke?
First, lets check to see that express-route is learning the default route from On-Prem and also lets check the effective routes of the VM in our hub vnet before enabling routing-intent on our vWAN hub!
![image](https://github.com/user-attachments/assets/33ab8e4e-3549-429f-839e-97ae6cc57bba)
We can see the first line, that the circuit is learning 0.0.0.0/ from on-premise and we can see the other two prefixes as well the 10.4.0.0/16 and 10.0.0.0/24 from On-Prem. Lets go ahead and check the VM effective routes of the VM in the hub as well:
![image](https://github.com/user-attachments/assets/0b45cb4d-ab1d-44cb-a7f7-954027aaa9f4)
<br>
We can see the VM is learnign the 0.0.0.0/0 with next hop the MSEE Physical Address. Egress traffic for Express-Route bypasses the GW and the next hop is the [MSEE physical address](https://github.com/adtork/ExpressRoute--What-is-this-IP-/blob/main/README.md).
Ignore the other prefixes, this same circuit is also hooked to another vWAN, so we can ignore for now. Now, lets enable routing intent for internet breakout in the Azure Firewall inside the vhub and re-check the circuit and VM effective routes!
<br>
![image](https://github.com/user-attachments/assets/937d73ee-f979-4b68-9d12-b948058f8f9b)

