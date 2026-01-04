# 💾 Data Platform & Analytics
## 📈 DynamoDB, Data Lake, Medallion, Data Vault, Data Mesh & ML

> **📖 Purpose**: This document describes data patterns for multi-tenant SaaS, including DynamoDB tenant isolation, data lake architecture, Medallion pattern, Data Vault modeling, Master Data Management (MDM), Data Mesh principles, and ML/Analytics pipelines.

---

## 🎯 Scope

- 🗄️ **DynamoDB Multi-Tenancy**: Tenant isolation patterns, partition keys, country residency
- 🏞️ **Data Lake Architecture**: S3, Glue, Athena, Lake Formation
- 🥇 **Medallion Architecture**: Bronze (raw), Silver (cleaned), Gold (curated)
- 🏗️ **Data Vault Modeling**: Hubs, links, satellites for historical data
- 👑 **Master Data Management (MDM)**: Single source of truth for master data
- 🕸️ **Data Mesh**: Domain-oriented data products, self-serve platform
- 🤖 **ML/Analytics Pipelines**: SageMaker, QuickSight, data products

---

## 🗄️ DynamoDB Multi-Tenant Patterns

```mermaid
graph TB
    subgraph "DynamoDB Table: user"
        PartitionKey[Partition Key: tenant_id]
        SortKey[Sort Key: entity_id]
        Attributes[Attributes: user_data, country_partition]
    end
    
    subgraph "Tenant Isolation"
        Tenant1[tenant_001<br/>US]
        Tenant2[tenant_002<br/>BR]
        Tenant3[tenant_003<br/>US]
    end
    
    Tenant1 -->|"user_001"| User1[User 1 Data]
    Tenant1 -->|"user_002"| User2[User 2 Data]
    Tenant2 -->|"user_003"| User3[User 3 Data]
    Tenant3 -->|"user_004"| User4[User 4 Data]
    
    PartitionKey --> Tenant1
    PartitionKey --> Tenant2
    PartitionKey --> Tenant3
```

### ✅ DynamoDB Partition Model

**Table Schema:**
```json
{
  "tenant_id": "tenant_001",        // Partition Key
  "entity_id": "user_123",           // Sort Key
  "country_partition": "US",         // Residency partition
  "user_data": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
```

### 🔐 Data Isolation Strategy

| Strategy | Implementation | Purpose |
|----------|---------------|---------|
| **Partition Key** | `tenant_id` | Ensures tenant isolation at DynamoDB level |
| **Sort Key** | `entity_id` | Unique entity within tenant |
| **Country Partition** | `country_partition` (US/BR) | Data residency compliance |
| **Application Enforcement** | JWT claims validation | Additional application-level isolation |

---

## 🏞️ Data Lake Architecture

```mermaid
graph TB
    subgraph "Data Sources"
        DynamoDB[(DynamoDB<br/>Operational Data)]
        APILogs[API Logs]
        EventLogs[Event Logs]
    end
    
    subgraph "S3 Data Lake"
        Bronze[Bronze Layer<br/>Raw Data]
        Silver[Silver Layer<br/>Cleaned Data]
        Gold[Gold Layer<br/>Curated Data]
    end
    
    subgraph "Processing"
        Glue[Glue ETL<br/>Data Transformation]
        GlueCatalog[Glue Catalog<br/>Metadata]
    end
    
    subgraph "Analytics"
        Athena[Athena<br/>SQL Queries]
        QuickSight[QuickSight<br/>Dashboards]
        SageMaker[SageMaker<br/>ML Models]
    end
    
    DynamoDB -->|"DynamoDB Streams"| Bronze
    APILogs -->|"CloudWatch Logs"| Bronze
    EventLogs -->|"EventBridge"| Bronze
    
    Bronze -->|"ETL"| Glue
    Glue --> Silver
    Silver -->|"ETL"| Glue
    Glue --> Gold
    
    Bronze --> GlueCatalog
    Silver --> GlueCatalog
    Gold --> GlueCatalog
    
    GlueCatalog --> Athena
    Athena --> QuickSight
    Gold --> SageMaker
```

### 📊 Data Lake Layers

| Layer | Purpose | Data Format | Retention |
|-------|---------|-------------|-----------|
| **🥇 Bronze** | Raw, unprocessed data | JSON, CSV, Parquet | 7 years |
| **🥈 Silver** | Cleaned, validated data | Parquet | 3 years |
| **🥉 Gold** | Curated, business-ready data | Parquet | 1 year |

---

## 🥇 Medallion Architecture

```mermaid
graph LR
    subgraph "Bronze Layer (Raw)"
        BronzeS3[S3: bronze/]
        BronzeData[Raw Data<br/>No Transformation]
    end
    
    subgraph "Silver Layer (Cleaned)"
        SilverS3[S3: silver/]
        SilverData[Cleaned Data<br/>Validated, Deduplicated]
    end
    
    subgraph "Gold Layer (Curated)"
        GoldS3[S3: gold/]
        GoldData[Curated Data<br/>Business-Ready]
    end
    
    BronzeS3 -->|"Glue ETL"| SilverS3
    SilverS3 -->|"Glue ETL"| GoldS3
    
    BronzeData --> SilverData
    SilverData --> GoldData
```

### 📊 Medallion Processing

**Bronze → Silver:**
- Data validation
- Deduplication
- Schema enforcement
- Data quality checks

**Silver → Gold:**
- Business logic transformation
- Aggregations
- Data enrichment
- Final data products

---

## 🏗️ Data Vault Modeling

```mermaid
graph TB
    subgraph "Data Vault Model"
        HubUser[Hub: User<br/>Business Key: user_id]
        HubTenant[Hub: Tenant<br/>Business Key: tenant_id]
        HubSubscription[Hub: Subscription<br/>Business Key: subscription_id]
        
        LinkUserTenant[Link: User-Tenant<br/>user_id + tenant_id]
        LinkUserSubscription[Link: User-Subscription<br/>user_id + subscription_id]
        
        SatUser[Satellite: User<br/>Name, Email, etc.]
        SatTenant[Satellite: Tenant<br/>Name, Country, etc.]
        SatSubscription[Satellite: Subscription<br/>Plan, Status, etc.]
    end
    
    HubUser --> LinkUserTenant
    HubTenant --> LinkUserTenant
    HubUser --> LinkUserSubscription
    HubSubscription --> LinkUserSubscription
    
    HubUser --> SatUser
    HubTenant --> SatTenant
    HubSubscription --> SatSubscription
```

### 📊 Data Vault Components

| Component | Purpose | Example |
|-----------|---------|---------|
| **Hub** | Business keys | User (user_id), Tenant (tenant_id) |
| **Link** | Relationships | User-Tenant, User-Subscription |
| **Satellite** | Descriptive attributes | User name, email, tenant name |

### ✅ Data Vault Benefits

- **✅ Historical Tracking**: Full history of changes
- **✅ Scalability**: Add new attributes without schema changes
- **✅ Flexibility**: Easy to add new relationships
- **✅ Audit Trail**: Complete audit trail of data changes

---

## 👑 Master Data Management (MDM)

```mermaid
graph TB
    subgraph "Source Systems"
        Identity[Identity Service]
        User[User Service]
        Billing[Billing Service]
    end
    
    subgraph "MDM Hub"
        MDMHub[MDM Hub<br/>Single Source of Truth]
        UserMaster[User Master Data]
        TenantMaster[Tenant Master Data]
    end
    
    subgraph "Consuming Systems"
        Analytics[Analytics]
        Reporting[Reporting]
        ML[ML Models]
    end
    
    Identity -->|"User Data"| MDMHub
    User -->|"User Data"| MDMHub
    Billing -->|"Tenant Data"| MDMHub
    
    MDMHub --> UserMaster
    MDMHub --> TenantMaster
    
    UserMaster --> Analytics
    UserMaster --> Reporting
    UserMaster --> ML
    TenantMaster --> Analytics
```

### 📊 MDM Strategy

| Master Data | Source | Consumers | Purpose |
|-------------|--------|-----------|---------|
| **User** | Identity, User services | Analytics, Reporting, ML | Single source of truth for user data |
| **Tenant** | Identity, Billing services | Analytics, Reporting | Single source of truth for tenant data |

---

## 🕸️ Data Mesh

```mermaid
graph TB
    subgraph "Data Products"
        UserDataProduct[User Data Product<br/>Domain: User]
        BillingDataProduct[Billing Data Product<br/>Domain: Billing]
        AnalyticsDataProduct[Analytics Data Product<br/>Domain: Analytics]
    end
    
    subgraph "Self-Serve Platform"
        Catalog[Data Catalog<br/>Glue Catalog]
        Storage[Storage<br/>S3]
        Compute[Compute<br/>Athena, EMR]
        Governance[Governance<br/>Lake Formation]
    end
    
    subgraph "Consumers"
        Analysts[Data Analysts]
        DataScientists[Data Scientists]
        ML[ML Engineers]
    end
    
    UserDataProduct --> Catalog
    BillingDataProduct --> Catalog
    AnalyticsDataProduct --> Catalog
    
    Catalog --> Storage
    Catalog --> Compute
    Catalog --> Governance
    
    Storage --> Analysts
    Compute --> DataScientists
    Governance --> ML
```

### 🎯 Data Mesh Principles

1. **✅ Domain Ownership**: Each domain owns its data products
2. **✅ Data as Product**: Data products are first-class citizens
3. **✅ Self-Serve Platform**: Centralized platform for data access
4. **✅ Federated Governance**: Domain-driven governance

### 📊 Data Products

| Data Product | Domain | Format | Consumers |
|--------------|--------|--------|-----------|
| **User Data Product** | User | Parquet (Gold layer) | Analytics, ML |
| **Billing Data Product** | Billing | Parquet (Gold layer) | Analytics, Reporting |
| **Analytics Data Product** | Analytics | Parquet (Gold layer) | Dashboards, ML |

---

## 📊 Analytics Pipeline

```mermaid
graph LR
    subgraph "Data Sources"
        DynamoDB[(DynamoDB)]
        Events[EventBridge Events]
        Logs[CloudWatch Logs]
    end
    
    subgraph "Ingestion"
        Streams[DynamoDB Streams]
        Kinesis[Kinesis Firehose]
    end
    
    subgraph "Storage"
        S3[S3 Data Lake]
    end
    
    subgraph "Processing"
        Glue[Glue ETL]
        EMR[EMR Spark]
    end
    
    subgraph "Analytics"
        Athena[Athena Queries]
        QuickSight[QuickSight Dashboards]
        SageMaker[SageMaker ML]
    end
    
    DynamoDB --> Streams
    Events --> Kinesis
    Logs --> Kinesis
    
    Streams --> S3
    Kinesis --> S3
    
    S3 --> Glue
    S3 --> EMR
    
    Glue --> S3
    EMR --> S3
    
    S3 --> Athena
    S3 --> QuickSight
    S3 --> SageMaker
```

### 📊 Analytics Use Cases

| Use Case | Data Source | Processing | Output |
|----------|-------------|------------|--------|
| **User Analytics** | User DynamoDB | Glue ETL | QuickSight dashboard |
| **Revenue Analytics** | Billing DynamoDB | Glue ETL | QuickSight dashboard |
| **Churn Prediction** | User + Billing data | SageMaker ML | ML model predictions |
| **Recommendation Engine** | User behavior data | SageMaker ML | Product recommendations |

---

## 🤖 ML/Analytics Pipelines

### 📊 SageMaker Workflow

```mermaid
graph TB
    subgraph "Data Preparation"
        S3Data[S3: gold/user-data/]
        Preprocessing[Data Preprocessing]
    end
    
    subgraph "Model Training"
        Training[Model Training<br/>SageMaker Training]
        Model[ML Model]
    end
    
    subgraph "Model Deployment"
        Endpoint[SageMaker Endpoint]
        Inference[Real-time Inference]
    end
    
    subgraph "Monitoring"
        ModelMonitor[Model Monitor]
        DriftDetection[Data Drift Detection]
    end
    
    S3Data --> Preprocessing
    Preprocessing --> Training
    Training --> Model
    Model --> Endpoint
    Endpoint --> Inference
    Inference --> ModelMonitor
    ModelMonitor --> DriftDetection
```

### 🎯 ML Use Cases

| Use Case | Model Type | Input | Output |
|----------|-----------|-------|--------|
| **Churn Prediction** | Binary Classification | User behavior, subscription data | Churn probability |
| **Recommendation Engine** | Collaborative Filtering | User interactions | Product recommendations |
| **Fraud Detection** | Anomaly Detection | Transaction data | Fraud score |
| **Revenue Forecasting** | Time Series | Historical billing data | Revenue forecast |

---

## 💡 Key Decisions

1. **✅ DynamoDB Tenant Partitioning**: Single table per domain with tenant_id partition key for isolation
2. **✅ Data Lake Pattern**: S3-based data lake with Medallion architecture (Bronze/Silver/Gold)
3. **✅ Data Vault Modeling**: Historical data tracking with hubs, links, satellites
4. **✅ Data Mesh**: Domain-oriented data products with self-serve platform
5. **✅ MDM Hub**: Single source of truth for master data (users, tenants)
6. **✅ Serverless Analytics**: Athena, Glue, QuickSight for serverless analytics
7. **✅ ML on AWS**: SageMaker for model training, deployment, and monitoring
8. **✅ Lake Formation**: Centralized governance and access control for data lake

---

## 🔗 Related Documentation

- 📄 [README.md](../README.md) - Central index
- ⚙️ [04-backend-ddd-microservices.md](04-backend-ddd-microservices.md) - Domain services and data patterns
- ☁️ [03-cloud-foundation-sre.md](03-cloud-foundation-sre.md) - Infrastructure and S3 setup
- 📊 [08-observability.md](08-observability.md) - Observability and data collection

---

> **💡 Tip**: Medallion architecture (Bronze/Silver/Gold) provides a clear data transformation pipeline. Data Mesh principles enable domain-oriented data products with self-serve analytics capabilities.

