# ☁️ Cloud Foundation & SRE
## 🌐 AWS Infrastructure, EKS, Istio & Site Reliability Engineering

> **📖 Purpose**: This document describes the AWS infrastructure architecture, EKS cluster design, Istio service mesh, networking topology, and SRE practices for operational excellence.

---

## 🎯 Scope

- ☁️ **AWS Infrastructure**: VPC, subnets, endpoints, security groups, networking
- 🎯 **EKS Cluster**: Cluster design, node groups, IRSA (IAM Roles for Service Accounts)
- 🔗 **Istio Service Mesh**: Ingress gateway, east-west traffic, mTLS, traffic management
- 🌐 **Networking**: Route53 → CloudFront → WAF → API Gateway → EKS topology
- 📊 **SRE Practices**: SLIs, SLOs, error budgets, runbooks, incident response

---

## 🌐 Network Topology (North-South Traffic)

```mermaid
graph TB
    subgraph "Edge Layer"
        Route53[Route53<br/>DNS]
        CloudFront[CloudFront<br/>CDN]
        WAF[WAF<br/>Web Application Firewall]
    end
    
    subgraph "API Layer"
        APIGW[API Gateway<br/>REST API]
        VPCLink[VPC Link<br/>Private Integration]
    end
    
    subgraph "VPC"
        subgraph "Public Subnets"
            NLB[NLB/ALB<br/>Load Balancer]
        end
        
        subgraph "Private Subnets"
            IstioGW[Istio Ingress Gateway]
        end
        
        subgraph "EKS Cluster"
            BFF[BFF Service]
            Identity[Identity Service]
            User[User Service]
            Billing[Billing Service]
            Notifications[Notifications Service]
        end
    end
    
    Route53 --> CloudFront
    CloudFront --> WAF
    WAF --> APIGW
    APIGW --> VPCLink
    VPCLink --> NLB
    NLB --> IstioGW
    IstioGW --> BFF
    BFF --> Identity
    BFF --> User
    BFF --> Billing
    BFF --> Notifications
```

### 📊 Traffic Flow

| Layer | Component | Purpose |
|-------|-----------|---------|
| **Edge** | Route53 | DNS resolution |
| **Edge** | CloudFront | CDN, caching, DDoS protection |
| **Edge** | WAF | Web application firewall, OWASP rules |
| **API** | API Gateway | REST API, rate limiting, API keys |
| **API** | VPC Link | Private integration to VPC |
| **Network** | NLB/ALB | Load balancing |
| **Mesh** | Istio Ingress Gateway | Service mesh ingress |
| **Application** | EKS Services | Domain services |

---

## 🔗 Istio Service Mesh (East-West Traffic)

```mermaid
graph TB
    subgraph "EKS Cluster"
        subgraph "istio-system namespace"
            IstioControlPlane[Istio Control Plane]
        end
        
        subgraph "bff namespace"
            BFFPod1[BFF Pod 1]
            BFFPod2[BFF Pod 2]
        end
        
        subgraph "identity namespace"
            IdentityPod1[Identity Pod 1]
            IdentityPod2[Identity Pod 2]
        end
        
        subgraph "user namespace"
            UserPod1[User Pod 1]
            UserPod2[User Pod 2]
        end
        
        subgraph "billing namespace"
            BillingPod1[Billing Pod 1]
            BillingPod2[Billing Pod 2]
        end
        
        subgraph "notifications namespace"
            NotificationsPod1[Notifications Pod 1]
            NotificationsPod2[Notifications Pod 2]
        end
    end
    
    IstioControlPlane -.->|"mTLS STRICT"| BFFPod1
    IstioControlPlane -.->|"mTLS STRICT"| IdentityPod1
    IstioControlPlane -.->|"mTLS STRICT"| UserPod1
    IstioControlPlane -.->|"mTLS STRICT"| BillingPod1
    IstioControlPlane -.->|"mTLS STRICT"| NotificationsPod1
    
    BFFPod1 -->|"mTLS via Istio"| IdentityPod1
    BFFPod1 -->|"mTLS via Istio"| UserPod1
    BFFPod1 -->|"mTLS via Istio"| BillingPod1
    BFFPod1 -->|"mTLS via Istio"| NotificationsPod1
```

### 🔒 Istio Security Configuration

**PeerAuthentication (mTLS STRICT):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: identity
spec:
  mtls:
    mode: STRICT
```

**AuthorizationPolicy (Service-to-Service):**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-bff-to-identity
  namespace: identity
spec:
  selector:
    matchLabels:
      app: identity-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bff/sa/bff-service"]
```

---

## 🎯 EKS Cluster Architecture

```mermaid
graph TB
    subgraph "EKS Cluster"
        subgraph "Control Plane"
            EKSControlPlane[EKS Control Plane<br/>Managed by AWS]
        end
        
        subgraph "Node Groups"
            subgraph "Managed Node Group 1"
                Node1[Node 1<br/>t3.large]
                Node2[Node 2<br/>t3.large]
            end
            
            subgraph "Managed Node Group 2"
                Node3[Node 3<br/>t3.large]
                Node4[Node 4<br/>t3.large]
            end
        end
        
        subgraph "Namespaces"
            BFFNS[bff namespace]
            IdentityNS[identity namespace]
            UserNS[user namespace]
            BillingNS[billing namespace]
            NotificationsNS[notifications namespace]
        end
    end
    
    EKSControlPlane --> Node1
    EKSControlPlane --> Node2
    EKSControlPlane --> Node3
    EKSControlPlane --> Node4
    
    Node1 --> BFFNS
    Node1 --> IdentityNS
    Node2 --> UserNS
    Node2 --> BillingNS
    Node3 --> NotificationsNS
    Node4 --> BFFNS
```

### 📊 EKS Configuration

| Component | Configuration | Purpose |
|-----------|--------------|---------|
| **Cluster** | EKS 1.28+ | Kubernetes orchestration |
| **Node Groups** | Managed node groups (t3.large) | Worker nodes |
| **IRSA** | IAM Roles for Service Accounts | AWS service access |
| **Networking** | VPC CNI, Calico | Pod networking |
| **Storage** | EBS CSI driver | Persistent volumes |

---

## 🔄 Traffic Flow (North-South & East-West)

```mermaid
sequenceDiagram
    participant User as End User
    participant Route53 as Route53
    participant CloudFront as CloudFront
    participant WAF as WAF
    participant APIGW as API Gateway
    participant NLB as NLB
    participant IstioGW as Istio Ingress Gateway
    participant BFF as BFF Service
    participant UserService as User Service
    
    User->>Route53: DNS lookup
    Route53->>CloudFront: Route to CloudFront
    CloudFront->>WAF: Check WAF rules
    WAF->>APIGW: Forward to API Gateway
    APIGW->>NLB: VPC Link to NLB
    NLB->>IstioGW: Load balance to Istio Gateway
    IstioGW->>BFF: Route to BFF service
    BFF->>UserService: mTLS via Istio
    UserService-->>BFF: Response
    BFF-->>IstioGW: Response
    IstioGW-->>NLB: Response
    NLB-->>APIGW: Response
    APIGW-->>WAF: Response
    WAF-->>CloudFront: Response
    CloudFront-->>User: Response
```

### 📊 Traffic Types

| Type | Path | Encryption |
|------|------|------------|
| **North-South** | User → Route53 → CloudFront → WAF → API Gateway → NLB → Istio Gateway → Services | TLS 1.3 |
| **East-West** | Service → Service (via Istio) | mTLS (Istio) |

---

## 🔐 Network Policies

```mermaid
graph TB
    subgraph "Default Deny"
        DefaultDeny[Default NetworkPolicy<br/>Deny All]
    end
    
    subgraph "Allowed Traffic"
        BFFToIdentity[BFF → Identity<br/>Allowed]
        BFFToUser[BFF → User<br/>Allowed]
        BFFToBilling[BFF → Billing<br/>Allowed]
        BFFToNotifications[BFF → Notifications<br/>Allowed]
        IstioToAll[Istio System → All<br/>Allowed]
    end
    
    DefaultDeny --> BFFToIdentity
    DefaultDeny --> BFFToUser
    DefaultDeny --> BFFToBilling
    DefaultDeny --> BFFToNotifications
    DefaultDeny --> IstioToAll
```

### 🛡️ NetworkPolicy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-bff-to-user
  namespace: user
spec:
  podSelector:
    matchLabels:
      app: user-service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: bff
      podSelector:
        matchLabels:
          app: bff-service
    ports:
    - protocol: TCP
      port: 8080
```

---

## 🔐 IAM Roles for Service Accounts (IRSA)

```mermaid
graph LR
    subgraph "EKS Cluster"
        ServiceAccount[Service Account<br/>user-service-sa]
        Pod[User Service Pod]
    end
    
    subgraph "AWS IAM"
        IAMRole[IAM Role<br/>user-service-role]
        Policy[IAM Policy<br/>DynamoDB Access]
    end
    
    subgraph "AWS Services"
        DynamoDB[(DynamoDB)]
        S3[(S3)]
    end
    
    ServiceAccount -->|"Assumes"| IAMRole
    Pod -->|"Uses"| ServiceAccount
    IAMRole -->|"Grants"| Policy
    Policy -->|"Allows"| DynamoDB
    Policy -->|"Allows"| S3
```

### 📊 IRSA Configuration

| Service | IAM Role | Permissions |
|---------|----------|------------|
| **User Service** | `user-service-role` | DynamoDB read/write (user table) |
| **Billing Service** | `billing-service-role` | DynamoDB read/write (billing table) |
| **Notifications Service** | `notifications-service-role` | DynamoDB read/write, SES send email |
| **BFF Service** | `bff-service-role` | No AWS service access (only calls other services) |

---

## 📊 SRE Practices

### 🎯 SLIs (Service Level Indicators)

| SLI | Measurement | Target |
|-----|-------------|--------|
| **Availability** | Uptime percentage | 99.9% (3 nines) |
| **Latency (p50)** | Median response time | < 200ms |
| **Latency (p99)** | 99th percentile | < 1000ms |
| **Error Rate** | 5xx errors / total requests | < 0.1% |
| **Throughput** | Requests per second | > 1000 RPS |

### 📈 SLOs (Service Level Objectives)

| Service | SLO | Error Budget |
|---------|-----|--------------|
| **BFF** | 99.9% availability | 43.2 minutes/month |
| **Identity** | 99.95% availability | 21.6 minutes/month |
| **User** | 99.9% availability | 43.2 minutes/month |
| **Billing** | 99.95% availability | 21.6 minutes/month |
| **Notifications** | 99.5% availability | 216 minutes/month |

### 📖 Runbooks

**Common Runbooks:**
- 🔴 **Service Down**: Check pods, check logs, check dependencies
- 🟡 **High Latency**: Check metrics, check database, check network
- 🟠 **High Error Rate**: Check logs, check dependencies, check configuration
- 🔵 **Capacity Issues**: Scale pods, check node capacity, check quotas

### 🚨 Incident Response

1. **Detection**: Alerts from CloudWatch, Prometheus
2. **Triage**: Identify affected services, scope impact
3. **Mitigation**: Rollback, scale up, fix configuration
4. **Resolution**: Root cause analysis, post-mortem
5. **Prevention**: Update runbooks, improve monitoring

---

## 💡 Key Decisions

1. **✅ AWS-Native Services**: Leverage managed services (EKS, EventBridge, DynamoDB) for operational simplicity
2. **✅ Istio Service Mesh**: Automatic mTLS, traffic management, observability
3. **✅ Defense in Depth**: Multiple security layers (WAF, API Gateway, Istio, NetworkPolicies)
4. **✅ IRSA**: IAM Roles for Service Accounts for least-privilege AWS access
5. **✅ Network Policies**: Default deny, explicit allows for network segmentation
6. **✅ SLO-Based Operations**: SLIs, SLOs, error budgets for reliability
7. **✅ Managed Node Groups**: AWS-managed EKS node groups for operational simplicity
8. **✅ VPC Endpoints**: Private connectivity to AWS services (S3, DynamoDB)

---

## 🔗 Related Documentation

- 📄 [README.md](../README.md) - Central index
- 🏛️ [01-enterprise-architecture.md](01-enterprise-architecture.md) - Enterprise architecture
- 🏛️ [02-overall-solution-architecture.md](02-overall-solution-architecture.md) - Overall solution architecture
- ⚙️ [04-backend-ddd-microservices.md](04-backend-ddd-microservices.md) - Domain services
- 🔒 [07-security-zero-trust.md](07-security-zero-trust.md) - Security model
- 📊 [08-observability.md](08-observability.md) - Observability and monitoring

---

> **💡 Tip**: Istio service mesh provides automatic mTLS for all service-to-service communication, eliminating the need for manual certificate management. NetworkPolicies add an additional layer of network segmentation.

