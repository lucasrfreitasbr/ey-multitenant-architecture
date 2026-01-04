# ⚙️ Backend: DDD & Microservices
## 🧩 Bounded Contexts, Event-Driven Architecture & K8s Mapping

> **📖 Purpose**: This document describes the Domain-Driven Design (DDD) approach, bounded contexts mapped to Kubernetes namespaces, microservices patterns, and event-driven communication via EventBridge and SQS.

---

## 🎯 Scope

- 🧩 **Domain-Driven Design (DDD)**: Bounded contexts, domain models, ubiquitous language
- 🏗️ **Bounded Contexts as K8s Namespaces**: Each bounded context = Kubernetes namespace
- ⚙️ **Microservices Patterns**: Command/query separation, CQRS, service design
- 📡 **Event-Driven Architecture**: EventBridge + SQS for domain events
- 🔐 **Multi-Tenancy Data Patterns**: DynamoDB tenant isolation
- 📦 **Outbox/Inbox Patterns**: Reliable event publishing and consumption

---

## 🏗️ Bounded Contexts in Kubernetes

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "identity namespace"
            IdentityPod1[Identity Service Pod 1]
            IdentityPod2[Identity Service Pod 2]
            IdentityDB[(Identity DynamoDB)]
        end
        
        subgraph "user namespace"
            UserPod1[User Service Pod 1]
            UserPod2[User Service Pod 2]
            UserDB[(User DynamoDB)]
        end
        
        subgraph "billing namespace"
            BillingPod1[Billing Service Pod 1]
            BillingPod2[Billing Service Pod 2]
            BillingDB[(Billing DynamoDB)]
        end
        
        subgraph "notifications namespace"
            NotificationsPod1[Notifications Service Pod 1]
            NotificationsPod2[Notifications Service Pod 2]
            NotificationsDB[(Notifications DynamoDB)]
        end
        
        subgraph "bff namespace"
            BFFPod1[BFF Service Pod 1]
            BFFPod2[BFF Service Pod 2]
        end
    end
    
    BFFPod1 -->|"mTLS via Istio"| IdentityPod1
    BFFPod1 -->|"mTLS via Istio"| UserPod1
    BFFPod1 -->|"mTLS via Istio"| BillingPod1
    BFFPod1 -->|"mTLS via Istio"| NotificationsPod1
    
    IdentityPod1 --> IdentityDB
    UserPod1 --> UserDB
    BillingPod1 --> BillingDB
    NotificationsPod1 --> NotificationsDB
```

### 📊 Bounded Context Mapping

| Bounded Context | K8s Namespace | Services | Purpose |
|----------------|---------------|-----------|---------|
| **🔐 Identity** | `identity` | Identity Service | Authentication, authorization, JWT tokens |
| **👤 User** | `user` | User Service | User profiles, tenant-scoped data |
| **💳 Billing** | `billing` | Billing Service | Subscriptions, invoices, payments |
| **📧 Notifications** | `notifications` | Notifications Service | Email, webhooks, alerts |
| **🎭 BFF** | `bff` | BFF Service | API aggregation, tenant context |

### 🏗️ Namespace Configuration

Each namespace includes:
- **Labels**: `domain: {context-name}`, `tenant-isolation: strict`
- **Istio PeerAuthentication**: `STRICT` mTLS mode
- **NetworkPolicy**: Default deny, explicit allows
- **AuthorizationPolicy**: Service-to-service allowlist

---

## 🏛️ C4 Container Diagram

```mermaid
C4Container
    title Container Diagram - Multi-Tenant SaaS Backend
    
    Person(user, "End User", "Uses the SaaS platform")
    
    System_Boundary(saas, "Multi-Tenant SaaS Platform") {
        Container(frontend, "React Frontend", "React + TypeScript", "User interface")
        Container(bff, "BFF Service", "Node.js + Express", "API aggregation, tenant context")
        
        ContainerDb(identityService, "Identity Service", "Node.js + TypeScript", "Authentication, authorization")
        ContainerDb(userService, "User Service", "Node.js + TypeScript", "User profiles, tenant data")
        ContainerDb(billingService, "Billing Service", "Node.js + TypeScript", "Subscriptions, invoices")
        ContainerDb(notificationsService, "Notifications Service", "Node.js + TypeScript", "Email, webhooks")
        
        ContainerDb(identityDb, "Identity DynamoDB", "DynamoDB", "Identity data, tenant-isolated")
        ContainerDb(userDb, "User DynamoDB", "DynamoDB", "User data, tenant-isolated")
        ContainerDb(billingDb, "Billing DynamoDB", "DynamoDB", "Billing data, tenant-isolated")
        ContainerDb(notificationsDb, "Notifications DynamoDB", "DynamoDB", "Notifications data, tenant-isolated")
        
        Container(eventbridge, "EventBridge", "AWS EventBridge", "Domain events bus")
        Container(sqs, "SQS Queues", "AWS SQS", "Event consumers")
    }
    
    Rel(user, frontend, "Uses", "HTTPS")
    Rel(frontend, bff, "Calls", "HTTPS/REST")
    
    Rel(bff, identityService, "Calls", "HTTPS/mTLS")
    Rel(bff, userService, "Calls", "HTTPS/mTLS")
    Rel(bff, billingService, "Calls", "HTTPS/mTLS")
    Rel(bff, notificationsService, "Calls", "HTTPS/mTLS")
    
    Rel(identityService, identityDb, "Reads/Writes", "DynamoDB API")
    Rel(userService, userDb, "Reads/Writes", "DynamoDB API")
    Rel(billingService, billingDb, "Reads/Writes", "DynamoDB API")
    Rel(notificationsService, notificationsDb, "Reads/Writes", "DynamoDB API")
    
    Rel(identityService, eventbridge, "Publishes", "EventBridge API")
    Rel(userService, eventbridge, "Publishes", "EventBridge API")
    Rel(billingService, eventbridge, "Publishes", "EventBridge API")
    
    Rel(eventbridge, sqs, "Routes to", "EventBridge Rules")
    Rel(sqs, notificationsService, "Consumes", "SQS API")
    Rel(sqs, userService, "Consumes", "SQS API")
    Rel(sqs, billingService, "Consumes", "SQS API")
```

---

## 📡 Event-Driven Architecture

```mermaid
sequenceDiagram
    participant Producer as User Service
    participant Outbox as DynamoDB Outbox
    participant EB as EventBridge
    participant Rule as EventBridge Rule
    participant SQS as SQS Queue
    participant Consumer as Notifications Service
    participant Inbox as DynamoDB Inbox
    
    Producer->>Outbox: Write event (UserCreated.v1)
    Producer->>Producer: Commit transaction
    Producer->>EB: Publish from outbox
    EB->>Rule: Route event
    Rule->>SQS: Send to queue
    SQS->>Consumer: Poll message
    Consumer->>Inbox: Check deduplication
    alt Event not processed
        Consumer->>Consumer: Process event
        Consumer->>Inbox: Mark as processed
        Consumer->>SQS: Delete message
    else Event already processed
        Consumer->>SQS: Delete message (idempotent)
    end
```

### 📡 Event Flow Pattern

1. **Producer**: Domain service writes event to DynamoDB outbox table
2. **Outbox Processor**: Polls outbox, publishes to EventBridge
3. **EventBridge**: Routes event to SQS queues via rules
4. **Consumer**: Polls SQS, checks inbox for deduplication
5. **Processing**: Processes event, marks as processed in inbox

### 🎯 Domain Events

| Event | Publisher | Consumers | Purpose |
|-------|-----------|-----------|---------|
| **UserCreated.v1** | Identity Service | User Service, Notifications Service | New user signup |
| **SubscriptionStarted.v1** | Billing Service | User Service, Notifications Service | Subscription activated |
| **InvoiceIssued.v1** | Billing Service | Notifications Service | Invoice generated |
| **NotificationRequested.v1** | Any Service | Notifications Service | Notification trigger |

---

## 📦 Outbox Pattern

```mermaid
sequenceDiagram
    participant Service as Domain Service
    participant DB as DynamoDB
    participant Outbox as Outbox Table
    participant Processor as Outbox Processor
    participant EB as EventBridge
    
    Service->>DB: Begin transaction
    Service->>DB: Write business data
    Service->>Outbox: Write event (pending)
    Service->>DB: Commit transaction
    
    loop Poll outbox
        Processor->>Outbox: Query pending events
        Processor->>EB: Publish event
        alt Success
            Processor->>Outbox: Mark as published
        else Failure
            Processor->>Outbox: Keep pending (retry)
        end
    end
```

### ✅ Outbox Pattern Benefits

- **✅ Transactional Guarantees**: Event written atomically with business data
- **✅ Reliability**: Events not lost if EventBridge is unavailable
- **✅ Idempotency**: Outbox processor can retry safely
- **✅ Ordering**: Events processed in order (optional)

### 📊 Outbox Table Schema

```json
{
  "event_id": "uuid",
  "domain": "user",
  "event_type": "UserCreated.v1",
  "payload": "{...}",
  "status": "pending|published|failed",
  "created_at": "timestamp",
  "published_at": "timestamp"
}
```

---

## 📥 Inbox Pattern (Deduplication)

```mermaid
sequenceDiagram
    participant SQS as SQS Queue
    participant Consumer as Domain Service
    participant Inbox as Inbox Table
    participant Business as Business Logic
    
    SQS->>Consumer: Receive message
    Consumer->>Inbox: Check if processed (event_id)
    alt Not processed
        Consumer->>Business: Process event
        Business->>Business: Execute business logic
        Consumer->>Inbox: Mark as processed
        Consumer->>SQS: Delete message
    else Already processed
        Consumer->>SQS: Delete message (idempotent)
    end
```

### ✅ Inbox Pattern Benefits

- **✅ Deduplication**: Prevents duplicate event processing
- **✅ Idempotency**: Safe to process same event multiple times
- **✅ Audit Trail**: Track all processed events
- **✅ Recovery**: Can replay events if needed

### 📊 Inbox Table Schema

```json
{
  "event_id": "uuid",
  "domain": "notifications",
  "event_type": "UserCreated.v1",
  "processed_at": "timestamp",
  "status": "processed|failed"
}
```

---

## 🗺️ DDD Context Map

```mermaid
graph TB
    subgraph "Bounded Contexts"
        Identity[🔐 Identity Context<br/>AuthN/AuthZ]
        User[👤 User Context<br/>User Profiles]
        Billing[💳 Billing Context<br/>Subscriptions]
        Notifications[📧 Notifications Context<br/>Alerts]
    end
    
    Identity -->|"Publishes: UserCreated"| User
    Identity -->|"Publishes: UserCreated"| Notifications
    User -->|"Publishes: UserUpdated"| Billing
    Billing -->|"Publishes: SubscriptionStarted"| User
    Billing -->|"Publishes: InvoiceIssued"| Notifications
    Billing -->|"Publishes: SubscriptionStarted"| Notifications
    
    Identity -.->|"Shared Kernel: JWT"| User
    Identity -.->|"Shared Kernel: JWT"| Billing
    Identity -.->|"Shared Kernel: JWT"| Notifications
```

### 🔗 Context Relationships

| Relationship | From | To | Type | Mechanism |
|--------------|------|-----|------|-----------|
| **UserCreated** | Identity | User, Notifications | Published Language | EventBridge |
| **SubscriptionStarted** | Billing | User, Notifications | Published Language | EventBridge |
| **InvoiceIssued** | Billing | Notifications | Published Language | EventBridge |
| **JWT Token** | Identity | All | Shared Kernel | JWT claims |

---

## 🔐 Multi-Tenancy Data Patterns

### 📊 DynamoDB Partition Model

```mermaid
graph LR
    subgraph "DynamoDB Table: user"
        Partition[Partition Key: tenant_id]
        Sort[Sort Key: entity_id]
        Attributes[Attributes: user_data]
    end
    
    Partition -->|"tenant_001"| Tenant1[Tenant 1 Data]
    Partition -->|"tenant_002"| Tenant2[Tenant 2 Data]
    Partition -->|"tenant_003"| Tenant3[Tenant 3 Data]
    
    Tenant1 -->|"user_001"| User1[User 1]
    Tenant1 -->|"user_002"| User2[User 2]
    Tenant2 -->|"user_003"| User3[User 3]
```

### ✅ Data Isolation Strategy

- **Partition Key**: `tenant_id` - Ensures tenant isolation
- **Sort Key**: `entity_id` - Unique entity within tenant
- **Access Control**: Application enforces tenant_id from JWT
- **Country Partition**: Additional `country_partition` (US/BR) for residency

### 📊 Table Schema Example

```json
{
  "tenant_id": "tenant_001",
  "entity_id": "user_123",
  "country_partition": "US",
  "user_data": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "created_at": "2024-01-01T00:00:00Z"
}
```

---

## ⚙️ Service Design Patterns

### 🎯 Command/Query Separation

| Pattern | Endpoint | Purpose |
|---------|----------|---------|
| **Command** | `POST /commands/create-user` | Mutate state, publish events |
| **Query** | `GET /read/users/:id` | Read state, no side effects |

### 📡 CQRS (Command Query Responsibility Segregation)

- **Commands**: Write to DynamoDB, publish events
- **Queries**: Read from DynamoDB (or read model if needed)
- **Events**: Drive eventual consistency across contexts

### 🔐 Tenant Context Middleware

```mermaid
sequenceDiagram
    participant Client as Client
    participant BFF as BFF Service
    participant Middleware as Tenant Middleware
    participant Service as Domain Service
    
    Client->>BFF: Request with JWT
    BFF->>Middleware: Extract tenant_id from JWT
    Middleware->>Middleware: Validate country_partition
    Middleware->>Service: Forward with tenant context
    Service->>Service: Enforce tenant_id in queries
```

---

## 💡 Key Decisions

1. **✅ Bounded Contexts = K8s Namespaces**: Clear domain boundaries, independent deployment
2. **✅ Event-Driven Communication**: EventBridge + SQS for loose coupling, no direct service calls
3. **✅ Outbox Pattern**: Reliable event publishing with transactional guarantees
4. **✅ Inbox Pattern**: Deduplication and idempotent event processing
5. **✅ DynamoDB Tenant Partitioning**: Single table per domain with tenant_id partition key
6. **✅ Command/Query Separation**: Clear separation of write and read operations
7. **✅ Tenant Context Middleware**: Automatic tenant extraction and validation
8. **✅ Istio mTLS**: All service-to-service communication encrypted and authenticated

---

## 🔗 Related Documentation

- 📄 [README.md](../README.md) - Central index
- 🏛️ [01-enterprise-architecture.md](01-enterprise-architecture.md) - EA and composable capabilities
- 🎨 [03-frontend-bff.md](03-frontend-bff.md) - BFF pattern and frontend
- ☁️ [04-cloud-foundation-sre.md](04-cloud-foundation-sre.md) - K8s and Istio infrastructure
- 💾 [05-data-platform-analytics-ml.md](05-data-platform-analytics-ml.md) - Data patterns
- 🔒 [08-security-zero-trust.md](08-security-zero-trust.md) - Security and multi-tenancy

---

> **💡 Tip**: Bounded contexts mapped to K8s namespaces enable clear domain boundaries and independent deployment. Event-driven communication via EventBridge + SQS provides loose coupling without Kafka complexity.

