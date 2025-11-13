# 1200 - Azure DMZ / Perimeter Network Architecture

## Overview

A DMZ (Demilitarized Zone), also called a Perimeter Network, is a network security architecture that adds an extra layer of protection between the internet and internal resources. The DMZ contains public-facing services and acts as a buffer zone, inspecting and filtering all traffic before it reaches internal systems. In Azure, DMZ architectures protect against external threats while allowing controlled access to services.

## Architecture Diagram Concept

```
                    [Internet]
                        ↓
            [Azure DDoS Protection]
                        ↓
            [Public IP / Azure Front Door]
                        ↓
        ┌───────────────────────────────┐
        │         PERIMETER/DMZ         │
        │  ┌─────────────────────────┐  │
        │  │ Azure Application       │  │
        │  │ Gateway + WAF           │  │
        │  └─────────────────────────┘  │
        │             ↓                 │
        │  ┌─────────────────────────┐  │
        │  │ Azure Firewall          │  │
        │  │ (Threat Intelligence)   │  │
        │  └─────────────────────────┘  │
        │             ↓                 │
        │  ┌─────────────────────────┐  │
        │  │ Network Virtual         │  │
        │  │ Appliances (NVA)        │  │
        │  │ (Optional: Palo Alto,   │  │
        │  │  Fortinet, etc.)        │  │
        │  └─────────────────────────┘  │
        └───────────────────────────────┘
                        ↓
            [Network Security Groups]
                        ↓
        ┌───────────────────────────────┐
        │      INTERNAL NETWORK         │
        │  ┌─────────────────────────┐  │
        │  │ Web Tier VMs            │  │
        │  │ (Private IPs)           │  │
        │  └─────────────────────────┘  │
        │             ↓                 │
        │  ┌─────────────────────────┐  │
        │  │ Application Tier VMs    │  │
        │  └─────────────────────────┘  │
        │             ↓                 │
        │  ┌─────────────────────────┐  │
        │  │ Data Tier               │  │
        │  │ (Databases)             │  │
        │  └─────────────────────────┘  │
        └───────────────────────────────┘
                        ↓
            [Azure Bastion] (Management)
```

## Key Components and Their Connections

### External Layer

#### 1. **Azure DDoS Protection**

- **Purpose**: Protect against Distributed Denial of Service attacks
- **Connection**:
  - Always-on traffic monitoring on public IPs
  - Automatic attack mitigation without user intervention
  - Integration with Azure Monitor for alerts
  - Protects all resources in VNet

**DDoS Protection Tiers:**

- **Basic**: Free, automatic protection for Azure infrastructure
- **Standard**: Advanced protection for VNet resources ($2,944/month)
  - Adaptive tuning based on traffic patterns
  - Attack analytics and metrics
  - Integration with Azure Firewall
  - 24/7 support during active attacks

**Protected Resources:**

- Public IP addresses
- Application Gateway
- Azure Firewall
- Load Balancer
- Any resource with public IP

**Attack Types Protected:**

- Volumetric attacks (UDP floods, amplification floods)
- Protocol attacks (SYN floods, fragmented packet attacks)
- Application layer attacks (HTTP floods) - when combined with WAF

#### 2. **Azure Front Door**

- **Purpose**: Global edge network for traffic distribution and DDoS mitigation
- **Connection**:
  - First point of contact for global users
  - Built-in DDoS protection at edge
  - WAF capabilities at edge locations
  - Routes traffic to regional Application Gateway or Azure Firewall
  - SSL/TLS termination

**Front Door as DMZ Component:**

- Acts as first line of defense
- Filters malicious traffic at edge (130+ locations)
- Reduces load on regional infrastructure
- Custom rules for geo-filtering
- Bot protection

### Perimeter/DMZ Layer

#### 3. **Azure Application Gateway with WAF**

- **Purpose**: Layer 7 (application) load balancer and web application firewall
- **Connection**:
  - Deployed in dedicated subnet (ApplicationGatewaySubnet)
  - Receives traffic from internet or Front Door
  - Inspects HTTP/HTTPS traffic for threats
  - Routes clean traffic to backend web tier
  - Public IP for internet-facing, private IP for internal

**Web Application Firewall (WAF) Protection:**

- **OWASP Core Rule Set (CRS)**: Protection against:
  - SQL injection
  - Cross-site scripting (XSS)
  - Command injection
  - HTTP request smuggling
  - Remote file inclusion
  - Session fixation
  - And 20+ other attack types

**WAF Modes:**

- **Detection Mode**: Logs threats but doesn’t block (testing)
- **Prevention Mode**: Blocks malicious requests (production)

**Custom WAF Rules:**

```
Rule 1: Block traffic from specific countries
Rule 2: Rate limiting per client IP (max 100 req/min)
Rule 3: Block requests with specific user agents
Rule 4: Allow only specific HTTP methods (GET, POST)
Rule 5: Block requests with suspicious patterns in URL
```

**Example WAF Configuration:**

```json
{
  "policySettings": {
    "mode": "Prevention",
    "state": "Enabled",
    "maxRequestBodySizeInKb": 128,
    "fileUploadLimitInMb": 100,
    "requestBodyCheck": true
  },
  "managedRules": {
    "managedRuleSets": [
      {
        "ruleSetType": "OWASP",
        "ruleSetVersion": "3.2"
      },
      {
        "ruleSetType": "Microsoft_BotManagerRuleSet",
        "ruleSetVersion": "1.0"
      }
    ]
  }
}
```

#### 4. **Azure Firewall**

- **Purpose**: Cloud-native stateful firewall for network and application layer filtering
- **Connection**:
  - Deployed in dedicated subnet (AzureFirewallSubnet)
  - All north-south (internet) and east-west (internal) traffic routes through it
  - Uses User Defined Routes (UDR) to force traffic through firewall
  - Provides centralized logging and monitoring

**Azure Firewall Capabilities:**

- **Network Rules**: IP, port, protocol filtering
- **Application Rules**: FQDN-based filtering (allow *.microsoft.com)
- **NAT Rules**: DNAT for inbound connections
- **Threat Intelligence**: Block known malicious IPs/domains
- **IDPS (Intrusion Detection and Prevention)**: Signature-based detection
- **TLS Inspection**: Decrypt, inspect, re-encrypt HTTPS traffic
- **DNS Proxy**: Prevent DNS tunneling and manipulation

**Firewall SKUs:**

- **Standard**: Basic firewall capabilities
- **Premium**: IDPS, TLS inspection, URL filtering, web categories

**Example Firewall Rules:**

```
Network Rule Collection: Outbound Internet
  Priority: 100
  Rule 1: Allow web tier to internet on ports 80, 443
  Rule 2: Deny all other outbound traffic
  
Application Rule Collection: SaaS Access
  Priority: 200
  Rule 1: Allow *.microsoft.com, *.azure.com
  Rule 2: Allow *.salesforce.com (specific apps)
  Rule 3: Deny all other FQDNs

NAT Rule Collection: Inbound Services
  Priority: 300
  Rule 1: DNAT public IP:443 → internal web server:443
  Rule 2: DNAT public IP:3389 → Bastion subnet (admin access)
```

**Threat Intelligence Feed:**

- Microsoft threat intelligence from across 8+ trillion signals/day
- Automatic blocking of known malicious IPs
- Alert and deny mode options
- Updated continuously

#### 5. **Network Virtual Appliances (NVA)**

- **Purpose**: Third-party security appliances for advanced features
- **Connection**:
  - Deployed in dedicated DMZ subnet
  - Positioned between Application Gateway and internal network
  - Active-Active or Active-Passive HA configuration
  - Azure Load Balancer distributes traffic to NVA instances

**Popular NVAs:**

- **Palo Alto Networks**: Advanced threat prevention, URL filtering
- **Fortinet FortiGate**: UTM (Unified Threat Management)
- **Check Point CloudGuard**: Advanced security features
- **Cisco ASA**: Traditional firewall functionality
- **F5 BIG-IP**: Advanced load balancing and security
- **Barracuda CloudGen Firewall**: Next-gen firewall

**When to Use NVAs:**

- Compliance requires specific firewall vendor
- Need features not in Azure Firewall (e.g., advanced IPS, SSL VPN)
- Existing licensing agreements with vendors
- Specialized traffic inspection requirements
- On-premises firewall expertise to leverage

**NVA High Availability:**

```
Internet → Application Gateway → Load Balancer → NVA1 (active)
                                               → NVA2 (standby)
                                 → Internal Network
```

#### 6. **Network Security Groups (NSGs) - DMZ**

- **Purpose**: Subnet-level firewall rules
- **Connection**:
  - Applied to Application Gateway subnet
  - Applied to Azure Firewall subnet
  - Applied to NVA subnet (if used)
  - Works in conjunction with Azure Firewall

**NSG Rules for Application Gateway Subnet:**

```
Priority 100: Allow inbound 443 from Internet
Priority 110: Allow inbound 80 from Internet (redirect to 443)
Priority 120: Allow inbound 65200-65535 from GatewayManager (management)
Priority 130: Allow inbound from AzureLoadBalancer
Priority 200: Deny all other inbound
Priority 300: Allow outbound to web tier subnet
Priority 400: Allow outbound to Azure Firewall
Priority 500: Deny all other outbound
```

**NSG Rules for Azure Firewall Subnet:**

```
Inbound:
  Priority 100: Allow all from Internet (firewall handles filtering)
Outbound:
  Priority 100: Allow all to Internet (firewall policy controls)
  Priority 110: Allow all to internal subnets
```

### Internal Network Layer

#### 7. **Web Tier Subnet**

- **Purpose**: Host public-facing web servers with no direct internet access
- **Connection**:
  - Receives traffic from Application Gateway/Firewall only
  - No public IP addresses
  - NSG allows traffic only from DMZ subnet
  - Can initiate outbound through Azure Firewall

**Web Tier NSG Rules:**

```
Inbound:
  Priority 100: Allow 443 from Application Gateway subnet
  Priority 110: Allow 80 from Application Gateway subnet
  Priority 120: Allow 22/3389 from Azure Bastion subnet (management)
  Priority 4096: Deny all other inbound

Outbound:
  Priority 100: Allow to Application Tier subnet
  Priority 110: Allow to Azure Firewall (for internet access)
  Priority 4096: Deny all other outbound
```

#### 8. **Application Tier Subnet**

- **Purpose**: Business logic and application processing
- **Connection**:
  - Receives traffic only from Web Tier
  - No public IP addresses
  - Connects to Data Tier for database queries
  - Outbound through Azure Firewall if needed

**Application Tier NSG Rules:**

```
Inbound:
  Priority 100: Allow application ports from Web Tier subnet
  Priority 110: Allow 22/3389 from Azure Bastion subnet
  Priority 4096: Deny all other inbound

Outbound:
  Priority 100: Allow to Data Tier subnet
  Priority 110: Allow to Azure Firewall
  Priority 4096: Deny all other outbound
```

#### 9. **Data Tier Subnet**

- **Purpose**: Databases and data storage
- **Connection**:
  - Receives connections only from Application Tier
  - Private endpoints for Azure SQL, Cosmos DB
  - No internet access
  - Highly restricted NSG rules

**Data Tier NSG Rules:**

```
Inbound:
  Priority 100: Allow SQL (1433) from Application Tier subnet
  Priority 110: Allow 22/3389 from Azure Bastion subnet (emergency)
  Priority 4096: Deny all other inbound

Outbound:
  Priority 100: Allow to Application Tier subnet (query responses)
  Priority 4096: Deny all other outbound (no internet)
```

### Management Layer

#### 10. **Azure Bastion**

- **Purpose**: Secure RDP/SSH access without exposing VMs to internet
- **Connection**:
  - Deployed in AzureBastionSubnet
  - Accessed via Azure Portal over HTTPS
  - Reaches all VMs via private IPs
  - No public IPs needed on VMs

**Bastion in DMZ Architecture:**

- Separate management plane from data plane
- All administrative access logged
- Integrates with Azure AD (MFA required)
- Cannot be bypassed by attackers
- No attack surface for brute force

#### 11. **Jump Box / Bastion Host (Alternative)**

- **Purpose**: Hardened VM for administrative access
- **Connection**:
  - Single point of entry for administrators
  - Sits in management subnet
  - Accessed via VPN or Azure Bastion
  - Minimal software installed (hardened)

**Jump Box Security:**

- Just-in-time (JIT) VM access
- Logged and monitored access
- Multi-factor authentication required
- No data stored on jump box
- Regularly patched and updated

### Monitoring and Logging

#### 12. **Azure Monitor / Log Analytics**

- **Purpose**: Centralized logging and monitoring
- **Connection**:
  - Collects logs from all DMZ components
  - NSG Flow Logs capture all traffic
  - Azure Firewall logs all connections
  - Application Gateway logs all requests
  - Creates alerts on suspicious activity

**Critical Logs to Monitor:**

- Failed login attempts
- NSG rule hits (especially denies)
- Firewall threat intelligence alerts
- WAF blocked requests
- Unusual traffic patterns
- Port scanning attempts
- DDoS attack metrics

#### 13. **Microsoft Defender for Cloud**

- **Purpose**: Security posture management and threat detection
- **Connection**:
  - Assesses DMZ security configuration
  - Provides hardening recommendations
  - Detects threats in real-time
  - Just-in-time VM access
  - Adaptive network hardening

**Defender for Cloud Features:**

- **Secure Score**: Measure security posture
- **Recommendations**: Harden DMZ configuration
- **Security Alerts**: Real-time threat detection
- **Regulatory Compliance**: PCI-DSS, HIPAA, etc.

#### 14. **Azure Sentinel**

- **Purpose**: Cloud-native SIEM for advanced threat detection
- **Connection**:
  - Ingests logs from Log Analytics
  - Machine learning detects anomalies
  - Automated incident response
  - Threat hunting capabilities

**Sentinel Use Cases in DMZ:**

- Detect brute force attacks
- Identify lateral movement attempts
- Correlate events across DMZ layers
- Automated blocking of malicious IPs
- Compliance reporting

### Additional Security Components

#### 15. **Private Endpoints**

- **Purpose**: Private connectivity to PaaS services
- **Connection**:
  - Azure SQL, Storage, Key Vault get private IPs
  - No public endpoints exposed
  - Traffic never leaves Azure backbone
  - Accessed from internal network only

#### 16. **Azure Key Vault**

- **Purpose**: Secure secrets and certificate management
- **Connection**:
  - Stores SSL certificates for Application Gateway
  - Connection strings for applications
  - API keys and passwords
  - Private endpoint for access from VNet

#### 17. **User Defined Routes (UDR)**

- **Purpose**: Force traffic through security appliances
- **Connection**:
  - Applied to subnets
  - Override default Azure routing
  - Force all traffic through Azure Firewall or NVA

**Example UDR for Web Tier:**

```
Route 1: 
  Destination: 0.0.0.0/0 (Internet)
  Next Hop: Virtual Appliance (Azure Firewall IP)
  
Route 2:
  Destination: 10.0.0.0/8 (Internal)
  Next Hop: Virtual Appliance (Azure Firewall IP)
```

## Traffic Flow Examples

### Example 1: Legitimate User Request

1. **User** accesses https://www.contoso.com
1. **DNS** resolves to Azure Front Door public IP
1. **Front Door** terminates SSL, applies WAF rules at edge
1. **Front Door** routes to nearest Azure region’s Application Gateway
1. **DDoS Protection** validates traffic volume is normal
1. **Application Gateway** applies regional WAF rules
1. **Application Gateway** checks backend health probes
1. **Application Gateway** SSL re-encryption, forwards to web tier VM
1. **NSG** on web tier subnet allows traffic from App Gateway
1. **Web Tier VM** processes request, calls application tier
1. **NSG** allows web to app tier communication
1. **Application Tier VM** queries database
1. **NSG** allows app to data tier communication
1. **Database** returns data through same path in reverse
1. **Response** cached at Front Door edge for future requests

**End-to-End Latency:** ~50-200ms depending on complexity

### Example 2: SQL Injection Attack (Blocked)

1. **Attacker** sends request: `https://www.contoso.com/search?q=' OR '1'='1`
1. **Front Door** receives request
1. **Front Door WAF** detects SQL injection pattern
1. **WAF** blocks request with 403 Forbidden
1. **Attack logged** in Azure Monitor
1. **Alert triggered** in Azure Sentinel
1. **Automated playbook** adds attacker IP to block list
1. **Security team notified** via email/Teams
1. **Request never reaches** backend infrastructure

### Example 3: DDoS Attack (Mitigated)

1. **Botnet** sends 100,000 requests/second to public IP
1. **DDoS Protection** detects abnormal traffic volume
1. **Mitigation activated** automatically within 2 minutes
1. **Scrubbing centers** filter malicious traffic
1. **Legitimate traffic** passes through to Application Gateway
1. **Attack metrics** visible in Azure Monitor
1. **DDoS rapid response team** engaged (if Standard plan)
1. **Post-attack report** generated automatically

### Example 4: Port Scan Attempt (Detected and Blocked)

1. **Attacker** scans ports 1-65535 on public IP
1. **Azure Firewall** receives scan packets
1. **Threat Intelligence** identifies scanner IP as malicious
1. **Firewall** blocks all traffic from scanner IP
1. **IDPS** detects port scanning pattern
1. **Alert** created in Defender for Cloud
1. **NSG Flow Logs** capture all blocked connections
1. **Sentinel** correlates with other attacks globally
1. **Automated response** adds IP to global block list

### Example 5: Lateral Movement Attempt (Blocked)

1. **Attacker** compromises web tier VM (hypothetically)
1. **Attacker** attempts to connect directly to database
1. **NSG** on data tier subnet denies traffic from web tier
1. **NSG Flow Logs** record denied connection
1. **Sentinel** detects anomalous connection attempt
1. **Alert** raised: “Unusual web tier to data tier direct connection”
1. **Automated response** isolates compromised VM
1. **Incident response team** notified
1. **VM reimaged** from clean image

### Example 6: Administrator Access

1. **Administrator** logs into Azure Portal
1. **Administrator** navigates to target VM
1. **Azure AD** authenticates with MFA
1. **Conditional Access** validates device compliance
1. **Azure Bastion** connection request
1. **JIT access** requested for 2 hours
1. **Approval** required from security team
1. **Approval granted**, temporary NSG rule created
1. **Bastion** establishes RDP connection over private IP
1. **Session recorded** for audit purposes
1. **After 2 hours**, NSG rule automatically removed

## DMZ Architecture Patterns

### Pattern 1: Single DMZ (Basic)

```
Internet → DDoS → App Gateway + WAF → Azure Firewall → Internal VMs
```

**Use Case:** Small to medium organizations, standard security requirements

### Pattern 2: Multi-Tier DMZ

```
Internet → DDoS → App Gateway + WAF → Azure Firewall → NVA → Internal VMs
```

**Use Case:** High-security environments, defense-in-depth requirements

### Pattern 3: Dual DMZ (Extranet)

```
Internet → External DMZ (public services) → Azure Firewall
Partner Network → Partner DMZ → Azure Firewall → Internal Network
```

**Use Case:** B2B integrations, partner access to specific services

### Pattern 4: Hub DMZ with Spoke Workloads

```
Internet → Hub VNet (DMZ: Firewall, App Gateway) → Spoke VNets (Workloads)
```

**Use Case:** Enterprise environments with multiple applications/departments

### Pattern 5: Global DMZ with Front Door

```
Internet → Front Door (Global) → Regional DMZ (App Gateway + Firewall) → VMs
```

**Use Case:** Global applications requiring edge security and performance

## Security Defense Layers

### Layer 1: Edge Protection

- Azure Front Door with WAF
- DDoS Protection
- Geo-filtering and bot protection

### Layer 2: Perimeter Security

- Azure Application Gateway with WAF
- Azure Firewall or NVA
- Threat intelligence and IDPS

### Layer 3: Network Segmentation

- Network Security Groups
- User Defined Routes
- Private endpoints for PaaS

### Layer 4: Host Security

- Antimalware and EDR agents
- Just-in-time VM access
- Patch management
- Disk encryption

### Layer 5: Application Security

- Input validation
- Output encoding
- Secure coding practices
- Dependency scanning

### Layer 6: Data Security

- Encryption at rest
- Encryption in transit
- Data classification
- Access controls

### Layer 7: Identity & Access

- Azure AD authentication
- Multi-factor authentication
- Conditional Access
- Privileged Identity Management

## Best Practices

### Network Design:

1. **Subnet Segmentation**: Separate subnet for each tier and DMZ component
1. **No Public IPs**: Internal VMs should never have public IPs
1. **Default Deny**: NSG rules should default to deny, explicit allow
1. **Forced Tunneling**: Route all internet traffic through firewall
1. **Private Endpoints**: Use for all PaaS services

### Security Configuration:

1. **WAF in Prevention Mode**: Block threats, not just detect
1. **Threat Intelligence**: Enable on Azure Firewall
1. **IDPS**: Enable if using Azure Firewall Premium
1. **TLS Inspection**: Decrypt and inspect HTTPS (where legally permitted)
1. **Patch Management**: Automated patching for all VMs

### Monitoring & Response:

1. **Centralized Logging**: All logs to Log Analytics
1. **Real-Time Alerts**: Alert on critical security events
1. **SIEM Integration**: Use Azure Sentinel for correlation
1. **Incident Response Plan**: Document and test regularly
1. **Regular Audits**: Security posture reviews monthly

### Compliance:

1. **Azure Policy**: Enforce security standards automatically
1. **Regulatory Frameworks**: Map to PCI-DSS, HIPAA, ISO 27001
1. **Audit Trails**: Retain logs per compliance requirements (often 7 years)
1. **Penetration Testing**: Regular external security assessments
1. **Compliance Reporting**: Automated compliance dashboards

## Common Mistakes to Avoid

1. **Public IPs on Internal VMs**: Creates bypass around DMZ
1. **Overly Permissive NSGs**: “Allow Any” defeats purpose of DMZ
1. **No Outbound Filtering**: Allows data exfiltration and C2 communication
1. **Single Point of Failure**: No redundancy for critical components
1. **Ignored Logs**: Generating logs but not monitoring them
1. **Misconfigured UDRs**: Traffic not actually going through firewall
1. **Disabled WAF**: WAF in detection-only mode doesn’t block
1. **No Segmentation**: All internal VMs in single subnet
1. **Weak Passwords**: Default or weak admin passwords
1. **No MFA**: Administrative access without multi-factor authentication

## Cost Considerations

### Monthly Cost Estimate (Medium Enterprise):

|Component                   |Monthly Cost     |
|----------------------------|-----------------|
|Azure Firewall Standard     |$1,000-2,000     |
|Azure Firewall Premium      |$2,500-4,000     |
|Application Gateway v2 + WAF|$500-1,500       |
|DDoS Protection Standard    |$2,944           |
|Azure Front Door            |$300-1,000       |
|Log Analytics (500GB/month) |$1,500           |
|Azure Bastion Standard      |$140             |
|Sentinel (500GB/month)      |$1,500           |
|Network bandwidth           |$500-2,000       |
|**Total**                   |**$8,000-15,000**|

### Cost Optimization:

1. **Right-Size Components**: Don’t over-provision
1. **Review Logs Retention**: 90 days vs 2 years makes huge difference
1. **Optimize Bandwidth**: Use CDN to reduce egress
1. **Reserved Instances**: Commit to 1-3 years for discounts
1. **Monitor Costs**: Alert on unexpected increases

## When to Use DMZ Architecture

✅ **Essential for:**

- Internet-facing applications handling sensitive data
- Organizations with compliance requirements (PCI-DSS, HIPAA)
- High-value targets (financial, healthcare, government)
- Multi-tenant environments
- Organizations experiencing active threats

❌ **May not need for:**

- Internal-only applications
- Development/test environments (use simpler security)
- Very small organizations with limited resources
- Applications with no sensitive data

## Evolution: Traditional vs Modern DMZ

### Traditional On-Premises DMZ:

- Physical hardware firewalls
- Complex configuration and management
- Limited scalability
- High capital expenditure
- Manual updates and patching

### Modern Azure DMZ:

- Cloud-native security services
- Simplified management via Azure Portal
- Automatic scaling
- Pay-as-you-go pricing
- Automatic updates and threat intelligence

## DMZ Validation Checklist

Before going live, verify:

- ✅ All public IPs protected by DDoS Standard
- ✅ WAF enabled in Prevention mode
- ✅ Azure Firewall threat intelligence enabled
- ✅ All internal VMs have no public IPs
- ✅ NSGs applied to all subnets with default deny
- ✅ UDRs force traffic through Azure Firewall
- ✅ Private endpoints for all PaaS services
- ✅ Azure Bastion deployed for management
- ✅ All logs flowing to Log Analytics
- ✅ Alerts configured for critical events
- ✅ Backup and disaster recovery tested
- ✅ Penetration test completed
- ✅ Incident response plan documented
- ✅ Security team trained on tools

## Learning Resources

- Microsoft Learn: [Azure DMZ architecture](https://learn.microsoft.com/azure/architecture/reference-architectures/dmz/secure-vnet-dmz)
- Azure Architecture Center: [Network DMZ](https://learn.microsoft.com/azure/architecture/example-scenario/gateway/firewall-application-gateway)
- [Azure Firewall documentation](https://learn.microsoft.com/azure/firewall/)
- [Application Gateway WAF documentation](https://learn.microsoft.com/azure/web-application-firewall/)
- [Azure DDoS Protection documentation](https://learn.microsoft.com/azure/ddos-protection/)
- [Network Security Best Practices](https://learn.microsoft.com/azure/security/fundamentals/network-best-practices)
- [Azure Security Benchmark](https://learn.microsoft.com/security/benchmark/azure/baselines/virtual-network-security-baseline)
