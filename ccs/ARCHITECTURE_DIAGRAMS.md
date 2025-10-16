# Veyron Architecture Diagrams
## Visual Reference for System Understanding

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph "Client Applications"
        A[Campaign UI]
        B[Journey Builder]
        C[Loyalty Module]
    end
    
    subgraph "Veyron CCS API Layer"
        D[MessageMetaController<br/>Create/Update Configs]
        E[MessageValidationController<br/>Validate Messages]
        F[DeliveryReportController<br/>Get Status]
    end
    
    subgraph "Facade Layer"
        G[MessageMetaFacade<br/>Config Management]
        H[TransactionalMessageFacade<br/>Txn Logic]
        I[PromotionalMessageFacade<br/>Promo Logic]
    end
    
    subgraph "Service Layer"
        J[MessageMetaConfigService<br/>CRUD Operations]
        K[MessageCreationService<br/>Create Messages]
        L[MessageExecutionService<br/>Execute Messages]
        M[ValidationServices<br/>Validate Requests]
    end
    
    subgraph "Queue Layer - Apache Camel"
        N[CreateMessageProcessor]
        O[MessageExecutionProcessor]
        P[TransactionalMessageProcessor]
        Q[PromotionalMessageProcessor]
    end
    
    subgraph "Data Layer"
        R[(MongoDB<br/>txn_message_meta_config)]
        S[(MongoDB<br/>promo_message_meta_config)]
        T[(MongoDB<br/>message_meta_details)]
        U[(MongoDB<br/>tag)]
    end
    
    subgraph "External Services"
        V[NSAdmin<br/>Transactional Delivery]
        W[Veneno<br/>Promotional Delivery]
        X[Intouch<br/>Org/User Data]
    end
    
    A --> D
    B --> D
    C --> D
    D --> G
    E --> H
    E --> I
    F --> W
    
    G --> J
    H --> K
    I --> K
    
    K --> N
    L --> O
    H --> P
    I --> Q
    
    N --> T
    O --> T
    P --> V
    Q --> W
    
    J --> R
    J --> S
    J --> U
    
    V --> X
    W --> X
```

---

## 2. Message Flow - Transactional vs Promotional

```mermaid
sequenceDiagram
    participant Client as Campaign/Journey
    participant API as Veyron API
    participant Facade as Message Facade
    participant Queue as Queue Processor
    participant DB as MongoDB
    participant NSAdmin as NSAdmin Service
    participant Veneno as Veneno Service
    
    %% Transactional Flow
    rect rgb(200, 220, 255)
        Note over Client,Veneno: TRANSACTIONAL FLOW
        Client->>API: POST /message/validate<br/>(MessageRequest)
        API->>Facade: TransactionalMessageFacade.validate()
        Facade-->>API: Validation OK
        
        Client->>Queue: Push to TransactionalMessageProcessor
        Queue->>Facade: TransactionalMessageFacade.sendMessage()
        Facade->>DB: Fetch MessageMetaConfig (APPROVED)
        DB-->>Facade: Config
        
        Facade->>Queue: Push to TransTagResolverProducer
        Queue->>NSAdmin: NsadminMessageBuilder.build()
        Note over NSAdmin: Apply delay<br/>Apply blackout time
        NSAdmin->>NSAdmin: Push to gateway queue
        NSAdmin-->>Client: Delivery status (async)
    end
    
    %% Promotional Flow
    rect rgb(255, 220, 200)
        Note over Client,Veneno: PROMOTIONAL FLOW
        Client->>API: POST /message/validate<br/>(SendPromotionalMessageRequest)
        API->>Facade: PromotionalMessageFacade.validate()
        Facade-->>API: Validation OK
        
        Client->>Queue: Push to PromotionalMessageProcessor
        Queue->>Facade: PromotionalMessageFacade.sendMessage()
        Facade->>DB: Fetch PromotionalMessageMetaConfig
        DB-->>Facade: Config with venenoReferenceId
        
        Facade->>Veneno: VenenoService.pushToVenenoBulkQueue()
        Note over Veneno: Build VenenoCustomerInfo<br/>with resolved tags
        Veneno->>Veneno: Push to Veneno queue
        Veneno-->>Client: Delivery status (async)
    end
```

---

## 3. Data Model - MessageMetaConfig Hierarchy

```mermaid
classDiagram
    class MessageMetaConfig {
        <<abstract>>
        +String id
        +Long orgId
        +Long ouId
        +String sourceEntityId
        +Channel channel
        +MessageContent messageContent
        +DeliverySettings deliverySettings
        +Map executionParams
        +MessageConfigStatus status
        +Module module
        +toRO() MessageMetaConfigRO
        +isApproved() boolean
    }
    
    class TransactionalMessageMetaConfig {
        +CommunicationType communicationType = TRANSACTION
    }
    
    class PromotionalMessageMetaConfig {
        +CommunicationType communicationType = PROMOTION
        +Long venenoReferenceId
    }
    
    class DeliverySettings {
        +ChannelSettings channelSettings
        +AdditionalSettings additionalSettings
        +DelayedScheduleSettings delayedScheduleSettings
    }
    
    class ChannelSettings {
        <<abstract>>
        +Channel channel
        +Long domainId
        +Long domainGatewayMapId
    }
    
    class SmsChannelSettings {
        +String gsmSenderId
        +String cdmaSenderId
        +Boolean targetNdnc
    }
    
    class EmailChannelSettings {
        +String senderId
        +String senderLabel
        +String replyToId
    }
    
    class AdditionalSettings {
        +boolean useTinyUrl
        +boolean encryptUrl
        +boolean linkTrackingEnabled
        +boolean userSubscriptionDisabled
    }
    
    class DelayedScheduleSettings {
        +Integer duration
        +TimeUnit timeUnit
    }
    
    MessageMetaConfig <|-- TransactionalMessageMetaConfig
    MessageMetaConfig <|-- PromotionalMessageMetaConfig
    MessageMetaConfig *-- DeliverySettings
    DeliverySettings *-- ChannelSettings
    DeliverySettings *-- AdditionalSettings
    DeliverySettings *-- DelayedScheduleSettings
    ChannelSettings <|-- SmsChannelSettings
    ChannelSettings <|-- EmailChannelSettings
```

---

## 4. Queue Architecture

```mermaid
graph LR
    subgraph "Inbound Queues"
        A[VeyronQueue.MESSAGE_CREATION]
        B[VeyronQueue.BULK_MESSAGE_CREATION]
        C[VeyronQueue.TRANSACTIONAL_MESSAGE]
        D[VeyronQueue.PROMOTIONAL_MESSAGE]
    end
    
    subgraph "Processors"
        E[CreateMessageProcessor]
        F[CreateBulkMessageProcessor]
        G[TransactionalMessageProcessor]
        H[PromotionalMessageProcessor]
    end
    
    subgraph "Execution Queues"
        I[VeyronQueue.MESSAGE_EXECUTION]
        J[VeyronQueue.BULK_MESSAGE_EXECUTION]
    end
    
    subgraph "Execution Processors"
        K[MessageExecutionProcessor]
        L[MessageExecutionBulkProcessor]
    end
    
    subgraph "External Queues"
        M[NSADMIN_RECIPIENTS_LIST_TRANS]
        N[NSADMIN_RECIPIENTS_LIST]
        O[VENENO_BULK_QUEUE]
    end
    
    A --> E
    B --> F
    C --> G
    D --> H
    
    E --> I
    F --> J
    G --> M
    H --> O
    
    I --> K
    J --> L
    
    K --> M
    K --> O
    L --> N
    L --> O
    
    style C fill:#ffcccc
    style D fill:#ffcccc
    style H fill:#ffcccc
    style O fill:#ffcccc
```

---

## 5. Validation Pipeline

```mermaid
flowchart TD
    A[Incoming MessageRequest] --> B{Is Valid JSON?}
    B -->|No| Z[Return 400 Error]
    B -->|Yes| C[MessageRequestValidatorService.validate]
    
    C --> D[JSR-303 Bean Validation]
    D --> E{Violations?}
    E -->|Yes| Z
    
    E -->|No| F[Channel-Specific Validator]
    
    F --> G{Channel Type}
    G -->|EMAIL| H[EmailRequestValidator]
    G -->|SMS| I[SmsRequestValidator]
    G -->|WHATSAPP| J[WhatsappRequestValidator]
    G -->|PUSH| K[PushRequestValidator]
    
    H --> H1{Is Promotional?}
    H1 -->|Yes| H2[Check UNSUBSCRIBE tag]
    H2 -->|Missing| Z
    H2 -->|Present| L
    H1 -->|No| L
    
    I --> L[MessageRequestValidator.validate]
    J --> L
    K --> L
    
    L --> M[Check Tags]
    L --> N[Check Execution Params]
    L --> O[Check Incentive Tags]
    
    M --> P{All Valid?}
    N --> P
    O --> P
    
    P -->|No| Z
    P -->|Yes| Q[Success - Message Validated]
    
    style H2 fill:#ffcccc
    style Z fill:#ff9999
    style Q fill:#99ff99
```

---

## 6. Tag System Architecture

```mermaid
graph TB
    subgraph "Tag Sources"
        A[Tags Enum<br/>100+ Pre-defined]
        B[MongoDB Tag Collection<br/>Custom Tags]
    end
    
    subgraph "Tag Usage"
        C[Campaign Module]
        D[Journey Module]
        E[Loyalty Module]
        F[CCS/Veyron]
    end
    
    subgraph "Tag Resolution"
        G[TagExtractorUtils<br/>Extract tags from content]
        H[Intouch/Veneno<br/>Resolve tag values]
    end
    
    subgraph "Tag Validation"
        I[MessageRequestValidator<br/>Check if tags supported]
        J[Channel Validators<br/>Check required tags]
    end
    
    A --> G
    B --> G
    
    C --> G
    D --> G
    E --> G
    F --> G
    
    G --> I
    I --> J
    J --> H
    
    style A fill:#ffffcc
    style B fill:#ffffcc
    Note1[Problem: No module-based<br/>filtering implemented]
    Note1 -.-> C
    Note1 -.-> D
```

---

## 7. Delay & Scheduling Flow

```mermaid
sequenceDiagram
    participant Config as MessageMetaConfig
    participant Builder as NsadminMessageBuilder
    participant Helper as SendMessageHelperService
    participant Org as Organization (Intouch)
    
    Note over Config: DelayedScheduleSettings:<br/>duration: 30<br/>timeUnit: MINUTES
    
    Config->>Builder: build(messageMetaConfig)
    Builder->>Helper: getOrganizationTimeStamps(orgId, channel)
    Helper->>Org: Get blackout time settings
    Org-->>Helper: minSMSHour, maxSMSHour, timezone
    
    Helper->>Helper: Calculate scheduledTimestamp<br/>based on blackout
    Helper-->>Builder: Return timestamps
    
    Builder->>Builder: applyDelayedScheduleIfApplicable()
    Note over Builder: currentTime = scheduledTimestamp<br/>Add 30 minutes<br/>Return new timestamp
    
    Builder->>Builder: Set message.scheduledTime
    Builder-->>Config: Message with delayed schedule
    
    rect rgb(255, 200, 200)
        Note over Config,Org: MISSING: Absolute scheduling<br/>Need: scheduledAt: "2025-12-25T09:00:00"
    end
```

---

## 8. MongoDB Collections Structure

```mermaid
erDiagram
    txn_message_meta_config ||--o{ message_meta_details : creates
    promo_message_meta_config ||--o{ message_meta_details : creates
    tag ||--o{ message_meta_config : references
    
    txn_message_meta_config {
        string id PK
        long orgId
        long ouId
        string sourceEntityId
        string channel
        object messageContent
        object deliverySettings
        string status
        date approvedAt
    }
    
    promo_message_meta_config {
        string id PK
        long orgId
        long ouId
        string sourceEntityId
        string channel
        object messageContent
        object deliverySettings
        long venenoReferenceId
        string status
        date approvedAt
    }
    
    message_meta_details {
        long id PK
        string messageMetaHashId
        long orgId
        long ouId
        string eventId
        string entityId
        string sourceEntityId
        string communicationType
        string channel
        object messageContent
        object deliverySettings
        long venenoReferenceId
        boolean test
    }
    
    tag {
        string id PK
        string tagName
        string groupName
        string description
        string clientName
        string type
    }
```

---

## 9. External Service Dependencies

```mermaid
graph LR
    subgraph "Veyron CCS"
        A[Veyron Service]
    end
    
    subgraph "Capillary Platform Services"
        B[Veneno<br/>Promotional Delivery]
        C[NSAdmin<br/>Gateway Management]
        D[Intouch<br/>Org/User Data]
        E[Template Service<br/>Template Management]
        F[Coupon Service<br/>Voucher Validation]
    end
    
    subgraph "External Gateways"
        G[SMS Gateways]
        H[Email Providers]
        I[WhatsApp Business API]
        J[Push Notification Services]
    end
    
    A -->|Promotional sends| B
    A -->|Transactional sends| C
    A -->|User/org lookup| D
    A -->|Template resolution| E
    A -->|Voucher validation| F
    
    B --> G
    B --> H
    B --> I
    B --> J
    
    C --> G
    C --> H
    C --> I
    C --> J
    
    style B fill:#ffcccc
    style A fill:#ccffcc
```

---

## 10. Feature Flags & Configuration

```mermaid
graph TB
    subgraph "AdditionalSettings - Feature Flags"
        A[useTinyUrl<br/>Shorten URLs]
        B[encryptUrl<br/>Encrypt URLs]
        C[linkTrackingEnabled<br/>Track Link Clicks]
        D[userSubscriptionDisabled<br/>Override Opt-out]
    end
    
    subgraph "Channel-Specific Settings"
        E[SMS: targetNdnc<br/>NDNC Check]
        F[Email: replyToId<br/>Reply Address]
        G[WhatsApp: accountId<br/>Business Account]
    end
    
    subgraph "Org-Level Settings"
        H[Blackout Time<br/>minSMSHour/maxSMSHour]
        I[Timezone<br/>Base Timezone]
        J[DLT Enabled<br/>Template Registration]
    end
    
    subgraph "Message Delivery Behavior"
        K[Final Message]
    end
    
    A --> K
    B --> K
    C --> K
    D --> K
    E --> K
    F --> K
    G --> K
    H --> K
    I --> K
    J --> K
    
    style D fill:#ffccaa
```

---

## 11. Implementation Roadmap - Gantt Chart

```mermaid
gantt
    title Veyron CCS Enhancement - Implementation Timeline
    dateFormat YYYY-MM-DD
    section Epic 1: Promotional Support
    Opt-out validation for all channels :a1, 2025-11-01, 3d
    Latency benchmarking :a2, after a1, 2d
    
    section Epic 2: Tag Unification
    Tag audit script :b1, 2025-11-01, 3d
    Module-based tag filtering API :b2, after b1, 4d
    Tag suggestion API :b3, after b2, 3d
    
    section Epic 3: Scheduling
    Add scheduledAt field to model :c1, 2025-11-06, 2d
    Scheduled message queue processor :c2, after c1, 5d
    Timezone conversion logic :c3, after c2, 3d
    
    section Epic 4: Delivery Settings
    Already complete :milestone, 2025-11-01, 0d
    
    section Epic 5: Reporting
    Verify promotional metrics :e1, 2025-11-06, 2d
    Module-level segmentation :e2, after e1, 3d
    Analytics dashboard :e3, after e2, 5d
    
    section Testing & Integration
    Integration testing :f1, 2025-11-15, 5d
    Load testing :f2, after f1, 3d
    UAT with marketing team :f3, after f2, 5d
```

---

## 12. Monitoring & Observability

```mermaid
graph TB
    subgraph "Application Metrics"
        A[MetricsService<br/>Prometheus]
        B[New Relic APM<br/>Transaction Traces]
        C[Application Logs<br/>SLF4J]
    end
    
    subgraph "Message Processors"
        D[MessageExecutionProcessor]
        E[PromotionalMessageProcessor]
        F[TransactionalMessageProcessor]
        G[CreateMessageProcessor]
    end
    
    subgraph "Metrics Collected"
        H[Execution Time<br/>P90, P95, P99]
        I[Error Rate<br/>Success/Failure/Retry]
        J[Message Volume<br/>Throughput]
        K[Queue Depth<br/>Backlog]
    end
    
    subgraph "Alerting"
        L[Prometheus Alerts<br/>Latency, Errors]
        M[New Relic Alerts<br/>APM Violations]
        N[PagerDuty<br/>On-call Rotation]
    end
    
    D --> A
    E --> A
    F --> A
    G --> A
    
    D --> B
    E --> B
    F --> B
    G --> B
    
    D --> C
    E --> C
    F --> C
    G --> C
    
    A --> H
    A --> I
    B --> H
    B --> I
    C --> J
    C --> K
    
    H --> L
    I --> L
    L --> N
    M --> N
```

---

## How to Use These Diagrams

### Viewing in GitHub
All diagrams use **Mermaid syntax** and render automatically in GitHub README files. To view:
1. Open this file in GitHub web interface
2. Diagrams will render automatically
3. Or use [Mermaid Live Editor](https://mermaid.live) to edit

### Updating Diagrams
1. Edit the Mermaid code directly in this markdown file
2. Test changes in [Mermaid Live Editor](https://mermaid.live)
3. Commit changes to see them render in GitHub

### Exporting
- Use Mermaid Live Editor to export as PNG/SVG
- Use VS Code with Mermaid extension for local editing
- Use Confluence/Notion Mermaid plugins for documentation

---

**Document Version**: 1.0  
**Last Updated**: October 16, 2025  
**Maintained By**: Engineering Team

