# 800 - Azure Big Data & Analytics Architecture

## Overview

Big Data & Analytics architecture handles massive volumes of structured and unstructured data, enabling organizations to derive insights, build predictive models, and make data-driven decisions. Azure provides a comprehensive suite of services for data ingestion, storage, processing, analysis, and visualization at petabyte scale.

## Architecture Diagram Concept

```
[Data Sources]
    ├─→ IoT Devices
    ├─→ Applications
    ├─→ Databases
    └─→ Files/Logs
         ↓
[Data Ingestion]
    ├─→ Azure Event Hubs
    ├─→ Azure Data Factory
    └─→ Azure IoT Hub
         ↓
[Data Storage]
    ├─→ Azure Data Lake Storage Gen2
    ├─→ Azure Synapse Analytics
    └─→ Azure Cosmos DB
         ↓
[Data Processing]
    ├─→ Azure Databricks
    ├─→ Azure Synapse Analytics
    ├─→ Azure HDInsight
    └─→ Azure Stream Analytics
         ↓
[Data Serving & Visualization]
    ├─→ Azure Synapse Analytics
    ├─→ Power BI
    ├─→ Azure Machine Learning
    └─→ APIs (API Management)
```

## Key Components and Their Connections

### Data Ingestion Layer

#### 1. **Azure Data Factory (ADF)**

- **Purpose**: Cloud-based ETL (Extract, Transform, Load) and data integration service
- **Connection**:
  - Connects to 90+ data sources (on-premises, cloud, SaaS)
  - Orchestrates data movement and transformation
  - Schedules and monitors data pipelines
  - Loads data into Data Lake, Synapse, or other destinations

**Key Features:**

- **Copy Activity**: Move data between sources and sinks
- **Data Flows**: Visual data transformation (mapping data flows)
- **Pipeline Orchestration**: Control flow with loops, conditions, branches
- **Triggers**: Schedule, tumbling window, event-based triggers
- **Integration Runtime**: Self-hosted for on-premises, Azure for cloud

**Common Patterns:**

- Daily batch ETL from SQL Server to Data Lake
- Real-time CDC (Change Data Capture) from databases
- File ingestion from FTP/SFTP to Azure Storage
- SAP data extraction to Azure

**Example Pipeline:**

```
Source: On-Premises SQL Server
  ↓ (Self-Hosted Integration Runtime)
Copy Activity → Transform in Data Flow
  ↓
Sink: Azure Data Lake Storage Gen2
  ↓
Trigger: Azure Synapse Pipeline for processing
```

#### 2. **Azure Event Hubs**

- **Purpose**: Real-time data streaming ingestion (millions of events/second)
- **Connection**:
  - IoT devices, applications send telemetry
  - Buffers events for downstream processing
  - Integrates with Stream Analytics, Databricks, Functions
  - Capture to Data Lake for cold path analytics

**Event Hubs in Analytics:**

- **Hot Path**: Stream Analytics → Real-time dashboard
- **Cold Path**: Capture to Data Lake → Batch processing
- **Lambda Architecture**: Combine hot and cold paths

#### 3. **Azure IoT Hub**

- **Purpose**: Managed service for IoT device connectivity
- **Connection**:
  - Receives telemetry from IoT devices
  - Routes messages to Event Hubs, Storage, Event Grid
  - Device management and security
  - Bidirectional communication

#### 4. **Azure Logic Apps**

- **Purpose**: Workflow automation for data ingestion
- **Connection**:
  - Schedule-based data pulls (e.g., daily API calls)
  - Event-driven ingestion (e.g., file uploaded to SharePoint)
  - Integrates with 400+ connectors
  - Low-code alternative to Data Factory for simple scenarios

### Data Storage Layer

#### 5. **Azure Data Lake Storage Gen2**

- **Purpose**: Massively scalable, cost-effective data lake for big data analytics
- **Connection**:
  - Built on Azure Blob Storage with hierarchical namespace
  - Stores raw, structured, and unstructured data
  - Integration with all Azure analytics services
  - POSIX permissions for fine-grained access control

**Data Lake Organization:**

```
/raw          - Raw ingested data (immutable)
  /2025
    /11
      /13
        /sensor-data.parquet
/bronze       - Validated and deduplicated
/silver       - Cleansed and conformed
/gold         - Business-level aggregates
```

**Key Features:**

- **Hierarchical Namespace**: Folders, not just flat blobs
- **Lifecycle Management**: Auto-tier to Cool/Archive based on age
- **RBAC**: Azure AD integration for access control
- **ACLs**: POSIX-style permissions on files/folders
- **Zone-Redundant Storage**: High availability

**File Formats:**

- **Parquet**: Columnar format, excellent compression
- **Delta Lake**: ACID transactions, time travel (with Databricks)
- **Avro**: Row-based, schema evolution support
- **CSV/JSON**: Human-readable, less efficient

#### 6. **Azure Synapse Analytics**

- **Purpose**: Unified analytics platform (data warehouse + big data)
- **Connection**:
  - Queries data in Data Lake without moving it (serverless SQL pool)
  - Dedicated SQL pool for data warehouse workloads
  - Apache Spark pools for big data processing
  - Pipelines for ETL orchestration (similar to Data Factory)

**Synapse Components:**

- **Serverless SQL Pool**: Query data in Data Lake with T-SQL (pay per query)
- **Dedicated SQL Pool**: Provisioned data warehouse (MPP architecture)
- **Apache Spark Pools**: Managed Spark for big data processing
- **Synapse Pipelines**: Data integration and orchestration
- **Synapse Studio**: Unified development experience

**Architecture Patterns:**

- **Data Warehouse**: Dimensional modeling (star schema) in dedicated SQL pool
- **Data Lakehouse**: Query data in place with serverless SQL pool
- **Hybrid**: Curated data in dedicated pool, raw data in lake

#### 7. **Azure Cosmos DB**

- **Purpose**: Globally distributed NoSQL database for operational analytics
- **Connection**:
  - Low-latency reads/writes for real-time analytics
  - Change feed for stream processing
  - Analytical storage for HTAP (Hybrid Transactional/Analytical Processing)
  - Multiple APIs (SQL, MongoDB, Cassandra)

**Cosmos DB for Analytics:**

- **Analytical Store**: Columnar storage auto-synced from transactional
- **Azure Synapse Link**: Query analytical store from Synapse (no ETL)
- **Change Feed**: Trigger Azure Functions or Stream Analytics on data changes

### Data Processing Layer

#### 8. **Azure Databricks**

- **Purpose**: Apache Spark-based analytics platform
- **Connection**:
  - Reads from Data Lake, Event Hubs, Cosmos DB
  - Distributed processing of big data
  - Machine learning and AI workloads
  - Writes results back to Data Lake or Synapse

**Databricks Features:**

- **Notebooks**: Interactive development (Python, Scala, SQL, R)
- **Jobs**: Scheduled batch processing
- **Delta Lake**: ACID transactions, time travel, schema enforcement
- **MLflow**: Machine learning lifecycle management
- **Unity Catalog**: Unified governance across workspaces

**Common Workloads:**

- ETL/ELT at scale (terabytes to petabytes)
- Data science and machine learning
- Real-time streaming with Structured Streaming
- Graph analytics with GraphX

**Cluster Types:**

- **All-Purpose**: Interactive development
- **Job Clusters**: Automated workloads (cheaper)
- **Serverless**: Managed by Databricks (fastest startup)

#### 9. **Azure HDInsight**

- **Purpose**: Managed Hadoop, Spark, Hive, Kafka, HBase clusters
- **Connection**:
  - Processes data in Data Lake
  - Supports open-source frameworks
  - Integration with on-premises Hadoop
  - Lower-level control than Databricks (more configuration)

**HDInsight Cluster Types:**

- **Spark**: General-purpose big data processing
- **Hadoop**: MapReduce batch processing
- **Hive**: SQL-like queries on large datasets (LLAP for interactive)
- **Kafka**: Distributed streaming platform
- **HBase**: NoSQL database on Hadoop
- **Storm**: Real-time stream processing

**When to Use:**

- Existing Hadoop expertise in organization
- Need specific Hadoop ecosystem tools
- Lift-and-shift from on-premises Hadoop
- Cost-sensitive (lower cost than Databricks for some workloads)

#### 10. **Azure Stream Analytics**

- **Purpose**: Real-time stream processing with SQL-like queries
- **Connection**:
  - Inputs: Event Hubs, IoT Hub, Blob Storage
  - Outputs: Data Lake, Cosmos DB, Power BI, Synapse, Event Hubs
  - Real-time dashboards and alerts
  - Windowing and aggregation

**Stream Analytics Features:**

- **Windowing**: Tumbling, hopping, sliding, session windows
- **Reference Data**: Join streaming data with static data
- **User-Defined Functions (UDFs)**: JavaScript or C#
- **Anomaly Detection**: Built-in ML models
- **Geo-Spatial**: Geographic queries

**Example Query:**

```sql
-- Real-time temperature monitoring
SELECT
    DeviceId,
    AVG(Temperature) AS AvgTemp,
    MAX(Temperature) AS MaxTemp,
    System.Timestamp AS WindowEnd
INTO
    PowerBIDashboard
FROM
    IoTHubInput TIMESTAMP BY EventTime
GROUP BY
    DeviceId,
    TumblingWindow(minute, 5)
HAVING
    AVG(Temperature) > 75
```

#### 11. **Azure Synapse Spark Pools**

- **Purpose**: Apache Spark within Synapse workspace
- **Connection**:
  - Native integration with Data Lake and dedicated SQL pools
  - Shared metadata with Synapse serverless SQL
  - Unified security and governance
  - Notebooks with multiple languages

**Synapse Spark vs Databricks:**

|Feature    |Synapse Spark           |Databricks                       |
|-----------|------------------------|---------------------------------|
|Integration|Deep Synapse integration|Separate service                 |
|Cost       |Pay per node-hour       |Pay per DBU                      |
|Ease of Use|Easier for SQL users    |More features for data scientists|
|Advanced ML|Basic                   |Advanced (MLflow, AutoML)        |

### Machine Learning Layer

#### 12. **Azure Machine Learning**

- **Purpose**: End-to-end machine learning platform
- **Connection**:
  - Reads data from Data Lake, Synapse, Cosmos DB
  - Train models at scale
  - Deploy models as web services or to IoT Edge
  - MLOps for model lifecycle management

**ML Workflow:**

1. **Data Preparation**: Datastore connects to Data Lake
1. **Feature Engineering**: Transform data in notebooks or pipelines
1. **Training**: Compute targets (CPU, GPU clusters)
1. **Hyperparameter Tuning**: Automated with HyperDrive
1. **Model Registration**: Version control in ML model registry
1. **Deployment**: Real-time (AKS, ACI) or batch (Synapse, Databricks)
1. **Monitoring**: Model drift detection and retraining

#### 13. **Azure Cognitive Services**

- **Purpose**: Pre-built AI services for common scenarios
- **Connection**:
  - Called from Data Factory, Databricks, Functions
  - Vision: Image classification, object detection, OCR
  - Language: Sentiment analysis, key phrase extraction, translation
  - Speech: Speech-to-text, text-to-speech

**Example: Document Processing Pipeline:**

```
Documents in Blob → Function triggered → Cognitive Services (OCR) →
Extract text → Text Analytics (sentiment) → Store in Cosmos DB
```

### Data Serving Layer

#### 14. **Power BI**

- **Purpose**: Business intelligence and data visualization
- **Connection**:
  - Direct Query or Import from Synapse, Data Lake, Cosmos DB
  - Real-time streaming from Event Hubs, Stream Analytics
  - Embedded in applications via Power BI Embedded
  - Paginated reports for operational reporting

**Power BI Integration:**

- **Synapse**: Native connector, optimized queries
- **Data Lake**: Via Dataflows or direct in Power BI Desktop
- **Databricks**: JDBC/ODBC connector
- **Streaming Datasets**: Real-time dashboards

**Deployment:**

- **Power BI Service**: Cloud-based sharing and collaboration
- **Power BI Report Server**: On-premises deployment
- **Power BI Embedded**: Embed in custom applications

#### 15. **Azure Analysis Services**

- **Purpose**: Enterprise-grade semantic model for BI
- **Connection**:
  - Tabular models on top of data warehouse
  - High-performance queries for Power BI
  - Row-level security and roles
  - Scale up/down based on query load

#### 16. **Azure API Management**

- **Purpose**: Expose analytics results as APIs
- **Connection**:
  - Front-end for Synapse dedicated SQL pool
  - Rate limiting and authentication
  - Transform JSON results
  - Developer portal for API consumers

### Governance and Security

#### 17. **Microsoft Purview**

- **Purpose**: Unified data governance and catalog
- **Connection**:
  - Scans Data Lake, Synapse, databases
  - Automatic data classification (PII, PHI)
  - Data lineage tracking
  - Business glossary

**Purview Features:**

- **Data Discovery**: Search across all data assets
- **Classification**: Automatic and manual tagging
- **Lineage**: Trace data from source to report
- **Sensitivity Labels**: Integrate with Azure Information Protection

#### 18. **Azure Key Vault**

- **Purpose**: Secure storage for connection strings and keys
- **Connection**:
  - Data Factory pipelines reference Key Vault
  - Databricks uses Key Vault-backed scopes
  - Synapse linked services use Key Vault
  - No secrets in code or configuration

#### 19. **Azure Active Directory**

- **Purpose**: Identity and access management
- **Connection**:
  - Authentication for all analytics services
  - RBAC for resource access (Data Lake, Synapse)
  - Conditional Access for data access
  - Service principals for automation

#### 20. **Azure Monitor**

- **Purpose**: Monitoring and diagnostics
- **Connection**:
  - Metrics from all analytics services
  - Log Analytics for centralized logging
  - Alerts on pipeline failures, query performance
  - Dashboards for operational monitoring

### Networking

#### 21. **Virtual Network Integration**

- **Purpose**: Secure analytics in private network
- **Connection**:
  - Synapse workspace in managed VNet
  - Databricks in VNet injection
  - Private endpoints for Data Lake, Synapse
  - No public internet exposure

#### 22. **Private Link**

- **Purpose**: Private connectivity to analytics services
- **Connection**:
  - Private endpoint for Data Lake in VNet
  - Private endpoint for Synapse workspace
  - Access from on-premises via ExpressRoute
  - No data traverses public internet

## Big Data Architecture Patterns

### Pattern 1: Lambda Architecture

```
Data Sources → Batch Layer (Data Lake + Synapse)
            ↘ 
             → Speed Layer (Event Hubs + Stream Analytics)
            ↗
            → Serving Layer (Synapse + Power BI)
```

**Characteristics:**

- Batch layer: Complete, accurate historical data
- Speed layer: Real-time, approximate results
- Serving layer: Merge batch and speed views

**Implementation:**

- Batch: Data Lake → Databricks (nightly) → Synapse
- Speed: Event Hubs → Stream Analytics → Cosmos DB
- Query: Synapse (historical) UNION Cosmos DB (recent)

### Pattern 2: Kappa Architecture

```
Data Sources → Event Hubs → Stream Processing → Serving Layer
```

**Characteristics:**

- Single stream processing path
- Reprocess historical data by replaying events
- Simpler than Lambda (no separate batch layer)

**Implementation:**

- Event Hubs with 7+ days retention
- Databricks Structured Streaming
- Delta Lake for results (serving layer)
- Reprocess by resetting checkpoints

### Pattern 3: Modern Data Warehouse

```
Sources → Data Factory → Data Lake (staging) → Databricks (transform) →
Synapse Dedicated SQL Pool (dimensional model) → Power BI
```

**Characteristics:**

- Traditional dimensional modeling (star schema)
- Batch processing (nightly or hourly)
- Optimized for BI and reporting

**Implementation:**

- Fact and dimension tables in Synapse
- Slowly Changing Dimensions (SCD) handling
- Incremental loads with watermarks
- Pre-aggregated tables for performance

### Pattern 4: Data Lakehouse

```
Sources → Data Lake (all data) → Databricks Delta Lake →
Synapse Serverless SQL (query in place) → Power BI
```

**Characteristics:**

- Store data once in Data Lake
- ACID transactions with Delta Lake
- Query with SQL (serverless) or Spark
- No data duplication

**Implementation:**

- Bronze: Raw data (append-only)
- Silver: Validated and deduplicated
- Gold: Business-level aggregates
- Query any layer with Synapse or Databricks

### Pattern 5: Real-Time Analytics

```
IoT Devices → Event Hubs → Stream Analytics → 
Cosmos DB (hot) + Data Lake (cold) → Power BI (real-time)
```

**Characteristics:**

- Sub-second latency for insights
- Scales to millions of events/second
- Continuous processing

**Implementation:**

- Hot path: Stream Analytics → Cosmos DB → Power BI streaming
- Cold path: Event Hubs Capture → Data Lake → batch analysis

### Pattern 6: Machine Learning Pipeline

```
Data Lake → Databricks (feature engineering) → 
Azure ML (training) → Model Registry → 
AKS (serving) / Synapse (batch scoring)
```

**Characteristics:**

- Automated, repeatable ML workflows
- Version control for models
- CI/CD for model deployment

**Implementation:**

- Azure ML Pipelines for orchestration
- Experiment tracking with MLflow
- Model monitoring and retraining
- A/B testing in production

## Traffic Flow Examples

### Example 1: IoT Telemetry Processing

1. **10,000 IoT sensors** send temperature data every 10 seconds
1. **IoT Hub** receives 1 million events/hour
1. **IoT Hub** routes to Event Hubs
1. **Event Hubs Capture** writes raw data to Data Lake every 5 minutes
1. **Stream Analytics** reads from Event Hubs:
- Calculates moving average per sensor
- Detects anomalies (temperature spikes)
- Aggregates by region every minute
1. **Stream Analytics** outputs to:
- Power BI for real-time dashboard (aggregates)
- Cosmos DB for quick queries (individual readings)
- Event Grid for alerts (anomalies)
1. **Databricks** nightly job processes Data Lake files:
- Joins with sensor metadata
- Calculates daily statistics
- Writes to Synapse dedicated SQL pool
1. **Power BI** reports combine:
- Real-time (last 24 hours) from Cosmos DB
- Historical (months/years) from Synapse

### Example 2: E-Commerce Clickstream Analysis

1. **Web application** logs user clicks to Event Hubs (100,000 events/sec)
1. **Event Hubs** buffers events across 16 partitions
1. **Databricks Structured Streaming** reads events:
- Sessionizes clicks (group by user, 30-min timeout)
- Enriches with product catalog
- Detects cart abandonment
1. **Databricks** writes to Delta Lake in Data Lake
1. **Synapse Serverless SQL** queries Delta Lake for ad-hoc analysis
1. **Azure ML** batch scoring job runs hourly:
- Product recommendation model
- Reads recent sessions from Delta Lake
- Scores and writes recommendations to Cosmos DB
1. **Web application** retrieves recommendations from Cosmos DB (< 10ms)

### Example 3: Financial Data Warehouse

1. **Data Factory** ingests data from multiple sources:
- Transactions from on-premises SQL Server (incremental)
- Customer data from Salesforce (full refresh)
- Market data from REST API (daily)
1. **Data Factory** lands raw data in Data Lake (raw zone)
1. **Databricks** processes data:
- Data quality checks and validation (bronze zone)
- Transformations and joins (silver zone)
- Business aggregates (gold zone)
1. **Databricks** loads dimension and fact tables to Synapse dedicated SQL pool
1. **Synapse** handles:
- Slowly Changing Dimensions (Type 2)
- Incremental fact table loads
- Pre-aggregations for performance
1. **Power BI** connects to Synapse:
- Semantic model with relationships
- Row-level security by region
- Scheduled refreshes
1. **Business users** access reports in Power BI Service

### Example 4: Log Analytics

1. **Application servers** send logs to Log Analytics workspace (Azure Monitor)
1. **Data Factory** exports logs to Data Lake (Parquet format)
1. **Databricks** processes logs:
- Parse unstructured log text
- Extract error patterns
- Detect anomalies with ML
1. **Databricks** writes results to:
- Synapse for long-term storage
- Cosmos DB for recent errors (fast queries)
1. **Stream Analytics** processes real-time logs:
- Filters for critical errors
- Aggregates error counts
- Outputs to Power BI for real-time dashboard
1. **Alert rules** in Azure Monitor trigger on thresholds

## Security Best Practices

### 1. **Network Security**

- Private endpoints for Data Lake, Synapse, Databricks
- VNet injection for Databricks clusters
- Managed VNet for Synapse workspace
- Firewall rules on all services (allow only specific IPs)
- No public access to storage accounts

### 2. **Access Control**

- Azure AD authentication for all services
- RBAC at subscription/resource group level
- ACLs on Data Lake folders (read/write/execute)
- Synapse workspace roles (Synapse Administrator, SQL Admin, Spark Admin)
- Just-in-time access with PIM

### 3. **Data Protection**

- Encryption at rest (enabled by default)
- Encryption in transit (TLS 1.2+)
- Customer-managed keys in Key Vault (optional)
- Dynamic data masking in Synapse
- Column-level security for sensitive data

### 4. **Monitoring & Auditing**

- Enable diagnostic logs for all resources
- Centralize logs in Log Analytics
- Audit data access with Azure Storage logs
- Track query execution in Synapse
- Compliance dashboards in Microsoft Purview

## Cost Optimization

### 1. **Storage Optimization**

- Lifecycle management: Auto-tier to Cool/Archive
- File formats: Parquet/Delta for compression
- Partition data for efficient querying
- Delete old/unused data

### 2. **Compute Optimization**

- Databricks: Autoscaling clusters, spot instances
- Synapse: Pause dedicated SQL pool when not in use
- Stream Analytics: Right-size streaming units
- Serverless SQL: Pay per query (better for ad-hoc)

### 3. **Reserved Capacity**

- Synapse dedicated SQL pool: Reserved capacity (up to 65% savings)
- Databricks: Pre-purchase DBUs (up to 37% savings)

### 4. **Query Optimization**

- Partition pruning in Data Lake queries
- Materialized views in Synapse
- Statistics and indexing
- Avoid SELECT * queries

### 5. **Development Optimization**

- Use dev/test pricing for non-production
- Share clusters in Databricks (interactive)
- Auto-terminate idle clusters
- Schedule jobs during off-peak (lower cost)

## Best Practices

1. **Data Lake Organization**: Bronze/Silver/Gold medallion architecture
1. **Partitioning**: Partition by date for time-series data
1. **File Sizes**: Target 128 MB - 1 GB files (avoid small files)
1. **Compression**: Use Snappy for Parquet (balance compression/speed)
1. **Schema Evolution**: Use Delta Lake or Avro for schema changes
1. **Incremental Processing**: Process only new/changed data
1. **Idempotency**: Pipelines should be rerunnable
1. **Testing**: Test with sample data before full runs
1. **Documentation**: Document data lineage and transformations
1. **Monitoring**: Set up alerts for pipeline failures

## When to Use Big Data Architecture

✅ **Good for:**

- Large data volumes (terabytes to petabytes)
- Multiple data sources and formats
- Real-time and batch analytics
- Machine learning at scale
- Complex transformations
- Need for data governance

❌ **Not ideal for:**

- Small datasets (< 100 GB)
- Simple reporting needs
- Transactional applications
- Low latency requirements (< 1ms)
- Limited budget for analytics

## Learning Resources

- Microsoft Learn: [Big data architectures](https://learn.microsoft.com/azure/architecture/data-guide/big-data/)
- Azure Architecture Center: [Analytics end-to-end](https://learn.microsoft.com/azure/architecture/example-scenario/dataplate2e/data-platform-end-to-end)
- [Azure Synapse Analytics documentation](https://learn.microsoft.com/azure/synapse-analytics/)
- [Azure Databricks documentation](https://learn.microsoft.com/azure/databricks/)
- [Data Lake Storage Gen2 documentation](https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction)
- [Stream Analytics documentation](https://learn.microsoft.com/azure/stream-analytics/)
