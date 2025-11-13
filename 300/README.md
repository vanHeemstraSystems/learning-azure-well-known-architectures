# 300 - Learning Our Subject

I’ll create a comprehensive table mapping well-known Azure architectures to their key building blocks, including the services you’re studying.

|**Architecture Pattern**  |**Key Azure Building Blocks**                                                                                                                                                                                     |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|**[100 - N-Tier (Multi-tier)](./100/README.md)**   |• Azure Virtual Machines<br>• Azure Load Balancer<br>• Azure Application Gateway<br>• Network Security Groups (NSGs)<br>• Azure Virtual Network<br>• Azure SQL Database<br>• Azure Bastion (for secure management)|
|**Microservices**         |• Azure Kubernetes Service (AKS)<br>• Azure Container Instances<br>• Azure API Management (API Gateway)<br>• Azure Service Bus<br>• Azure Application Gateway<br>• Azure Front Door<br>• Azure Container Registry |
|**Hub-Spoke Network**     |• Azure Virtual Network (multiple VNets)<br>• Azure Virtual Network Peering<br>• Azure Firewall<br>• Azure VPN Gateway<br>• Azure ExpressRoute<br>• Network Security Groups<br>• Azure Bastion (in hub)           |
|**Web Application (PaaS)**|• Azure App Service<br>• Azure Application Gateway (with WAF)<br>• Azure CDN<br>• Azure SQL Database<br>• Azure Key Vault<br>• Network Security Groups                                                            |
|**High Availability (HA)**|• Azure Load Balancer<br>• Azure Traffic Manager<br>• Azure Availability Zones<br>• Azure Virtual Machines (in availability sets)<br>• Azure Site Recovery                                                        |
|**Hybrid Cloud**          |• Azure VPN Gateway<br>• Azure ExpressRoute<br>• Azure Arc<br>• Azure Stack<br>• Azure Virtual Network<br>• Network Security Groups                                                                               |
|**Event-Driven**          |• Azure Event Grid<br>• Azure Event Hubs<br>• Azure Functions<br>• Azure Logic Apps<br>• Azure Service Bus<br>• Azure API Management                                                                              |
|**Big Data & Analytics**  |• Azure Synapse Analytics<br>• Azure Data Lake Storage<br>• Azure Databricks<br>• Azure HDInsight<br>• Azure Data Factory<br>• Azure Virtual Network (for private endpoints)                                      |
|**Serverless**            |• Azure Functions<br>• Azure Logic Apps<br>• Azure API Management<br>• Azure Cosmos DB<br>• Azure Event Grid<br>• Azure Storage                                                                                   |
|**DevOps/CI-CD**          |• Azure DevOps<br>• Azure Container Registry<br>• Azure Kubernetes Service<br>• Azure Virtual Machines (build agents)<br>• Azure Key Vault<br>• Network Security Groups                                           |
|**Zero Trust Security**   |• Azure Bastion<br>• Azure Firewall<br>• Network Security Groups<br>• Azure Private Link<br>• Azure AD (Entra ID)<br>• Azure Key Vault<br>• Azure Policy                                                          |
|**DMZ/Perimeter Network** |• Azure Firewall<br>• Azure Application Gateway (with WAF)<br>• Network Security Groups<br>• Azure Virtual Network<br>• Azure DDoS Protection<br>• Azure Bastion                                                  |

**Key Notes for Your Study:**

- **Azure Bastion** appears primarily in secure management scenarios (Hub-Spoke, Zero Trust) - it provides secure RDP/SSH access without exposing VMs to the public internet
- **Azure API Management (API Gateway)** is central to microservices and event-driven architectures
- **Network Security Groups** are foundational across almost all architectures for subnet/NIC-level security
- **Azure Virtual Machines** form the backbone of IaaS-based patterns (N-Tier, HA, Hybrid)

This mapping should help you understand how these components fit into broader architectural contexts as you prepare for your studies and potential interviews.​​​​​​​​​​​​​​​​
