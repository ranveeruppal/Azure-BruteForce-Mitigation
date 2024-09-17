# Azure Brute Force Mitigation Project using Fortinet Next-Generation Firewall and Microsoft Sentinel

## Skills Learned
- Network security and traffic flow management in Azure.
- Firewall configuration, routing, and access control.
- Log management and integration with Microsoft Sentinel.
- Custom intrusion detection rules.
- Incident response workflow in a cloud environment.

## Tools Used
- **Azure**: Resource groups, virtual networks, routing tables, Log Analytics, Azure Monitor, Microsoft Sentinel.
- **Fortinet Next-Generation Firewall**: Intrusion prevention system, traffic policies, and logging.
- **Windows & Linux Virtual Machines**: RDP and syslog configuration.
- **Azure Monitor Agent**: Log forwarding and management.
- **Custom IPS Rules**: Detection and prevention of brute-force attacks.

---

## Objective
This Azure brute force mitigation project aimed to establish a functional virtual network in Azure that contains the Fortinet NGFW configured to prevent brute force RDP attacks on our Windows virtual machine. The focus was to utilize the Intrusion Prevention System (IPS) on the NGFW to block brute-force attacks and create logs that are sent to Microsoft Sentinel on the Azure cloud to create alerts for security analysts to conduct real-time incident response.

---

## Steps

1. First, I created a new resource group called `SOC_lab` to contain all the resources required for this project.

2. I then created a VNet with the following security settings, along with three different subnets: one for the Bastion service, the WAN, and the DMZ, respectively.

3. I then installed the FortiGate Next-Generation Firewall, the configuration was as follows:
    - The external subnet (connected to the internet) is going to be the `LAB_WAN` subnet created in step two.
    - The internal subnet will be the `LAB_DMZ`. The subnet IPs are `10.10.0.0/24` and `10.10.200.0/24` respectively.
    - The public IP assigned to the Fortinet firewall by Azure will be translated to its IP within the external subnet `10.10.0.4/24`.

4. The Fortinet Next-Generation Firewall was successfully deployed and can be accessed through its public IP address.

5. As displayed in the diagram, we have two interfaces: port 1 and port 2. To configure these for this lab, I disabled HTTPS on port 1 (connected to the internet) and disabled SSH on the DMZ port (port 2).

6. The static routing is described in the screenshot below. All traffic from the internet will go through the Azure gateway in the `LAB_WAN` subnet, which will then (through the firewall) be routed through the `LAB_DMZ` gateway to reach our virtual machines.

7. I then added a Windows virtual machine to the resource group. Because I only want to expose this virtual machine to the internet through the firewall (as opposed to the Azure network), I blocked public inbound ports. The settings and configurations for the virtual machine are also attached.

8. I set up a second Linux virtual machine with the following configurations.

9. Now that all the subnets, firewalls, and virtual machines have been deployed into my resource group, I had to make sure that the flow of the traffic was configured correctly, where all traffic flows through the Fortinet firewall. This was done through route tables.

10. I first created a route table called `DMZ-route-table` for the DMZ subnet. This table routes all traffic from the DMZ subnet destined for the internet through the firewall (IP address: `10.10.200.4`) in a route called `to-internet`. This route table was associated with the DMZ subnet.

11. A second route table (`WAN-route-table`) was created to route traffic coming from the internet destined for our virtual machines through the Fortinet firewall and then through the DMZ.

12. I then tested the connection between the Windows machine and the Fortinet firewall and between the Windows machine and the internet to ensure the pings were going through. However, for this to be possible, I had to allow internet-bound traffic to pass through the firewall by creating a new policy. This policy allows incoming traffic through the DMZ port and outgoing traffic through the WAN port. For testing purposes, I allowed all sources and all destinations, which wouldn't normally be the case in a production environment.

13. Since the incoming traffic also needs to be checked to see if it can reach the Windows virtual machine, I had to configure a virtual IP where the external IP of the firewall is mapped to the local IP of the Windows machine. I tested the incoming connection by using RDP on port 3389, which is why I also enabled port forwarding so traffic on the WAN subnet on port 3389 would be mapped to port 3389 on the Windows machine.

14. After disabling the Windows firewall on the virtual machine, I was able to successfully connect to the Azure Windows machine through an RDP connection from my local Windows 11 machine.

15. Before conducting the brute-force attacks, I created a new IPS (Intrusion Prevention System) policy to block these attacks and monitor them. The IPS signature offered by Fortinet for preventing brute-force attacks was not sensitive enough, so I created a custom signature. A brute-force attack would be detected if 5 wrong attempts were made within 20 seconds.

16. Testing confirmed that my IP was blocked after 5 failed attempts within 20 seconds.

The next stage of this project was to utilize Microsoft Sentinel to ingest the logs from Fortinet to create an alerting system and perform incident response. However, to send logs into Microsoft Sentinel, I had to configure a Log Analytics workspace along with Azure Monitor. Since Azure cannot directly ingest logs from the Fortinet firewall, the logs must first pass through syslogs on the Linux VM, which are then sent to the workspace in Azure.

17. To get the logs to the Log Analytics workspace, I installed the Azure Monitor Agent (as the Log Analytics workspace agent was deprecated). The Linux machine and Fortinet configurations were done according to the official Fortinet documentation.

18. I verified that the logs were being received on the Linux VM by checking the `/var/log` directory, and the logs were successfully forwarded.

19. On the Sentinel page in Azure, I verified that the logs were being ingested into Sentinel via the CEF (Common Event Format) connector.

20. Finally, I created a Sentinel query rule based on the IPS rule created earlier to block brute-force RDP attempts on the Windows machine. This query creates an alert anytime the IPS from the firewall is triggered. The alert is configured to map to the source IP that triggered the brute-force attack.

21. Checking the incidents in Sentinel, I confirmed that a new ticket had been created based on the IPS being triggered by the brute-force attack. This ticket can now be used to conduct incident response. The source IP has also been included in the ticket.

---

## Reference

Documentation and resources:
- [Fortinet FortiGate Integration with Microsoft Sentinel](https://community.fortinet.com/t5/FortiGate/Technical-Tip-FortiGate-Integration-with-Microsoft-Sentinel-via/ta-p/324681)
