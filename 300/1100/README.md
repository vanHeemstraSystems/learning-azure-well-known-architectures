# 1100 - Azure Zero Trust Security Architecture

## Overview

Zero Trust is a security model based on the principle “never trust, always verify.” Instead of assuming everything inside a network is safe, Zero Trust verifies every user, device, and connection attempting to access resources, regardless of location. In Azure, this involves identity verification, device compliance, least-privilege access, and microsegmentation.

## Core Zero Trust Principles

1. **Verify Explicitly**: Always authenticate and authorize based on all available data points
1. **Use Least Privilege Access**: Limit user access with Just-In-Time and Just-Enough-Access (JIT/JEA)
1. **Assume Breach**: Minimize blast radius and segment access; verify end-to-end encryption

## Architecture Diagram Concept

```
        [User with Compliant Device]
                    ↓
        [Azure AD (Entra ID) MFA]
                    ↓
        [Conditional Access Policies]
                    ↓
        [Azure Firewall / NSGs]
                    ↓
        [Azure Private Link]
                    ↓
    ┌──────────────┴──────────────┐
    │                             │
[App with Managed Identity]  [Azure Bastion]
    │                             │
[Azure Key Vault]          [Monitored VMs]
```

## Key Components and Their Connections

### Identity & Access Control

#### 1. **Azure Active Directory (Azure AD / Entra ID)**

- **Purpose**: Identity and access management foundation
- **Connection**:
  - Central authentication for all users and services
  - Issues tokens for accessing Azure resources
  - Enforces conditional access policies
  - Integrates with on-premises AD via Azure AD Connect

**Zero Trust Features:**

- **Multi-Factor Authentication (MFA)**: Requires 2+ verification factors
- **Passwordless Authentication**: Windows Hello, FIDO2 keys, Microsoft Authenticator
- **Risk-Based Conditional Access**: Adaptive policies based on sign-in risk
- **Identity Protection**: Detects and remediates identity risks
- **Privileged Identity Management (PIM)**: Just-in-time admin access

#### 2. **Conditional Access Policies**

- **Purpose**: Policy enforcement engine for access decisions
- **Connection**:
  - Evaluates signals from users, devices, locations, applications
  - Enforces controls before granting access
  - Integrated with Azure AD authentication flow

**Policy Components:**

- **Assignments**: Who/what is subject to policy (users, groups, apps)
- **Conditions**: When policy applies (location, device, risk level)
- **Controls**: What happens (grant access, require MFA, block)

**Example Policies:**

```
Policy 1: Admin Access
- Assignment: Admin users
- Conditions: Accessing Azure Portal
- Controls: Require MFA + compliant device + approved location

Policy 2: Risky Sign-In
- Assignment: All users
- Conditions: Sign-in risk = High
- Controls: Block access or require password change + MFA

Policy 3: External Users
- Assignment: Guest users
- Conditions: Accessing any app
- Controls: Require MFA + terms of use acceptance
```

#### 3. **Privileged Identity Management (PIM)**

- **Purpose**: Just-in-time privileged access with approval workflows
- **Connection**:
  - Manages Azure AD roles and Azure resource roles
  - Requires justification for role activation
  - Time-limited role assignments (e.g., 8 hours)
  - Approval workflow for sensitive roles

**PIM Benefits:**

- Eliminates standing admin privileges
- Audit trail of all privilege activations
- Alerts on suspicious privilege use
- Regular access reviews

#### 4. **Azure AD Identity Protection**

- **Purpose**: Detect and respond to identity-based risks
- **Connection**:
  - Analyzes trillions of signals from Microsoft threat intelligence
  - Calculates user and sign-in risk scores
  - Feeds risk data to Conditional Access
  - Automates response to detected risks

**Risk Detections:**

- Atypical travel (impossible travel between locations)
- Anonymous IP address usage (Tor, VPN)
- Leaked credentials (found on dark web)
- Malware-linked IP addresses
- Unfamiliar sign-in properties

### Device Trust

#### 5. **Microsoft Intune (Endpoint Manager)**

- **Purpose**: Mobile device management and app management
- **Connection**:
  - Enrolls and manages Windows, macOS, iOS, Android devices
  - Enforces device compliance policies
  - Integrates with Conditional Access (compliant device requirement)
  - Deploys security baselines and configurations

**Compliance Policies:**

- OS version requirements
- Encryption enabled
- Antivirus installed and updated
- Password complexity
- Jailbreak/root detection
- Device health attestation

#### 6. **Azure AD Device Registration**

- **Purpose**: Establish device identity in Azure AD
- **Connection**:
  - Devices register with Azure AD
  - Receive device certificate for authentication
  - Enable device-based Conditional Access

**Device States:**

- **Azure AD Registered**: User owns device (BYOD)
- **Azure AD Joined**: Organization owns device (cloud-only)
- **Hybrid Azure AD Joined**: Joined to on-prem AD and Azure AD

### Network Security

#### 7. **Azure Firewall**

- **Purpose**: Cloud-native, stateful firewall as a service
- **Connection**:
  - Deployed in hub VNet (hub-spoke topology)
  - All traffic routed through firewall via UDRs
  - Inspects and filters based on FQDN, IP, port
  - Integration with Azure Sentinel for threat intelligence

**Zero Trust Features:**

- **Application Rules**: FQDN-based filtering (allow *.microsoft.com)
- **Network Rules**: Traditional IP/port filtering
- **IDPS**: Intrusion detection and prevention
- **TLS Inspection**: Decrypt and inspect encrypted traffic
- **Threat Intelligence**: Block known malicious IPs
- **DNS Proxy**: Prevent DNS tunneling

#### 8. **Network Security Groups (NSGs)**

- **Purpose**: Subnet and NIC-level firewall
- **Connection**:
  - Applied to subnets and network interfaces
  - Default deny with explicit allow rules
  - Stateful packet filtering

**Zero Trust NSG Strategy:**

- **Microsegmentation**: Separate NSG per subnet/workload
- **Least Privilege**: Only allow required ports/protocols
- **Service Tags**: Use Azure service tags instead of IPs
- **Application Security Groups (ASGs)**: Logical grouping of VMs

**Example NSG Rules:**

```
Priority 100: Allow HTTPS from Azure Front Door service tag
Priority 200: Allow SQL from Application ASG
Priority 300: Allow RDP from Azure Bastion subnet
Priority 4096: Deny all other traffic
```

#### 9. **Azure Bastion**

- **Purpose**: Secure, passwordless RDP/SSH access
- **Connection**:
  - Deployed in dedicated subnet (AzureBastionSubnet)
  - Provides browser-based access via Azure Portal
  - No public IPs on VMs needed
  - Integrates with Azure AD for authentication

**Zero Trust Benefits:**

- **No Exposed RDP/SSH**: VMs have no public IP or open ports
- **Just-In-Time Access**: Combined with PIM for time-limited access
- **MFA Required**: Azure AD MFA enforced before connection
- **Audit Logging**: All sessions logged to Azure Monitor
- **No VPN Required**: Access from anywhere via browser

#### 10. **Azure Private Link / Private Endpoints**

- **Purpose**: Private connectivity to PaaS services
- **Connection**:
  - PaaS services (SQL, Storage, Key Vault) get private IP in VNet
  - Traffic never traverses public internet
  - Accessible from on-premises via VPN/ExpressRoute

**Zero Trust Benefits:**

- **Network Isolation**: Services not exposed to internet
- **No Firewall Exceptions**: Access via private IP
- **Compliance**: Meets data residency requirements
- **Reduced Attack Surface**: Eliminates public endpoints

#### 11. **Azure DDoS Protection**

- **Purpose**: Protection against distributed denial of service attacks
- **Connection**:
  - DDoS Standard plan on VNets
  - Always-on traffic monitoring
  - Automatic mitigation of attacks
  - Integration with Azure Firewall

### Data Protection

#### 12. **Azure Key Vault**

- **Purpose**: Centralized secrets and key management
- **Connection**:
  - Stores secrets, keys, certificates
  - Accessed via Azure AD authenticated identities
  - Private endpoint for network isolation
  - Audit logs to Azure Monitor

**Zero Trust Features:**

- **RBAC**: Granular permissions (Key Vault Reader, Secrets User)
- **Access Policies**: Control which identities access which secrets
- **Soft Delete**: Prevent accidental deletion
- **Purge Protection**: Prevent permanent deletion during retention
- **Network Rules**: Restrict access to specific VNets/IPs
- **Managed HSM**: FIPS 140-2 Level 3 hardware security modules

#### 13. **Managed Identities**

- **Purpose**: Eliminate credentials in code and configuration
- **Connection**:
  - Azure resources authenticate using Azure AD identity
  - No passwords or keys to manage
  - Used for service-to-service authentication

**Types:**

- **System-Assigned**: Tied to resource lifecycle
- **User-Assigned**: Shared across resources

**Example Usage:**

```
VM with Managed Identity → Azure AD → Key Vault
(No credentials stored on VM)
```

#### 14. **Azure Information Protection**

- **Purpose**: Classify and protect documents and emails
- **Connection**:
  - Classifies data (Public, Confidential, Highly Confidential)
  - Applies encryption and access policies
  - Labels follow data wherever it goes

#### 15. **Azure SQL Database Security**

- **Purpose**: Secure data at rest and in transit
- **Zero Trust Features**:
  - **Transparent Data Encryption (TDE)**: Encrypt database at rest
  - **Always Encrypted**: Encrypt sensitive columns (client-side)
  - **Dynamic Data Masking**: Hide data from non-privileged users
  - **Row-Level Security**: Users only see authorized rows
  - **Azure AD Authentication**: No SQL logins, use managed identities
  - **Private Endpoint**: Network isolation
  - **Advanced Threat Protection**: Detect anomalous activities

### Monitoring & Governance

#### 16. **Azure Monitor**

- **Purpose**: Unified monitoring and logging
- **Connection**:
  - Collects logs from all Azure resources
  - Centralized log analytics workspace
  - Real-time alerts on suspicious activity
  - Integration with Azure Sentinel

**Zero Trust Monitoring:**

- **Sign-in logs**: Track authentication attempts
- **Audit logs**: Track resource changes
- **Network traffic logs**: NSG flow logs, firewall logs
- **Security alerts**: Anomaly detection

#### 17. **Microsoft Defender for Cloud**

- **Purpose**: Cloud security posture management (CSPM)
- **Connection**:
  - Continuously assesses security posture
  - Provides security recommendations
  - Detects threats across Azure resources
  - Secure score to measure security

**Defender Plans:**

- **Defender for Servers**: VM protection, vulnerability scanning
- **Defender for SQL**: Database threat detection
- **Defender for Storage**: Malware scanning, anomaly detection
- **Defender for Key Vault**: Unusual access pattern detection

#### 18. **Azure Sentinel**

- **Purpose**: Cloud-native SIEM and SOAR
- **Connection**:
  - Ingests logs from Azure Monitor
  - Machine learning-based threat detection
  - Automated incident response
  - Threat hunting capabilities

**Zero Trust Use Cases:**

- Detect impossible travel login attempts
- Alert on privilege escalation
- Identify lateral movement patterns
- Automated remediation playbooks

#### 19. **Azure Policy**

- **Purpose**: Enforce governance and compliance
- **Connection**:
  - Define and assign policies at scale
  - Audit non-compliant resources
  - Deny deployment of non-compliant resources

**Zero Trust Policies:**

```
- Require encryption on storage accounts
- Require private endpoints for PaaS services
- Deny public IP addresses on VMs
- Require NSGs on all subnets
- Require MFA for privileged accounts
- Require specific Azure regions only
```

## Zero Trust Traffic Flow Examples

### Example 1: User Accessing Azure Portal

1. **User** navigates to portal.azure.com
1. **Azure AD** challenges for username/password
1. **Conditional Access** evaluates:
- Is device compliant? (checks Intune)
- Is location trusted? (checks IP geolocation)
- Is sign-in risky? (checks Identity Protection)
1. **Conditional Access** requires MFA (phone notification)
1. **User** approves MFA
1. **Azure AD** issues access token
1. **User** accesses portal (session monitored continuously)

### Example 2: Application Accessing Database

1. **App Service** needs to query Azure SQL
1. **App Service** uses managed identity (no password)
1. **Azure AD** authenticates managed identity
1. **App Service** connects to SQL via private endpoint (private IP)
1. **Azure Firewall** inspects traffic (if in path)
1. **Azure SQL** verifies identity and RBAC permissions
1. **Connection established** over TLS 1.2+
1. **All activity logged** to Azure Monitor

### Example 3: Admin Accessing VM

1. **Admin** needs to troubleshoot VM
1. **Admin** requests role activation in PIM
1. **PIM** sends approval request to manager
1. **Manager** approves with justification required
1. **Role activated** for 4 hours
1. **Admin** connects to Azure Bastion in portal
1. **Conditional Access** requires MFA + compliant device
1. **Azure Bastion** establishes RDP to VM (no public IP)
1. **Session recorded** and logged
1. **Role expires** after 4 hours (automatic de-escalation)

### Example 4: Detecting and Responding to Threat

1. **Attacker** obtains leaked credentials
1. **Attacker** attempts sign-in from suspicious location
1. **Identity Protection** detects high-risk sign-in
1. **Conditional Access** blocks access or requires password reset
1. **Alert sent** to security team via Azure Sentinel
1. **Sentinel playbook** automatically disables account
1. **Incident created** for investigation
1. **User notified** to reset credentials

## Implementing Zero Trust Step-by-Step

### Phase 1: Identity Foundation (Weeks 1-4)

1. Enable Azure AD MFA for all users
1. Configure Conditional Access policies (start with report-only mode)
1. Implement PIM for privileged roles
1. Enable Azure AD Identity Protection
1. Register all devices with Azure AD

### Phase 2: Device Compliance (Weeks 5-8)

1. Deploy Microsoft Intune
1. Define device compliance policies
1. Enforce compliant device requirement in Conditional Access
1. Deploy security baselines to managed devices
1. Implement mobile application management (MAM)

### Phase 3: Network Security (Weeks 9-12)

1. Deploy Azure Firewall in hub VNet
1. Implement hub-spoke network topology
1. Configure NSGs with least-privilege rules
1. Deploy Azure Bastion for VM access
1. Remove public IPs from VMs
1. Implement private endpoints for PaaS services

### Phase 4: Data Protection (Weeks 13-16)

1. Migrate secrets to Key Vault
1. Enable managed identities for applications
1. Configure private endpoints for Key Vault, SQL, Storage
1. Enable Always Encrypted for sensitive SQL data
1. Implement Azure Information Protection

### Phase 5: Monitoring & Response (Weeks 17-20)

1. Configure centralized logging (Log Analytics)
1. Enable Microsoft Defender for Cloud on all subscriptions
1. Deploy Azure Sentinel
1. Create detection rules and alerts
1. Build automated response playbooks
1. Conduct security drill exercises

## Best Practices

### Identity:

1. **Enforce MFA** for all users, especially admins
1. **Use Conditional Access** for risk-based policies
1. **Eliminate standing privileges** with PIM
1. **Regular access reviews** (quarterly)
1. **Block legacy authentication** protocols

### Network:

1. **Default deny** all traffic, explicitly allow required flows
1. **Microsegmentation** with NSGs per workload
1. **No public IPs** on VMs (use Azure Bastion)
1. **Private endpoints** for all PaaS services
1. **Inspect all traffic** with Azure Firewall

### Data:

1. **Encrypt everything** at rest and in transit
1. **Use managed identities** (no passwords in code)
1. **Key Vault** for all secrets
1. **Classify data** with sensitivity labels
1. **Regular backups** with immutable storage

### Monitoring:

1. **Centralize logs** in Log Analytics
1. **Enable diagnostic settings** on all resources
1. **Real-time alerts** for suspicious activity
1. **Regular security reviews** of Defender for Cloud recommendations
1. **Incident response plan** with regular drills

## Common Challenges

1. **User Experience Impact**: MFA and device compliance can frustrate users
- **Mitigation**: Use passwordless auth, clear communication, exceptions for low-risk scenarios
1. **Legacy Application Compatibility**: Old apps may not support modern auth
- **Mitigation**: Application proxy, gradual migration, risk acceptance with compensating controls
1. **Complexity**: Zero Trust involves many interconnected components
- **Mitigation**: Phased implementation, documentation, training
1. **Cost**: Security tooling (Sentinel, Defender, PIM) adds expense
- **Mitigation**: Start with essentials, use cost management, justify with risk reduction

## Measuring Zero Trust Maturity

### Level 0 - Traditional Security:

- Perimeter-based security
- Trust internal network
- Static credentials

### Level 1 - Initial:

- MFA for external access
- Basic Conditional Access
- Some PaaS services

### Level 2 - Advanced:

- MFA for all users
- Risk-based Conditional Access
- PIM for privileged access
- Network microsegmentation
- Private endpoints

### Level 3 - Optimal:

- Passwordless authentication
- Real-time risk assessment
- Full microsegmentation
- Automated threat response
- Continuous compliance validation

## When to Implement Zero Trust

✅ **Essential for:**

- Organizations handling sensitive data (healthcare, finance)
- Regulated industries (GDPR, HIPAA, PCI DSS)
- Companies with remote workforce
- Cloud-first or cloud-native organizations
- Organizations with previous security incidents

❌ **May not be priority for:**

- Small organizations with limited resources
- Non-critical development environments
- Organizations without security expertise (need to build capability first)

## Learning Resources

- Microsoft Learn: [Zero Trust security](https://learn.microsoft.com/security/zero-trust/)
- Azure Architecture Center: [Zero Trust network](https://learn.microsoft.com/azure/architecture/framework/security/design-network)
- [Microsoft Zero Trust deployment guide](https://learn.microsoft.com/security/zero-trust/deploy/overview)
- [Azure Security Benchmark](https://learn.microsoft.com/security/benchmark/azure/)
- [Microsoft Security documentation](https://learn.microsoft.com/security/)
