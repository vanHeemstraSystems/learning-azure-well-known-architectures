# 700 - Azure Event-Driven Architecture

## Overview

Event-Driven Architecture (EDA) is a design pattern where systems communicate through events rather than direct calls. An event represents a significant change in state (e.g., “order placed”, “file uploaded”, “user registered”). Services produce and consume events asynchronously, enabling loose coupling, scalability, and resilience. This pattern is fundamental to modern distributed systems and microservices.

## Architecture Diagram Concept

```
[Event Producers]
    ├─→ Web App
    ├─→ IoT Devices
    ├─→ Mobile Apps
    └─→ External Systems
         ↓
[Event Ingestion]
    ├─→ Azure Event Hubs (streaming)
    ├─→ Azure Event Grid (discrete events)
    └─→ Azure Service Bus (messaging)
         ↓
[Event Processing]
    ├─→ Azure Functions
    ├─→ Logic Apps
    ├─→ Stream Analytics
    └─→ Azure Databricks
         ↓
[Event Storage & Consumers]
    ├─→ Azure Cosmos DB
    ├─→ Azure Storage
    ├─→ Power BI
    └─→ External Applications
```

## Key Components and Their Connections

### Event Ingestion Services

#### 1. **Azure Event Grid**

- **Purpose**: Intelligent event routing service for reactive programming
- **Connection**:
  - Receives events from Azure services or custom applications
  - Routes events to handlers based on subscriptions and filters
  - Provides reliable delivery with retry and dead-lettering
  - Near real-time delivery (<1 second)

**Event Grid Concepts:**

- **Event Source**: Where events originate (Storage, Resource Group, custom topics)
- **Topic**: Endpoint where events are sent
- **Event Subscription**: Defines which events go to which handlers
- **Event Handler**: Destination for events (Functions, Logic Apps, webhooks)

**Built-in Event Sources:**

- Azure Subscriptions (resource creation/deletion)
- Resource Groups
- Blob Storage (file created/deleted/renamed)
- Event Hubs
- Azure Maps
- Media Services
- IoT Hub
- Container Registry
- Service Bus

**Event Schema:**

```json
{
  "id": "unique-event-id",
  "eventType": "Microsoft.Storage.BlobCreated",
  "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
  "eventTime": "2025-11-13T10:30:00Z",
  "data": {
    "api": "PutBlob",
    "contentType": "image/jpeg",
    "contentLength": 524288,
    "url": "https://storage.blob.core.windows.net/images/photo.jpg"
  },
  "dataVersion": "1.0",
  "metadataVersion": "1",
  "topic": "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/storage"
}
```

**Filtering Capabilities:**

- **Subject Filtering**: Route based on event subject
- **Event Type Filtering**: Filter by eventType field
- **Advanced Filtering**: Filter on any field in data section

**Delivery Guarantees:**

- At-least-once delivery
- Automatic retry with exponential backoff
- Dead-letter queue for failed deliveries
- Maximum event size: 1 MB

#### 2. **Azure Event Hubs**

- **Purpose**: Big data streaming platform and event ingestion service
- **Connection**:
  - Receives millions of events per second
  - Buffers events in partitions for parallel processing
  - Captures events to Blob Storage or Data Lake
  - Integrates with Apache Kafka protocol

**Event Hubs Concepts:**

- **Namespace**: Container for multiple event hubs
- **Event Hub**: Stream of events (like a Kafka topic)
- **Partition**: Ordered sequence of events (enables parallelism)
- **Consumer Group**: View of entire event hub (multiple consumers)
- **Throughput Units (TUs)**: Capacity units (1 TU = 1 MB/s ingress)

**Key Features:**

- **Capture**: Automatically save events to Blob Storage/Data Lake
- **Auto-Inflate**: Automatically scale throughput units
- **Kafka Support**: Use Kafka client libraries
- **Geo-Disaster Recovery**: Paired namespaces across regions
- **Event Retention**: Store events for 1-7 days (up to 90 days premium)

**Use Cases:**

- Telemetry and log ingestion
- IoT device streaming
- Click-stream analysis
- Real-time analytics pipeline
- Event sourcing pattern

#### 3. **Azure Service Bus**

- **Purpose**: Enterprise messaging service with queues and topics
- **Connection**:
  - Provides reliable message delivery between services
  - Supports FIFO ordering, transactions, duplicate detection
  - Decouples sender from receiver
  - Message can be up to 256 KB (1 MB in premium)

**Service Bus vs Event Hubs vs Event Grid:**

- **Service Bus**: Enterprise messaging, FIFO, transactions, sessions
- **Event Hubs**: High-throughput streaming, big data ingestion
- **Event Grid**: Event routing, reactive programming, Azure-native

**Service Bus Features:**

- **Dead-Letter Queue**: Store undeliverable messages
- **Message Deferral**: Postpone message processing
- **Scheduled Messages**: Deliver at specific time
- **Duplicate Detection**: Prevent processing same message twice
- **Sessions**: FIFO guarantee across related messages
- **Auto-Forward**: Chain queues/topics together

**Messaging Patterns:**

- **Point-to-Point (Queues)**: One sender, one receiver
- **Publish-Subscribe (Topics)**: One sender, multiple filtered subscribers

#### 4. **Azure IoT Hub**

- **Purpose**: Managed service for bi-directional IoT communication
- **Connection**:
  - Receives telemetry from millions of IoT devices
  - Routes messages to Event Hubs, Service Bus, Event Grid, or Storage
  - Supports device management and command/control
  - Built-in security with per-device identity

**IoT Hub Routing:**

- Device-to-cloud messages routed based on message properties
- Can route to Event Hubs, Service Bus, Blob Storage, Event Grid
- Multiple endpoints with custom routing queries

### Event Processing Services

#### 5. **Azure Functions**

- **Purpose**: Serverless event processing
- **Connection**:
  - Triggered by Event Grid, Event Hubs, Service Bus, or Storage events
  - Processes events with custom code
  - Automatically scales based on event volume
  - Multiple language support (C#, Python, JavaScript, Java)

**Event Triggers:**

- **Event Grid Trigger**: Process Event Grid events
- **Event Hub Trigger**: Process streaming events with checkpointing
- **Service Bus Trigger**: Process queue/topic messages with auto-complete
- **Blob Trigger**: React to blob creation/modification
- **Timer Trigger**: Scheduled event processing

**Example Function (Event Grid Trigger):**

```csharp
[FunctionName("ProcessBlobEvent")]
public static void Run(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    [Blob("{data.url}", FileAccess.Read)] Stream blobStream,
    ILogger log)
{
    log.LogInformation($"Processing blob: {eventGridEvent.Subject}");
    // Process the blob
}
```

#### 6. **Azure Logic Apps**

- **Purpose**: Visual workflow orchestration for event-driven processes
- **Connection**:
  - Triggered by events from Event Grid, Service Bus, HTTP
  - Orchestrates complex business processes
  - 400+ connectors to SaaS and enterprise systems
  - Low-code/no-code approach

**Common Event-Driven Workflows:**

- When file uploaded → scan for viruses → notify team
- When order placed → validate inventory → process payment → ship
- When threshold exceeded → create incident → notify on-call
- When form submitted → validate → save to database → send confirmation

**Logic App Triggers:**

- Event Grid event received
- Service Bus message received
- HTTP request received
- Recurrence (schedule)
- When file created (OneDrive, SharePoint, FTP)

#### 7. **Azure Stream Analytics**

- **Purpose**: Real-time stream processing and analytics
- **Connection**:
  - Reads from Event Hubs, IoT Hub, Blob Storage
  - Performs real-time aggregations, filtering, windowing
  - Writes results to output sinks (Cosmos DB, Power BI, Storage)
  - SQL-like query language

**Stream Analytics Features:**

- **Windowing**: Tumbling, hopping, sliding, session windows
- **Joins**: Join multiple streams or reference data
- **Built-in ML**: Anomaly detection functions
- **Geo-spatial**: Geographic queries and functions

**Example Query:**

```sql
SELECT
    DeviceId,
    AVG(Temperature) AS AvgTemp,
    COUNT(*) AS EventCount
INTO
    OutputBlob
FROM
    IoTHubInput
GROUP BY
    DeviceId,
    TumblingWindow(minute, 5)
HAVING
    AVG(Temperature) > 75
```

#### 8. **Azure Databricks**

- **Purpose**: Big data analytics and complex event processing
- **Connection**:
  - Reads from Event Hubs or Kafka
  - Processes large-scale streaming data
  - Machine learning on event streams
  - Structured Streaming (Spark)

### Event Storage and State Management

#### 9. **Azure Cosmos DB**

- **Purpose**: Globally distributed, multi-model database for event data
- **Connection**:
  - Stores processed events and application state
  - Change Feed triggers downstream processing
  - Low-latency reads/writes (<10ms)
  - Global distribution for multi-region events

**Cosmos DB Change Feed:**

- Ordered list of all changes to container
- Can trigger Azure Functions for further processing
- Enables event sourcing pattern
- Guaranteed ordering per partition key

**Event Sourcing with Cosmos DB:**

```
Event → Store in Cosmos → Change Feed → Materialize Views
```

#### 10. **Azure Storage (Blob, Queue, Table)**

- **Purpose**: Durable storage for events and messages
- **Connection**:
  - **Blob Storage**: Store large events, Event Hubs capture
  - **Queue Storage**: Simple message queuing
  - **Table Storage**: Key-value store for event metadata

**Storage Event Grid Integration:**

- Blob created/deleted/renamed events
- Subscribe to specific containers or blob name patterns
- Near real-time notifications

#### 11. **Azure Data Lake Storage Gen2**

- **Purpose**: Big data storage for event archives
- **Connection**:
  - Hierarchical namespace for organized event storage
  - Event Hubs capture destination
  - Integration with analytics tools (Databricks, Synapse)
  - Lifecycle management policies

### Supporting Services

#### 12. **Azure API Management**

- **Purpose**: API gateway for event-driven APIs
- **Connection**:
  - Exposes event-driven APIs to external consumers
  - Publishes events to Event Grid on API calls
  - Rate limiting and throttling
  - Webhook validation for event subscriptions

#### 13. **Azure Monitor & Application Insights**

- **Purpose**: End-to-end event tracing and monitoring
- **Connection**:
  - Tracks events across all services
  - Distributed tracing with correlation IDs
  - Custom metrics on event processing
  - Alerts on event processing failures

**Key Metrics to Monitor:**

- Event ingestion rate
- Processing latency (end-to-end)
- Failed deliveries and dead-letters
- Consumer lag (for Event Hubs)
- Function execution counts and duration

#### 14. **Azure Key Vault**

- **Purpose**: Secure storage for event system credentials
- **Connection**:
  - Stores connection strings for Event Hubs, Service Bus
  - API keys for external systems
  - Accessed via managed identities
  - Event Grid can notify on secret changes

#### 15. **Virtual Network Integration**

- **Purpose**: Secure event processing in isolated networks
- **Connection**:
  - Event Hubs/Service Bus with private endpoints
  - Functions with VNet integration
  - Network Security Groups for event processor subnets

## Event-Driven Architecture Patterns

### Pattern 1: Event Notification

**Purpose:** Notify other systems that something happened

```
Producer → Event Grid → Multiple Subscribers
```

**Example:**

1. User uploads document to Blob Storage
1. Event Grid publishes BlobCreated event
1. Subscribers notified:
- Function 1: Generate thumbnail
- Function 2: Extract metadata
- Logic App: Send notification email
- External webhook: Update CRM

### Pattern 2: Event-Carried State Transfer

**Purpose:** Include all necessary data in the event

```
Order Service → Event (with full order data) → Shipping Service
```

**Example:**

```json
{
  "eventType": "OrderPlaced",
  "data": {
    "orderId": "12345",
    "customerId": "C789",
    "items": [...],
    "totalAmount": 199.99,
    "shippingAddress": {...}
  }
}
```

**Benefits:**

- Consumers don’t need to query original service
- Reduced coupling
- Better performance

### Pattern 3: Event Sourcing

**Purpose:** Store all changes as sequence of events

```
Command → Events → Event Store (Cosmos DB) → Projections/Views
```

**Example: Bank Account:**

```
Events:
1. AccountOpened (balance: 0)
2. MoneyDeposited (amount: 1000)
3. MoneyWithdrawn (amount: 200)
4. MoneyDeposited (amount: 500)

Current State = Replay all events = Balance: 1300
```

**Implementation with Cosmos DB:**

- Store events in Cosmos DB with partition key = aggregate ID
- Use Change Feed to materialize read models
- Event store is source of truth

### Pattern 4: CQRS (Command Query Responsibility Segregation)

**Purpose:** Separate read and write models

```
Commands → Write Model → Events → Read Model (optimized for queries)
```

**Example:**

```
Write Side:
  HTTP POST /orders → Command Handler → Store Event → Event Grid

Read Side:
  Event Grid → Update Read Model (Cosmos DB) → HTTP GET /orders
```

**Benefits:**

- Optimized read and write operations independently
- Scale read and write sides separately
- Multiple read models for different use cases

### Pattern 5: Complex Event Processing (CEP)

**Purpose:** Detect patterns across multiple events

```
Event Stream → Stream Analytics → Pattern Detection → Action
```

**Example: Fraud Detection:**

```
Events: Transaction, Login, Location Change
Pattern: 
  IF (Transaction.amount > 10000 
      AND Location.distance > 500 miles 
      WITHIN 1 hour)
  THEN alert("Possible fraud")
```

### Pattern 6: Choreography

**Purpose:** Services react to events without central orchestrator

```
Service A → Event → Service B → Event → Service C
```

**Example: Order Processing:**

1. Order Service: Order placed → Publishes “OrderPlaced” event
1. Inventory Service: Subscribes to “OrderPlaced” → Reserves inventory → Publishes “InventoryReserved”
1. Payment Service: Subscribes to “InventoryReserved” → Processes payment → Publishes “PaymentCompleted”
1. Shipping Service: Subscribes to “PaymentCompleted” → Ships order

**Characteristics:**

- No central coordinator
- Services independently react to events
- Loose coupling

### Pattern 7: Orchestration with Saga

**Purpose:** Coordinate distributed transactions with compensation

```
Orchestrator → Command to Service A → Event from A → 
              Command to Service B → Event from B → ...
              (If failure → Compensating transactions)
```

**Example with Durable Functions:**

```csharp
[FunctionName("OrderSaga")]
public static async Task<string> Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    
    try
    {
        await context.CallActivityAsync("ReserveInventory", order);
        await context.CallActivityAsync("ProcessPayment", order);
        await context.CallActivityAsync("ShipOrder", order);
        return "Success";
    }
    catch
    {
        await context.CallActivityAsync("ReleaseInventory", order);
        await context.CallActivityAsync("RefundPayment", order);
        return "Failed - Compensated";
    }
}
```

## Traffic Flow Examples

### Example 1: Real-Time Analytics Pipeline

1. **IoT devices** send temperature telemetry to IoT Hub (1000 events/sec)
1. **IoT Hub** routes to Event Hubs
1. **Event Hubs** buffers events across 4 partitions
1. **Stream Analytics** reads from Event Hubs:
- Calculates average temperature per device per 5-minute window
- Filters for temperatures > 80°F
1. **Stream Analytics** outputs to:
- Cosmos DB for real-time dashboard
- Power BI for live visualization
- Event Grid if threshold exceeded
1. **Event Grid** triggers Function for alert notification
1. **Function** sends alert via SendGrid email and Twilio SMS

**End-to-End Latency:** <5 seconds from sensor to alert

### Example 2: E-Commerce Order Processing

1. **Customer** places order via web app
1. **Web app** publishes “OrderCreated” event to Event Grid
1. **Event Grid** fans out to multiple subscribers:
   
   **Subscriber 1: Inventory Function**
- Checks inventory availability
- Reserves items
- Publishes “InventoryReserved” or “InventoryUnavailable” to Event Grid
   
   **Subscriber 2: Email Function**
- Sends order confirmation email immediately
1. **Payment Function** subscribed to “InventoryReserved”:
- Charges credit card
- Publishes “PaymentSucceeded” or “PaymentFailed”
1. **Shipping Logic App** subscribed to “PaymentSucceeded”:
- Creates shipping label
- Calls shipping API
- Publishes “OrderShipped”
- Sends tracking email
1. **Analytics Function** subscribed to all order events:
- Stores events in Cosmos DB for reporting
- Updates real-time dashboard

### Example 3: File Processing Pipeline

1. **User** uploads CSV file to Blob Storage
1. **Blob Storage** triggers Event Grid “BlobCreated” event
1. **Event Grid** routes to Function 1
1. **Function 1** downloads file and validates format:
- If valid: Publishes “FileValidated” to Service Bus
- If invalid: Publishes “FileRejected” to Service Bus
1. **Function 2** subscribed to “FileValidated”:
- Processes CSV line by line
- Publishes “RecordProcessed” events to Event Grid
1. **Stream Analytics** aggregates “RecordProcessed” events:
- Counts processed records per minute
- Outputs to Power BI dashboard
1. **Function 3** subscribed to “FileValidated”:
- Archives original file to Azure Storage (Cool tier)
- Updates processing status in Cosmos DB

### Example 4: Multi-Region Event Processing

1. **Events** ingested in East US region to Event Hubs (Geo-DR enabled)
1. **Event Hubs** replicates to West US region (disaster recovery)
1. **Functions** in both regions process events (active-active):
- Use consumer groups to prevent duplicate processing
- Checkpoint progress in Storage Account
1. **Cosmos DB** (multi-region) stores results:
- Write to nearest region
- Global consistency ensures all regions eventually see data
1. **Event Grid** publishes completion events:
- Subscribers in multiple regions receive notifications

## Security Best Practices

### 1. **Authentication & Authorization**

- **Managed Identities**: Event processors authenticate to Event Hubs/Service Bus
- **Shared Access Signatures (SAS)**: Granular permissions (Send, Listen)
- **Azure AD**: Role-based access control
- **Event Grid**: Webhook validation with validation codes

### 2. **Network Security**

- **Private Endpoints**: Event Hubs, Service Bus not exposed to internet
- **VNet Integration**: Functions process events in isolated VNet
- **IP Filtering**: Restrict Event Hubs/Service Bus access by IP
- **Service Endpoints**: Secure traffic between services

### 3. **Data Protection**

- **Encryption in Transit**: TLS 1.2+ for all connections
- **Encryption at Rest**: Events encrypted in Event Hubs/Service Bus
- **Event Payload Encryption**: Encrypt sensitive data in event payload
- **Key Vault**: Store connection strings and encryption keys

### 4. **Event Validation**

- **Schema Validation**: Validate event schema at ingestion
- **Input Sanitization**: Prevent injection attacks
- **Idempotency**: Handle duplicate events safely
- **Rate Limiting**: Prevent event flooding (API Management)

## Monitoring & Observability

### Key Metrics to Track:

1. **Ingestion Metrics:**
- Events received per second
- Ingestion errors
- Throttling occurrences
1. **Processing Metrics:**
- Processing latency (P50, P95, P99)
- Failed processing attempts
- Dead-letter queue size
- Consumer lag (Event Hubs)
1. **Delivery Metrics:**
- Successful deliveries
- Failed deliveries
- Retry attempts
- Dead-letter events

### Distributed Tracing:

```
Correlation ID: 12345-67890-abcdef

Event Producer → [12345] → Event Grid → [12345] → Function A → [12345] → 
Service Bus → [12345] → Function B → [12345] → Cosmos DB

All logs tagged with same correlation ID for end-to-end tracing
```

### Alerting Strategy:

- **Critical**: Dead-letter queue growing, all processing failed
- **Warning**: Processing latency > 30 seconds, retry rate > 10%
- **Info**: Throughput approaching limits, new event types detected

## Best Practices

1. **Design for Idempotency**: Events may be delivered more than once
1. **Use Correlation IDs**: Track events across entire system
1. **Schema Versioning**: Support multiple event schema versions
1. **Small Event Payloads**: Keep events lightweight, use references for large data
1. **Event Retention**: Configure appropriate retention based on use case
1. **Error Handling**: Implement dead-letter queues and retry logic
1. **Partitioning Strategy**: Choose partition keys carefully (Event Hubs)
1. **Testing**: Test failure scenarios, simulate event storms
1. **Documentation**: Document all event types and their schemas
1. **Monitoring**: Comprehensive observability from ingestion to processing

## Scaling Considerations

### Event Hubs Scaling:

- **Throughput Units**: Scale from 1 TU (1 MB/s) to 40 TUs
- **Partitions**: More partitions = more parallel processing (max 32)
- **Auto-Inflate**: Automatically increase TUs based on load
- **Premium Tier**: Up to 100 TUs, dedicated capacity

### Service Bus Scaling:

- **Premium Tier**: Dedicated resources, predictable performance
- **Messaging Units**: Scale capacity independently
- **Partitioned Entities**: Distribute across multiple message brokers

### Functions Scaling:

- **Consumption Plan**: Automatic scaling (0 to 200 instances)
- **Premium Plan**: Pre-warmed instances, faster scaling
- **Scaling Rules**: Based on event count in Event Hubs/Service Bus

## Common Challenges

1. **Event Ordering**: Difficult to maintain order across distributed system
- **Mitigation**: Use sessions (Service Bus), single partition (Event Hubs), or accept eventual consistency
1. **Duplicate Events**: At-least-once delivery guarantees
- **Mitigation**: Implement idempotent processing, use deduplication IDs
1. **Event Schema Evolution**: Changing event structure breaks consumers
- **Mitigation**: Schema versioning, backward compatibility, schema registry
1. **Debugging**: Harder to trace issues in event-driven systems
- **Mitigation**: Correlation IDs, comprehensive logging, distributed tracing
1. **Event Storm**: Sudden spike in events overwhelms system
- **Mitigation**: Rate limiting, backpressure, auto-scaling, dead-lettering
1. **Data Consistency**: Eventual consistency challenges
- **Mitigation**: Design for eventual consistency, saga pattern for transactions

## When to Use Event-Driven Architecture

✅ **Good for:**

- Microservices communication
- Real-time data processing
- IoT and telemetry systems
- Loosely coupled systems
- Scalable, distributed applications
- Integration with multiple systems
- Asynchronous processing needs

❌ **Not ideal for:**

- Simple CRUD applications
- Strong consistency requirements
- Synchronous request-response patterns
- Small applications with few components
- Teams without distributed systems experience

## Cost Optimization

1. **Right-Size Event Hubs**: Start with fewer TUs, enable auto-inflate
1. **Service Bus Tier**: Standard for most cases, Premium only if needed
1. **Event Grid**: Pay-per-operation (very cost-effective)
1. **Functions Consumption**: Only pay for executions
1. **Event Retention**: Reduce retention period to lower storage costs
1. **Batch Processing**: Process events in batches to reduce function executions

## Learning Resources

- Microsoft Learn: [Event-driven architecture style](https://learn.microsoft.com/azure/architecture/guide/architecture-styles/event-driven)
- Azure Architecture Center: [Event-driven scenarios](https://learn.microsoft.com/azure/architecture/serverless/event-hubs-functions/resilient-design)
- [Event Hubs documentation](https://learn.microsoft.com/azure/event-hubs/)
- [Event Grid documentation](https://learn.microsoft.com/azure/event-grid/)
- [Service Bus documentation](https://learn.microsoft.com/azure/service-bus-messaging/)
- [Stream Analytics documentation](https://learn.microsoft.com/azure/stream-analytics/)
