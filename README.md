# Deploying and Configuring an Azure Firewall to Protect a Virtual Network
# Intro
In this write-up, I deploy an Azure Firewall to control access, block unauthorized traffic, and protect my virtual network from threats. This is part of the security measures for my [Azure Honeynet SIEM Project]().

Below is the final architecture, showing the virtual network with two subnets: one for lab resources and another for the firewall, along with their IP ranges.

![project architecture](/images/FW%20Network%20Topology%20Final%20.jpg)

# Deploying the Azure Firewall and Setting Up Routing / Networking
First, a new subnet is created for the firewall.
![Firewall subnet creation](/images/FW-Subnet-Creation.png)

Next, the firewall is deployed and details like its name, virtual network, and public IP are specified.  
![Firewall deployment](/images/FW-Deploy.png)

To route all traffic through the firewall, a routing table is created with a default route (`0.0.0.0/0`), using the firewall’s private IP as the next hop. This route is then associated with the `default` subnet which contains the resources used in the Azure Honeynet SIEM Project. 
# Configuring the Firewall Rules and Policies
Firewall rules are organized into collections for DNAT, Network, and Application rules. A few test rules are configured within each collection.
![Firewall rule collections](/images/Firewall-Rules-Collections.png)

Two DNAT rules are set up to allow access to Azure resources in the `default` subnet:

- **RDP**: Maps the Windows VM’s private IP to port 3389.
- **SSH**: Maps the Linux VM’s private IP to port 22.

Traffic to the firewall’s public IP is translated based on destination port to the corresponding VM.  The Network Security Group (NSG) is also configured to allow traffic, e.g., SSH access to the Linux VM from a specific management IP
![DNAT Rules](/images/DNAT-Rules.png)
![Adding SSH DNAT rule](/images/Add-SSH-rule.png)

The **Application Rule** allows or denies traffic based on FQDNs or web categories. A test rule permitting `google.com` blocks other sites by default. Attempting to access `portal.azure.com` results in a deny action:  
![Testing App rule Google.com](/images/FW-App-Rule-Test.png)

The rule also applies to the Linux VM, where non-Google requests fail:  
![Testing App rule on Linux](/images/Linux-FW-Test.png)
# IDPS Configuration and Testing and TLS Inspection
Azure Firewall Premium enables **TLS Inspection** and **Intrusion Detection & Prevention (IDPS)**. TLS Inspection allows the firewall to decrypt web traffic and inspect the entire packet, making it possible to detect threats hidden within encrypted communications. This is important for identifying malicious payloads, enforcing security policies, and preventing data exfiltration. Another consideration though, is the risk of firewall compromise. If all traffic, including business-critical data, is decrypted and inspected, the firewall itself becomes a high-value target.

IDPS operates using predefined signatures to detect and respond to threats in real time. Depending on its configuration, it can either generate alerts or actively block suspicious traffic. This is vital for identifying known attack patterns, such as exploit attempts, malware downloads, and unauthorized access. For this project, I enable IDPS in the firewall policy and trigger a test rule by downloading an ELF file from my Oracle VPS over HTTP (port 8081). The firewall logs confirm the detection, showcasing the effectiveness of IDPS in identifying potentially malicious activity.
![IDPS rule](/images/IDPS-Rule.png)

A simple ELF file is compiled and served via a Python simple HTTP server. The incoming connection from the Azure VM is visible, originating from the firewall’s public IP:  
![Python simple server http connections](/images/ELF%20Download%20Python%20Server.png)
![Downloading ELF file from Windows](/images/WindowsVM-Download-ELF.png)  
After downloading the file (`TEST_ELF`), firewall logs confirm the triggered rule, showing details like the source IP (VPS hosting the ELF file) and the rule signature ID along with a description of the rule.  
![Azure Logs ELF Download Rule](/images/ELF%20Download%20Logs.png)
# Monitoring
Firewall logs are collected via a **diagnostic rule** and sent to **Log Analytics Workspace** for queries and analysis. A sample query retrieves log details, showing web traffic from the Windows VM (`10.0.0.5`) to an external IP (`20.15.141.193`):  
![Firewall Azure Analytics Log Query](/images/FW-KQL.png)

A **workbook dashboard** visualizes firewall activity with key metrics:
### Allow vs Deny Actions

![Firewall Allow vs Deny Pie chart](/images/Azure%20Firewall%20Allows%20vs%20Denies.png)

```
AzureDiagnostics 
| where Category contains "AZFW" 
| summarize Count = count() by Action_s 
| render piechart
```
### Traffic Trends

![Firewall Traffic Trends Chart](/images/Traffic%20Trends.png)
```
AzureDiagnostics 
| where Category contains "AZFW" 
| summarize TotalTraffic = count() by bin(TimeGenerated, 1min), Action_s 
| render timechart
```
### Blocked Traffic by Destination IP

![Firewall Blocked Traffic by Desination IP Chart](/images/Blocked%20Traffic%20by%20Desination%20IP.png)
```
AzureDiagnostics 
| where Category contains "AZFWN" and Action_s == "Deny" 
| summarize BlockCount = count() by DestinationIp_s 
| top 10 by BlockCount 
| render barchart
```
# Further Improvements and Conclusion
- Fully implement **TLS Inspection**.
- Refine **workbook queries** for deeper insights.
- **Integrate with Microsoft Sentinel** to set up alerts and analytics rules using firewall logs.

This project demonstrated the deployment and configuration of an Azure Firewall, focusing on network security, traffic filtering, TLS inspection, and Intrusion Detection & Prevention (IDPS). By setting up firewall rules, DNAT policies, and Network Security Groups, I ensured controlled access to Azure resources. Testing IDPS with a simulated ELF file download validated its ability to detect and log suspicious activity. Diagnostic rules were also configured allowing ingestion into a Log Analytics workspace so that logs could be queried and custom workbooks could be generated to provided insights into firewall activity.
