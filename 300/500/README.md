# 500 - Azure High Availability (HA) Architecture

## Overview

High Availability (HA) architecture ensures applications remain accessible and operational even during failures, maintenance, or disasters. Azure provides multiple mechanisms to achieve HA including availability zones, availability sets, load balancing, and geo-redundancy. This pattern is critical for mission-critical applications requiring 99.9%+ uptime.

## Architecture Diagram Concept

```
                    [Azure Traffic Manager]
                     (Global DNS-based)
                ┌─────────────────────────────┐
                │                             │
         [Region 1 (Primary)]          [Region 2 (Secondary)]
                │                             │
        [Azure Load Balancer]         [Azure Load Balancer]
                │                             │
    ┌───────────┴───────────┐     ┌──────────┴──────────┐
    │                       │     │                      │
[Zone 1 VMs]          [Zone 2 VMs]  [Zone 1 VMs]   [Zone 2 VMs]
    │                       │     │                      │
    └───────────┬───────────┘     └──────────┬───────────┘
                │                             │
        [Azure SQL - ZRS]              [Azure SQL - Geo-Replica]
```

## Key HA Concepts in Azure

### 1. **Availability Zones**

- **Definition**: Physically separate datacenters within an Azure region
- **Purpose**: Protect against datacenter-level failures
- **SLA**: 99.99% uptime when using 2+ zones
- **Latency**: <2ms between zones in same region

### 2. **Availability Sets**

- **Definition**: Logical grouping of VMs within a datacenter
- **Purpose**: Protect against hardware failures and maintenance
- **SLA**: 99.95% uptime
- **Components**:
  - **Fault Domains**: Separate power/network sources (max 3 per set)
  - **Update Domains**: Separate maintenance windows (max 20 per set)

### 3. **Azure Regions**

- **Definition**: Geographic location containing multiple datacenters
- **Purpose**: Geographic redundancy and disaster recovery
- **Paired Regions**: Each region paired with another for DR (e.g., East US ↔ West US)

## Key Components and Their Connections

### 1. **Azure Traffic Manager**

- **Purpose**: Global DNS-based load balancer and failover manager
- **Connection**:
  - Entry point for all user traffic
  - Routes users to healthy, best-performing Azure region
  - Performs health checks on regional endpoints
  - No traffic passes through Traffic Manager (DNS-only)

**Traffic Manager Routing Methods:**

- **Priority**: Primary region with automatic failover to secondary
- **Performance**: Route to region with lowest latency
- **Geographic**: Route based on user’s geographic location
- **Weighted**: Distribute traffic percentage across regions
- **Subnet**: Route based on user’s IP subnet
- **MultiValue**: Return multiple healthy endpoints

**Health Probing:**

- Continuously checks endpoint health (HTTP/HTTPS probe)
- Configurable probe interval (10-30 seconds)
- Automatic failover on endpoint failure
- Can probe custom health check endpoints

### 2. **Azure Front Door (Alternative)**

- **Purpose**: Global HTTP/HTTPS load balancer with Layer 7 capabilities
- **Connection**:
  - Traffic actually passes through Front Door (unlike Traffic Manager)
  - SSL termination at edge
  - WAF protection
  - Caching at edge locations
  - Instant global failover

**Front Door vs Traffic Manager:**

- **Traffic Manager**: DNS-based, any protocol, slower failover
- **Front Door**: Application layer, HTTP/HTTPS only, instant failover

### 3. **Azure Load Balancer (Regional)**

- **Purpose**: Layer 4 load balancer within a region
- **Connection**:
  - Deployed in each region
  - Distributes traffic across VMs in multiple availability zones
  - Performs health probes on backend VMs
  - Supports both public and internal load balancing

**Load Balancer Types:**

- **Public Load Balancer**: Internet-facing with public IP
- **Internal Load Balancer**: Private IP for internal traffic

**Load Balancer SKUs:**

- **Basic**: Free, limited features, no zone redundancy
- **Standard**: Production-ready, zone-redundant, 99.99% SLA

**Key Features:**

- **Zone Redundancy**: Frontend IP survives zone failures
- **Availability Set Support**: Distribute across fault domains
- **Health Probes**: HTTP, HTTPS, TCP probes
- **Outbound Rules**: Manage outbound internet connectivity
- **HA Ports**: Load balance all ports for internal LB

### 4. **Virtual Machines in Availability Zones**

- **Purpose**: Compute resources distributed across zones
- **Connection**:
  - Each VM deployed in specific availability zone (1, 2, or 3)
  - Behind load balancer for traffic distribution
  - Identical configuration across all zones
  - Shared managed disks with zone-redundant storage (ZRS)

**Zone Deployment Strategy:**

- **Minimum**: 2 VMs in 2 different zones (99.99% SLA)
- **Recommended**: 3 VMs in 3 zones (better fault tolerance)
- **Load Balancer**: Distributes traffic evenly across zones

**Example Configuration:**

```
- VM1 in Zone 1 (10.0.1.4)
- VM2 in Zone 2 (10.0.1.5)
- VM3 in Zone 3 (10.0.1.6)
- Load Balancer Frontend: 20.30.40.50
```

### 5. **Virtual Machines in Availability Sets (Legacy/Alternative)**

- **Purpose**: HA within single datacenter (when zones not available)
- **Connection**:
  - VMs distributed across fault domains and update domains
  - Behind load balancer
  - 99.95% SLA (lower than zones)

**When to Use Availability Sets:**

- Azure region doesn’t support availability zones
- Cost-sensitive scenarios (zones may have data transfer costs)
- Legacy applications

### 6. **Virtual Machine Scale Sets (VMSS)**

- **Purpose**: Auto-scaling group of identical VMs
- **Connection**:
  - Can span availability zones automatically
  - Integrated with Load Balancer
  - Auto-scales based on metrics (CPU, memory, schedule)
  - Supports up to 1,000 VM instances

**VMSS Features:**

- **Auto-Healing**: Automatically replaces unhealthy instances
- **Rolling Updates**: Update instances gradually
- **Zone Balancing**: Distribute evenly across zones
- **Custom Auto-Scale Rules**: Scale on any Azure Monitor metric

### 7. **Azure Virtual Network (VNet)**

- **Purpose**: Network foundation for all resources
- **Connection**:
  - Spans all availability zones in a region
  - Subnets automatically zone-redundant
  - Can peer VNets across regions for DR

### 8. **Network Security Groups (NSGs)**

- **Purpose**: Firewall rules for subnets and NICs
- **Connection**:
  - Applied to VM subnets in all zones
  - Identical rules across all zones
  - Allow load balancer health probes

**Critical NSG Rules for HA:**

- Allow Azure Load Balancer IP (168.63.129.16) for health probes
- Allow traffic between zones (same VNet)
- Allow management access via Azure Bastion

### 9. **Azure Managed Disks**

- **Purpose**: Durable storage for VM data
- **HA Options**:
  - **Zone-Redundant Storage (ZRS)**: Replicates across 3 zones (99.9999999999% durability)
  - **Locally Redundant Storage (LRS)**: 3 copies in single zone (99.999999999% durability)
  - **Geo-Redundant Storage (GRS)**: Additional copies in paired region

**Recommendation:** Use ZRS for data disks when using availability zones

### 10. **Azure SQL Database (HA Configuration)**

- **Purpose**: Highly available managed database
- **Connection**:
  - Zone-redundant deployment in premium/business critical tiers
  - Automatic failover to secondary replica
  - Accessed via connection string (failover transparent to app)

**Azure SQL HA Features:**

- **Zone-Redundant**: Replicas in multiple zones (99.995% SLA)
- **Active Geo-Replication**: Read replicas in other regions
- **Auto-Failover Groups**: Automatic failover to secondary region
- **Point-in-Time Restore**: Restore to any point in last 7-35 days

**Geo-Replication Setup:**

- **Primary Region**: Read-write database
- **Secondary Region**: Read-only replica
- **Failover**: Manual or automatic via failover group
- **RPO**: <5 seconds (data loss window)
- **RTO**: <30 seconds (failover time)

### 11. **Azure Storage Accounts**

- **Purpose**: Store application data, logs, backups
- **HA Options**:
  - **LRS**: 3 copies in single zone
  - **ZRS**: 3 copies across zones in region (99.9999999999% durability)
  - **GRS**: 6 copies across 2 paired regions
  - **GZRS**: ZRS in primary + LRS in secondary region
  - **RA-GRS**: GRS with read access to secondary

### 12. **Azure Site Recovery**

- **Purpose**: Disaster recovery orchestration
- **Connection**:
  - Replicates VMs to secondary region
  - Automated failover and failback
  - Regular DR drills without affecting production

**ASR Capabilities:**

- **RPO**: Continuous replication (typically <15 minutes)
- **RTO**: Automated failover in minutes
- **VM Replication**: Azure-to-Azure, on-premises to Azure
- **Recovery Plans**: Orchestrate multi-tier app failover

### 13. **Azure Backup**

- **Purpose**: Backup and restore for disaster recovery
- **Connection**:
  - Backs up VMs, SQL databases, files
  - Stores backups in geo-redundant storage
  - Point-in-time restore capability

### 14. **Azure Application Gateway (with HA)**

- **Purpose**: Layer 7 load balancer with WAF
- **Connection**:
  - Zone-redundant deployment (v2 SKU)
  - Receives traffic from Traffic Manager or Front Door
  - Routes to backend VMs across zones

### 15. **Azure Bastion (HA Configuration)**

- **Purpose**: Secure RDP/SSH access
- **Connection**:
  - Can be deployed zone-redundant (Standard SKU)
  - No single point of failure for management access

## Traffic Flow Examples

### Example 1: Normal Operations

1. **User** DNS query to www.contoso.com
1. **Traffic Manager** returns IP of primary region’s load balancer
1. **User** connects to primary region load balancer
1. **Load Balancer** health-checks all VMs in all zones
1. **Load Balancer** distributes request to healthy VM in Zone 1
1. **VM in Zone 1** processes request, queries zone-redundant SQL
1. **Response** returns through load balancer to user

### Example 2: Single VM Failure

1. **VM1 in Zone 1** crashes or becomes unhealthy
1. **Load Balancer** detects failure via health probe (within 10-30 seconds)
1. **Load Balancer** stops sending traffic to VM1
1. **Load Balancer** routes all traffic to VM2 (Zone 2) and VM3 (Zone 3)
1. **VM1** auto-heals if using VMSS
1. **Service continues** with no user impact (traffic routed to healthy VMs)

### Example 3: Availability Zone Failure

1. **Entire Zone 1** experiences power failure
1. **Load Balancer** detects VM1 unhealthy (health probe fails)
1. **Load Balancer** automatically routes all traffic to Zone 2 and Zone 3
1. **Zone-Redundant SQL** continues operating (replicas in Zone 2/3)
1. **Users experience** no downtime (transparent failover)
1. **When Zone 1 recovers**, load balancer automatically includes it again

### Example 4: Regional Disaster

1. **Entire primary region** becomes unavailable (natural disaster, major outage)
1. **Traffic Manager** health probe to primary region fails
1. **Traffic Manager** updates DNS to point to secondary region
1. **Users** resolve DNS and connect to secondary region
1. **Geo-replicated SQL** promoted to primary in secondary region
1. **DNS TTL** determines switchover speed (typically 1-5 minutes)

### Example 5: Planned Maintenance

1. **Azure notifies** of upcoming maintenance on Zone 1
1. **Administrator** can manually drain traffic from Zone 1 VMs
1. **Load Balancer** stops routing to Zone 1
1. **Maintenance completed** on Zone 1 VMs
1. **VMs brought back** into rotation via load balancer
1. **Process repeated** for Zone 2 and Zone 3 (rolling maintenance)

## SLA Calculations

### Single VM with Premium SSD:

- **SLA**: 99.9% = ~8.76 hours downtime/year

### 2 VMs in Availability Set:

- **SLA**: 99.95% = ~4.38 hours downtime/year

### 2 VMs in 2 Availability Zones:

- **SLA**: 99.99% = ~52.6 minutes downtime/year

### Multi-Region with Traffic Manager:

- **SLA**: 99.999%+ = ~5.26 minutes downtime/year (with proper configuration)

## High Availability Checklist

### Infrastructure Layer:

- ✅ Deploy VMs across multiple availability zones (or availability sets)
- ✅ Use Azure Load Balancer with zone-redundant frontend
- ✅ Use managed disks with zone-redundant storage (ZRS)
- ✅ Configure auto-scaling with VMSS
- ✅ Implement health checks on all endpoints

### Application Layer:

- ✅ Design stateless applications (store state externally)
- ✅ Implement retry logic with exponential backoff
- ✅ Use circuit breaker pattern for dependencies
- ✅ Handle transient failures gracefully
- ✅ Implement health check endpoints

### Data Layer:

- ✅ Use zone-redundant Azure SQL or geo-replication
- ✅ Use zone-redundant storage (ZRS) or GRS
- ✅ Regular backups with geo-redundant backup storage
- ✅ Test restore procedures regularly
- ✅ Implement connection pooling and retry logic

### Monitoring & Alerting:

- ✅ Configure Azure Monitor alerts for health issues
- ✅ Set up availability tests (synthetic monitoring)
- ✅ Dashboard for real-time health visibility
- ✅ Runbooks for automated remediation
- ✅ Post-mortem process for incidents

### Disaster Recovery:

- ✅ Define RPO (Recovery Point Objective) and RTO (Recovery Time Objective)
- ✅ Implement Azure Site Recovery for region failover
- ✅ Regular DR drills and documentation
- ✅ Automated failover testing
- ✅ Communication plan for outages

## Cost Optimization for HA

1. **Right-Size VMs**: Start with smaller VMs, scale up as needed
1. **Reserved Instances**: Save up to 72% with 1-3 year commitment
1. **Azure Hybrid Benefit**: Use existing Windows/SQL licenses
1. **Auto-Shutdown**: Stop non-production VMs outside business hours
1. **Spot VMs**: Use for fault-tolerant workloads (up to 90% savings)
1. **Standard Load Balancer**: More cost-effective than multiple Basic LBs

## Best Practices

1. **Test Failover Regularly**: Monthly DR drills
1. **Automate Everything**: Infrastructure as Code (Terraform, Bicep)
1. **Monitor Continuously**: Proactive alerting and dashboards
1. **Document Procedures**: Runbooks for incident response
1. **Implement Chaos Engineering**: Test failure scenarios (Azure Chaos Studio)
1. **Update Dependencies**: Keep OS and applications patched
1. **Use Managed Services**: Azure SQL, App Service have built-in HA
1. **Avoid Single Points of Failure**: Review architecture for bottlenecks

## Common Pitfalls

1. **Not testing DR procedures**: Theory vs. reality
1. **Single region deployment**: No protection against regional outages
1. **Ignoring health probes**: Incorrect probe configuration causes issues
1. **Shared state on VMs**: Stateful apps don’t failover cleanly
1. **Long DNS TTL**: Slow failover with Traffic Manager
1. **No connection retry logic**: Applications crash on transient failures
1. **Inadequate monitoring**: Can’t fix what you can’t see

## When to Use HA Architecture

✅ **Good for:**

- Mission-critical production applications
- Applications with uptime SLA requirements
- Customer-facing services
- Financial or healthcare applications
- E-commerce platforms

❌ **Not necessary for:**

- Development and test environments
- Internal tools with low usage
- Batch processing with tolerance for delays
- Cost-sensitive non-critical workloads

## Learning Resources

- Microsoft Learn: [Design for high availability](https://learn.microsoft.com/azure/architecture/framework/resiliency/design-requirements)
- Azure Architecture Center: [High availability patterns](https://learn.microsoft.com/azure/architecture/framework/resiliency/reliability-patterns)
- [Azure SLA documentation](https://azure.microsoft.com/support/legal/sla/)
- [Availability Zones overview](https://learn.microsoft.com/azure/reliability/availability-zones-overview)
