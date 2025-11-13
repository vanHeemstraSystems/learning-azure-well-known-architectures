# 400 - Azure Web Application (PaaS) Architecture

## Overview

The Azure Web Application (PaaS) architecture leverages Platform-as-a-Service offerings to build scalable, secure web applications without managing underlying infrastructure. This pattern is ideal for modern web applications, APIs, and SaaS solutions that need rapid deployment and automatic scaling.

## Architecture Diagram Concept

```
Internet
    ↓
[Azure Front Door / CDN]
    ↓
[Azure Application Gateway + WAF]
    ↓
[Azure App Service (Web App)]
    ↓
[Azure SQL Database]
    
Supporting Services:
- Azure Key Vault (secrets)
- Azure Storage (static content)
- Azure Cache for Redis (caching)
- Application Insights (monitoring)
```

## Key Components and Their Connections

### 1. **Azure Virtual Network (VNet)**

- **Purpose**: Provides network isolation for PaaS services
- **Connection**:
  - App Service integrates with VNet for outbound traffic
  - Private endpoints for Azure SQL, Storage, Key Vault
  - Subnet for Application Gateway
- **Note**: VNet integration is optional but recommended for security

### 2. **Azure Front Door**

- **Purpose**: Global load balancer, CDN, and web acceleration
- **Connection**:
  - Entry point for users worldwide
  - Routes traffic to nearest Azure region
  - Caches static content at edge locations
  - Forwards dynamic requests to Application Gateway or App Service

**Front Door Features:**

- **Global Load Balancing**: Route to best performing backend
- **SSL Offload**: Terminate SSL at the edge
- **URL Rewriting**: Modify URLs as needed
- **Session Affinity**: Stick users to same backend
- **Custom Domains**: Support your own domain names

### 3. **Azure CDN (Alternative to Front Door)**

- **Purpose**: Content delivery network for static assets
- **Connection**:
  - Pulls content from Azure Storage or App Service
  - Caches images, CSS, JavaScript at edge
  - Reduces latency for global users
  - Can work alongside Front Door or independently

### 4. **Azure Application Gateway**

- **Purpose**: Regional Layer 7 load balancer with security features
- **Connection**:
  - Deployed in VNet subnet (ApplicationGatewaySubnet)
  - Receives traffic from Front Door or directly from internet
  - Routes to App Service backend pool
  - Provides Web Application Firewall (WAF) protection

**Application Gateway Key Features:**

- **WAF Protection**: OWASP Top 10 protection
- **SSL Termination**: Decrypt traffic at gateway
- **URL-Based Routing**: Route based on URL path
- **Multi-Site Hosting**: Host multiple websites
- **Cookie-Based Session Affinity**: Maintain user sessions
- **Health Probes**: Automatic health checking

**WAF Rule Sets:**

- **OWASP Core Rule Set (CRS)**: Protection against common attacks
- **Bot Protection**: Block malicious bots
- **Custom Rules**: Create organization-specific rules
- **Geo-Filtering**: Block traffic from specific countries

### 5. **Azure App Service**

- **Purpose**: Managed platform for hosting web applications
- **Connection**:
  - Receives traffic from Application Gateway
  - Integrates with VNet for secure backend access
  - Connects to Azure SQL via private endpoint
  - Accesses Key Vault for secrets

**App Service Plans:**

- **Free/Shared**: Development and testing
- **Basic**: Low-traffic apps, no auto-scaling
- **Standard**: Production apps with auto-scaling
- **Premium**: Enhanced performance and VNet integration
- **Isolated**: Dedicated environment in App Service Environment (ASE)

**App Service Features:**

- **Auto-Scaling**: Scale based on metrics (CPU, requests)
- **Deployment Slots**: Blue-green deployments (staging/production)
- **Continuous Deployment**: Integrate with Azure DevOps, GitHub
- **Custom Domains & SSL**: Bring your own domain and certificates
- **Authentication**: Built-in auth with Azure AD, Google, Facebook

### 6. **VNet Integration (App Service)**

- **Purpose**: Secure outbound connectivity from App Service
- **Connection**:
  - App Service joins VNet subnet
  - Enables access to resources on private IPs
  - Routes traffic through VNet to databases, on-premises resources
  - Requires Standard tier or higher

### 7. **Private Endpoints**

- **Purpose**: Private IP addresses for PaaS services
- **Connection**:
  - App Service accesses databases via private IP
  - No traffic traverses public internet
  - Each PaaS service (SQL, Storage, Key Vault) gets endpoint in VNet subnet

**Private Endpoint Benefits:**

- Enhanced security (no public exposure)
- Access from on-premises via VPN/ExpressRoute
- Simplified NSG rules
- Compliance requirements

### 8. **Network Security Groups (NSGs)**

- **Purpose**: Control traffic to subnets
- **Rules**:
  - **Application Gateway Subnet**: Allow inbound 443/80, Azure infrastructure
  - **Private Endpoint Subnet**: Allow App Service subnet traffic
  - **App Service Integration Subnet**: Allow outbound to private endpoints

### 9. **Azure SQL Database**

- **Purpose**: Managed relational database
- **Connection**:
  - Accessed by App Service via private endpoint or VNet service endpoint
  - Uses Azure AD authentication or SQL authentication
  - Firewall rules allow only App Service IPs (if not using private endpoint)
  - Connection strings stored in Key Vault

**Azure SQL Features:**

- **Automatic Backups**: Point-in-time restore
- **Geo-Replication**: Read replicas in other regions
- **Auto-Tuning**: Performance optimization
- **Elastic Pools**: Share resources among databases
- **Threat Detection**: Anomaly detection and alerts

**Security Best Practices:**

- Enable Transparent Data Encryption (TDE)
- Use Azure AD authentication with managed identity
- Implement private endpoint for network isolation
- Enable Azure SQL Auditing
- Use Always Encrypted for sensitive data

### 10. **Azure Key Vault**

- **Purpose**: Centralized secrets management
- **Connection**:
  - App Service accesses via managed identity
  - Stores connection strings, API keys, certificates
  - Private endpoint for secure access
  - Integrated with App Service via app settings

**Key Vault Integration:**

```csharp
// App Service uses managed identity to access Key Vault
// No credentials in code or config files
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/sqlconnection/)
```

### 11. **Azure Storage Account**

- **Purpose**: Store static content, user uploads, logs
- **Connection**:
  - App Service uploads files to Blob Storage
  - CDN/Front Door serves static content from Storage
  - Private endpoint for secure access
  - Blob Storage can serve as website origin

**Storage Types:**

- **Blob Storage**: Unstructured data (images, videos, documents)
- **File Storage**: SMB file shares
- **Queue Storage**: Message queuing
- **Table Storage**: NoSQL key-value store

### 12. **Azure Cache for Redis**

- **Purpose**: In-memory cache for performance optimization
- **Connection**:
  - App Service connects to Redis for session state, caching
  - Reduces database load
  - Improves response times
  - Private endpoint for secure connectivity

**Redis Use Cases:**

- Session state management
- Output caching
- Rate limiting
- Real-time analytics

### 13. **Application Insights**

- **Purpose**: Application performance monitoring (APM)
- **Connection**:
  - Integrated with App Service via SDK or auto-instrumentation
  - Collects telemetry (requests, dependencies, exceptions)
  - Distributed tracing across components
  - Real-time metrics and alerting

**Monitoring Capabilities:**

- **Live Metrics**: Real-time performance view
- **Application Map**: Visualize dependencies
- **Transaction Diagnostics**: End-to-end tracing
- **Availability Tests**: Synthetic monitoring
- **Custom Metrics**: Business KPIs

### 14. **Azure Monitor**

- **Purpose**: Unified monitoring platform
- **Connection**:
  - Aggregates logs from all Azure resources
  - Creates alerts based on metrics and logs
  - Dashboards for visualization
  - Integration with Azure Logic Apps for automation

### 15. **Azure AD (Entra ID)**

- **Purpose**: Identity and access management
- **Connection**:
  - Authenticates users to web application
  - App Service uses Easy Auth feature
  - Azure SQL uses Azure AD authentication
  - Managed identity for service-to-service auth

## Traffic Flow Examples

### Example 1: User Request for Dynamic Content

1. **User** requests https://www.contoso.com/products
1. **Front Door** receives request at edge location
1. **Front Door** routes to Application Gateway in home region
1. **Application Gateway** applies WAF rules, checks for threats
1. **Application Gateway** forwards to App Service
1. **App Service** processes request, queries Azure SQL via private endpoint
1. **Azure SQL** returns data
1. **App Service** renders page, returns to Application Gateway
1. **Application Gateway** returns to Front Door
1. **Front Door** returns to user

### Example 2: Static Content Delivery

1. **User** requests https://www.contoso.com/images/logo.png
1. **Front Door/CDN** checks cache at edge location
1. **Cache hit**: Returns image directly from edge (fast!)
1. **Cache miss**: Pulls from Azure Storage origin, caches, returns to user
1. **Subsequent requests**: Served from edge cache

### Example 3: Background Job Processing

1. **User** uploads file to App Service
1. **App Service** saves file to Blob Storage via private endpoint
1. **App Service** writes message to Storage Queue
1. **Azure Function** triggered by queue message
1. **Function** processes file, updates Azure SQL
1. **Function** sends notification via Logic App

### Example 4: Secure Database Connection

1. **App Service** needs database connection string
1. **App Service** uses managed identity to authenticate to Key Vault
1. **Key Vault** returns connection string via private endpoint
1. **App Service** connects to Azure SQL using connection string
1. **Azure SQL** authenticates App Service via managed identity
1. **Connection established** over private IP (no internet traversal)

## Security Implementation

### 1. **Network Security**

- Private endpoints for all PaaS services
- VNet integration for App Service
- NSGs on all subnets
- Application Gateway with WAF enabled
- Disable public access to databases and storage

### 2. **Identity Security**

- Azure AD authentication for users
- Managed identities for service-to-service
- No credentials in code or configuration
- Key Vault for all secrets

### 3. **Data Security**

- TLS 1.2+ for all connections
- Encryption at rest for SQL, Storage
- Always Encrypted for sensitive SQL columns
- Private endpoints for data services

### 4. **Application Security**

- WAF protection against OWASP Top 10
- DDoS protection (standard on Front Door/App Gateway)
- Regular security updates (managed by Azure)
- Vulnerability scanning via Defender for Cloud

## Scaling Strategies

### 1. **App Service Scaling**

- **Scale Up**: Increase instance size (more CPU/RAM)
- **Scale Out**: Add more instances (up to 30 in Premium)
- **Auto-Scale Rules**: Based on CPU, memory, HTTP queue length, custom metrics
- **Schedule-Based Scaling**: Scale based on time of day

### 2. **Database Scaling**

- **DTU-Based**: Scale DTUs up/down
- **vCore-Based**: Scale vCores and storage independently
- **Serverless**: Auto-pause during inactivity
- **Hyperscale**: Up to 100TB databases with fast scaling

### 3. **Global Scaling**

- Deploy App Service in multiple regions
- Use Front Door for global load balancing
- Geo-replicate Azure SQL Database
- Use Azure Traffic Manager for DNS-based routing

## High Availability

### 1. **App Service HA**

- Deploy to multiple availability zones (zone-redundant)
- Use deployment slots for zero-downtime deployments
- Health checks with automatic instance replacement

### 2. **Database HA**

- Zone-redundant deployment
- Geo-replication for disaster recovery
- Automatic failover groups

### 3. **Storage HA**

- Locally Redundant Storage (LRS) - 3 copies in region
- Geo-Redundant Storage (GRS) - 6 copies across regions
- Zone-Redundant Storage (ZRS) - 3 copies across zones

## Cost Optimization

1. **Right-Size App Service Plan**: Start small, scale as needed
1. **Reserved Instances**: Save up to 55% with 1-3 year commitment
1. **Auto-Scaling**: Scale down during off-peak hours
1. **Serverless SQL**: Pay only when database is active
1. **CDN Caching**: Reduce App Service load and costs
1. **Azure Hybrid Benefit**: Use existing licenses

## Best Practices

1. **Enable Diagnostic Logging**: App Service, SQL, Application Gateway
1. **Use Deployment Slots**: Test changes before production
1. **Implement Circuit Breakers**: Handle dependency failures gracefully
1. **Health Endpoints**: Expose /health endpoints for monitoring
1. **Connection Pooling**: Reuse database connections
1. **Async Operations**: Use async/await for I/O operations
1. **Regular Backups**: Automated backups for App Service and SQL
1. **Security Scanning**: Use Defender for Cloud

## When to Use PaaS Architecture

✅ **Good for:**

- Modern web applications and APIs
- SaaS applications
- Organizations wanting minimal infrastructure management
- Rapid development and deployment needs
- Applications requiring auto-scaling

❌ **Not ideal for:**

- Legacy applications requiring specific OS configurations
- Applications needing full server control
- Extremely cost-sensitive scenarios (VMs may be cheaper)
- Applications with complex networking requirements

## Common Patterns

### 1. **Blue-Green Deployment**

- Use deployment slots
- Deploy to staging slot
- Test thoroughly
- Swap staging to production (instant)

### 2. **Canary Deployment**

- Route 10% traffic to new version
- Monitor for errors
- Gradually increase traffic
- Rollback if issues detected

### 3. **Multi-Region Active-Active**

- Deploy to multiple regions
- Front Door routes to nearest region
- Write to primary database, read from replicas
- Automatic failover on region failure

## Learning Resources

- Microsoft Learn: [Azure App Service](https://learn.microsoft.com/azure/app-service/)
- Azure Architecture Center: [Basic web application](https://learn.microsoft.com/azure/architecture/reference-architectures/app-service-web-app/basic-web-app)
- [Application Gateway documentation](https://learn.microsoft.com/azure/application-gateway/)
- [Azure SQL Database best practices](https://learn.microsoft.com/azure/sql-database/)
