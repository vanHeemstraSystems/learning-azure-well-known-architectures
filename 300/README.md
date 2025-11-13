# 300 - Learning Our Subject

I’ll create a comprehensive table mapping well-known Azure architectures to their key building blocks, including the services you’re studying.

|**Architecture Pattern**  |**Key Azure Building Blocks**                                                                                                                                                                                     |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**[100 - N-Tier (Multi-tier)](./100/README.md)**   |• Azure Virtual Machines<br>• Azure Load Balancer<br>• Azure Application Gateway<br>• Network Security Groups (NSGs)<br>• Azure Virtual Network<br>• Azure SQL Database<br>• Azure Bastion (for secure management)|
|**[200 - Microservices](./200/README.md)**         |• Azure Kubernetes Service (AKS)<br>• Azure Container Instances<br>• Azure API Management (API Gateway)<br>• Azure Service Bus<br>• Azure Application Gateway<br>• Azure Front Door<br>• Azure Container Registry |
|**[300 - Hub-Spoke Network](./300/README.md)**     |• Azure Virtual Network (multiple VNets)<br>• Azure Virtual Network Peering<br>• Azure Firewall<br>• Azure VPN Gateway<br>• Azure ExpressRoute<br>• Network Security Groups<br>• Azure Bastion (in hub)           |
|**[400 - Web Application (PaaS)](./400/README.md)**|• Azure App Service<br>• Azure Application Gateway (with WAF)<br>• Azure CDN<br>• Azure SQL Database<br>• Azure Key Vault<br>• Network Security Groups                                                            |
|**[500 - High Availability (HA)](./500/README.md)**|• Azure Load Balancer<br>• Azure Traffic Manager<br>• Azure Availability Zones<br>• Azure Virtual Machines (in availability sets)<br>• Azure Site Recovery                                                        |
|**[600 - Hybrid Cloud](./600/README.md)**          |• Azure VPN Gateway<br>• Azure ExpressRoute<br>• Azure Arc<br>• Azure Stack<br>• Azure Virtual Network<br>• Network Security Groups                                                                               |
|**[700 - Event-Driven](./700/README.md)**          |• Azure Event Grid<br>• Azure Event Hubs<br>• Azure Functions<br>• Azure Logic Apps<br>• Azure Service Bus<br>• Azure API Management                                                                              |
|**[800 - Big Data & Analytics](./800/README.md)**  |• Azure Synapse Analytics<br>• Azure Data Lake Storage<br>• Azure Databricks<br>• Azure HDInsight<br>• Azure Data Factory<br>• Azure Virtual Network (for private endpoints)                                      |
|**[900 - Serverless](./900/README.md)**            |• Azure Functions<br>• Azure Logic Apps<br>• Azure API Management<br>• Azure Cosmos DB<br>• Azure Event Grid<br>• Azure Storage                                                                                   |
|**[1000 - DevOps/CI-CD](./1000/README.md)**          |• Azure DevOps<br>• Azure Container Registry<br>• Azure Kubernetes Service<br>• Azure Virtual Machines (build agents)<br>• Azure Key Vault<br>• Network Security Groups                                           |
|**[1100 - Zero Trust Security](./1100/README.md)**   |• Azure Bastion<br>• Azure Firewall<br>• Network Security Groups<br>• Azure Private Link<br>• Azure AD (Entra ID)<br>• Azure Key Vault<br>• Azure Policy                                                          |
|**[1200 - DMZ/Perimeter Network](./1200/README.md)** |• Azure Firewall<br>• Azure Application Gateway (with WAF)<br>• Network Security Groups<br>• Azure Virtual Network<br>• Azure DDoS Protection<br>• Azure Bastion                                                  |

**Key Notes for Your Study:**

- **Azure Bastion** appears primarily in secure management scenarios (Hub-Spoke, Zero Trust) - it provides secure RDP/SSH access without exposing VMs to the public internet
- **Azure API Management (API Gateway)** is central to microservices and event-driven architectures
- **Network Security Groups** are foundational across almost all architectures for subnet/NIC-level security
- **Azure Virtual Machines** form the backbone of IaaS-based patterns (N-Tier, HA, Hybrid)

This mapping should help you understand how these components fit into broader architectural contexts as you prepare for your studies and potential interviews.​​​​​​​​​​​​​​​​
