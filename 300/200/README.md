# 200 - Azure Microservices Architecture

## Overview

Microservices architecture breaks down applications into small, independent services that can be developed, deployed, and scaled independently. Each service focuses on a specific business capability and communicates with other services through well-defined APIs.

## Architecture Diagram Concept

```
Internet
    ↓
[Azure Front Door / Application Gateway]
    ↓
[Azure API Management]
    ↓
    ├─→ [Microservice A in AKS]
    ├─→ [Microservice B in AKS]
    └─→ [Microservice C in Container Instance]
    ↓
[Azure Service Bus / Event Grid]
    ↓
[Azure Cosmos DB / Azure SQL]
```

## Key Components and Their Connections

### 1. **Azure Virtual Network (VNet)**

- **Purpose**: Provides network isolation for all microservices
- **Connection**:
  - Contains all Azure resources
  - Segmented into multiple subnets for different services
  - Enables private communication between services

### 2. **Azure Front Door**

- **Purpose**: Global load balancer and CDN with intelligent routing
- **Connection**:
  - Entry point for global users
  - Routes traffic to nearest Azure region
  - Provides SSL termination and caching
  - Forwards traffic to API Management or Application Gateway

### 3. **Azure Application Gateway**

- **Purpose**: Regional Layer 7 load balancer with WAF
- **Connection**:
  - Receives traffic from Front Door or directly from users
  - Routes traffic to API Management
  - Provides Web Application Firewall protection
  - Performs path-based routing

### 4. **Azure API Management (API Gateway)**

- **Purpose**: Central API gateway for all microservices
- **Connection**:
  - Receives all external API requests
  - Authenticates and authorizes requests
  - Routes requests to appropriate microservices
  - Applies rate limiting, throttling, and caching
  - Transforms requests/responses
  - Deployed in the VNet (internal mode) or external mode

**Key API Management Features:**

- **Policies**: Apply transformations, authentication, rate limiting
- **Products**: Group APIs for different consumer audiences
- **Subscriptions**: Control access with API keys
- **Developer Portal**: Self-service API documentation

### 5. **Azure Kubernetes Service (AKS)**

- **Purpose**: Managed Kubernetes cluster for container orchestration
- **Connection**:
  - Hosts multiple microservices as containerized applications
  - Deployed in its own subnet within the VNet
  - Receives traffic from API Management
  - Manages scaling, health checks, and rolling updates
  - Uses Azure Container Registry for container images

**AKS Components:**

- **Pods**: Individual microservice instances
- **Services**: Load balance traffic to pods
- **Ingress Controller**: Routes external traffic to services
- **Namespaces**: Logical separation of microservices

### 6. **Azure Container Registry (ACR)**

- **Purpose**: Private Docker registry for container images
- **Connection**:
  - Stores Docker images for all microservices
  - AKS pulls images from ACR during deployment
  - Connected via service principal or managed identity
  - Can use Private Link for secure access

### 7. **Azure Container Instances (ACI)**

- **Purpose**: Serverless containers for lightweight microservices
- **Connection**:
  - Hosts simple microservices that don’t need full orchestration
  - Deployed in VNet subnet
  - Accessed through API Management
  - Faster startup than AKS for bursty workloads

### 8. **Network Security Groups (NSGs)**

- **Purpose**: Control traffic between microservices and subnets
- **Rules**:
  - **AKS Subnet**: Allow traffic from API Management, deny direct internet access
  - **Container Instance Subnet**: Allow traffic from API Management
  - **Database Subnet**: Allow traffic only from microservice subnets

### 9. **Azure Service Bus**

- **Purpose**: Message broker for asynchronous communication between microservices
- **Connection**:
  - Microservices publish messages to topics/queues
  - Other microservices subscribe to receive messages
  - Enables decoupled, event-driven communication
  - Uses Premium tier for VNet integration

**Service Bus Patterns:**

- **Queues**: Point-to-point messaging
- **Topics/Subscriptions**: Publish-subscribe pattern
- **Dead Letter Queue**: Handle failed messages

### 10. **Azure Event Grid**

- **Purpose**: Event routing service for reactive programming
- **Connection**:
  - Microservices publish domain events
  - Subscribers receive events and react
  - Integrates with Azure services and custom webhooks
  - Provides event filtering and routing

### 11. **Azure Cosmos DB**

- **Purpose**: Globally distributed NoSQL database
- **Connection**:
  - Each microservice can have its own Cosmos DB container/collection
  - Accessed via private endpoint in VNet
  - Provides low-latency, high-throughput data access
  - Supports multiple consistency models

### 12. **Azure SQL Database**

- **Purpose**: Managed relational database for microservices requiring SQL
- **Connection**:
  - Individual databases per microservice (database-per-service pattern)
  - Accessed via private endpoint or VNet service endpoint
  - Microservices connect using managed identity

### 13. **Azure Key Vault**

- **Purpose**: Centralized secrets management
- **Connection**:
  - Stores API keys, connection strings, certificates
  - Microservices access via managed identity
  - AKS uses Key Vault provider for Secrets Store CSI driver
  - Private endpoint for secure access

### 14. **Azure Monitor & Application Insights**

- **Purpose**: Observability and monitoring
- **Connection**:
  - Collects logs and metrics from all microservices
  - Distributed tracing across service calls
  - Performance monitoring and alerting
  - Integrated with AKS, Container Instances, and API Management

### 15. **Azure Bastion**

- **Purpose**: Secure management access to AKS nodes
- **Connection**:
  - Deployed in dedicated subnet
  - Provides SSH access to AKS node VMs for troubleshooting
  - No public IPs needed on worker nodes

## Traffic Flow Example

### Synchronous Request Flow:

1. **User** sends HTTPS request to Front Door endpoint
1. **Front Door** routes to nearest region’s Application Gateway
1. **Application Gateway** forwards to API Management
1. **API Management** authenticates request, applies policies, routes to appropriate microservice
1. **AKS Ingress Controller** receives request, routes to correct pod
1. **Microservice Pod** processes request, queries Cosmos DB if needed
1. **Response flows back** through AKS → API Management → Application Gateway → Front Door → User

### Asynchronous Event Flow:

1. **Microservice A** publishes event to Service Bus topic
1. **Service Bus** stores message durably
1. **Microservice B** subscribes to topic, receives message
1. **Microservice B** processes event, updates its own database
1. **Microservice B** publishes completion event to Event Grid
1. **Microservice C** receives Event Grid notification, performs additional processing

### Inter-Service Communication:

1. **Microservice A** needs data from Microservice B
1. **Microservice A** calls API Management internal endpoint
1. **API Management** routes to Microservice B in AKS
1. **Microservice B** returns data
1. **Alternative**: Microservice A directly calls Microservice B using internal AKS service DNS

## Communication Patterns

### 1. **API Gateway Pattern**

- All external traffic goes through API Management
- Provides single entry point
- Handles cross-cutting concerns (auth, logging, rate limiting)

### 2. **Service Mesh (Optional)**

- Tools like Istio or Linkerd on AKS
- Handles service-to-service communication
- Provides traffic management, security, observability

### 3. **Event-Driven Pattern**

- Services publish/subscribe to events
- Loose coupling between services
- Eventual consistency

### 4. **Database per Service**

- Each microservice owns its database
- No shared databases between services
- Maintains service independence

## Security Implementation

### 1. **Network Security**

- NSGs on all subnets
- Private endpoints for PaaS services
- No direct internet access to microservices
- API Management as security gateway

### 2. **Identity & Access**

- Managed identities for Azure resource access
- Azure AD authentication for API Management
- OAuth 2.0 / OpenID Connect for user authentication
- Service principal for CI/CD pipelines

### 3. **Secrets Management**

- All secrets in Key Vault
- No hardcoded credentials
- Regular secret rotation
- AKS secrets mounted from Key Vault

### 4. **API Security**

- JWT token validation in API Management
- Rate limiting per API/subscription
- IP whitelisting for admin APIs
- TLS 1.2+ enforcement

## Scaling Strategies

### 1. **AKS Scaling**

- **Horizontal Pod Autoscaler (HPA)**: Scale pods based on CPU/memory
- **Cluster Autoscaler**: Add/remove nodes based on demand
- **Virtual Nodes**: Burst to Azure Container Instances

### 2. **API Management Scaling**

- Premium tier supports multiple units
- Auto-scaling based on request rate
- Multi-region deployment for global scale

### 3. **Database Scaling**

- Cosmos DB: Autoscale throughput
- Azure SQL: Elastic pools, hyperscale tier

## Deployment Pipeline

1. **Developer** commits code to Azure DevOps / GitHub
1. **CI Pipeline** builds Docker image, pushes to Container Registry
1. **CD Pipeline** updates Kubernetes manifests
1. **AKS** performs rolling update of pods
1. **API Management** automatically routes to new instances
1. **Zero downtime** deployment achieved

## Best Practices

1. **Design for Failure**: Implement circuit breakers, retries, timeouts
1. **Distributed Tracing**: Use correlation IDs across all services
1. **API Versioning**: Support multiple API versions in API Management
1. **Health Checks**: Implement liveness and readiness probes in AKS
1. **Logging**: Structured logging with consistent format
1. **Documentation**: Keep API documentation up-to-date in API Management
1. **Testing**: Implement contract testing between services
1. **Resource Limits**: Set CPU/memory limits on all containers

## When to Use Microservices Architecture

✅ **Good for:**

- Large, complex applications with multiple teams
- Applications requiring independent scaling of components
- Cloud-native applications built for agility
- Organizations with DevOps maturity

❌ **Not ideal for:**

- Small applications with limited complexity
- Teams without container/Kubernetes expertise
- When strong consistency is critical across all data
- Projects with tight deadlines and limited resources

## Common Challenges

1. **Complexity**: More moving parts than monolithic apps
1. **Distributed Transactions**: Eventual consistency can be challenging
1. **Testing**: Integration testing is more complex
1. **Debugging**: Requires distributed tracing tools
1. **Data Consistency**: Maintaining consistency across services
1. **Network Latency**: Inter-service calls add latency

## Learning Resources

- Microsoft Learn: [Microservices architecture on AKS](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices)
- Azure Architecture Center: [Design a microservices architecture](https://learn.microsoft.com/azure/architecture/microservices/)
- [AKS documentation](https://learn.microsoft.com/azure/aks/)
- [API Management documentation](https://learn.microsoft.com/azure/api-management/)
