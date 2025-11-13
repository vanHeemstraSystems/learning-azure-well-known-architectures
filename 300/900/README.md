# 900 - Azure Serverless Architecture

## Overview

Serverless architecture allows you to build and run applications without managing infrastructure. You write code that responds to events, and Azure automatically provisions compute resources, scales them, and manages the underlying servers. You only pay for actual compute time used, making it highly cost-effective for variable workloads.

## Architecture Diagram Concept

```
         [Event Sources]
    ┌──────────┬──────────┬──────────┐
    │          │          │          │
[HTTP Trigger] [Timer]  [Queue]  [Blob Storage]
    │          │          │          │
    └──────────┴──────────┴──────────┘
                    ↓
            [Azure Functions]
                    ↓
         ┌──────────┴──────────┐
         │                     │
    [Cosmos DB]          [Logic Apps]
         │                     │
    [Event Grid]       [Service Bus]
```

## Key Components and Their Connections

### Compute Services

#### 1. **Azure Functions**

- **Purpose**: Event-driven, serverless compute service
- **Connection**:
  - Triggered by various event sources (HTTP, timer, queue, blob, etc.)
  - Executes code in response to events
  - Automatically scales based on demand
  - Can call other Azure services or external APIs

**Supported Languages:**

- C#, Java, JavaScript/TypeScript, Python, PowerShell

**Hosting Plans:**

- **Consumption Plan**: Pay-per-execution, automatic scaling, 5-minute timeout
- **Premium Plan**: Pre-warmed instances, unlimited execution time, VNet connectivity
- **Dedicated Plan**: Run on App Service plan (predictable pricing)

**Function Triggers (Event Sources):**

- **HTTP Trigger**: Respond to HTTP requests (REST APIs)
- **Timer Trigger**: Run on schedule (cron expressions)
- **Queue Trigger**: Process messages from Azure Storage Queue or Service Bus
- **Blob Trigger**: React to new/modified files in Blob Storage
- **Event Grid Trigger**: React to Azure events
- **Event Hub Trigger**: Process streaming data
- **Cosmos DB Trigger**: React to database changes

**Bindings:**

- **Input Bindings**: Read data from services (Cosmos DB, Blob Storage, SQL)
- **Output Bindings**: Write data to services without explicit connection code

#### 2. **Azure Logic Apps**

- **Purpose**: Visual workflow orchestration with 400+ connectors
- **Connection**:
  - Triggered by schedules, events, or HTTP requests
  - Orchestrates complex business processes
  - Integrates with Azure Functions, SaaS apps, on-premises systems
  - No code required (low-code/no-code platform)

**Common Triggers:**

- HTTP request received
- Schedule (recurrence)
- Event Grid event
- Service Bus message
- Email received (Office 365, Gmail)
- File created (OneDrive, SharePoint)

**Common Actions:**

- Send email
- Create database record
- Call Azure Function
- Transform data
- Conditional logic (if/else, loops)
- Parallel branches

**Logic Apps vs Functions:**

- **Logic Apps**: Workflow orchestration, visual designer, low-code
- **Functions**: Custom code execution, developer-focused

#### 3. **Azure Container Instances (ACI)**

- **Purpose**: Serverless containers for specific workloads
- **Connection**:
  - Runs containerized applications without managing VMs
  - Fast startup (seconds)
  - Triggered by Functions or Logic Apps
  - Useful for batch processing, scheduled jobs

### Data Services

#### 4. **Azure Cosmos DB**

- **Purpose**: Globally distributed, multi-model NoSQL database
- **Connection**:
  - Serverless pricing tier (pay per request)
  - Automatic scaling
  - Functions use Cosmos DB triggers and bindings
  - Low latency for global users

**Cosmos DB Features:**

- **APIs**: SQL, MongoDB, Cassandra, Gremlin, Table
- **Consistency Models**: 5 levels from eventual to strong
- **Change Feed**: Triggers Functions on data changes
- **Global Distribution**: Multi-region writes

**Serverless Pricing:**

- Pay per Request Unit (RU) consumed
- No minimum throughput charge
- Auto-scales from zero

#### 5. **Azure Table Storage**

- **Purpose**: NoSQL key-value store (cheaper alternative to Cosmos DB)
- **Connection**:
  - Functions read/write via bindings
  - Good for structured non-relational data
  - Lower cost than Cosmos DB

#### 6. **Azure Blob Storage**

- **Purpose**: Object storage for files, images, videos
- **Connection**:
  - Functions triggered on blob upload
  - Store function output to blobs
  - Hosting for static websites
  - Lifecycle management (auto-tiering to cool/archive)

### Messaging & Events

#### 7. **Azure Event Grid**

- **Purpose**: Event routing service for reactive programming
- **Connection**:
  - Publishes events from Azure services
  - Routes events to Functions, Logic Apps, webhooks
  - Near real-time event delivery (<1 second)
  - Built-in retry and dead-lettering

**Event Sources:**

- Azure subscriptions and resource groups
- Blob Storage (file created/deleted)
- Event Hubs
- Service Bus
- Azure Maps
- Custom topics (your application)

**Event Handlers:**

- Azure Functions
- Logic Apps
- Azure Automation
- Webhooks
- Event Hubs
- Service Bus

**Event Schema:**

```json
{
  "topic": "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}",
  "subject": "/blobServices/default/containers/{container}/blobs/{file}.txt",
  "eventType": "Microsoft.Storage.BlobCreated",
  "eventTime": "2025-11-13T10:30:00.0000000Z",
  "id": "unique-event-id",
  "data": {
    "api": "PutBlob",
    "url": "https://myaccount.blob.core.windows.net/container/file.txt"
  }
}
```

#### 8. **Azure Service Bus**

- **Purpose**: Enterprise message broker with queues and topics
- **Connection**:
  - Functions triggered by queue/topic messages
  - Reliable message delivery
  - FIFO ordering (sessions)
  - Duplicate detection
  - Dead-letter queues

**Queue vs Topic:**

- **Queue**: Point-to-point (one sender, one receiver)
- **Topic**: Pub-sub (one sender, multiple subscribers with filters)

#### 9. **Azure Event Hubs**

- **Purpose**: Big data streaming platform (millions of events/second)
- **Connection**:
  - Functions process streaming data
  - Real-time analytics
  - Event capture to Blob Storage
  - Integration with Azure Stream Analytics

**Use Cases:**

- Telemetry ingestion
- Log aggregation
- Click stream analysis
- IoT data processing

### Integration & Orchestration

#### 10. **Durable Functions**

- **Purpose**: Stateful workflows in serverless Functions
- **Connection**:
  - Orchestrates multiple functions
  - Maintains state across function executions
  - Built-in checkpointing and replay
  - Supports long-running processes

**Durable Function Patterns:**

1. **Function Chaining**: Call functions in sequence
1. **Fan-out/Fan-in**: Parallel execution, aggregate results
1. **Async HTTP APIs**: Long-running HTTP operations
1. **Monitoring**: Periodic polling with backoff
1. **Human Interaction**: Wait for external approval

**Example Orchestration:**

```csharp
[FunctionName("OrderProcessing")]
public static async Task<object> Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    // Step 1: Validate inventory
    var isValid = await context.CallActivityAsync<bool>("ValidateInventory", order);
    if (!isValid) return "Inventory unavailable";
    
    // Step 2: Charge payment
    await context.CallActivityAsync("ChargePayment", order);
    
    // Step 3: Ship order
    await context.CallActivityAsync("ShipOrder", order);
    
    return "Order completed";
}
```

#### 11. **Azure API Management**

- **Purpose**: API gateway for serverless functions
- **Connection**:
  - Provides single endpoint for multiple functions
  - Rate limiting and throttling
  - Authentication and authorization
  - Request/response transformation
  - API versioning

### Supporting Services

#### 12. **Azure Key Vault**

- **Purpose**: Secrets management
- **Connection**:
  - Functions access secrets via managed identity
  - Stores API keys, connection strings, certificates
  - No secrets in application code or configuration

**Key Vault References in Functions:**

```
@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/apikey/)
```

#### 13. **Application Insights**

- **Purpose**: Application performance monitoring
- **Connection**:
  - Automatically integrated with Functions
  - Distributed tracing across functions
  - Live metrics stream
  - Custom telemetry

**Monitoring Metrics:**

- Execution count
- Success/failure rate
- Duration
- Dependencies (SQL, HTTP, Storage)
- Exceptions

#### 14. **Azure Monitor**

- **Purpose**: Centralized logging and alerting
- **Connection**:
  - Collects logs from all serverless components
  - Creates alerts on function failures
  - Log Analytics queries for troubleshooting

## Serverless Architecture Patterns

### Pattern 1: HTTP API Backend

```
Client → API Management → Azure Functions → Cosmos DB
```

**Use Case:** REST API for mobile/web apps

**Implementation:**

- HTTP-triggered functions for CRUD operations
- API Management for security and rate limiting
- Cosmos DB for data storage
- Application Insights for monitoring

### Pattern 2: Event-Driven Data Processing

```
Blob Upload → Event Grid → Function → Process → Cosmos DB
```

**Use Case:** Image processing, file conversion

**Implementation:**

1. User uploads file to Blob Storage
1. Event Grid publishes BlobCreated event
1. Function triggered, downloads blob
1. Function processes file (resize, convert, analyze)
1. Function saves results to Cosmos DB
1. Function saves processed file back to Blob Storage

### Pattern 3: Scheduled Batch Processing

```
Timer Trigger → Function → Query Database → Generate Report → Send Email
```

**Use Case:** Nightly reports, data cleanup

**Implementation:**

- Timer-triggered function (cron: `0 0 2 * * *` = 2 AM daily)
- Query data from SQL or Cosmos DB
- Generate report (PDF, Excel)
- Save to Blob Storage
- Send email via SendGrid or Logic Apps

### Pattern 4: Queue-Based Work Distribution

```
Web App → Queue → Function 1 → Function 2 → Function 3
```

**Use Case:** Order processing, background jobs

**Implementation:**

1. Web app places message in Service Bus queue
1. Function 1 validates order
1. Function 1 places message in queue for Function 2
1. Function 2 processes payment
1. Function 2 places message in queue for Function 3
1. Function 3 ships order

### Pattern 5: Stream Processing

```
IoT Devices → Event Hubs → Function → Stream Analytics → Power BI
```

**Use Case:** Real-time analytics, IoT telemetry

**Implementation:**

- IoT devices send telemetry to Event Hubs
- Function processes each event (filtering, enrichment)
- Stream Analytics performs aggregations
- Results written to Cosmos DB
- Power BI visualizes real-time dashboards

### Pattern 6: Saga Pattern with Durable Functions

```
Orchestrator Function → Activity 1 → Activity 2 → Activity 3
                     ↓ (on failure)
                  Compensate 2 → Compensate 1
```

**Use Case:** Distributed transactions

**Implementation:**

- Orchestrator manages workflow
- Each activity is a function
- On failure, orchestrator runs compensation functions
- Maintains consistency across services

## Traffic Flow Examples

### Example 1: File Upload Processing

1. **User** uploads image to Blob Storage via web app
1. **Blob Storage** triggers Event Grid event
1. **Event Grid** routes BlobCreated event to Function
1. **Function** downloads image from Blob Storage
1. **Function** resizes image using image library
1. **Function** uploads thumbnail to different blob container
1. **Function** writes metadata to Cosmos DB
1. **Total time**: <2 seconds from upload to completion

### Example 2: Order Placement

1. **Customer** submits order via mobile app
1. **API Management** receives HTTP POST request
1. **API Management** routes to “PlaceOrder” Function
1. **Function** validates order (inventory check)
1. **Function** writes order to Cosmos DB
1. **Function** publishes “OrderCreated” event to Event Grid
1. **Event Grid** triggers “ProcessPayment” Function
1. **ProcessPayment** Function charges credit card
1. **ProcessPayment** Function publishes “PaymentCompleted” event
1. **Event Grid** triggers “FulfillOrder” Logic App
1. **Logic App** sends confirmation email and notifies warehouse

### Example 3: Scheduled Report Generation

1. **Timer Trigger** fires at 6 AM daily
1. **Function** queries Azure SQL Database
1. **Function** generates Excel report
1. **Function** saves report to Blob Storage
1. **Function** publishes to Service Bus topic
1. **Logic App** (subscribed to topic) sends report via email
1. **Execution time**: 2 minutes
1. **Cost**: ~$0.01 per day

## Security Best Practices

### 1. **Authentication & Authorization**

- Enable Azure AD authentication on Functions
- Use managed identities for Azure resource access
- API Management for API key management
- Function-level authorization codes (dev/test only)

### 2. **Network Security**

- Premium plan: VNet integration for outbound traffic
- Private endpoints for Cosmos DB, Storage
- IP restrictions on Functions
- API Management as security gateway

### 3. **Secrets Management**

- Store all secrets in Key Vault
- Reference secrets via Key Vault references
- Never hardcode credentials
- Rotate secrets regularly

### 4. **Input Validation**

- Validate all input in Functions
- Sanitize data to prevent injection
- Rate limiting via API Management
- Maximum execution timeout

## Cost Optimization

### 1. **Choose Right Plan**

- **Consumption**: Variable workloads, unpredictable traffic
- **Premium**: Consistent traffic, need VNet
- **Dedicated**: Already have App Service plan

### 2. **Optimize Function Execution**

- Minimize cold start with keep-alive or Premium plan
- Reduce execution time (faster = cheaper)
- Use async operations
- Connection pooling for databases

### 3. **Serverless vs Always-On**

- Serverless: <100 executions/day or highly variable
- Always-On (App Service): >10,000 executions/day

### 4. **Data Transfer Costs**

- Keep Functions and data in same region
- Use Cosmos DB serverless for unpredictable traffic
- Blob lifecycle management (cool/archive tiers)

### Example Cost Calculation (Consumption Plan):

```
Function executions: 1 million/month
Execution time: 500ms average
Memory: 512 MB

Cost breakdown:
- Executions: $0.20 (first 1M free, then $0.20/million)
- Compute: ~$6.00
- Total: ~$6.20/month

Compare to:
- Basic App Service: $13/month (always running)
- Standard App Service: $74/month
```

## Monitoring & Troubleshooting

### 1. **Application Insights**

- Live metrics for real-time debugging
- Transaction tracing across functions
- Custom metrics and events
- Availability tests

### 2. **Log Analytics**

- Query logs with Kusto Query Language (KQL)
- Create dashboards
- Long-term log retention

### 3. **Common Issues**

- **Cold Start**: First execution slow (use Premium or keep-alive)
- **Timeout**: Increase timeout or use Durable Functions
- **Connection Limits**: Use connection pooling
- **Trigger Not Firing**: Check permissions, storage account connectivity

## Best Practices

1. **Keep Functions Small**: Single responsibility principle
1. **Idempotency**: Functions should handle duplicate events safely
1. **Error Handling**: Try-catch blocks, poison queue for failures
1. **Async/Await**: Use async for I/O operations
1. **Connection Pooling**: Reuse database connections
1. **Logging**: Structured logging with correlation IDs
1. **Testing**: Unit tests for functions, integration tests for workflows
1. **Deployment**: Use deployment slots for zero-downtime deployments

## When to Use Serverless

✅ **Good for:**

- Event-driven applications
- Variable/unpredictable workloads
- Microservices and APIs
- Background processing and batch jobs
- Scheduled tasks
- Real-time file/data processing

❌ **Not ideal for:**

- Long-running processes (>10 minutes) - use Batch or VMs
- CPU-intensive workloads - use VMs or AKS
- Consistent high-volume traffic - App Service may be cheaper
- Applications requiring stateful connections

## Learning Resources

- Microsoft Learn: [Azure Functions documentation](https://learn.microsoft.com/azure/azure-functions/)
- [Serverless architecture patterns](https://learn.microsoft.com/azure/architecture/serverless-quest/serverless-overview)
- [Durable Functions](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview)
- [Logic Apps documentation](https://learn.microsoft.com/azure/logic-apps/)
- [Event Grid documentation](https://learn.microsoft.com/azure/event-grid/)
