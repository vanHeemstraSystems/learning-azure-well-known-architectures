# 100 - Azure N-Tier (Multi-tier) Architecture

## Overview

N-Tier architecture is a traditional architectural pattern that separates an application into logical layers, with each layer running on separate infrastructure. In Azure, this typically consists of a presentation tier, application/business logic tier, and data tier.

## Architecture Diagram Concept

```
Internet
    ↓
[Azure Application Gateway + WAF]
    ↓
[Azure Load Balancer]
    ↓
[Web Tier VMs] ← [Azure Bastion] (for management)
    ↓
[Azure Load Balancer]
    ↓
[Application Tier VMs]
    ↓
[Azure SQL Database / Data Tier]
```

## Key Components and Their Connections

### 1. **Azure Virtual Network (VNet)**

- **Purpose**: Provides network isolation and segmentation
- **Connection**: Contains all other resources; divided into multiple subnets
- **Subnets typically include**:
  - Gateway subnet (for Application Gateway)
  - Web tier subnet
  - Application tier subnet
  - Data tier subnet
  - Management subnet (for Azure Bastion)

### 2. **Azure Application Gateway**

- **Purpose**: Layer 7 load balancer with Web Application Firewall (WAF)
- **Connection**:
  - Receives HTTPS traffic from the internet
  - Routes traffic to the web tier load balancer or directly to web tier VMs
  - Performs SSL termination
  - Deployed in its own subnet within the VNet

### 3. **Azure Load Balancer (Web Tier)**

- **Purpose**: Layer 4 load balancer for distributing traffic across web servers
- **Connection**:
  - Receives traffic from Application Gateway
  - Distributes traffic evenly across multiple web tier VMs
  - Performs health probes to ensure VMs are responsive

### 4. **Web Tier Virtual Machines**

- **Purpose**: Host the presentation layer (web servers like IIS, Apache, Nginx)
- **Connection**:
  - Deployed in web tier subnet
  - Receive traffic from the web tier load balancer
  - Connect to application tier VMs for business logic
  - Protected by Network Security Groups (NSGs)
  - Can be accessed via Azure Bastion for management

### 5. **Network Security Groups (NSGs) - Web Tier**

- **Purpose**: Control inbound and outbound traffic to web tier subnet
- **Rules**:
  - **Inbound**: Allow HTTPS (443) from Application Gateway
  - **Outbound**: Allow traffic to application tier subnet
  - **Deny**: All other traffic by default

### 6. **Azure Load Balancer (Application Tier)**

- **Purpose**: Distributes requests from web tier to application servers
- **Connection**:
  - Receives requests from web tier VMs
  - Balances load across application tier VMs
  - Internal load balancer (not exposed to internet)

### 7. **Application Tier Virtual Machines**

- **Purpose**: Execute business logic and application processing
- **Connection**:
  - Deployed in application tier subnet
  - Receive requests from web tier VMs via load balancer
  - Connect to Azure SQL Database for data operations
  - Protected by NSGs
  - Accessed via Azure Bastion for management

### 8. **Network Security Groups (NSGs) - Application Tier**

- **Purpose**: Control traffic to application tier subnet
- **Rules**:
  - **Inbound**: Allow traffic only from web tier subnet
  - **Outbound**: Allow traffic to data tier (SQL Database)
  - **Deny**: All other traffic by default

### 9. **Azure SQL Database**

- **Purpose**: Managed relational database service for data storage
- **Connection**:
  - Deployed in data tier subnet (or uses Azure Private Link)
  - Receives connections only from application tier VMs
  - Uses Azure SQL firewall rules to restrict access
  - Can use VNet service endpoints or Private Link for secure connectivity

### 10. **Network Security Groups (NSGs) - Data Tier**

- **Purpose**: Control access to database resources
- **Rules**:
  - **Inbound**: Allow SQL traffic (1433) only from application tier subnet
  - **Deny**: All other traffic

### 11. **Azure Bastion**

- **Purpose**: Provides secure RDP/SSH access to VMs without exposing them to the internet
- **Connection**:
  - Deployed in a dedicated subnet called “AzureBastionSubnet”
  - Connects to all VMs in the VNet over private IP addresses
  - Users connect to Azure Bastion via the Azure portal using HTTPS
  - No public IP addresses needed on VMs

### 12. **Azure Key Vault**

- **Purpose**: Stores secrets, connection strings, and certificates
- **Connection**:
  - VMs access Key Vault to retrieve secrets
  - Uses Managed Identity for authentication
  - Accessed via VNet service endpoint or Private Link

## Traffic Flow Example

### User Request Flow:

1. **User** sends HTTPS request to public IP of Application Gateway
1. **Application Gateway** terminates SSL, applies WAF rules, forwards to Web Load Balancer
1. **Web Load Balancer** selects a healthy web tier VM
1. **Web Tier VM** processes request, calls Application Load Balancer
1. **Application Load Balancer** forwards to an application tier VM
1. **Application Tier VM** executes business logic, queries Azure SQL Database
1. **Azure SQL Database** returns data to application tier VM
1. **Response flows back** through application tier → web tier → Application Gateway → User

### Management Access Flow:

1. **Administrator** logs into Azure Portal
1. **Administrator** navigates to Azure Bastion service
1. **Azure Bastion** establishes RDP/SSH connection to target VM over private IP
1. **Administrator** manages VM securely without exposing it to the internet

## Security Benefits

- **Defense in Depth**: Multiple layers of security (NSGs, WAF, firewall rules)
- **Network Segmentation**: Each tier isolated in separate subnets
- **No Direct Internet Exposure**: Application and data tiers not accessible from internet
- **Secure Management**: Azure Bastion eliminates need for public IPs on VMs
- **Least Privilege**: NSGs enforce minimal required connectivity

## Scaling Considerations

- **Horizontal Scaling**: Add more VMs behind load balancers
- **Vertical Scaling**: Increase VM sizes for more compute power
- **Auto-scaling**: Use Virtual Machine Scale Sets for automatic scaling
- **Database Scaling**: Azure SQL Database supports elastic pools and hyperscale tiers

## Best Practices

1. **Use managed disks** for all VMs
1. **Enable Azure Monitor** for logging and diagnostics
1. **Implement backup strategy** using Azure Backup
1. **Use Availability Zones** for high availability
1. **Apply NSG rules** at both subnet and NIC level
1. **Rotate secrets** in Key Vault regularly
1. **Enable SSL/TLS** end-to-end where possible
1. **Use Azure Policy** to enforce governance standards

## When to Use N-Tier Architecture

✅ **Good for:**

- Traditional enterprise applications
- Applications with clear separation of concerns
- When you need full control over infrastructure
- Migrating on-premises applications to Azure (lift-and-shift)

❌ **Not ideal for:**

- Cloud-native applications (consider microservices)
- Applications requiring extreme scalability
- When you want to minimize infrastructure management

## Learning Resources

- Microsoft Learn: [N-tier architecture style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/n-tier)
- Azure Architecture Center: [N-tier application with Azure SQL Database](https://learn.microsoft.com/azure/architecture/reference-architectures/n-tier/n-tier-sql-server)
