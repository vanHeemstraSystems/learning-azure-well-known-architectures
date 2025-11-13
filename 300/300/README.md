# 300 - Azure Hub-Spoke Network Architecture

## Overview

Hub-Spoke is a networking architecture pattern where a central hub virtual network connects to multiple spoke virtual networks. The hub acts as a central point for managing connectivity, security, and shared services, while spokes host workloads in isolation. This pattern is ideal for enterprises with multiple departments, applications, or environments.

## Architecture Diagram Concept

```
                    [On-Premises Network]
                            ↕
                    [ExpressRoute/VPN Gateway]
                            ↕
        ┌───────────────────────────────────────┐
        │         HUB VIRTUAL NETWORK           │
        │  ┌─────────────────────────────────┐  │
        │  │   Azure Firewall                │  │
        │  │   Azure Bastion                 │  │
        │  │   VPN/ExpressRoute Gateway      │  │
        │  │   Shared Services (DNS, AD)     │  │
        │  └─────────────────────────────────┘  │
        └───────────────────────────────────────┘
                ↕           ↕           ↕
        ┌───────────┐  ┌────────────┐  ┌──────────┐
        │  SPOKE 1  │  │  SPOKE 2   │  │ SPOKE 3  │
        │  (Prod)   │  │  (Dev)     │  │ (Test)   │
        │  VNet     │  │  VNet      │  │  VNet    │
        └───────────┘  └────────────┘  └──────────┘
```

## Key Components and Their Connections

### Hub Virtual Network Components

#### 1. **Hub Virtual Network (VNet)**

- **Purpose**: Central connectivity and security point
- **Connection**:
  - Peered to all spoke VNets
  - Connected to on-premises via VPN Gateway or ExpressRoute
  - Contains shared services used by all spokes
  - Typically has address space like 10.0.0.0/16

**Hub Subnets:**

- `AzureFirewallSubnet` (must be exactly this name)
- `AzureBastionSubnet` (must be exactly this name)
- `GatewaySubnet` (must be exactly this name)
- Shared services subnet (for DNS, Domain Controllers, etc.)

#### 2. **Azure Firewall**

- **Purpose**: Centralized network security and traffic inspection
- **Connection**:
  - Deployed in hub VNet’s AzureFirewallSubnet
  - All spoke-to-spoke and spoke-to-internet traffic routes through it
  - Has both public and private IP addresses
  - Central point for network and application rules

**Firewall Capabilities:**

- **Network Rules**: Allow/deny traffic based on IP, port, protocol
- **Application Rules**: Filter traffic based on FQDN (e.g., allow only *.microsoft.com)
- **NAT Rules**: DNAT for inbound internet traffic
- **Threat Intelligence**: Block known malicious IPs
- **DNS Proxy**: Firewall acts as DNS proxy for all spokes

**Traffic Flow Through Firewall:**

1. Spoke VNet uses User Defined Route (UDR)
1. UDR sets next hop to Azure Firewall private IP
1. All outbound traffic goes to Firewall first
1. Firewall evaluates rules and forwards or drops traffic

#### 3. **Azure Bastion**

- **Purpose**: Secure RDP/SSH access to VMs across all VNets
- **Connection**:
  - Deployed in hub VNet’s AzureBastionSubnet
  - Can access VMs in hub and all peered spoke VNets
  - Eliminates need for public IPs on VMs
  - Users connect via Azure Portal over HTTPS (443)

**Bastion Advantages in Hub-Spoke:**

- **Single Bastion Host**: One deployment serves all VNets
- **Cost Effective**: No need for Bastion in each spoke
- **Centralized Access Control**: Manage access from one location
- **Audit Logging**: All connections logged in hub

#### 4. **VPN Gateway or ExpressRoute Gateway**

- **Purpose**: Connect Azure to on-premises networks
- **Connection**:
  - Deployed in hub VNet’s GatewaySubnet
  - Provides hybrid connectivity
  - Routes on-premises traffic to appropriate spokes via hub

**VPN Gateway:**

- Site-to-Site VPN for encrypted internet-based connection
- Point-to-Site VPN for remote users
- Lower cost, good for smaller bandwidth needs

**ExpressRoute:**

- Private, dedicated connection via connectivity provider
- Higher bandwidth and reliability
- Lower latency than VPN
- More expensive, suitable for production workloads

#### 5. **Azure Virtual Network Peering (Hub to Spokes)**

- **Purpose**: Connect hub to each spoke VNet
- **Connection**:
  - Dedicated Azure backbone connection (not internet)
  - Low latency, high bandwidth
  - No gateways or public IPs needed
  - Each spoke peers with hub (not with other spokes)

**Peering Configuration:**

- **Hub to Spoke**: Allow gateway transit, allow forwarded traffic
- **Spoke to Hub**: Use remote gateway, allow forwarded traffic

#### 6. **Shared Services Subnet**

- **Purpose**: Host centralized services used by all spokes
- **Services Include**:
  - Active Directory Domain Controllers
  - DNS servers
  - Monitoring tools
  - Backup infrastructure
  - Update management servers

### Spoke Virtual Network Components

#### 7. **Spoke Virtual Networks**

- **Purpose**: Isolate workloads by environment, department, or application
- **Connection**:
  - Peered to hub VNet only
  - Cannot directly communicate with other spokes (by default)
  - Route through hub for spoke-to-spoke communication

**Common Spoke Scenarios:**

- **Production Spoke**: Production workloads
- **Development Spoke**: Development environments
- **Testing Spoke**: QA and testing
- **Department Spokes**: Finance, HR, IT departments

**Spoke Address Spaces:**

- Spoke 1: 10.1.0.0/16
- Spoke 2: 10.2.0.0/16
- Spoke 3: 10.3.0.0/16
- No overlapping IP addresses

#### 8. **Spoke Workload Resources**

- **Components**: Virtual Machines, App Services, AKS clusters, databases
- **Connection**:
  - Deployed in spoke subnets
  - Access shared services in hub
  - Internet access via hub’s Azure Firewall
  - No public IP addresses needed

#### 9. **Network Security Groups (NSGs) in Spokes**

- **Purpose**: Subnet-level security within each spoke
- **Connection**:
  - Applied to each spoke subnet
  - Control inbound/outbound traffic
  - Work in conjunction with Azure Firewall

**Example NSG Rules:**

- Allow traffic from hub subnet (for Bastion, shared services)
- Allow traffic between subnets in same spoke
- Deny traffic from other spokes (unless specifically allowed)
- All other traffic routed via hub’s firewall

#### 10. **User Defined Routes (UDRs)**

- **Purpose**: Force traffic through Azure Firewall
- **Connection**:
  - Applied to each spoke subnet
  - Override Azure’s default routing
  - Direct traffic to firewall for inspection

**Example UDR:**

```
Destination: 0.0.0.0/0 (all traffic)
Next Hop: Virtual Appliance (Azure Firewall private IP)
```

### Additional Components

#### 11. **Azure DDoS Protection**

- **Purpose**: Protect against DDoS attacks
- **Connection**:
  - DDoS Standard plan enabled on hub VNet
  - Protection automatically extends to peered spoke VNets
  - Monitors traffic to public IPs

#### 12. **Azure Monitor & Network Watcher**

- **Purpose**: Monitoring and diagnostics
- **Connection**:
  - Collects logs from all VNets
  - Flow logs for NSGs
  - Connection monitoring
  - Traffic analytics

#### 13. **Azure Policy**

- **Purpose**: Enforce governance across all VNets
- **Connection**:
  - Applied at subscription or resource group level
  - Ensures compliance (e.g., all VNets must peer with hub)
  - Prevents non-compliant resources

## Traffic Flow Examples

### Example 1: Spoke-to-Internet Traffic

1. **VM in Spoke 1** initiates connection to internet (e.g., updates)
1. **UDR in Spoke 1** routes traffic to Azure Firewall in hub
1. **Azure Firewall** evaluates application rules
1. **Firewall allows** traffic (e.g., to *.windowsupdate.com)
1. **Firewall performs SNAT**, sends traffic to internet with firewall’s public IP
1. **Response traffic** returns to firewall, forwarded back to spoke VM

### Example 2: Spoke-to-Spoke Communication

1. **VM in Spoke 1** needs to connect to VM in Spoke 2
1. **UDR in Spoke 1** routes to Azure Firewall
1. **Azure Firewall** evaluates network rules
1. **Firewall allows** connection (if rule permits)
1. **Traffic forwarded** to Spoke 2 VNet via peering
1. **VM in Spoke 2** receives connection, responds through firewall

### Example 3: On-Premises to Spoke

1. **User on-premises** connects to ExpressRoute/VPN Gateway
1. **Gateway in hub** receives traffic
1. **Hub routing** directs to appropriate spoke via peering
1. **Spoke VM** receives connection
1. **Response traffic** routes back through hub gateway

### Example 4: Administrator Access via Bastion

1. **Admin** logs into Azure Portal
1. **Admin** navigates to target VM in Spoke 2
1. **Admin** clicks “Connect” via Bastion
1. **Azure Bastion in hub** establishes RDP/SSH to spoke VM over private IP
1. **Connection traverses** VNet peering to spoke
1. **No public IP** needed on spoke VM

### Example 5: Shared Service Access

1. **VM in Spoke 3** needs to resolve DNS
1. **VM configured** to use hub’s DNS server (e.g., 10.0.2.10)
1. **DNS query** sent to hub via VNet peering
1. **DNS server in hub** resolves query, returns result
1. **Alternative**: VM uses Azure AD Domain Services in hub

## Security Architecture

### Defense in Depth Layers:

1. **Azure Firewall**: Network perimeter security
1. **NSGs**: Subnet-level micro-segmentation
1. **Bastion**: Secure management access (no RDP/SSH exposure)
1. **Azure DDoS**: DDoS attack protection
1. **Network Watcher**: Traffic monitoring and alerting

### Zero Trust Implementation:

- No implicit trust between spokes
- All traffic inspected by firewall
- Least privilege access via NSGs
- Just-in-time VM access via Bastion

## Scaling the Architecture

### Adding New Spokes:

1. Create new VNet with non-overlapping address space
1. Peer new spoke to hub VNet
1. Configure UDR to route through Azure Firewall
1. Apply NSGs to spoke subnets
1. Update firewall rules to allow required traffic

### Multi-Region Hub-Spoke:

- Deploy hub-spoke in each Azure region
- Connect regional hubs via Global VNet Peering
- Use Traffic Manager or Front Door for global load balancing

## Cost Optimization

1. **Single Bastion**: One deployment serves all VNets (~$140/month)
1. **Single Firewall**: Shared across all spokes (based on usage)
1. **VNet Peering**: Charged per GB transferred (minimal cost for management traffic)
1. **Avoid Gateway Transit**: Spokes use hub’s gateway (no duplicate gateways)

## Best Practices

1. **Plan IP Address Space**: Reserve sufficient space for growth
1. **Document Network Topology**: Maintain diagrams and documentation
1. **Use Azure Policy**: Enforce peering and security standards
1. **Monitor Traffic**: Enable NSG flow logs and Traffic Analytics
1. **Regular Firewall Rule Reviews**: Remove unused rules
1. **Tag Resources**: Use tags for cost allocation and management
1. **Implement RBAC**: Separate permissions for hub vs. spoke management
1. **Backup Configurations**: Document UDRs, NSG rules, firewall rules

## When to Use Hub-Spoke Architecture

✅ **Good for:**

- Enterprise environments with multiple departments/applications
- Organizations requiring centralized security and governance
- Hybrid cloud scenarios with on-premises connectivity
- Environments with shared services (AD, DNS, monitoring)
- Organizations with compliance requirements

❌ **Not ideal for:**

- Small organizations with few workloads
- Simple architectures with no hybrid connectivity needs
- Scenarios requiring direct spoke-to-spoke communication (consider mesh)
- Cost-sensitive environments with minimal traffic between VNets

## Common Challenges

1. **Complexity**: More components to manage than flat networks
1. **Single Point of Failure**: Hub firewall/gateway failure impacts all spokes
1. **Latency**: Additional hop through firewall adds latency
1. **Troubleshooting**: More complex to diagnose connectivity issues
1. **Cost**: Firewall and gateway costs can be significant

## Migration Path

### From Flat Network to Hub-Spoke:

1. Create hub VNet and deploy Azure Firewall
1. Deploy Azure Bastion in hub
1. Peer existing VNets to hub (they become spokes)
1. Gradually add UDRs to route through firewall
1. Remove public IPs from VMs (use Bastion instead)
1. Update NSG rules to leverage centralized firewall

## Learning Resources

- Microsoft Learn: [Hub-spoke network topology](https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/hub-spoke)
- Azure Architecture Center: [Hub-spoke with Azure Firewall](https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/hub-spoke-firewall)
- [Azure Virtual Network documentation](https://learn.microsoft.com/azure/virtual-network/)
- [Azure Firewall documentation](https://learn.microsoft.com/azure/firewall/)
