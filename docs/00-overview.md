# 📋 Project Overview
## 🎯 System Context & Technology Stack

> **📖 Purpose**: This document provides a high-level overview of the multi-tenant SaaS reference architecture, its goals, system boundaries, and technology choices.

---

## 🎯 Scope

- ✅ **Project Purpose**: Reference architecture for enterprise-grade multi-tenant SaaS on AWS
- ✅ **Target Audience**: Enterprise architects, senior engineers, interview candidates
- ✅ **Technology Stack**: AWS-native services, EKS, Istio, EventBridge, DynamoDB
- ✅ **Architecture Style**: Domain-Driven Design (DDD) with bounded contexts, event-driven communication
- ✅ **Non-Goals**: This is documentation-only; no code implementation included

---

## 🏛️ System Context (C4 Level 1)

```mermaid
C4Context
    title System Context - Multi-Tenant SaaS Platform
    
    Person(endUsers, "End Users", "Multi-tenant SaaS customers accessing the platform")
    Person(tenantAdmins, "Tenant Administrators", "Managing tenant configuration and users")
    Person(platformAdmins, "Platform Administrators", "Managing platform infrastructure")
    
    System(saasPlatform, "Multi-Tenant SaaS Platform", "AWS-based SaaS platform with DDD, EKS, Istio")
    
    System_Ext(authProvider, "OAuth Provider", "External identity provider (e.g., Okta, Auth0)")
    System_Ext(paymentGateway, "Payment Gateway", "External payment processing (e.g., Stripe)")
    System_Ext(emailService, "Email Service", "External email delivery (e.g., SES, SendGrid)")
    
    SystemDb(eventbridge, "EventBridge", "AWS EventBridge for domain events")
    SystemDb(dynamodb, "DynamoDB", "AWS DynamoDB for tenant-isolated operational data")
    SystemDb(s3Lake, "S3 Data Lake", "AWS S3 for analytics, ML, and data products")
    
    Rel(endUsers, saasPlatform, "Uses", "HTTPS/REST")
    Rel(tenantAdmins, saasPlatform, "Manages", "HTTPS/REST")
    Rel(platformAdmins, saasPlatform, "Administers", "HTTPS/CLI")
    
    Rel(saasPlatform, authProvider, "Validates identity", "OAuth 2.0")
    Rel(saasPlatform, paymentGateway, "Processes payments", "HTTPS/REST")
    Rel(saasPlatform, emailService, "Sends emails", "HTTPS/REST")
    
    Rel(saasPlatform, eventbridge, "Publishes domain events", "EventBridge API")
    Rel(saasPlatform, dynamodb, "Reads/Writes tenant data", "DynamoDB API")
    Rel(saasPlatform, s3Lake, "Writes analytics data", "S3 API")
```

### 🎯 Key Actors

| Actor | Role | Interactions |
|-------|------|--------------|
| **End Users** | Multi-tenant SaaS customers | Use the platform via web/mobile |
| **Tenant Administrators** | Manage tenant configuration | Admin dashboard, user management |
| **Platform Administrators** | Manage infrastructure | AWS Console, Terraform, kubectl |
| **OAuth Provider** | External identity | OAuth 2.0 authentication flows |
| **Payment Gateway** | Payment processing | Payment transactions |
| **Email Service** | Email delivery | Notification emails |

---

## 📊 Technology Stack

```mermaid
graph TB
    subgraph "Frontend Layer"
        React[React + TypeScript]
        Vite[Vite Build Tool]
    end
    
    subgraph "API Layer"
        BFF[BFF - Node.js + Express]
    end
    
    subgraph "Application Layer"
        Identity[Identity Service]
        User[User Service]
        Billing[Billing Service]
        Notifications[Notifications Service]
    end
    
    subgraph "Platform Layer"
        EKS[EKS Cluster]
        Istio[Istio Service Mesh]
        K8s[Kubernetes Namespaces]
    end
    
    subgraph "Data Layer"
        DynamoDB[DynamoDB - Operational Data]
        S3[S3 - Data Lake]
        EventBridge[EventBridge - Events]
        SQS[SQS - Event Consumers]
    end
    
    subgraph "Infrastructure Layer"
        Route53[Route53 - DNS]
        CloudFront[CloudFront - CDN]
        WAF[WAF - Web Application Firewall]
        APIGW[API Gateway]
        VPC[VPC - Networking]
    end
    
    subgraph "Observability Layer"
        OpenTelemetry[OpenTelemetry]
        CloudWatch[CloudWatch Logs]
        Prometheus[Prometheus Metrics]
        Grafana[Grafana Dashboards]
    end
    
    React --> BFF
    BFF --> Identity
    BFF --> User
    BFF --> Billing
    BFF --> Notifications
    
    Identity --> EKS
    User --> EKS
    Billing --> EKS
    Notifications --> EKS
    
    EKS --> Istio
    EKS --> K8s
    
    Identity --> DynamoDB
    User --> DynamoDB
    Billing --> DynamoDB
    Notifications --> DynamoDB
    
    Identity --> EventBridge
    User --> EventBridge
    Billing --> EventBridge
    
    EventBridge --> SQS
    SQS --> Notifications
    
    BFF --> APIGW
    APIGW --> WAF
    WAF --> CloudFront
    CloudFront --> Route53
    
    Identity --> OpenTelemetry
    User --> OpenTelemetry
    Billing --> OpenTelemetry
    Notifications --> OpenTelemetry
    
    OpenTelemetry --> CloudWatch
    OpenTelemetry --> Prometheus
    Prometheus --> Grafana
```

### 🛠️ Technology Choices

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Frontend** | React + TypeScript | Modern, type-safe, component-based UI |
| **BFF** | Node.js + Express | API aggregation, tenant context propagation |
| **Backend** | Node.js + TypeScript | Consistent stack, type safety, microservices |
| **Platform** | EKS + Istio | Managed Kubernetes, service mesh for mTLS |
| **Data** | DynamoDB | Serverless, scalable, tenant partitioning |
| **Events** | EventBridge + SQS | AWS-native, no Kafka complexity, fanout pattern |
| **Analytics** | S3 + Glue + Athena | Data lake pattern, serverless analytics |
| **IaC** | Terraform | Infrastructure as code, version control |
| **Observability** | OpenTelemetry | Vendor-neutral, distributed tracing |

---

## 💡 Key Decisions

1. **✅ Domain-Driven Design (DDD)**: Bounded contexts mapped to K8s namespaces for clear domain boundaries
2. **✅ Event-Driven Architecture**: EventBridge + SQS instead of Kafka for AWS-native simplicity
3. **✅ Multi-Tenancy**: Data isolation via DynamoDB partition keys + application-level tenant context
4. **✅ Service Mesh**: Istio for automatic mTLS, traffic management, and observability
5. **✅ BFF Pattern**: Single backend entry point for frontend, aggregates APIs, enforces security
6. **✅ Zero Trust Security**: Defense in depth with multiple security layers
7. **✅ OpenTelemetry**: Vendor-neutral observability with distributed tracing
8. **✅ AWS-Native**: Leverage managed services (EKS, EventBridge, DynamoDB) for operational simplicity
9. **✅ Terraform IaC**: Infrastructure as code for reproducibility and version control
10. **✅ Composable Architecture**: Lower-level domains compose into higher-level business capabilities

---

## 🎯 Project Goals

### Primary Goals

- ✅ **Reference Architecture**: Comprehensive documentation for multi-tenant SaaS patterns
- ✅ **Interview Preparation**: Interview-grade documentation covering all architectural disciplines
- ✅ **EA Maturity**: Demonstrate composable capabilities, value chain, portfolio management
- ✅ **Production Patterns**: Real-world patterns (outbox, inbox, CQRS, multi-tenancy)
- ✅ **Security First**: Zero Trust model with defense in depth

### Success Criteria

- 📊 **Documentation Coverage**: All 8 discipline documents completed
- 🎨 **Visual Diagrams**: 36+ Mermaid diagrams across all documents
- 🔗 **Cross-References**: All documents linked and cross-referenced
- 💬 **Interview Ready**: Clear talking points and presentation guide

---

## 🚫 Non-Goals

- ❌ **Code Implementation**: This is documentation-only; no actual code
- ❌ **Full Production System**: Reference architecture, not production deployment
- ❌ **All AWS Services**: Focus on core services; not exhaustive AWS coverage
- ❌ **Specific Business Domain**: Generic patterns applicable to any multi-tenant SaaS

---

## 💬 Interview Presentation Guide

### 🎯 10-Bullet Presentation Script

1. **🏛️ Architecture Overview**: "This is a multi-tenant SaaS reference architecture on AWS using DDD, EKS, and Istio. Bounded contexts map to K8s namespaces, enabling clear domain boundaries."

2. **🧩 Composable Capabilities**: "Lower-level domains (identity, user, billing) compose into higher-level business capabilities (customer onboarding, revenue management), demonstrating EA maturity."

3. **📡 Event-Driven Communication**: "We use EventBridge + SQS for loose coupling. EventBridge routes domain events to SQS queues per consumer, avoiding Kafka complexity while maintaining scalability."

4. **🔐 Multi-Tenancy**: "Tenant isolation at data layer (DynamoDB partition keys) and application layer (JWT claims). We also enforce country residency partitions (US/BR) with geo-validation."

5. **🛡️ Zero Trust Security**: "Defense in depth: WAF → API Gateway → Istio mTLS → NetworkPolicies → Pod Security. Every request is verified, no implicit trust."

6. **🔗 Service Mesh**: "Istio provides automatic mTLS for east-west traffic, traffic management, and observability. All service-to-service calls are encrypted and authorized."

7. **📊 Observability**: "OpenTelemetry for distributed tracing across services and event publishing. We track requests from BFF through domain services to event publishing."

8. **💾 Data Platform**: "Medallion architecture (Bronze/Silver/Gold) with Data Mesh principles. Domain-oriented data products enable self-serve analytics."

9. **🔄 DevSecOps**: "Shift-left security: SAST, SCA, IaC scanning, and container scanning in CI/CD. Security checks happen early, not just before deployment."

10. **☁️ Infrastructure**: "Terraform for IaC, EKS for orchestration, AWS-native services for operational simplicity. Everything is version-controlled and reproducible."

---

## 🚀 Next Steps for Implementation

### 📋 Implementation Roadmap

1. **🏗️ Infrastructure Setup**
   - ✅ Terraform modules for VPC, EKS, networking
   - ✅ EKS cluster with node groups and IRSA
   - ✅ Istio installation and configuration

2. **🔐 Security Foundation**
   - ✅ WAF rules and API Gateway policies
   - ✅ IAM roles and IRSA for services
   - ✅ Secrets management (Secrets Manager, SSM)

3. **⚙️ Backend Services**
   - ✅ Domain services (identity, user, billing, notifications)
   - ✅ BFF service with tenant context middleware
   - ✅ Event adapters (EventBridge publisher, SQS consumer)

4. **🎨 Frontend**
   - ✅ React app with tenant selection
   - ✅ BFF client integration
   - ✅ Pages for Users, Billing, Notifications

5. **📊 Observability**
   - ✅ OpenTelemetry instrumentation
   - ✅ CloudWatch logs and Prometheus metrics
   - ✅ Grafana dashboards and SLO definitions

6. **🔄 CI/CD**
   - ✅ GitHub Actions workflows
   - ✅ Docker image builds and ECR pushes
   - ✅ Terraform plan/apply automation

7. **💾 Data Platform**
   - ✅ DynamoDB tables per domain
   - ✅ S3 buckets for data lake
   - ✅ Glue catalog and Athena queries

8. **🧪 Testing**
   - ✅ Unit tests for services
   - ✅ Integration tests for event flows
   - ✅ Contract tests for APIs

9. **📈 Performance**
   - ✅ Load testing and optimization
   - ✅ Caching strategies
   - ✅ Database query optimization

10. **✅ Production Hardening**
    - ✅ Real JWT validation (not simulated)
    - ✅ Real EventBridge integration (not console.log)
    - ✅ Comprehensive error handling
    - ✅ Disaster recovery procedures
    - ✅ Cost optimization

---

## 🔗 Related Documentation

- 📄 [README.md](../README.md) - Central index and navigation
- 🏛️ [01-enterprise-architecture.md](01-enterprise-architecture.md) - EA frameworks and composable capabilities
- ⚙️ [02-backend-ddd-microservices.md](02-backend-ddd-microservices.md) - DDD and microservices patterns
- 🎨 [03-frontend-bff.md](03-frontend-bff.md) - Frontend and BFF architecture
- ☁️ [04-cloud-foundation-sre.md](04-cloud-foundation-sre.md) - AWS infrastructure and SRE
- 💾 [05-data-platform-analytics-ml.md](05-data-platform-analytics-ml.md) - Data platform and analytics
- 📊 [06-observability.md](06-observability.md) - Observability and OpenTelemetry
- 🔄 [07-devsecops.md](07-devsecops.md) - CI/CD and DevSecOps
- 🔒 [08-security-zero-trust.md](08-security-zero-trust.md) - Zero Trust security model

---

> **💡 Tip**: Start here to understand the system context, then dive into specific disciplines based on your interests or interview focus.

