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

## Network Topology

![Network Topology](https://i.imgur.com/your-image-id.png)

---

## Steps

1. First, I created a new resource group called `SOC_lab` to contain all the resources required for this project.

<img width="476" alt="Screenshot 2024-09-17 at 6 00 44 PM" src="https://github.com/user-attachments/assets/ec601d8a-1b45-4d51-953b-4ae77f16729b">

2. I then created a VNet with the following security settings, along with three different subnets: one for the Bastion service, the WAN, and the DMZ, respectively.

<img width="610" alt="Screenshot 2024-09-17 at 6 01 50 PM" src="https://github.com/user-attachments/assets/9ab92bc0-ff65-4998-914e-8500efd7ebe4">

3. I then installed the FortiGate Next-Generation Firewall, the configuration was as follows:
    - The external subnet (connected to the internet) is going to be the `LAB_WAN` subnet created in step two. The internal subnet will be the `LAB_DMZ`. The subnet IPs are `10.10.0.0/24` and 
      `10.10.200.0/24` respectively.
    - The public IP assigned to the Fortinet firewall by Azure will be translated to its IP within the external subnet `10.10.0.4/24`.

4. The Fortinet Next-Generation Firewall was successfully deployed and can be accessed through its public IP address.

<img width="984" alt="Screenshot 2024-09-17 at 6 03 46 PM" src="https://github.com/user-attachments/assets/28cf4edb-139e-44d9-8768-5fea498b4c24">

5. As displayed in the network topology, the firewall has two interfaces: port 1 and port 2. To configure these for this lab, I disabled HTTPS on port 1 (connected to the internet) and disabled SSH on the DMZ port (port 2).

6. The static routing is described in the screenshot below. All traffic from the internet will go through the Azure gateway in the `LAB_WAN` subnet, which will then (through the firewall) be routed through the `LAB_DMZ` gateway to reach our virtual machines.

<img width="1266" alt="Screenshot 2024-09-17 at 6 06 19 PM" src="https://github.com/user-attachments/assets/49abddd6-9ef1-49ea-840c-71b9591d13cc">

7. I then added a Windows virtual machine to the resource group. Because I only want to expose this virtual machine to the internet through the firewall (as opposed to the Azure network), I blocked public inbound ports. The settings and configurations for the virtual machine are also attached.

<img width="765" alt="Screenshot 2024-09-17 at 6 07 51 PM" src="https://github.com/user-attachments/assets/03364982-4667-4196-8ad4-1cdc9e14e0ef">

8. I set up a second Linux virtual machine with the following configurations.

<img width="380" alt="Screenshot 2024-09-17 at 6 08 31 PM" src="https://github.com/user-attachments/assets/6f46b3c7-403c-4dd0-b83f-0d30b8ced33e">

9. Now that all the subnets, firewalls, and virtual machines have been deployed into my resource group, I had to make sure that the flow of the traffic was configured correctly, where all traffic flows through the Fortinet firewall. This was done through route tables.

10. I first created a route called `To-Internet` for the DMZ subnet. This table routes all traffic from the DMZ subnet destined for the internet through the firewall (IP address: `10.10.200.4`) in a route called `to-internet`. In other words, all outbound traffic will have the next hop set as the DMZ port of the firewall (port 2).

<img width="918" alt="Screenshot 2024-09-17 at 6 09 11 PM" src="https://github.com/user-attachments/assets/83c9e4d2-247a-4b79-b61f-4e46ef40437e">

11. A second route (`to-DMZ`) was created to route traffic coming from the internet destined for our virtual machines through the Fortinet firewall and then through the DMZ.

<img width="457" alt="Screenshot 2024-09-17 at 6 11 48 PM" src="https://github.com/user-attachments/assets/cbed0eb4-116f-430e-8cac-d1d9c28c01e1">

12. I then tested the connection between the Windows machine and the Fortinet firewall and between the Windows machine and the internet to ensure pings were going through. However, for this to be possible, I had to allow internet-bound traffic to pass through the firewall by creating a new policy. This policy allows incoming traffic through the DMZ port and outgoing traffic through the WAN port. For testing purposes, I allowed all sources and all destinations, which wouldn't normally be the case in a production environment.

<img width="509" alt="Screenshot 2024-09-17 at 6 12 47 PM" src="https://github.com/user-attachments/assets/534c439f-7bf1-431b-803e-d6f2184922cd">

13. Since the incoming traffic also needs to be checked to see if it can reach the Windows virtual machine, I had to configure a virtual IP where the external IP of the firewall is mapped to the local IP of the Windows machine. I tested the incoming connection by using RDP on port 3389, which is why I also enabled port forwarding so traffic on the WAN subnet on port 3389 would be mapped to port 3389 on the Windows machine.

<img width="639" alt="Screenshot 2024-09-17 at 6 14 08 PM" src="https://github.com/user-attachments/assets/8712b9cb-7650-494d-b264-913eaeaa3d88">

14. After disabling the Windows firewall (Not our NGFW) on the virtual machine, I was able to successfully connect to the Azure Windows machine through an RDP connection from my local Windows 11 machine.

<img width="985" alt="Screenshot 2024-09-17 at 6 15 19 PM" src="https://github.com/user-attachments/assets/f28be326-a1ff-417b-84c3-cd657425e911">

15. Before conducting the brute-force attacks, I created a new IPS (Intrusion Prevention System) policy to block these attacks and monitor them. The IPS signature offered by Fortinet for preventing brute-force attacks was not sensitive enough, so I created a custom signature. A brute-force attack would be detected if 5 wrong attempts were made within 20 seconds.

<img width="883" alt="Screenshot 2024-09-17 at 6 17 05 PM" src="https://github.com/user-attachments/assets/0470d4dc-6654-48f7-a5b0-65e453741421">

16. Testing confirmed that my IP was blocked after 5 failed attempts within 20 seconds.

<img width="1006" alt="Screenshot 2024-09-17 at 10 42 42 PM" src="https://github.com/user-attachments/assets/3e138757-b699-4a06-b891-ddc6b109ebc0">

The next stage of this project was to utilize Microsoft Sentinel to ingest the logs from Fortinet to create an alerting system and perform incident response. However, to send logs into Microsoft Sentinel, I had to configure a Log Analytics workspace along with Azure Monitor. Since Azure cannot directly ingest logs from the Fortinet firewall, the logs must first pass through syslogs on the Linux VM, which are then sent to the workspace in Azure.

17. To get the logs to the Log Analytics workspace, I installed the Azure Monitor Agent (as the Log Analytics workspace agent was deprecated). The Linux machine and Fortinet configurations were done according to the official Fortinet documentation.

18. I verified that the logs were being received on the Linux VM by checking the `/var/log` directory, and the logs were successfully forwarded.

<img width="1384" alt="Screenshot 2024-09-17 at 6 18 29 PM" src="https://github.com/user-attachments/assets/dfcc2601-96b6-4bce-ba89-0c97757f4a07">

19. On the Sentinel page in Azure, I verified that the logs were being ingested into Sentinel via the CEF (Common Event Format) connector.

<img width="952" alt="Screenshot 2024-09-17 at 6 18 58 PM" src="https://github.com/user-attachments/assets/828d04d1-4fd1-4070-a580-4889f091db9b">

20. Finally, I created a Sentinel query rule based on the IPS rule created earlier to block brute-force RDP attempts on the Windows machine. This query creates an alert anytime the IPS from the firewall is triggered. The alert is configured to map to the source IP that triggered the brute-force attack.

<img width="991" alt="Screenshot 2024-09-17 at 6 19 13 PM" src="https://github.com/user-attachments/assets/ac440e49-6c63-4f2f-9c55-5dead2b7e65c">

21. Checking the incidents in Sentinel, I confirmed that a new ticket had been created based on the IPS being triggered by the brute-force attack. This ticket can now be used to conduct incident response. The source IP has also been included in the ticket.
<img width="502" alt="Screenshot 2024-09-17 at 6 19 55 PM" src="https://github.com/user-attachments/assets/2361b7c1-e305-4b2e-b5a8-a931f4fc350f">

---

## Reference

Documentation and resources:
- [Fortinet FortiGate Integration with Microsoft Sentinel](https://community.fortinet.com/t5/FortiGate/Technical-Tip-FortiGate-Integration-with-Microsoft-Sentinel-via/ta-p/324681)
