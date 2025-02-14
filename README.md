# vWAN-Routing-Intent-with-Forced-Tunneling


# Intro
In this article we are going to discuss about Azure Virtual WAN with Routing Intent enabled For Internet inspection (Internet Breakout) while also force tunneling from on-premise. Often there are requirements to force tunnel onprem for inspection or regulatory requirements. For this reason, folks sometimes want to do both. They want to inspect inside the vhub, but also still send that traffic on-premise to be inspected! This force tunneling can be over IPSEC, SDWan, or Express-Route. For demonstration purposes, I am going to advertise a 0.0.0.0/0 from on-premise over express-route via BGP while also enabling routing intent on the virtual hub and checking that behavior. We will go over various work-arounds to this behavior and also other gotchas that it affects as well! Information on what routing intent is, can be found [here](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-routing-policies)

> [!NOTE]
> Officially advertising a 0.0.0.0/0 from On-Premise while also enabling routing-intent for internet breakout is NOT supported at the time of this article, and we will demonstrate below why! This is mentioned via this [link](https://learn.microsoft.com/en-us/azure/firewall/forced-tunneling)
> ![image](https://github.com/user-attachments/assets/8d1be4b9-3d90-4524-9987-f0e4d135c011)

> [!NOTE]
> The other point to make, if this was dual vhub topology (inter-hub) the 0.0.0.0/0 route does not [propogate across hubs](https://learn.microsoft.com/en-us/azure/virtual-wan/about-virtual-hub-routing#considerations). The quad zero route is confined per hub!

# Simple Base Topology
![image](https://github.com/user-attachments/assets/4cb363ae-a85a-499a-8ad7-46405b435035)

# What happens when I advertise 0/0 from OnPrem and RI injects 0/0 down Express-Route and the Spoke?

> [!NOTE]
> When testing propogation of the default route (0.0.0.0/0), this is assuming you are allowing default route propogation at the circuit level in the vwan hub is using Express-Route, and also at the connetion level. If either of these are turned off, the default route will not be propogated either by the circuit level, or from the secured vhub injecting the 0.0.0.0/0 route to spokes that are vnet peered to the vhub!
> <br>
> <br>
Circuit Level at the vHub:
> <br>
> <br>
> ![image](https://github.com/user-attachments/assets/233ad03d-3ac5-4550-9026-d038da26c2e6)
> <br>
> <br>
Vnet Peering to the vHub:
> <br>
> ![image](https://github.com/user-attachments/assets/7c093b4e-5d5a-45be-bd0c-1c5b7df7eed1)

First, lets check to see that express-route is learning the default route from On-Prem and also lets check the effective routes of the VM in our hub vnet before enabling routing-intent on our vWAN hub!
![image](https://github.com/user-attachments/assets/33ab8e4e-3549-429f-839e-97ae6cc57bba)
We can see the first line, that the circuit is learning 0.0.0.0/ from on-premise and we can see the other two prefixes as well the 10.4.0.0/16 and 10.0.0.0/24 from On-Prem. Lets go ahead and check the VM effective routes of the VM in the hub as well:
![image](https://github.com/user-attachments/assets/0b45cb4d-ab1d-44cb-a7f7-954027aaa9f4)
<br>
We can see the VM is learnign the 0.0.0.0/0 with next hop the MSEE Physical Address. Egress traffic for Express-Route bypasses the GW and the next hop is the [MSEE physical address](https://github.com/adtork/ExpressRoute--What-is-this-IP-/blob/main/README.md).
Ignore the other prefixes, this same circuit is also hooked to another vWAN, so we can ignore for now.
<br>
<br>
**Now, lets enable routing intent for internet breakout in the Azure Firewall inside the vhub and re-check the circuit, on-premise and VM effective routes!**
<br>
<br>
Now when we check on-premise routes, even though I am still advertising the 0.0.0.0/0 from on-prem, since we enabled Routing-Intent for internet breakout, on-prem is learning the 0.0.0.0/0 from the vWAN hub!
![image](https://github.com/user-attachments/assets/0c28d053-54e2-478f-91b7-630c4896cfef)
Next, lets take a look at the express-route circuit learned routes to see what changed too!
![image](https://github.com/user-attachments/assets/c5f18bb0-64a0-49bb-9fe8-e6907b8d1693)
Now we can see that we are no longer learning the 0.0.0.0/0 from On-Prem anymore and the express-route gateway is injecting those routes to the MSEE, which in turn sends them down to on-prem. Now, lets look at the VM effective routes in the hub as well!
<br>
<br>
![image](https://github.com/user-attachments/assets/0adda907-25b6-47e5-920a-4679f7256078)
<br>
We can see the VM is now learning the 0.0.0.0/0 via the internal FW IP inside the vhub and no longer hairpinning down and following on-prem like before. So, what does this really mean overall? It means the 0.0.0.0/0 being injected via routing-itent won over the 0.0.0.0/ being advertised from on-premise and you cannot do both! As we saw, the VM picks the default route being injected by vWAN and no longer learns from on-prem. On-prem in turns learns the 0.0.0.0/0 as well from the vhub! 

# But I want to do both, are there any work-arounds? 
The short answer, sort of! If you still want to advertise a default route from on-premise and send the traffic through the Azure Firewall or NVA inside the vhub, you can technically deploy an Azure firewall or NVA inside the vWAN hub and NOT turn on routing intent. Then from spoke Vnets, you can create a regular UDR, 0.0.0.0/0 next hop Azure Firewall or NVA inside the vhub. Keep in mind though, this is leaving the secured hub in sort of a "classic mode" and not using routing intent! So, if you then later on need multiple vhubs, you will **not be able to do interhub inspection that way**. You can also use a stacked or tiered vnet design, where you break the Azure Firewall or NVA out in a spoke and steer with UDRs. So, you would put a UDR on the default route table on the vhub pointing to a supernet of your spokes. Then, on the vnet connetions peered to the vhub, you would put each spoke address, next hop Azure Firewall/NVA. That design is following this article [here](https://learn.microsoft.com/en-us/azure/virtual-wan/scenario-route-through-nva). This is of course assuming in both scenarios you are leaving on default route propogation which I talked about above.

> [!NOTE]
There is also another sudo work-around, but I have tested this and it **breaks Azure Bastion** for example if you're using that service! I would **not** recommended this either as some other componnet services inside Azure are dependant on reaching out to Azure IPs for control plane updates and doing this could potentially cause issues. Technically though, you can split the 0.0.0.0/0 into smaller networks of 128.0.0.0/1 and 0.0.0.0/1. Since those networks are more specific then 0.0.0.0/0, traffic will still follow that route back on-premise, as well as go through the NVA/ Azure Firewall inside the vhub. This however totally breaks Bastion and I have not tested it on other services! 

Officially though as we saw, you cannot do both unless you don't turn off routing intent, or break the Azure Firewall/NVA out in a spoke which sort of defeats the purpose of using routing intent! 




