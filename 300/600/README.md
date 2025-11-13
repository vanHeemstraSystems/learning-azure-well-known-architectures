# 600 - Azure Hybrid Cloud Architecture

## Overview

Hybrid Cloud Architecture connects on-premises infrastructure with Azure cloud services, enabling organizations to extend their datacenter to the cloud. This approach allows gradual cloud adoption, meets data residency requirements, maintains legacy system compatibility, and provides flexibility in workload placement. Hybrid is the reality for most enterprises during their cloud journey.

## Architecture Diagram Concept

```
┌─────────────────────────────────────────┐
│        ON-PREMISES DATACENTER           │
│  ┌──────────────────────────────────┐  │
│  │  VMs, Databases, Applications    │  │
│  │  Active Directory                │  │
│  │  File Servers                    │  │
│  └──────────────────────────────────┘  │
└──────────────┬──────────────────────────┘
               ↕
    [ExpressRoute / VPN Gateway]
               ↕
┌──────────────┴──────────────────────────┐
│           AZURE CLOUD                   │
│  ┌──────────────────────────────────┐  │
│  │  Hub VNet (Hybrid Connectivity)  │  │
│  │  - VPN/ER Gateway                │  │
│  │  - Azure Firewall                │  │
│  │  - Azure AD Connect              │  │
│  └──────────────────────────────────┘  │
│               ↕                         │
│  ┌──────────────────────────────────┐  │
│  │  Spoke VNets (Workloads)         │  │
│  │  - Azure VMs                     │  │
│  │  - Azure SQL                     │  │
│  │  - App Services                  │  │
│  └──────────────────────────────────┘  │
│  ┌──────────────────────────────────┐  │
│  │  Azure Arc (Manage On-Prem)     │  │
│  │  Azure Backup                    │  │
│  │  Azure Monitor                   │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Key Components and Their Connections

### Connectivity Layer

#### 1. **Azure VPN Gateway**

- **Purpose**: Encrypted connectivity between on-premises and Azure over internet
- **Connection**:
  - Deployed in hub VNet’s GatewaySubnet
  - Terminates Site-to-Site VPN tunnels from on-premises
  - IPsec/IKE encryption for all traffic
  - BGP support for dynamic routing

**VPN Gateway Types:**

- **Site-to-Site (S2S)**: Connect entire on-premises network to Azure
- **Point-to-Site (P2S)**: Individual users/devices connect to Azure
- **VNet-to-VNet**: Connect Azure VNets across regions

**VPN Gateway SKUs:**

- **Basic**: 10 tunnels, 100 Mbps (deprecated, avoid for new deployments)
- **VpnGw1-5**: 30-100 tunnels, 650 Mbps - 10 Gbps
- **VpnGw1-5AZ**: Zone-redundant versions (higher SLA)

**Configuration Requirements:**

- On-premises VPN device (Cisco, Palo Alto, pfSense, etc.)
- Public IP address for on-premises VPN device
- Non-overlapping IP address spaces
- Pre-shared key for authentication

**Routing:**

- **Static Routing**: Manually configured routes
- **BGP (Border Gateway Protocol)**: Dynamic route exchange (recommended)

#### 2. **Azure ExpressRoute**

- **Purpose**: Private, dedicated connection to Azure (not over internet)
- **Connection**:
  - Deployed in hub VNet’s GatewaySubnet
  - Physical connection via connectivity provider (AT&T, Verizon, Equinix)
  - Dedicated bandwidth from 50 Mbps to 100 Gbps
  - Predictable latency and reliability

**ExpressRoute vs VPN:**

|Feature   |ExpressRoute        |VPN Gateway            |
|----------|--------------------|-----------------------|
|Connection|Private, dedicated  |Over public internet   |
|Bandwidth |Up to 100 Gbps      |Up to 10 Gbps          |
|Latency   |Low, predictable    |Variable               |
|Cost      |$$ (higher)         |$ (lower)              |
|Setup Time|Weeks (provider)    |Hours                  |
|Best For  |Production workloads|Dev/test, remote access|

**ExpressRoute Peering Types:**

- **Private Peering**: Connect to Azure VNets (VMs, databases)
- **Microsoft Peering**: Connect to Microsoft 365, Dynamics 365, Azure PaaS

**ExpressRoute SKUs:**

- **Local**: Connect to Azure region in same metro
- **Standard**: Connect to any Azure region in geopolitical region
- **Premium**: Global connectivity, more routes, Office 365 connectivity

**High Availability:**

- Dual circuits for redundancy
- Zone-redundant gateways
- Multiple peering locations

#### 3. **Azure Virtual WAN**

- **Purpose**: Unified connectivity and routing across branches, VNets, and users
- **Connection**:
  - Hub-and-spoke as a service
  - Integrates VPN, ExpressRoute, P2S VPN
  - Optimized routing between locations
  - Global transit network

**Virtual WAN Components:**

- **Virtual Hub**: Managed hub VNet with routing
- **VPN Gateway**: S2S and P2S VPN in hub
- **ExpressRoute Gateway**: ER connectivity in hub
- **Azure Firewall**: Integrated security

**Use Cases:**

- Branch office connectivity (SD-WAN)
- Global enterprise networks
- Multi-region hybrid deployments
- Simplified management vs. manual hub-spoke

### Identity Integration

#### 4. **Azure AD Connect**

- **Purpose**: Synchronize on-premises Active Directory to Azure AD
- **Connection**:
  - Installed on Windows Server in on-premises network
  - Syncs users, groups, passwords to Azure AD
  - Enables single sign-on across on-premises and cloud

**Sync Methods:**

- **Password Hash Synchronization (PHS)**: Hash of password synced to Azure AD (recommended)
- **Pass-Through Authentication (PTA)**: Password verified on-premises
- **Federation (ADFS)**: On-premises ADFS validates credentials

**Features:**

- **Selective Sync**: Sync specific OUs or groups
- **Attribute Filtering**: Control which attributes sync
- **Device Writeback**: Sync Azure AD registered devices to on-prem AD
- **Group Writeback**: Office 365 groups to on-prem AD

**High Availability:**

- Staging server for disaster recovery
- Multiple sync servers (one active, others standby)

#### 5. **Azure AD Domain Services**

- **Purpose**: Managed Active Directory domain services in Azure
- **Connection**:
  - Provides domain join, LDAP, Kerberos, NTLM
  - Syncs from Azure AD
  - No need to deploy/manage domain controllers in Azure
  - Integrates with on-premises AD via Azure AD Connect

**Use Cases:**

- Lift-and-shift applications requiring domain join
- Applications using LDAP, Kerberos
- Legacy applications needing Windows authentication

#### 6. **Azure Active Directory (Azure AD)**

- **Purpose**: Cloud-based identity and access management
- **Connection**:
  - Central authentication for Azure and Microsoft 365
  - Integrates with on-premises AD via Azure AD Connect
  - Conditional Access for hybrid scenarios
  - Single Sign-On (SSO) across cloud and on-premises

### Data Services

#### 7. **Azure SQL Managed Instance**

- **Purpose**: Near 100% compatible SQL Server in Azure
- **Connection**:
  - Deployed in VNet with private IP
  - Transactional replication from on-premises SQL Server
  - VNet peering or VPN/ER for connectivity
  - Supports linked servers to on-premises

**Migration Paths:**

- **Online**: Minimal downtime using Azure Database Migration Service
- **Offline**: Backup/restore from on-premises
- **Transactional Replication**: Keep on-prem as primary, Azure as secondary

#### 8. **Azure Files / Azure File Sync**

- **Purpose**: Cloud file shares with SMB protocol
- **Connection**:
  - Azure File Sync agent on on-premises file servers
  - Bidirectional sync between on-prem and Azure Files
  - Cloud tiering: Move cold data to Azure, keep hot data on-prem
  - Accessible via SMB from on-premises

**Use Cases:**

- Lift-and-shift file server workloads
- Multi-site file sharing
- Backup and disaster recovery for file shares
- Cloud tiering for storage optimization

#### 9. **Azure Stack HCI**

- **Purpose**: Hyperconverged infrastructure running on-premises with Azure integration
- **Connection**:
  - On-premises hardware running Azure Stack HCI OS
  - Integrates with Azure for backup, monitoring, updates
  - Run VMs on-premises with Azure-consistent management
  - Stretch clusters across sites for HA

### Management & Monitoring

#### 10. **Azure Arc**

- **Purpose**: Extend Azure management to on-premises and multi-cloud
- **Connection**:
  - Lightweight agent on on-premises servers, Kubernetes clusters
  - Manage servers and Kubernetes from Azure portal
  - Apply Azure Policy, RBAC to on-premises resources
  - Deploy Azure services to on-premises (Arc-enabled data services)

**Azure Arc Capabilities:**

- **Arc-enabled Servers**: Manage Windows/Linux servers anywhere
- **Arc-enabled Kubernetes**: Manage K8s clusters anywhere
- **Arc-enabled Data Services**: Run Azure SQL MI, PostgreSQL on-premises
- **Arc-enabled SQL Server**: Manage SQL Servers with Azure tools

**Benefits:**

- Unified management plane (Azure portal)
- Consistent governance (Azure Policy)
- Azure services on-premises
- Hybrid security posture management

#### 11. **Azure Monitor**

- **Purpose**: Unified monitoring for hybrid environments
- **Connection**:
  - Log Analytics agent on on-premises servers
  - Collects logs, metrics from on-premises and Azure
  - Single pane of glass for entire infrastructure
  - Alerts span on-premises and cloud

**Monitoring Capabilities:**

- VM Insights: Performance monitoring
- Network Performance Monitor: Monitor VPN/ER connectivity
- Service Map: Visualize dependencies across hybrid
- Log Analytics: Query logs across all resources

#### 12. **Azure Automation**

- **Purpose**: Runbook automation across hybrid environments
- **Connection**:
  - Hybrid Runbook Worker on on-premises servers
  - Execute runbooks against on-premises resources
  - Update management for on-premises servers
  - Configuration management with Desired State Configuration (DSC)

**Automation Use Cases:**

- Start/stop VMs on-premises and Azure
- Patch management across hybrid
- Configuration drift remediation
- Backup orchestration

#### 13. **Azure Backup**

- **Purpose**: Backup and restore for hybrid workloads
- **Connection**:
  - MARS agent for file/folder backup from on-premises
  - Azure Backup Server for comprehensive backup (VMs, SQL, SharePoint)
  - Backup data stored in Recovery Services vault in Azure
  - Instant restore to on-premises or Azure

**Backup Capabilities:**

- On-premises VMs (Hyper-V, VMware)
- Physical servers (Windows, Linux)
- SQL Server, Exchange, SharePoint
- Long-term retention (up to 99 years)
- Immutable backups (ransomware protection)

#### 14. **Azure Site Recovery**

- **Purpose**: Disaster recovery orchestration
- **Connection**:
  - Configuration server on-premises
  - Replicates VMs to Azure continuously
  - Automated failover to Azure on disaster
  - Failback to on-premises when recovered

**DR Scenarios:**

- On-premises VMware to Azure
- On-premises Hyper-V to Azure
- Physical servers to Azure
- Azure to Azure (region to region)

### Security Services

#### 15. **Azure Firewall**

- **Purpose**: Central network security for hybrid traffic
- **Connection**:
  - Deployed in hub VNet
  - All hybrid traffic routes through firewall
  - Filters traffic between on-premises and Azure
  - Application and network rules

**Firewall Rules for Hybrid:**

- Allow on-premises subnets to specific Azure services
- Block unauthorized access from Azure to on-premises
- FQDN filtering for cloud services from on-premises

#### 16. **Azure Bastion**

- **Purpose**: Secure RDP/SSH access to Azure VMs
- **Connection**:
  - Provides access to Azure VMs without VPN
  - Useful for remote administrators
  - No need for VPN connection for VM management
  - MFA enforced via Azure AD

#### 17. **Microsoft Defender for Cloud**

- **Purpose**: Security posture management across hybrid
- **Connection**:
  - Agents on on-premises and Azure servers
  - Unified security recommendations
  - Threat protection for hybrid workloads
  - Just-in-time VM access

#### 18. **Azure Key Vault**

- **Purpose**: Centralized secrets management
- **Connection**:
  - Applications in Azure and on-premises access secrets
  - Private endpoint for secure access from on-premises
  - HSM-backed key storage
  - Certificate management for hybrid applications

### Network Services

#### 19. **Azure DNS Private Zones**

- **Purpose**: Private DNS resolution for hybrid scenarios
- **Connection**:
  - On-premises DNS forwards to Azure DNS
  - Azure VMs resolve on-premises names
  - Conditional forwarding between on-prem and Azure
  - Auto-registration for Azure resources

**DNS Configuration:**

```
On-Premises DNS:
  - Forward *.privatelink.* zones to Azure DNS (168.63.129.16)
  - Conditional forwarders for Azure private zones

Azure DNS Private Zones:
  - Auto-register Azure VM names
  - Link to VNets for resolution
```

#### 20. **Azure Load Balancer**

- **Purpose**: Distribute traffic across hybrid resources
- **Connection**:
  - Backend pools can include Azure VMs and on-premises servers
  - Requires VPN/ExpressRoute connectivity
  - Health probes ensure only healthy endpoints receive traffic

#### 21. **Azure Application Gateway**

- **Purpose**: Layer 7 load balancing for hybrid web apps
- **Connection**:
  - Backend pools include Azure and on-premises web servers
  - Path-based routing can direct traffic between locations
  - WAF protection for all backends

### Storage Services

#### 22. **Azure Storage Accounts**

- **Purpose**: Cloud storage accessible from on-premises
- **Connection**:
  - On-premises applications access via REST API
  - Private endpoints for secure access
  - StorSimple for hybrid storage appliances (legacy)

#### 23. **Azure Data Box**

- **Purpose**: Physical data transfer to/from Azure
- **Connection**:
  - Physical device shipped to on-premises
  - Load data locally, ship to Azure datacenter
  - Data uploaded to Azure Storage
  - Useful for TB/PB-scale initial migrations

## Hybrid Architecture Patterns

### Pattern 1: Lift and Shift (Rehost)

```
On-Premises VMs → Azure Migrate → Azure VMs
```

**Characteristics:**

- Minimal changes to applications
- Fastest cloud migration path
- Uses Azure VMs, managed disks
- Maintains hybrid connectivity for dependencies

### Pattern 2: Hybrid Data Tier

```
On-Premises Apps → VPN/ER → Azure SQL Database
```

**Characteristics:**

- Apps stay on-premises
- Database moves to Azure
- Reduced on-premises infrastructure
- Requires low-latency connection

### Pattern 3: Cloud Bursting

```
On-Premises (Primary) → Scale to Azure (Burst) during peak
```

**Characteristics:**

- On-premises handles normal load
- Azure handles peak/overflow traffic
- Azure Load Balancer distributes
- Cost-effective for variable workloads

### Pattern 4: Disaster Recovery

```
On-Premises (Primary) → Azure Site Recovery → Azure (DR Site)
```

**Characteristics:**

- Continuous replication to Azure
- Failover on disaster
- Failback when recovered
- No secondary datacenter needed

### Pattern 5: Hybrid Backup

```
On-Premises → Azure Backup → Recovery Services Vault
```

**Characteristics:**

- Eliminate tape backups
- Long-term retention in Azure
- Ransomware protection (immutable)
- Restore to on-prem or Azure

### Pattern 6: Dev/Test in Cloud

```
Production (On-Premises) ← Data → Dev/Test (Azure)
```

**Characteristics:**

- Production stays on-premises
- Dev/test moves to Azure
- Rapid environment provisioning
- Cost savings (Azure Dev/Test pricing)

## Traffic Flow Examples

### Example 1: User Accessing Hybrid Application

1. **User** on-premises accesses web application in Azure
1. **On-premises DNS** resolves app.contoso.com to Azure Application Gateway
1. **Traffic** routes through ExpressRoute to Azure
1. **Azure Firewall** inspects traffic (if in path)
1. **Application Gateway** routes to App Service in Azure
1. **App Service** queries Azure SQL Managed Instance
1. **App Service** also queries on-premises SQL Server via VPN/ER
1. **Response** returns through same path

### Example 2: Azure AD Authentication

1. **User** on-premises logs into Windows (domain-joined)
1. **User** opens Office 365 in browser
1. **Azure AD** recognizes user (synced via Azure AD Connect)
1. **Seamless SSO** authenticates without re-entering password
1. **User** accesses Azure resources with same identity

### Example 3: File Access via Azure File Sync

1. **User** on-premises accesses file on local file server
1. **File** exists on-premises (hot data)
1. **Azure File Sync** keeps file synced to Azure Files
1. **User in another office** accesses same file from their local server
1. **Azure File Sync** provides file (synced from Azure)
1. **Both users** see consistent file version

### Example 4: Disaster Recovery Failover

1. **On-premises datacenter** experiences disaster (fire, flood)
1. **Azure Site Recovery** detects loss of connectivity
1. **Administrator** initiates failover in Azure portal
1. **ASR** brings up VMs in Azure from replicated disks
1. **DNS** updated to point to Azure VMs
1. **Users** access applications running in Azure
1. **Business continues** with minimal downtime (RTO ~15 min)

### Example 5: Backup and Restore

1. **SQL Server** on-premises backed up to Azure Backup
1. **Backup** stored in Recovery Services vault (encrypted)
1. **Disaster** corrupts on-premises SQL Server
1. **Administrator** initiates restore from Azure portal
1. **Options**: Restore to on-premises OR restore to Azure SQL Managed Instance
1. **Data recovered** with RPO of backup frequency (e.g., 12 hours)

## Security Best Practices

### 1. **Network Security**

- Use ExpressRoute for production (private connectivity)
- Deploy Azure Firewall in hub for traffic inspection
- NSGs on all subnets (Azure and on-premises if possible)
- Private endpoints for PaaS services
- Disable RDP/SSH from internet (use Azure Bastion)

### 2. **Identity Security**

- Azure AD Connect with PHS (most secure sync method)
- Conditional Access for hybrid scenarios
- MFA for all cloud access
- Privileged Identity Management for admins
- Regular access reviews

### 3. **Data Security**

- Encryption in transit (TLS 1.2+, IPsec)
- Encryption at rest (Azure Storage, SQL TDE)
- Private endpoints for data services
- Azure Information Protection for document classification
- Regular backups with immutable storage

### 4. **Compliance**

- Azure Policy for governance across hybrid
- Microsoft Defender for Cloud for compliance posture
- Azure Arc for on-premises policy enforcement
- Audit logs centralized in Log Analytics
- Regular compliance assessments

## Cost Optimization

1. **Right-Size Connectivity**:
- VPN for dev/test ($100-200/month)
- ExpressRoute for production ($50-10,000+/month based on bandwidth)
1. **Hybrid Benefit**:
- Use existing Windows Server licenses (up to 40% savings)
- Use existing SQL Server licenses (up to 55% savings)
1. **Reserved Instances**:
- Commit to 1-3 years for Azure VMs (up to 72% savings)
1. **Azure File Sync Cloud Tiering**:
- Keep hot data on-premises (fast access)
- Tier cold data to Azure (cheaper storage)
1. **Backup Optimization**:
- Differential/incremental backups
- Lifecycle policies for old backups (archive tier)
1. **Start Small**:
- Pilot workloads before full migration
- Incremental approach reduces risk and cost

## Common Challenges and Solutions

### Challenge 1: Network Latency

**Problem:** High latency over VPN impacts application performance
**Solutions:**

- Upgrade to ExpressRoute for lower latency
- Cache data locally where possible
- Optimize application to reduce chatty calls
- Consider moving latency-sensitive apps fully to Azure

### Challenge 2: Bandwidth Limitations

**Problem:** Limited bandwidth between on-premises and Azure
**Solutions:**

- Increase ExpressRoute bandwidth
- Implement caching and compression
- Schedule large data transfers during off-peak
- Use Azure Data Box for initial bulk transfer

### Challenge 3: Identity Integration Complexity

**Problem:** Complex on-premises AD with multiple forests
**Solutions:**

- Use Azure AD Connect with multi-forest sync
- Federation for complex scenarios
- Azure AD B2B for external users
- Gradual migration to cloud-only identities

### Challenge 4: Application Dependencies

**Problem:** Application has dependencies on on-premises services
**Solutions:**

- Map dependencies with Azure Migrate
- Migrate dependent services together
- Use hybrid connectivity temporarily
- Refactor application to remove dependencies

### Challenge 5: Compliance and Data Residency

**Problem:** Data must remain on-premises for compliance
**Solutions:**

- Use Azure Stack Hub/HCI for on-premises Azure services
- Hybrid data tier (compute in Azure, data on-prem)
- Azure Arc for on-premises management
- Choose Azure regions that meet compliance requirements

## Migration Phases

### Phase 1: Assess (Weeks 1-4)

- Inventory on-premises resources
- Identify dependencies with Azure Migrate
- Assess cloud readiness
- Estimate costs
- Define migration strategy

### Phase 2: Pilot (Weeks 5-12)

- Establish hybrid connectivity (VPN then ExpressRoute)
- Deploy hub-spoke network in Azure
- Migrate non-critical workloads
- Test disaster recovery
- Validate security and compliance

### Phase 3: Migrate (Months 3-12)

- Migrate applications in waves
- Tier 3 (least critical) → Tier 2 → Tier 1 (most critical)
- Use Azure Migrate for VMs
- Use Azure Database Migration Service for databases
- Implement Azure Backup and Site Recovery

### Phase 4: Optimize (Months 12+)

- Right-size resources based on actual usage
- Implement cost optimization
- Refactor applications for cloud-native services
- Decommission on-premises infrastructure
- Continuous monitoring and improvement

## When to Use Hybrid Cloud

✅ **Good for:**

- Organizations with existing on-premises investments
- Gradual cloud migration strategy
- Compliance or data residency requirements
- Applications with on-premises dependencies
- Disaster recovery and business continuity
- Dev/test in cloud, production on-premises

❌ **Not ideal for:**

- Greenfield applications (start cloud-native)
- Organizations ready for full cloud migration
- Simple applications without dependencies
- When hybrid connectivity costs outweigh benefits

## Learning Resources

- Microsoft Learn: [Hybrid cloud infrastructure](https://learn.microsoft.com/azure/architecture/solution-ideas/articles/hybrid-connectivity)
- Azure Architecture Center: [Hybrid networking](https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/)
- [Azure Hybrid Cloud documentation](https://azure.microsoft.com/solutions/hybrid-cloud-app/)
- [Azure Arc documentation](https://learn.microsoft.com/azure/azure-arc/)
- [Azure Migrate documentation](https://learn.microsoft.com/azure/migrate/)
- [ExpressRoute documentation](https://learn.microsoft.com/azure/expressroute/)
