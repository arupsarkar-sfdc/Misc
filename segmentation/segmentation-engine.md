# Salesforce Data Cloud: Segmentation Engine Architecture & Design Patterns

## Reference

This analysis uses design pattern definitions and concepts from:
- **Refactoring.Guru Design Patterns**: https://refactoring.guru/design-patterns
- Gang of Four (GoF) Design Patterns
- Modern Architectural Patterns
- Salesforce Data Cloud Segmentation Service Implementation

---

## Table of Contents

### [Part 1: How Segmentation Works](#part-1-how-segmentation-works)
1. [System Overview](#system-overview)
2. [Core Operations: Counting vs Materialization](#core-operations-counting-vs-materialization)
3. [End-to-End Flow](#end-to-end-flow)
4. [Consent, Subscriptions & Preferences Integration](#consent-subscriptions--preferences-integration)
5. [Data Enrichment & Mastering Processes](#data-enrichment--mastering-processes)
6. [Contact & Account Master (CAM) Architecture Integration](#contact--account-master-cam-architecture-integration)

### [Part 2: Design Patterns Implementation](#part-2-design-patterns-implementation)
**Pattern Coverage:** 12 out of 16 patterns (75% coverage)
- **Structural Patterns:** 4 out of 6 (67%)
- **Behavioral Patterns:** 6 out of 7 (86%)
- **Creational Patterns:** 1 out of 2 (50%)
- **Architectural Patterns:** 1 (Repository Pattern)

#### Structural Patterns
4. [Adapter Pattern](#4-adapter-pattern)
5. [Facade Pattern](#5-facade-pattern)
6. [Proxy Pattern](#6-proxy-pattern)
7. [Bridge Pattern](#7-bridge-pattern)

#### Behavioral Patterns
8. [Strategy Pattern](#8-strategy-pattern)
9. [Observer Pattern](#9-observer-pattern)
10. [Command Pattern](#10-command-pattern)
11. [Template Method Pattern](#11-template-method-pattern)
12. [Chain of Responsibility Pattern](#12-chain-of-responsibility-pattern)
13. [Mediator Pattern](#13-mediator-pattern)

#### Creational Patterns
14. [Factory Pattern](#14-factory-pattern)

#### Architectural Patterns
15. [Repository Pattern](#15-repository-pattern)

### [Part 3: Segmentation Product Lines](#part-3-segmentation-product-lines)
16. [Segmentation Engine in Data Cloud Ecosystem](#segmentation-engine-in-data-cloud-ecosystem)
17. [Conclusion](#conclusion)

---

<a id="part-1-how-segmentation-works"></a>
# Part 1: How Segmentation Works

The Segmentation Engine is built on the **Scone microservices framework** (Spring Boot) and orchestrates segment population calculation through two distinct operations: **Counting** (preview/estimate) and **Materialization** (activation-ready).

---

## System Overview

### System Components & Data Flow

```mermaid
graph TB
    subgraph "CDP (Customer Data Platform)"
        Metadata[🔢 Step 1<br/>Metadata Service<br/>Schema & Config]
        Admin[🔢 Step 2<br/>Admin Service<br/>Token Management]
        Auth[🔢 Step 3<br/>Auth Service<br/>User Authorization]
    end
    
    subgraph "Segmentation Engine (Scone-based)"
        REST[🔢 Step 4<br/>REST API<br/>Request Handler]
        JPA[🔢 Step 5<br/>JPA - Persistence<br/>Data Layer]
        JMS[🔢 Step 6<br/>JMS - Message Queue<br/>Async Processing]
        Kafka[🔢 Step 11<br/>KafkaListener<br/>Event Streaming]
        gRPC[🔢 Step 9<br/>gRPC<br/>Service Communication]
        Template[🔢 Step 8<br/>RestTemplate<br/>HTTP Client]
    end
    
    subgraph "AWS Services"
        RDS[🔢 Step 7<br/>RDS Database<br/>Persistent Storage]
        SQS[🔢 Step 10<br/>SQS Queue<br/>Job Processing]
    end
    
    subgraph "Query Execution"
        QueryEngine[🔢 Step 12<br/>Data Cloud Query Engine<br/>SQL Execution]
    end
    
    Core[Core Service] -->|1️⃣ REST Request| REST
    REST -->|2️⃣ Persist| JPA
    REST -->|3️⃣ Queue Job| JMS
    JPA -->|4️⃣ Store Data| RDS
    JMS -->|5️⃣ Send Message| SQS
    SQS -->|6️⃣ Process Jobs| Kafka
    Kafka -->|7️⃣ Query Execution| QueryEngine
    QueryEngine -->|8️⃣ Results| Kafka
    Template -->|🌐 HTTP Calls| QueryEngine
    Template -->|🔐 Get Token| Admin
    Kafka -->|9️⃣ Materialize Complete| KafkaOut[🔢 Step 13<br/>Kafka Topic<br/>Event Publishing]
    KafkaOut -->|🔟 Activation| ActEngine[🔢 Step 14<br/>Activation Engine<br/>Marketing Channels]
    
    %% Authentication & Config Flow
    Admin -->|🔐 Token| Template
    Metadata -->|📊 Schema| REST
    Auth -->|🔒 Authorize| REST
    gRPC -->|⚡ Fast Comms| QueryEngine
    
    style REST fill:#166534,color:#ffffff
    style QueryEngine fill:#92400e,color:#ffffff
```

### Step-by-Step Flow Analysis

**🔍 Request Processing Flow (Steps 1-6)**

| Step | Component | Action | Technical Details |
|------|-----------|--------|-------------------|
| **1** | **Metadata Service** | Schema validation | Validates segment definition against data model, checks field availability |
| **2** | **Admin Service** | Token management | Retrieves Data Cloud workspace token for query execution authentication |
| **3** | **Auth Service** | User authorization | Validates user permissions for segment operations and data access |
| **4** | **REST API** | Request handling | Processes segment CRUD operations (count/materialize requests) |
| **5** | **JPA Persistence** | Data operations | Stores segment definitions, job status, and metadata in RDS |
| **6** | **JMS Message Queue** | Async processing | Queues jobs for background processing via SQS |

**⚙️ Query Execution Flow (Steps 7-10)**

| Step | Component | Action | Technical Details |
|------|-----------|--------|-------------------|
| **7** | **RDS Database** | Persistent storage | Stores segment definitions, job history, and population counts |
| **8** | **RestTemplate** | HTTP communication | Makes authenticated calls to Data Cloud Query Engine for execution |
| **9** | **gRPC** | High-performance comms | Fast service-to-service communication for real-time operations |
| **10** | **SQS Queue** | Job processing | Manages async job queue for segment counting and materialization |

**📡 Event & Activation Flow (Steps 11-14)**

| Step | Component | Action | Technical Details |
|------|-----------|--------|-------------------|
| **11** | **KafkaListener** | Event streaming | Listens for query completion and materialization events |
| **12** | **Data Cloud Query Engine** | SQL execution | Executes segment queries against unified customer profiles |
| **13** | **Kafka Topic** | Event publishing | Publishes materialization completion events |
| **14** | **Activation Engine** | Marketing activation | Sends segments to Marketing Cloud, Google Ads, Facebook, LinkedIn, etc. |

---

## Core Operations: Counting vs Materialization

### Operation Comparison

| Aspect | Counting | Materialization |
|--------|----------|-----------------|
| **Purpose** | Preview/Estimate population | Activation-ready segment |
| **Response Time** | Immediate (202 Accepted) | Immediate (202 Accepted) |
| **Processing** | Async | Async |
| **Database Update** | Population count only | Full segment membership |
| **S3 Write** | ❌ No | ✅ Yes (full records) |
| **Activation Trigger** | ❌ No | ✅ Yes (via Kafka event) |
| **Use Case** | Testing segment criteria | Publishing segment for campaigns |
| **Caching** | ✅ Possible | Limited |

---

## End-to-End Flow

### Complete Segmentation Process Flow

```mermaid
sequenceDiagram
    participant U as 🔢 Step 1<br/>User/Core Service
    participant SE as 🔢 Step 2<br/>Segmentation Engine
    participant RDS as 🔢 Step 3<br/>RDS Database
    participant SQS as 🔢 Step 4<br/>SQS Queue
    participant K as 🔢 Step 5<br/>Kafka Listener
    participant D as 🔢 Step 6<br/>Data Cloud Query Engine
    participant AE as 🔢 Step 7<br/>Activation Engine
    
    U->>SE: 1️⃣ POST /segments/count
    SE->>RDS: 2️⃣ Store segment definition
    SE->>SQS: 3️⃣ Queue count job
    SE-->>U: 4️⃣ 202 Accepted (jobId)
    
    SQS->>K: 5️⃣ Process job message
    K->>D: 6️⃣ Execute SQL query
    D-->>K: 7️⃣ Return count results
    K->>RDS: 8️⃣ Update segment count
    
    Note over U,AE: For Materialization:
    U->>SE: 9️⃣ POST /segments/materialize
    SE->>SQS: 🔟 Queue materialize job
    K->>D: 1️⃣1️⃣ Execute full query
    D-->>K: 1️⃣2️⃣ Return full records
    K->>K: 1️⃣3️⃣ Write to S3
    K->>AE: 1️⃣4️⃣ Kafka event (segment ready)
    AE->>AE: 1️⃣5️⃣ Activate to channels
```

### Step-by-Step Process Details

**📊 Count Operation Flow (Steps 1-8)**

| Step | Action | Component | Result |
|------|--------|-----------|--------|
| **1** | User requests segment count | Core Service → REST API | HTTP POST with filter criteria |
| **2** | Store segment definition | Segmentation Engine → RDS | Persisted segment metadata |
| **3** | Queue count job | Segmentation Engine → SQS | Async job queued |
| **4** | Return job ID | REST API → User | 202 Accepted response |
| **5** | Process job message | SQS → Kafka Listener | Job dequeued for processing |
| **6** | Execute SQL query | Kafka Listener → Data Cloud Query Engine | COUNT(*) query execution |
| **7** | Return count results | Data Cloud Query Engine → Kafka Listener | Population count number |
| **8** | Update segment count | Kafka Listener → RDS | Count stored for retrieval |

**🚀 Materialization Flow (Steps 9-15)**

| Step | Action | Component | Result |
|------|--------|-----------|--------|
| **9** | User requests materialization | Core Service → REST API | HTTP POST for full segment |
| **10** | Queue materialize job | Segmentation Engine → SQS | Full processing job queued |
| **11** | Execute full query | Kafka Listener → Data Cloud Query Engine | SELECT * query execution |
| **12** | Return full records | Data Cloud Query Engine → Kafka Listener | Complete customer records |
| **13** | Write to S3 | Kafka Listener → S3 | Segment data file created |
| **14** | Publish ready event | Kafka Listener → Kafka Topic | Segment available for activation |
| **15** | Activate to channels | Activation Engine → Marketing Channels | Push to Marketing Cloud, Google Ads, etc. |

---

## Consent, Subscriptions & Preferences Integration

### Why Consent Matters in Segmentation

Segmentation without consent compliance creates **unmarketable segments**. A segment of 1M customers means nothing if only 100K are legally marketable due to consent status.

### Consent State Flow Integration

```mermaid
sequenceDiagram
    participant U as 🔢 Step 1<br/>User Form Submission
    participant CR as 🔢 Step 2<br/>Consent Rules Engine
    participant CS as 🔢 Step 3<br/>Consent Status Assignment
    participant SE as 🔢 Step 4<br/>Segmentation Engine
    participant MC as 🔢 Step 5<br/>Marketing Cloud
    
    U->>CR: 1️⃣ Submit form (country: Germany)
    CR->>CR: 2️⃣ Apply country rules (Opt-In required)
    CR->>CS: 3️⃣ Assign status: "Opt-Out Auto"
    CS->>SE: 4️⃣ Update marketability filter
    SE->>SE: 5️⃣ Apply consent filter to segments
    SE->>MC: 6️⃣ Only marketable profiles activated
    
    Note over U,MC: Germany = Opt-In Required
    Note over SE: Segment: 1M profiles → 200K marketable
```

### Marketability Processing Flow

```mermaid
flowchart TB
    subgraph "🌍 Country-Based Rules"
        OptOut[🔢 Step 1<br/>Opt-Out Countries<br/>🇺🇸 US, 🇦🇺 Australia<br/>Default: Marketable]
        OptIn[🔢 Step 2<br/>Opt-In Countries<br/>🇫🇷 France, 🇬🇧 UK<br/>Default: Not Marketable]
        OptInDouble[🔢 Step 3<br/>Double Opt-In<br/>🇩🇪 Germany<br/>Default: Requires Confirmation]
    end
    
    subgraph "🎯 Consent Status Assignment"
        ExplicitIn[🔢 Step 4<br/>Opt-In Explicit<br/>✅ Checkbox Checked<br/>All Countries]
        ImpliedIn[🔢 Step 5<br/>Opt-In Implied<br/>📝 Form Submitted<br/>Opt-Out Countries Only]
        AutoOut[🔢 Step 6<br/>Opt-Out Auto<br/>❌ No Consent Given<br/>Opt-In Countries]
        ExplicitOut[🔢 Step 7<br/>Opt-Out Explicit<br/>🚫 User Unsubscribed<br/>All Countries]
    end
    
    subgraph "📊 Segmentation Impact"
        Marketable[🔢 Step 8<br/>Marketable Audience<br/>✅ Explicit + Implied<br/>+ Activity < 12 months]
        NotMarketable[🔢 Step 9<br/>Non-Marketable<br/>❌ Auto + Explicit Out<br/>+ Inactive > 12 months]
    end
    
    %% Country to Status Flow
    OptOut -->|Form Submitted| ImpliedIn
    OptOut -->|Checkbox Checked| ExplicitIn
    OptIn -->|No Checkbox| AutoOut
    OptIn -->|Checkbox Checked| ExplicitIn
    OptInDouble -->|Email Confirmed| ExplicitIn
    
    %% Status to Marketability
    ExplicitIn -->|+ Recent Activity| Marketable
    ImpliedIn -->|+ Recent Activity| Marketable
    AutoOut --> NotMarketable
    ExplicitOut --> NotMarketable
    
    %% Activity Check
    ExplicitIn -->|Inactive 12+ months| NotMarketable
    ImpliedIn -->|Inactive 12+ months| NotMarketable
    
    style Marketable fill:#166534,color:#ffffff
    style NotMarketable fill:#991b1b,color:#ffffff
    style ExplicitIn fill:#166534,color:#ffffff
    style ImpliedIn fill:#065f46,color:#ffffff
    style AutoOut fill:#7f1d1d,color:#ffffff
    style ExplicitOut fill:#991b1b,color:#ffffff
```

### Segment Filtering with Consent

```mermaid
flowchart LR
    subgraph "🎯 Raw Segment"
        RawSeg[🔢 Step 1<br/>High-Value Customers<br/>1,000,000 profiles<br/>LTV > $1,000]
    end
    
    subgraph "🔍 Consent Filtering"
        ConsentFilter[🔢 Step 2<br/>Apply Consent Rules<br/>✅ Opt-In Explicit: 300K<br/>✅ Opt-In Implied: 500K<br/>❌ Opt-Out Auto: 150K<br/>❌ Opt-Out Explicit: 50K]
    end
    
    subgraph "⏰ Activity Filtering"
        ActivityFilter[🔢 Step 3<br/>Activity Check<br/>📧 Email opened < 12mo<br/>🌐 Web activity < 12mo<br/>💳 Purchase < 12mo]
    end
    
    subgraph "📤 Final Marketable"
        MarketableSeg[🔢 Step 4<br/>Marketable Segment<br/>220,000 profiles<br/>22% of original segment]
    end
    
    RawSeg -->|1️⃣ Apply Consent| ConsentFilter
    ConsentFilter -->|2️⃣ Check Activity| ActivityFilter
    ActivityFilter -->|3️⃣ Final Filter| MarketableSeg
    
    style RawSeg fill:#374151,color:#ffffff
    style ConsentFilter fill:#1e40af,color:#ffffff
    style ActivityFilter fill:#92400e,color:#ffffff
    style MarketableSeg fill:#166534,color:#ffffff
```

### Key Integration Points

**📊 Segmentation Query Enhancement:**
```sql
-- Original Segment Query
SELECT customer_id FROM unified_profiles 
WHERE lifetime_value > 1000

-- Consent-Enhanced Query
SELECT up.customer_id 
FROM unified_profiles up
JOIN consent_status cs ON up.customer_id = cs.customer_id
WHERE up.lifetime_value > 1000
  AND cs.consent_status IN ('opt_in_explicit', 'opt_in_implied')
  AND cs.last_activity_date > DATE_SUB(NOW(), INTERVAL 12 MONTH)
```

**🎯 Real-World Impact:**
- **Total Database**: ~140M subscribers
- **Truly Marketable**: ~12.2M (8.7%)
- **Segment Effectiveness**: Consent filtering can reduce segment size by 70-90%

---

<a id="part-2-design-patterns-implementation"></a>
# Part 2: Design Patterns Implementation

---

## 4. ADAPTER PATTERN

### Why Adapter Pattern?

**Business Problem:** Data Cloud segments are defined using business-friendly JSON criteria, but query engines expect SQL. Different data sources (Snowflake, Redshift, Databricks) have varying SQL dialects and connection methods.

**Technical Challenge:** 
- Business users define segments as: `"customers where LTV > 1000 AND churn_risk = high"`
- Query engines need: `SELECT customer_id FROM unified_profiles WHERE lifetime_value > 1000 AND churn_probability > 0.8`
- Each data warehouse has different table schemas and SQL syntax

**Solution:** Adapter Pattern provides translation layers that convert between incompatible interfaces without changing existing code.

### Where Applied
- **Query_Builder**: Translates segment filter criteria to SQL
- **Data Cloud Query Client**: Adapts internal requests to Data Cloud Query API
- **RDS/SQS/MDS Integration**: Adapts to AWS services

### Real Implementation

**Segment Filter (Business Logic):**
```json
{
  "segmentId": "SEG-HIGH-VALUE-001",
  "name": "High-Value At-Risk Customers",
  "criteria": {
    "operator": "AND",
    "conditions": [
      {
        "field": "LifetimeValue__c",
        "operator": ">",
        "value": 100000
      },
      {
        "field": "ChurnProbability__c",
        "operator": ">",
        "value": 0.7
      },
      {
        "field": "Industry__c",
        "operator": "IN",
        "values": ["Technology", "Finance"]
      }
    ]
  }
}
```

**Query_Builder Adapter Output (SQL for Data Cloud Query Engine):**
```sql
SELECT 
    IndividualId,
    FirstName,
    LastName,
    Email,
    LifetimeValue__c,
    ChurnProbability__c,
    Industry__c,
    AccountName
FROM UnifiedIndividual__dmo
WHERE LifetimeValue__c > 100000
  AND ChurnProbability__c > 0.7
  AND Industry__c IN ('Technology', 'Finance')
```

### Diagram

```mermaid
graph LR
    Filter[🔢 Step 1<br/>Segment Filter<br/>Business Rules] -->|1️⃣ Adapter Transform| QB[🔢 Step 2<br/>Query_Builder<br/>Translation Layer]
    QB -->|2️⃣ Generates| SQL[🔢 Step 3<br/>SQL Query<br/>Data Cloud Format]
    SQL -->|3️⃣ Executes| QueryEngine[🔢 Step 4<br/>Data Cloud Query Engine<br/>Query Processor]
    QueryEngine -->|4️⃣ Returns| Results[🔢 Step 5<br/>Query Results<br/>Customer Records]
    
    style QB fill:#c026d3,stroke:#ffffff,stroke-width:4px,color:#ffffff
```

**Benefits:**
- Business users define criteria without writing SQL
- Single source of truth for query generation
- Can switch query engines without changing filter format

---

## 5. FACADE PATTERN

### Why Facade Pattern?

**Business Problem:** Segment creation involves 10+ complex steps across multiple services (Auth, MDS, Query Engine, RDS, SQS, Kafka). Core Service needs simple operations like "count segment" or "materialize segment" without understanding the complexity.

**Technical Challenge:**
- Each segment operation requires orchestrating multiple microservices
- Different error handling for each service integration
- Complex state management during async processing
- Authentication, authorization, and resource management

**Solution:** Facade Pattern provides a single, simplified interface that hides complex subsystem interactions behind clean API endpoints.

### Where Applied
- **Segmentation_Engine REST API**: Simplified interface to complex operations
- **Core Service Integration**: Single entry point for segment operations

### Real Implementation

**Complex Internal Operations Hidden:**
```
Behind the Facade:
1. Validate segment definition
2. Check permissions (Auth Service)
3. Get partition attributes (MDS)
4. Generate SQL (Query_Builder)
5. Get Data Cloud token (Admin Service)
6. Execute query (Data Cloud Query Engine)
7. Process results
8. Update database (RDS)
9. Write to S3 (materialization)
10. Publish event (Kafka)
```

**Simple External Interface:**
```bash
# Core Service makes simple REST call
POST /api/v1/segments/{segmentId}/materialize
Authorization: Bearer {token}

Response: 202 Accepted
{
  "jobId": "JOB-98765",
  "segmentId": "SEG-HIGH-VALUE-001",
  "status": "PROCESSING",
  "estimatedCompletion": "2025-10-23T14:35:00Z"
}
```

### Sequence Flow (Facade Simplifies Complexity)

```mermaid
sequenceDiagram
    participant Core
    participant Facade as Segmentation Engine<br/>(FACADE)
    participant Internal as Internal Components<br/>(Hidden Complexity)
    
    Core->>Facade: POST /segments/{id}/materialize
    Facade->>Core: 202 Accepted (immediate)
    
    Note over Facade,Internal: Complex processing hidden from Core
    
    Facade->>Internal: 1. Validate segment
    Facade->>Internal: 2. Get partition attributes
    Facade->>Internal: 3. Generate SQL
    Facade->>Internal: 4. Get Data Cloud token
    Facade->>Internal: 5. Execute query
    Facade->>Internal: 6. Write to S3
    Facade->>Internal: 7. Publish Kafka event
    
    Internal->>Facade: All steps complete
    
    Note over Core: Core doesn't know about<br/>10+ internal steps
```

---

## 6. PROXY PATTERN

### Why Proxy Pattern?

**Business Problem:** Direct access to Data Cloud Query Engine would expose authentication tokens, create security vulnerabilities, and lack proper monitoring. Each service would need to implement its own authentication, rate limiting, and error handling.

**Technical Challenge:**
- Data Cloud requires workspace-specific authentication tokens
- Need centralized monitoring and logging of all query operations
- Rate limiting to prevent query engine overload
- Circuit breaking for resilience during Data Cloud outages
- Consistent error handling and retry logic

**Solution:** Proxy Pattern controls access to Data Cloud Query Engine, adding authentication, monitoring, rate limiting, and resilience without changing the query interface.

### Where Applied
- **RestTemplate to Data Cloud**: Acts as proxy for query execution
- **Admin_Service Token Retrieval**: Proxies authentication
- **Data Cloud Workspace Access**: Controls and monitors access

### Real Implementation

**Without Proxy (Direct Access - Problematic):**
```
Segmentation Engine → Direct Data Cloud Query
Problems:
- No authentication management
- No retry logic
- No circuit breaking
- No monitoring
- Hard to switch query engines
```

**With Proxy Pattern:**

```mermaid
graph TB
    SegEngine[Segmentation Engine] -->|Request| Proxy[Datorama Proxy<br/>RestTemplate]
    Proxy -->|1. Get Token| Admin[Admin Service]
    Admin -->|2. Token| Proxy
    Proxy -->|3. Auth Query| Datorama[Datorama Engine]
    Datorama -->|4. Results| Proxy
    Proxy -->|5. Filtered| SegEngine
    
    Proxy -->|Log| Monitor[Monitoring/Splunk]
    
    style Proxy fill:#ec4899,stroke:#ffffff,stroke-width:4px,color:#ffffff
```

**Real Data Example:**

**Step 1: Get Datorama Workspace Token**
```json
Request to Admin_Service:
GET /api/v1/workspace/token
{
  "workspaceId": "WS-12345",
  "userId": "segmentation-engine-sa"
}

Response:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresAt": "2025-10-23T15:00:00Z"
}
```

**Step 2: Execute Query via Proxy**
```json
Request to Datorama (with token):
POST /api/v1/query
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
{
  "sql": "SELECT IndividualId, FirstName, LastName, Email...",
  "workspaceId": "WS-12345",
  "jobId": "JOB-98765"
}

Response:
{
  "queryId": "QUERY-55501",
  "status": "COMPLETED",
  "recordCount": 1247,
  "executionTime": "2.3s"
}
```

**Benefits:**
- Centralized authentication management
- Query monitoring and logging
- Retry and circuit breaker logic
- Can cache results for counting operations

---

## 7. BRIDGE PATTERN

### Why Bridge Pattern?

**Business Problem:** Segment definitions must work across different data warehouses (Snowflake, Redshift, Databricks) with varying SQL dialects, table schemas, and optimization strategies. The business logic should remain stable regardless of the underlying query implementation.

**Technical Challenge:**
- Same segment criteria needs different SQL for different warehouses
- Query optimization strategies vary by platform
- Schema differences between data sources
- Need to add new data warehouses without changing business logic

**Solution:** Bridge Pattern separates the abstraction (segment criteria) from the implementation (SQL generation), allowing both to vary independently.

### Where Applied
- **Segment Definition (Abstraction) ↔ Query Execution (Implementation)**
- **Query_Builder bridges business logic to SQL**

### Real Implementation

**Abstraction: Segment Criteria**
```json
{
  "segmentId": "SEG-HIGH-VALUE-001",
  "criteria": {
    "operator": "AND",
    "conditions": [...]
  }
}
```

**Implementation A: SQL for Datorama**
```sql
SELECT IndividualId, ... FROM UnifiedIndividual__dmo WHERE ...
```

**Implementation B: Could support other query engines**
```
Future: Switch to Snowflake, Data Cloud native query, etc.
Bridge pattern allows changing implementation without changing segment definition
```

---

## 8. STRATEGY PATTERN

### Why Strategy Pattern?

**Business Problem:** Segment processing has fundamentally different requirements based on use case:
- **Count**: Fast preview for testing (seconds)
- **Materialization**: Full data export for activation (minutes)
- **Scheduled**: Batch processing during off-peak hours
- **On-Demand**: Real-time processing for immediate campaigns

**Technical Challenge:**
- Each strategy has different performance requirements
- Different caching policies (count results can be cached, materialization cannot)
- Different resource allocation and queue priorities
- Different S3 write patterns and activation triggers

**Solution:** Strategy Pattern encapsulates different algorithms for segment processing, allowing runtime selection based on business requirements.

### Where Applied
- **Counting vs Materialization**: Different processing strategies
- **Query Caching Strategy**: Cached vs Fresh queries
- **Scheduled vs On-Demand**: Different execution strategies

### Real Implementation

**Strategy Context: Segment Processing**

```mermaid
graph TB
    Context[Segment Processing Context] -->|Choose Strategy| Interface[Processing Strategy]
    
    Interface -->|Strategy 1| Count[Count Strategy<br/>Fast, No Activation]
    Interface -->|Strategy 2| Materialize[Materialize Strategy<br/>Full, With Activation]
    Interface -->|Strategy 3| Scheduled[Scheduled Strategy<br/>Periodic Refresh]
    
    Count -->|Output| DB1[Update RDS<br/>Population Count]
    Materialize -->|Output| Full[Write S3 + Kafka Event]
    Scheduled -->|Output| Refresh[Periodic S3 + Kafka]
    
    style Interface fill:#ea580c,stroke:#ffffff,stroke-width:4px,color:#ffffff
```

### Strategy 1: Count (Preview/Estimate)

**Purpose:** Quick population estimate for testing segment criteria

**Real Data Flow:**
```json
// Step 1: Core Request
POST /api/v1/segments/SEG-HIGH-VALUE-001/count
{
  "userId": "user@acme.com"
}

// Step 2: Generate SQL
{
  "operation": "count",
  "sql": "SELECT IndividualId FROM UnifiedIndividual__dmo WHERE LifetimeValue__c > 100000 AND ChurnProbability__c > 0.7"
}

// Step 3: Datorama Execution
{
  "queryType": "included_population",
  "result": {
    "count": 1247,
    "executionTime": "1.8s"
  }
}

// Step 4: Update RDS
UPDATE segments SET population_count = 1247, last_counted_at = NOW() WHERE segment_id = 'SEG-HIGH-VALUE-001';

// Step 5: Response to Core
{
  "segmentId": "SEG-HIGH-VALUE-001",
  "populationCount": 1247,
  "totalPopulation": 125000,
  "percentage": 1.0,
  "status": "COUNTED"
}

// NO S3 write
// NO Kafka event
// NO activation
```

### Strategy 2: Materialize (Activation-Ready)

**Purpose:** Full segment execution for campaign activation

**Real Data Flow:**
```json
// Step 1: Core Request
POST /api/v1/segments/SEG-HIGH-VALUE-001/materialize
{
  "userId": "user@acme.com",
  "activationTargets": ["marketing_cloud", "google_ads"]
}

// Step 2: Generate SQL with jobId
{
  "operation": "materialize",
  "jobId": "JOB-98765",
  "sql": "SELECT IndividualId, FirstName, LastName, Email, LifetimeValue__c, ChurnProbability__c, Industry__c, AccountName FROM UnifiedIndividual__dmo WHERE LifetimeValue__c > 100000 AND ChurnProbability__c > 0.7"
}

// Step 3: Create Datorama Data Stream
{
  "streamId": "STREAM-98765",
  "jobId": "JOB-98765",
  "sql": "...",
  "outputFormat": "parquet",
  "compression": "snappy"
}

// Step 4: Datorama Processes & Returns Data Stream
{
  "streamId": "STREAM-98765",
  "status": "COMPLETED",
  "recordCount": 1247,
  "dataLocation": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
  "executionTime": "3.2s"
}

// Step 5: Write Full Records to S3
S3 Path: s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/part-00000.parquet

Sample Records:
[
  {
    "IndividualId": "IND-UNIFIED-001",
    "FirstName": "Jane",
    "LastName": "Austin",
    "Email": "jane.austin@acme.com",
    "LifetimeValue__c": 135000.00,
    "ChurnProbability__c": 0.78,
    "Industry__c": "Technology",
    "AccountName": "Acme Corporation"
  },
  {
    "IndividualId": "IND-UNIFIED-002",
    "FirstName": "John",
    "LastName": "Smith",
    "Email": "john.smith@techcorp.com",
    "LifetimeValue__c": 145000.00,
    "ChurnProbability__c": 0.82,
    "Industry__c": "Finance",
    "AccountName": "TechCorp Inc"
  }
  // ... 1245 more records
]

// Step 6: Update RDS
UPDATE segments 
SET population_count = 1247, 
    last_materialized_at = NOW(), 
    s3_location = 's3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/',
    status = 'MATERIALIZED'
WHERE segment_id = 'SEG-HIGH-VALUE-001';

// Step 7: Publish Kafka Event
{
  "eventType": "SegmentMaterializationComplete",
  "timestamp": "2025-10-23T14:35:00Z",
  "segmentId": "SEG-HIGH-VALUE-001",
  "jobId": "JOB-98765",
  "populationCount": 1247,
  "s3Location": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
  "activationTargets": ["marketing_cloud", "google_ads"]
}

// Step 8: Activation Engine Consumes Kafka Event
// → Triggers: Marketing Cloud journey enrollment
// → Triggers: Google Ads audience upload
// → Triggers: Facebook Custom Audience sync
```

### Strategy 3: Scheduled Materialize

**Purpose:** Periodic refresh of segment membership

**Real Data Flow:**
```json
// Step 1: Scheduler Scans for Due Segments
{
  "scanTime": "2025-10-23T02:00:00Z",
  "segmentsDue": [
    {
      "segmentId": "SEG-HIGH-VALUE-001",
      "refreshFrequency": "daily",
      "lastMaterialized": "2025-10-22T02:00:00Z",
      "nextDue": "2025-10-23T02:00:00Z"
    }
  ]
}

// Step 2: Enqueue Materialize Event to SQS
{
  "eventType": "ScheduledMaterialize",
  "segmentId": "SEG-HIGH-VALUE-001",
  "triggeredBy": "scheduler",
  "priority": "normal"
}

// Step 3-8: Same as Materialize Strategy
// ... follows exact same flow as on-demand materialization
```

**Strategy Selection Logic:**
```java
public interface SegmentProcessingStrategy {
    ProcessingResult execute(Segment segment);
}

public class CountStrategy implements SegmentProcessingStrategy {
    public ProcessingResult execute(Segment segment) {
        // Fast count only
        String sql = queryBuilder.generateCountSQL(segment);
        int count = datorama.executeCount(sql);
        return new ProcessingResult(count, null, false);
    }
}

public class MaterializeStrategy implements SegmentProcessingStrategy {
    public ProcessingResult execute(Segment segment) {
        // Full materialization
        String sql = queryBuilder.generateFullSQL(segment);
        DataStream stream = datorama.executeStream(sql);
        String s3Location = s3Writer.write(stream);
        kafka.publish(new MaterializationCompleteEvent(segment, s3Location));
        return new ProcessingResult(stream.recordCount(), s3Location, true);
    }
}

// Context selects strategy
public class SegmentProcessor {
    public ProcessingResult process(Segment segment, OperationType operation) {
        SegmentProcessingStrategy strategy;
        
        if (operation == OperationType.COUNT) {
            strategy = new CountStrategy();
        } else if (operation == OperationType.MATERIALIZE) {
            strategy = new MaterializeStrategy();
        } else if (operation == OperationType.SCHEDULED_MATERIALIZE) {
            strategy = new MaterializeStrategy(); // Same as on-demand
        }
        
        return strategy.execute(segment);
    }
}
```

---

## 9. OBSERVER PATTERN

### Why Observer Pattern?

**Business Problem:** When a segment is materialized, multiple downstream systems need to be notified:
- **Marketing Cloud**: For email campaigns
- **Google Ads**: For digital advertising
- **Facebook Ads**: For social media targeting
- **Analytics Dashboard**: For reporting updates
- **Data Quality Service**: For monitoring segment health

**Technical Challenge:**
- Tight coupling if Segmentation Engine directly calls each system
- Different systems have different availability and processing speeds
- Need to add new activation channels without modifying core segmentation logic
- Failure in one system shouldn't block others

**Solution:** Observer Pattern decouples the Segmentation Engine from activation systems using Kafka events, allowing multiple subscribers to react independently.

### Where Applied
- **Kafka Event Publication**: Materialization completion triggers multiple observers
- **Activation Engine**: Observes segment completion events

### Real Implementation

**Subject: Segmentation Engine**
**Observers: Activation Targets**

```mermaid
sequenceDiagram
    participant Seg as Segmentation Engine<br/>(SUBJECT)
    participant Kafka
    participant Obs1 as Marketing Cloud<br/>(OBSERVER 1)
    participant Obs2 as Google Ads<br/>(OBSERVER 2)
    participant Obs3 as Facebook Ads<br/>(OBSERVER 3)
    participant Obs4 as LinkedIn Ads<br/>(OBSERVER 4)
    
    Seg->>Kafka: Publish: SegmentMaterializationComplete
    
    Note over Kafka: Event distributed to all observers
    
    Kafka->>Obs1: Segment SEG-HIGH-VALUE-001 ready
    Kafka->>Obs2: Segment SEG-HIGH-VALUE-001 ready
    Kafka->>Obs3: Segment SEG-HIGH-VALUE-001 ready
    Kafka->>Obs4: Segment SEG-HIGH-VALUE-001 ready
    
    Obs1->>Obs1: Enroll 1247 contacts in VIP journey
    Obs2->>Obs2: Upload audience (1247 matched)
    Obs3->>Obs3: Create custom audience
    Obs4->>Obs4: Sync matched company list
    
    Note over Obs1,Obs4: All observers process independently
```

**Real Kafka Event:**
```json
{
  "eventType": "SegmentMaterializationComplete",
  "eventId": "EVT-55501",
  "timestamp": "2025-10-23T14:35:00Z",
  "segmentId": "SEG-HIGH-VALUE-001",
  "segmentName": "High-Value At-Risk Customers",
  "jobId": "JOB-98765",
  "populationCount": 1247,
  "s3Location": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
  "fileFormat": "parquet",
  "compression": "snappy",
  "activationTargets": [
    "marketing_cloud",
    "google_ads",
    "facebook_ads",
    "linkedin_ads"
  ],
  "metadata": {
    "orgId": "00D5e000000abcd",
    "userId": "user@acme.com",
    "materializedAt": "2025-10-23T14:35:00Z"
  }
}
```

**Observer Reactions:**

**Observer 1: Marketing Cloud Activation**
```json
{
  "observerId": "marketing-cloud-activator",
  "action": "EnrollInJourney",
  "input": {
    "s3Location": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
    "segmentId": "SEG-HIGH-VALUE-001"
  },
  "processing": {
    "recordsRead": 1247,
    "recordsMatched": 1247,
    "recordsEnrolled": 1247
  },
  "output": {
    "journeyId": "VIP-RETENTION",
    "journeyName": "VIP Customer Retention",
    "enrollmentStatus": "SUCCESS",
    "enrolledAt": "2025-10-23T14:36:30Z"
  }
}
```

**Observer 2: Google Ads Activation**
```json
{
  "observerId": "google-ads-activator",
  "action": "UploadAudience",
  "input": {
    "s3Location": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
    "segmentId": "SEG-HIGH-VALUE-001"
  },
  "processing": {
    "recordsRead": 1247,
    "emailsHashed": 1247,
    "matchRate": 0.89
  },
  "output": {
    "audienceId": "12345678",
    "audienceName": "High-Value At-Risk Customers",
    "matchedUsers": 1110,
    "uploadStatus": "SUCCESS",
    "uploadedAt": "2025-10-23T14:37:15Z"
  }
}
```

---

## 10. COMMAND PATTERN

### Why Command Pattern?

**Business Problem:** Segment processing requests need to be queued, retried, logged, and potentially undone. Operations like "count segment" or "materialize segment" should be executable at different times with full context preservation.

**Technical Challenge:**
- Async processing requires preserving request context
- Need retry logic for failed operations
- Audit trail for all segment operations
- Ability to cancel long-running materialization jobs
- Priority-based queue processing

**Solution:** Command Pattern encapsulates segment requests as objects, enabling queuing, retry, undo, and logging without coupling the request creation from execution.

### Where Applied
- **Count Segment Request**: Encapsulated as command
- **Materialize Segment Request**: Encapsulated as command
- **SQS Message Queue**: Commands queued for async execution

### Real Implementation

**Command Objects:**

```java
public interface SegmentCommand {
    void execute();
    void undo();
    CommandResult getResult();
}

// Command 1: Count Segment
public class CountSegmentCommand implements SegmentCommand {
    private Segment segment;
    private QueryBuilder queryBuilder;
    private DatoramaService datorama;
    private SegmentRepository repository;
    
    public void execute() {
        String sql = queryBuilder.generateCountSQL(segment);
        int count = datorama.executeCount(sql);
        repository.updatePopulation(segment.getId(), count);
    }
    
    public void undo() {
        // Rollback count update
        repository.revertPopulationUpdate(segment.getId());
    }
}

// Command 2: Materialize Segment
public class MaterializeSegmentCommand implements SegmentCommand {
    private Segment segment;
    private String jobId;
    private QueryBuilder queryBuilder;
    private DatoramaService datorama;
    private S3Writer s3Writer;
    private KafkaProducer kafka;
    
    public void execute() {
        String sql = queryBuilder.generateFullSQL(segment, jobId);
        DataStream stream = datorama.executeStream(sql);
        String s3Location = s3Writer.write(stream);
        kafka.publish(new MaterializationCompleteEvent(segment, s3Location));
    }
    
    public void undo() {
        // Compensating transaction
        s3Writer.delete(s3Location);
        kafka.publish(new MaterializationRolledBackEvent(segment));
    }
}
```

**Command Invoker (SQS Queue):**

```json
// Command queued in SQS
{
  "messageId": "MSG-12345",
  "commandType": "MaterializeSegment",
  "payload": {
    "segmentId": "SEG-HIGH-VALUE-001",
    "jobId": "JOB-98765",
    "requestedBy": "user@acme.com",
    "timestamp": "2025-10-23T14:30:00Z"
  },
  "retryCount": 0,
  "maxRetries": 3
}
```

---

## 11. TEMPLATE METHOD PATTERN

### Why Template Method Pattern?

**Business Problem:** All segment processing follows the same fundamental workflow (validate → authenticate → query → process → store → notify), but counting and materialization have different implementations for some steps.

**Technical Challenge:**
- Common workflow steps should be reused to ensure consistency
- Specific steps need different implementations (counting vs materialization)
- New processing types should follow established patterns
- Audit and logging should be consistent across all processing types

**Solution:** Template Method Pattern defines the skeleton of segment processing workflow, allowing subclasses to override specific steps while maintaining overall structure.

### Where Applied
- **Segment Processing Pipeline**: Standard workflow with customizable steps

### Real Implementation

**Template: Standard Segment Processing**

```java
public abstract class SegmentProcessor {
    
    // Template Method - defines skeleton
    public final ProcessingResult process(Segment segment) {
        // Step 1: Validate (hook - can be overridden)
        if (shouldValidate()) {
            validate(segment);
        }
        
        // Step 2: Get Partition Attributes (concrete)
        PartitionAttributes partitions = getPartitionAttributes(segment);
        
        // Step 3: Generate SQL (abstract - must implement)
        String sql = generateSQL(segment, partitions);
        
        // Step 4: Get Datorama Token (concrete)
        String token = getDatoramaToken();
        
        // Step 5: Execute Query (abstract - must implement)
        QueryResult result = executeQuery(sql, token);
        
        // Step 6: Process Result (abstract - must implement)
        ProcessedData data = processResult(result);
        
        // Step 7: Persist (concrete)
        persist(data);
        
        // Step 8: Notify (hook - can be overridden)
        if (shouldNotify()) {
            notify(data);
        }
        
        return new ProcessingResult(data);
    }
    
    // Concrete methods (same for all)
    protected final PartitionAttributes getPartitionAttributes(Segment segment) {
        return mdsService.getPartitionAttributes(segment);
    }
    
    protected final String getDatoramaToken() {
        return adminService.getWorkspaceToken();
    }
    
    protected final void persist(ProcessedData data) {
        repository.save(data);
    }
    
    // Abstract methods (subclasses must implement)
    protected abstract String generateSQL(Segment segment, PartitionAttributes partitions);
    protected abstract QueryResult executeQuery(String sql, String token);
    protected abstract ProcessedData processResult(QueryResult result);
    
    // Hook methods (optional override)
    protected boolean shouldValidate() { return true; }
    protected void validate(Segment segment) { /* default validation */ }
    protected boolean shouldNotify() { return false; }
    protected void notify(ProcessedData data) { /* default: no-op */ }
}
```

**Concrete Implementation 1: Count Processor**

```java
public class CountProcessor extends SegmentProcessor {
    
    @Override
    protected String generateSQL(Segment segment, PartitionAttributes partitions) {
        return "SELECT COUNT(DISTINCT IndividualId) FROM " + 
               segment.getSourceObject() + " WHERE " + segment.getCriteria();
    }
    
    @Override
    protected QueryResult executeQuery(String sql, String token) {
        return datorama.executeCountQuery(sql, token);
    }
    
    @Override
    protected ProcessedData processResult(QueryResult result) {
        return new ProcessedData(result.getCount(), null);
    }
    
    @Override
    protected boolean shouldNotify() {
        return false; // No notification for counts
    }
}
```

**Concrete Implementation 2: Materialize Processor**

```java
public class MaterializeProcessor extends SegmentProcessor {
    
    @Override
    protected String generateSQL(Segment segment, PartitionAttributes partitions) {
        return "SELECT IndividualId, FirstName, LastName, Email, " +
               segment.getFields() + " FROM " + segment.getSourceObject() + 
               " WHERE " + segment.getCriteria();
    }
    
    @Override
    protected QueryResult executeQuery(String sql, String token) {
        return datorama.executeStreamQuery(sql, token, jobId);
    }
    
    @Override
    protected ProcessedData processResult(QueryResult result) {
        String s3Location = s3Writer.write(result.getDataStream());
        return new ProcessedData(result.getCount(), s3Location);
    }
    
    @Override
    protected boolean shouldNotify() {
        return true; // Publish Kafka event
    }
    
    @Override
    protected void notify(ProcessedData data) {
        kafka.publish(new MaterializationCompleteEvent(segment, data.getS3Location()));
    }
}
```

---

## 12. CHAIN OF RESPONSIBILITY PATTERN

### Why Chain of Responsibility Pattern?

**Business Problem:** Segment requests must pass through multiple validation and processing steps:
- **Authentication**: Valid user token
- **Schema Validation**: Proper segment criteria format
- **Rate Limiting**: User hasn't exceeded query limits
- **Permissions**: User can access specified data fields
- **Resource Availability**: System has capacity for processing

**Technical Challenge:**
- Each validation has different logic and dependencies
- Order of validation matters (don't check permissions before authentication)
- Some validations are optional or configurable
- Need to add new validation steps without modifying existing code

**Solution:** Chain of Responsibility Pattern passes requests through a chain of handlers, where each handler processes what it can and passes the request along.

### Where Applied
- **Segment Validation Pipeline**
- **Request Processing Chain**

### Real Implementation

**Chain: Segment Request Validation**

```mermaid
graph LR
    Request[Segment Request] --> H1[Auth Handler]
    H1 -->|Pass| H2[Validation Handler]
    H2 -->|Pass| H3[Rate Limit Handler]
    H3 -->|Pass| H4[Permission Handler]
    H4 -->|Pass| Process[Process Request]
    
    H1 -->|Fail| Reject[Reject: Unauthorized]
    H2 -->|Fail| Reject2[Reject: Invalid]
    H3 -->|Fail| Reject3[Reject: Rate Limited]
    H4 -->|Fail| Reject4[Reject: No Permission]
    
    style H1 fill:#166534,color:#ffffff
    style H2 fill:#166534,color:#ffffff
    style H3 fill:#166534,color:#ffffff
    style H4 fill:#166534,color:#ffffff
```

**Real Data Example:**

```json
// Incoming Request
{
  "segmentId": "SEG-HIGH-VALUE-001",
  "operation": "materialize",
  "userId": "user@acme.com",
  "orgId": "00D5e000000abcd"
}

// Handler 1: Auth Validation
{
  "handler": "AuthHandler",
  "check": "ValidateToken",
  "result": "PASS",
  "userId": "user@acme.com",
  "roles": ["SegmentAdmin", "MarketingUser"]
}

// Handler 2: Segment Validation
{
  "handler": "ValidationHandler",
  "checks": [
    {"check": "SegmentExists", "result": "PASS"},
    {"check": "SegmentNotDisabled", "result": "PASS"},
    {"check": "CriteriaValid", "result": "PASS"},
    {"check": "NoCircularDependency", "result": "PASS"}
  ],
  "result": "PASS"
}

// Handler 3: Rate Limit Check
{
  "handler": "RateLimitHandler",
  "check": "RequestRateLimit",
  "result": "PASS",
  "currentRate": 15,
  "limit": 100,
  "window": "1 hour"
}

// Handler 4: Permission Check
{
  "handler": "PermissionHandler",
  "checks": [
    {"check": "CanReadSegment", "result": "PASS"},
    {"check": "CanMaterializeSegment", "result": "PASS"},
    {"check": "CanAccessOrgData", "result": "PASS"}
  ],
  "result": "PASS"
}

// All handlers passed → Process request
{
  "status": "ACCEPTED",
  "jobId": "JOB-98765",
  "message": "Materialization job queued"
}
```

---

## 13. MEDIATOR PATTERN

### Why Mediator Pattern?

**Business Problem:** Core Service, Data Cloud Query Engine, RDS, SQS, and Kafka all need to interact, but direct communication creates tight coupling and complex dependencies. Each service would need to know about all others.

**Technical Challenge:**
- Core Service shouldn't directly manage query engine authentication
- Query results need coordinated storage in both RDS and S3
- Kafka events need correlation with database records
- Error handling requires coordination across multiple systems

**Solution:** Mediator Pattern centralizes complex communications and control logic in the Segmentation Engine, reducing dependencies between components.

### Where Applied
- **Segmentation_Engine**: Central mediator between components
- **Coordinates**: Core ↔ Data Cloud Query Engine ↔ RDS ↔ SQS ↔ Kafka

### Real Implementation

**Without Mediator:**
```
Core → Direct to Datorama (coupling)
Core → Direct to RDS (coupling)
Core → Direct to Kafka (coupling)
Result: Tight coupling, complex interactions
```

**With Mediator (Segmentation Engine):**

```mermaid
graph TB
    Core[Core Service] -->|Single Interface| Med[Segmentation Engine<br/>MEDIATOR]
    
    Med -->|Coordinates| Datorama[Datorama]
    Med -->|Coordinates| RDS[RDS]
    Med -->|Coordinates| SQS[SQS]
    Med -->|Coordinates| Kafka[Kafka]
    Med -->|Coordinates| Admin[Admin Service]
    Med -->|Coordinates| MDS[MDS]
    
    style Med fill:#92400e,stroke:#ffffff,stroke-width:4px,color:#ffffff
```

**Benefits:**
- Core doesn't need to know about Datorama, SQS, Kafka
- Segmentation Engine handles all coordination
- Reduced complexity from O(n²) to O(n)

---

## 14. FACTORY PATTERN

### Why Factory Pattern?

**Business Problem:** Segment operations require different types of queries and commands:
- **Count Queries**: Simple COUNT(*) for population estimates
- **Materialization Queries**: Full SELECT with joins for complete records
- **Streaming Queries**: Incremental processing for large segments
- **Cached Queries**: Optimized versions for repeated operations

**Technical Challenge:**
- Complex logic to determine appropriate query type based on segment criteria
- Different optimization strategies for different operation types
- Query construction involves multiple parameters and validations
- Need consistent object creation patterns

**Solution:** Factory Pattern encapsulates the creation logic for different query and command types, ensuring appropriate objects are created based on operation context.

### Where Applied
- **Query Generation**: Creates different query types based on operation
- **Command Creation**: Creates appropriate command objects

### Real Implementation

```java
public class QueryFactory {
    
    public Query createQuery(Segment segment, OperationType operation) {
        switch(operation) {
            case COUNT:
                return createCountQuery(segment);
            
            case MATERIALIZE:
                return createMaterializeQuery(segment);
            
            case SCHEDULED_MATERIALIZE:
                return createScheduledMaterializeQuery(segment);
            
            default:
                throw new IllegalArgumentException("Unknown operation: " + operation);
        }
    }
    
    private Query createCountQuery(Segment segment) {
        return new CountQuery.Builder()
            .select("COUNT(DISTINCT IndividualId)")
            .from(segment.getSourceObject())
            .where(segment.getCriteria())
            .build();
    }
    
    private Query createMaterializeQuery(Segment segment) {
        return new MaterializeQuery.Builder()
            .select(segment.getAllFields())
            .from(segment.getSourceObject())
            .where(segment.getCriteria())
            .withJobId(generateJobId())
            .build();
    }
}
```

---

## 15. REPOSITORY PATTERN

### Why Repository Pattern?

**Business Problem:** Segment data needs to be persisted and retrieved from RDS, but business logic shouldn't be coupled to specific database implementation details. Team needs ability to optimize queries, change databases, or add caching without affecting service logic.

**Technical Challenge:**
- Raw JPA/Hibernate code scattered throughout service logic
- Complex queries for segment filtering and job management
- Need consistent error handling for database operations
- Caching strategies for frequently accessed segment definitions
- Database schema changes shouldn't break business logic

**Solution:** Repository Pattern provides a collection-like interface for accessing domain objects, abstracting database operations and enabling cleaner separation of concerns.

### Where Applied
- **JPA/RDS Integration**: Abstracts database operations
- **Segment Repository**: CRUD operations for segments

### Real Implementation

```java
public interface SegmentRepository {
    Segment findById(String segmentId);
    List<Segment> findByOrg(String orgId);
    List<Segment> findDueForMaterialization();
    void save(Segment segment);
    void updatePopulation(String segmentId, int count);
    void updateMaterializationStatus(String segmentId, String s3Location);
}

@Repository
public class JpaSegmentRepository implements SegmentRepository {
    
    @Autowired
    private EntityManager entityManager;
    
    @Override
    public Segment findById(String segmentId) {
        return entityManager.find(Segment.class, segmentId);
    }
    
    @Override
    public List<Segment> findDueForMaterialization() {
        return entityManager.createQuery(
            "SELECT s FROM Segment s WHERE s.nextMaterializationDue <= :now AND s.enabled = true",
            Segment.class
        ).setParameter("now", LocalDateTime.now()).getResultList();
    }
    
    @Override
    public void updatePopulation(String segmentId, int count) {
        entityManager.createQuery(
            "UPDATE Segment s SET s.populationCount = :count, s.lastCountedAt = :now WHERE s.id = :id"
        ).setParameter("count", count)
         .setParameter("now", LocalDateTime.now())
         .setParameter("id", segmentId)
         .executeUpdate();
    }
}
```

---

## Complete Flow Examples

### Counting Flow (Complete Sequence)

```mermaid
sequenceDiagram
    participant Core
    participant Seg as Segmentation_Engine
    participant QB as Query_Builder
    participant RDS
    participant MDS
    participant SQS
    participant Admin
    participant Datorama
    
    Core->>Seg: 1. Count Segment Request {segment_id}
    Seg->>Core: 2. 202 Response (Accepted)
    
    Note over Seg,Datorama: Process Count Async
    
    Seg->>Seg: 2. Retrieve Segment from RDS
    Seg->>Seg: Check: Segment not disabled
    Seg->>MDS: 3. Get Partition Attributes
    MDS->>Seg: 4. OK Response
    
    Seg->>QB: 5. Generate SQL from filter
    QB->>Seg: SQL: SELECT COUNT...
    
    Seg->>RDS: 6. Write Count Job Data
    Seg->>SQS: 7. Enqueue Count Segment Event
    
    SQS->>Seg: 9. Handle Count Segment Event
    Seg->>RDS: 10. Get Count Job Data
    Seg->>Admin: 11. Get Datorama Workspace Token
    Admin->>Seg: 12. OK Response (token)
    
    Seg->>Datorama: 13. Request Included Population (with SQL)
    Note over Datorama: Datorama executes query<br/>may use cached result
    Datorama->>Seg: 14. 200 Response (count: 1247)
    
    Seg->>Datorama: 15. Request Total Population
    Datorama->>Seg: 16. 200 Response (total: 125000)
    
    Seg->>RDS: 17. Update Segment Job Status & Population
    
    Note over Core: Core polls status or receives callback
```

**Real Data at Each Step:**

**Step 1: Count Request**
```json
POST /api/v1/segments/SEG-HIGH-VALUE-001/count
{
  "userId": "user@acme.com",
  "orgId": "00D5e000000abcd"
}
```

**Step 2: Immediate Response**
```json
HTTP 202 Accepted
{
  "jobId": "COUNT-JOB-12345",
  "segmentId": "SEG-HIGH-VALUE-001",
  "status": "QUEUED",
  "estimatedCompletion": "2025-10-23T14:32:00Z"
}
```

**Step 5: Generated SQL**
```sql
SELECT COUNT(DISTINCT IndividualId) as population_count
FROM UnifiedIndividual__dmo
WHERE LifetimeValue__c > 100000
  AND ChurnProbability__c > 0.7
  AND Industry__c IN ('Technology', 'Finance')
```

**Step 13: Datorama Request**
```json
POST /api/v1/query/execute
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
{
  "workspaceId": "WS-12345",
  "sql": "SELECT COUNT(DISTINCT IndividualId)...",
  "queryType": "count",
  "cacheEnabled": true
}
```

**Step 14: Datorama Response**
```json
{
  "queryId": "QUERY-55501",
  "status": "COMPLETED",
  "result": {
    "population_count": 1247
  },
  "executionTime": "1.8s",
  "cached": false
}
```

**Step 17: Database Update**
```sql
UPDATE segments 
SET population_count = 1247,
    last_counted_at = '2025-10-23T14:31:45Z',
    status = 'COUNTED'
WHERE segment_id = 'SEG-HIGH-VALUE-001';
```

### Materialization Flow (Complete Sequence)

```mermaid
sequenceDiagram
    participant Core
    participant Seg as Segmentation_Engine
    participant QB as Query_Builder
    participant MDS
    participant RDS
    participant SQS
    participant Admin
    participant Datorama
    participant S3
    participant Kafka
    participant Act as Activation_Engine
    
    Core->>Seg: 1. Materialize Segment Request
    
    Seg->>MDS: 2. Get Partition Attributes
    MDS->>Seg: 3. OK Response
    
    Seg->>QB: 4. Generate SQL from filter
    QB->>Seg: SQL with jobId
    
    Seg->>RDS: 5. Write Materialize Job Data
    Seg->>SQS: 6. Enqueue Materialize Segment Event
    Seg->>Core: 7. 202 Response
    
    Note over Seg,Kafka: Process Materialize Async
    
    SQS->>Seg: 8. Handle Materialize Segment Event
    Seg->>RDS: 9. Get Materialize Job Data
    Seg->>Admin: 10. Get Datorama Workspace Token
    Admin->>Seg: 11. OK Response
    
    Seg->>Datorama: 12. Patch Data Stream (updated SQL with jobId)
    Datorama->>Seg: 13. 200 Response
    
    Seg->>Datorama: 14. Process Data Stream
    Note over Datorama: Execute query<br/>Return full records
    Datorama->>Seg: 15. 200 Response (stream complete)
    
    Note over Seg,S3: Handle Materialize Completion
    
    Seg->>Seg: 16. Handle Data Stream Completion Event
    Seg->>Datorama: 17. Request Total Population Count
    Datorama->>Seg: 18. 200 Response (count: 125000)
    
    Seg->>RDS: 19. Update Segment Job Status & Population
    Seg->>Kafka: 20. Enqueue Materialization Complete Event
    
    Kafka->>Act: Materialization Complete
    Note over Act: Activates to Marketing Cloud,<br/>Google Ads, Facebook, etc.
```

**Step 12: Create Data Stream in Datorama**
```json
PATCH /api/v1/datastream
{
  "streamId": "STREAM-98765",
  "workspaceId": "WS-12345",
  "sql": "SELECT IndividualId, FirstName, LastName, Email, LifetimeValue__c, ChurnProbability__c, Industry__c, AccountName FROM UnifiedIndividual__dmo WHERE LifetimeValue__c > 100000 AND ChurnProbability__c > 0.7 AND Industry__c IN ('Technology', 'Finance')",
  "jobId": "JOB-98765",
  "outputFormat": "parquet",
  "compression": "snappy",
  "outputLocation": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/"
}
```

**Step 14-15: Datorama Processing**
```json
// Datorama executes query and writes to S3
{
  "streamId": "STREAM-98765",
  "status": "COMPLETED",
  "recordCount": 1247,
  "outputFiles": [
    "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/part-00000.parquet",
    "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/part-00001.parquet"
  ],
  "executionTime": "3.2s",
  "dataSize": "2.1 MB"
}
```

**Step 20: Kafka Event Published**
```json
{
  "eventType": "SegmentMaterializationComplete",
  "eventId": "EVT-55501",
  "timestamp": "2025-10-23T14:35:00Z",
  "segmentId": "SEG-HIGH-VALUE-001",
  "segmentName": "High-Value At-Risk Customers",
  "jobId": "JOB-98765",
  "populationCount": 1247,
  "totalPopulation": 125000,
  "percentage": 1.0,
  "s3Location": "s3://segmentation-output/SEG-HIGH-VALUE-001/JOB-98765/",
  "fileFormat": "parquet",
  "compression": "snappy",
  "activationTargets": ["marketing_cloud", "google_ads", "facebook_ads", "linkedin_ads"],
  "metadata": {
    "orgId": "00D5e000000abcd",
    "userId": "user@acme.com",
    "materializedAt": "2025-10-23T14:35:00Z",
    "dataSize": "2.1 MB"
  }
}
```

---

## Pattern Summary Table

| # | Pattern | Implementation | Key Benefit |
|---|---------|----------------|-------------|
| 1 | Adapter | Query_Builder translates filter to SQL | Business users define criteria, not SQL |
| 2 | Facade | REST API hides 10+ internal operations | Simple interface for Core service |
| 3 | Proxy | Datorama access via RestTemplate | Authentication, monitoring, caching |
| 4 | Bridge | Segment definition ↔ Query execution | Can switch query engines |
| 5 | Strategy | Count vs Materialize vs Scheduled | Different processing based on needs |
| 6 | Observer | Kafka event → Multiple activations | One event triggers multiple platforms |
| 7 | Command | SQS queued operations | Async processing, retry logic |
| 8 | Template Method | Standard processing pipeline | Consistent workflow, customizable steps |
| 9 | Chain of Responsibility | Request validation chain | Sequential checks, fail-fast |
| 10 | Mediator | Segmentation Engine coordinates all | Reduced coupling, O(n) complexity |
| 11 | Factory | Query type creation | Creates appropriate query objects |
| 12 | Repository | JPA/RDS abstraction | Database independence |

---

## Technology Stack

**Framework:**
- **Scone** (Salesforce microservices framework on Spring Boot)
- Spring MVC for REST APIs
- Spring JPA for database
- Spring JMS for messaging

**Data Storage:**
- **RDS** (PostgreSQL): Segment definitions, job status, populations
- **S3**: Materialized segment data (Parquet format)

**Message Queue:**
- **SQS**: Async job processing
- **Kafka**: Event streaming for activations

**Query Execution:**
- **Datorama**: Query engine that processes SQL against Data Cloud data

**Integration:**
- **gRPC**: High-performance service-to-service communication
- **REST**: External API for Core service
- **RestTemplate**: HTTP client for Datorama, Admin Service

---

## Key Architectural Decisions

### 1. Why Two Operations (Count vs Materialize)?

**Count:**
- Fast preview for testing segment criteria
- No resource-intensive S3 writes
- No activation triggering
- Can be cached for performance

**Materialize:**
- Full execution for campaign activation
- Writes complete records to S3
- Triggers downstream systems via Kafka
- Required for actual marketing use

### 2. Why Async Processing (202 Accepted)?

```
Synchronous would require:
- Core waits for query execution (2-5 seconds)
- Blocks on S3 writes (variable time)
- Holds connection during Kafka publish

Async benefits:
- Immediate response (< 100ms)
- Core can poll status or receive callback
- Better resource utilization
- Retry logic in SQS
```

### 3. Why Datorama for Query Execution?

```
Datorama provides:
- Optimized query engine for Data Cloud
- Handles partition management
- Supports streaming results to S3
- Built-in caching for count queries
- Workspace-based multi-tenancy
```

### 4. Why S3 for Materialized Data?

```
S3 benefits:
- Scalable storage for large segments
- Cost-effective for infrequent access
- Standard format (Parquet) for activation engines
- Supports parallel reads
- Automatic lifecycle management
```

---

## Performance Characteristics

### Counting Operation

| Metric | Value |
|--------|-------|
| Response Time | < 100ms (202 Accepted) |
| Query Execution | 1-3 seconds |
| Cache Hit | < 500ms |
| Max Segment Size | 10M+ records |

### Materialization Operation

| Metric | Value |
|--------|-------|
| Response Time | < 100ms (202 Accepted) |
| Query Execution | 2-5 seconds |
| S3 Write | 1-10 seconds (varies by size) |
| Kafka Publish | < 100ms |
| End-to-End | 5-20 seconds |
| Max Segment Size | 10M+ records |

---

<a id="part-3-segmentation-product-lines"></a>
# Part 3: Segmentation Product Lines

### Segmentation Engine in Data Cloud Ecosystem

```mermaid
graph TB
    subgraph "Data Cloud Core"
        UP[Unified Profiles]
        CI[Calculated Insights]
        IR[Identity Resolution]
    end
    
    subgraph "Segmentation Engine"
        SegAPI[Segmentation API]
        QB[Query Builder]
        Proc[Processing Engine]
    end
    
    subgraph "Activation"
        DA[Data Actions]
        MC[Marketing Cloud]
        Ads[Ad Platforms]
    end
    
    UP -->|Query| SegAPI
    CI -->|Enrich| SegAPI
    SegAPI -->|Filter| QB
    QB -->|Execute| Proc
    Proc -->|Materialize| DA
    DA -->|Activate| MC
    DA -->|Activate| Ads
    
    style SegAPI fill:#166534,color:#ffffff
```

**Integration Points:**
1. **Unified Profiles** → Source data for segmentation
2. **Calculated Insights** → Used in segment criteria (LTV, Churn Risk)
3. **Identity Resolution** → Ensures single view per individual
4. **Data Actions** → Consumes segment materializations for activation

### Contact & Account Master (CAM) Architecture Integration

The segmentation engine leverages Data Cloud's **Contact & Account Master (CAM)** architecture for unified data processing:

```mermaid
flowchart TB
    subgraph "🌍 DATA SOURCES"
        direction LR
        CRM[🔗 CRM Connector<br/>📊 Org62<br/>Contacts, Accounts, Leads]
        MC[🔗 MC Connector<br/>📧 Marketing Cloud<br/>Email Engagement Data]
        S3[🔗 S3 Connector<br/>☁️ External Data<br/>3rd Party Sources]
        Snowflake[🔗 Snowflake<br/>🏢 Data Federation<br/>Enterprise Data Warehouse]
    end
    
    subgraph "� DATA INGESTION & TRANSFORMATION"
        direction TB
        subgraph "Default Data Space"
            StagingDLO[📦 Staging DLO<br/>Raw Data Storage<br/>As-Is Format]
            
            subgraph "Truth Profile Process"
                direction LR
                TransformDLO[⚙️ Transform DLO<br/>Data Processing Engine]
                BatchProcess[⏱️ Batch<br/>Scheduled Processing]
                StreamProcess[🌊 Stream<br/>Real-time Processing]
                Survivorship[🏆 Survivorship<br/>Master Record Rules]
                BatchAirflow[⏱️ Batch - Airflow<br/>Orchestrated Jobs]
            end
            
            MasterDLO[🎯 Master DLO<br/>Clean Processed Data<br/>Quality Validated]
        end
        
        subgraph "CAM ONLY Processing"
            direction LR
            EmailExclusion[⚠️ Email Exclusion<br/>Bounce/Unsubscribe Filtering]
            IndividualDLO[👤 Individual DLO<br/>Person-Level Aggregation]
        end
    end
    
    subgraph "� DATA MODEL"
        direction TB
        Individual[💡 Individual<br/>Master Person Record<br/>Cross-Channel Identity]
        ContactMaster[📱 Contact Master<br/>CRM Contact Integration<br/>Sales Context]
        Lead[🎯 Lead<br/>Prospect Management<br/>Marketing Qualified]
        MACD[🏢 Master Account<br/>Contact Data MACD<br/>Business Relationships]
        CAMIndividual[🔴 CAM Individual<br/>Unified Customer Profile<br/>360 Degree View]
        CAMUnified[🔗 CAM Unified Link<br/>Individual Relationships<br/>Household/Account Links]
    end
    
    subgraph "� IDENTITY RESOLUTIONS"
        direction LR
        ContactDedup[🧹 Contact Dedup<br/>IDR Process<br/>Duplicate Management]
    end
    
    subgraph "� INSIGHTS OR ACTIVATION LAYER"
        direction TB
        subgraph "Marketing Activation"
            SFMC_Table[📧 SFMC Data Extension<br/>Individual Table<br/>Journey Builder Ready]
        end
        subgraph "Sales Activation"  
            Org2_CRM[⚡ Org2<br/>CRM SObject<br/>Sales Cloud Integration]
        end
    end
    
    %% Data Flow Connections
    CRM -.->|Ingest| StagingDLO
    MC -.->|Ingest| StagingDLO
    S3 -.->|Ingest| StagingDLO
    Snowflake -.->|Federation| TransformDLO
    
    StagingDLO -->|Raw Data| TransformDLO
    TransformDLO -->|Batch Jobs| BatchProcess
    TransformDLO -->|Real-time| StreamProcess
    BatchProcess -->|Rules Engine| Survivorship
    StreamProcess -->|Orchestration| BatchAirflow
    Survivorship -->|Clean Data| MasterDLO
    
    MasterDLO -->|CAM Processing| EmailExclusion
    EmailExclusion -->|Filtered Data| IndividualDLO
    
    IndividualDLO -->|Master Records| Individual
    IndividualDLO -->|CRM Sync| ContactMaster
    IndividualDLO -->|Lead Gen| Lead
    IndividualDLO -->|Account Data| MACD
    
    Individual -->|Unified View| CAMIndividual
    ContactMaster -->|Link Creation| CAMUnified
    MACD -->|Business Context| CAMUnified
    
    Individual -->|Deduplication| ContactDedup
    
    CAMIndividual -.->|🎯 Segmentation Query| SFMC_Table
    CAMUnified -.->|🎯 Segmentation Query| SFMC_Table
    ContactDedup -->|Clean Contacts| Org2_CRM
    
    %% Styling
    style CAMIndividual fill:#dc2626,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style CAMUnified fill:#dc2626,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style ContactDedup fill:#7c3aed,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style SFMC_Table fill:#166534,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style Org2_CRM fill:#166534,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style TransformDLO fill:#1e40af,color:#ffffff,stroke:#ffffff,stroke-width:2px
    style Survivorship fill:#92400e,color:#ffffff,stroke:#ffffff,stroke-width:2px
    style EmailExclusion fill:#991b1b,color:#ffffff,stroke:#ffffff,stroke-width:2px
```

### CAM Data Processing Sequence for Segmentation

```mermaid
sequenceDiagram
    participant CRM as 🔢 Step 1<br/>CRM Connector<br/>Org62
    participant MC as 🔢 Step 2<br/>MC Connector<br/>Marketing Cloud
    participant Stage as 🔢 Step 3<br/>Staging DLO<br/>Raw Storage
    participant Transform as 🔢 Step 4<br/>Transform DLO<br/>Processing Engine
    participant Survive as 🔢 Step 5<br/>Survivorship<br/>Master Rules
    participant EmailEx as 🔢 Step 6<br/>Email Exclusion<br/>CAM Only
    participant IndDLO as 🔢 Step 7<br/>Individual DLO<br/>Person Aggregation
    participant CAMInd as 🔢 Step 8<br/>CAM Individual<br/>Unified Profile
    participant CAMLink as 🔢 Step 9<br/>CAM Unified Link<br/>Relationships
    participant Dedup as 🔢 Step 10<br/>Contact Dedup IDR<br/>Identity Resolution
    participant SFMC as 🔢 Step 11<br/>SFMC Data Extension<br/>Individual Table
    participant Org2 as 🔢 Step 12<br/>Org2 CRM<br/>SObject
    
    Note over CRM,Org2: Daily CAM Processing Cycle - 6AM PST
    
    %% Data Ingestion Phase
    CRM->>Stage: 1️⃣ Ingest CRM data - Contacts 2.3M records
    MC->>Stage: 2️⃣ Ingest email engagement - Opens 450K events
    Stage->>Transform: 3️⃣ Raw data transformation & validation
    
    %% Truth Profile Processing
    Transform->>Transform: 4️⃣ Apply data quality rules
    Transform->>Survive: 5️⃣ Survivorship processing - cross-source dedup
    Survive->>Survive: 6️⃣ Master record selection - email priority
    
    %% CAM-Specific Processing
    Survive->>EmailEx: 7️⃣ Apply email exclusion filters
    EmailEx->>EmailEx: 8️⃣ Remove bounced/unsubscribed - 15K filtered
    EmailEx->>IndDLO: 9️⃣ Person-level aggregation
    
    %% Data Model Creation
    IndDLO->>CAMInd: 🔟 Create unified individual profiles
    IndDLO->>CAMLink: 1️⃣1️⃣ Establish household/account relationships
    IndDLO->>Dedup: 1️⃣2️⃣ Identity resolution processing
    
    Note over CAMInd,SFMC: Segmentation and Activation - Real-time
    
    %% Segmentation Query
    Note right of CAMInd: Query: High-Value Customers<br/>LTV > $1000 + Email Consent
    CAMInd->>CAMInd: 1️⃣3️⃣ Apply segment criteria - 2.3M to 1.2M matches
    CAMLink->>CAMInd: 1️⃣4️⃣ Enrich with relationship data
    CAMInd->>CAMInd: 1️⃣5️⃣ Apply consent filtering - 1.2M to 850K marketable
    
    %% Activation
    CAMInd->>SFMC: 1️⃣6️⃣ Push to Individual Table - 850K profiles
    SFMC->>SFMC: 1️⃣7️⃣ Create journey audience and trigger campaigns
    
    Dedup->>Org2: 1️⃣8️⃣ Sync clean contacts to Sales Cloud - 2.1M records
    
    Note over CRM,Org2: Stream Processing - Real-time Updates
    
    CRM->>Transform: 1️⃣9️⃣ Real-time contact update - Web form submission
    Transform->>CAMInd: 2️⃣0️⃣ Update individual record - 5min latency
    CAMInd->>SFMC: 2️⃣1️⃣ Trigger segment refresh - plus 1 new contact
```

**Key CAM Integration Benefits for Segmentation:**

🎯 **Unified Data Model**: CAM Individual provides single customer view across all touchpoints  
📊 **Identity Resolution**: Contact Dedup IDR ensures accurate individual identification  
🔄 **Real-time Processing**: Stream processing enables near real-time segment updates  
📈 **Master Data Management**: Clean, deduplicated data improves segment quality  
🌐 **Multi-Org Support**: Segmentation works across Salesforce orgs and external systems

**Timing & Performance Impact:**
- **Batch Processing**: Daily CAM updates complete by 6 AM PST
- **Segmentation Query**: ~2-3 seconds for 1M+ profile segments
- **Real-time Updates**: Stream processing adds new contacts within 5 minutes
- **Activation Latency**: SFMC sync completes within 15 minutes

---

## Data Enrichment & Mastering Processes

### Why Data Enrichment Matters for Segmentation

**Quality segmentation requires clean, unified data**. Without proper mastering, a single customer appears as multiple records (CRM contact + SFMC subscriber + Lead), creating fragmented segments and inaccurate audience sizes.

### Data Enrichment Overview

Data Enrichment in the Unified Profile Team focuses on refining and enhancing contact and account data by integrating information from multiple sources, such as **Org62 UUID Contact**, **Leads**, and **SFMC records**. The process ensures that each record is accurate, complete and standardized, reducing duplication and inconsistencies through three mastering methods:

### Three-Pillar Mastering Architecture

```mermaid
flowchart TB
    subgraph "🔢 Step 1: Data Sources Integration"
        direction LR
        Org62[🏢 Org62 UUID<br/>CRM Contacts & Leads<br/>Sales Priority Data]
        SFMC[📧 SFMC Records<br/>Email Engagement<br/>Marketing Campaigns]
        EDW[🏗️ EDW Data<br/>Enterprise Data Warehouse<br/>Historical Records]
        NGMA[📊 NGMA Datasets<br/>Analytics Platform<br/>Behavioral Data]
    end
    
    subgraph "🔢 Step 2: Mastering Processes"
        direction TB
        ContactMaster[👤 Contact Mastering<br/>🎯 Goal: Single Contact View<br/>📍 Replaces Harte Hanks Process]
        LeadMaster[🎯 Lead Mastering<br/>📈 Goal: Unified Lead Master<br/>🔄 Phases Implementation]
        AccountMaster[🏢 Account Mastering<br/>🌐 Goal: Master Account ID<br/>📊 Customer & GTM Data Team]
    end
    
    subgraph "🔢 Step 3: Golden Record Creation"
        direction LR
        GoldenContact[✨ Golden Contact Record<br/>📋 Survivorship Rules<br/>🔑 Unique Contact Master ID]
        GoldenLead[✨ Golden Lead Record<br/>📋 Master Lead ID<br/>🎯 Final Lead DLO]
        GoldenAccount[✨ Golden Account Record<br/>📋 Master Account ID<br/>🔗 Parent-Child Relationships]
    end
    
    subgraph "🔢 Step 4: CAM Integration"
        direction TB
        CAMIndividual[🔴 CAM Individual<br/>Unified Customer Profile<br/>360 Degree View]
        CAMUnified[🔗 CAM Unified Link<br/>Individual Relationships<br/>Household/Account Links]
    end
    
    subgraph "🔢 Step 5: Segmentation Ready"
        direction LR
        CleanData[📊 Clean Unified Data<br/>No Duplicates<br/>Accurate Counts]
        SegmentEngine[🎯 Segmentation Engine<br/>Quality Segments<br/>Precise Targeting]
    end
    
    %% Data Flow
    Org62 --> ContactMaster
    SFMC --> ContactMaster
    EDW --> LeadMaster
    Org62 --> LeadMaster
    
    Org62 --> AccountMaster
    EDW --> AccountMaster
    
    ContactMaster --> GoldenContact
    LeadMaster --> GoldenLead
    AccountMaster --> GoldenAccount
    
    GoldenContact --> CAMIndividual
    GoldenLead --> CAMIndividual
    GoldenAccount --> CAMUnified
    
    CAMIndividual --> CleanData
    CAMUnified --> CleanData
    CleanData --> SegmentEngine
    
    %% Styling
    style ContactMaster fill:#dc2626,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style LeadMaster fill:#dc2626,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style AccountMaster fill:#dc2626,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style CAMIndividual fill:#166534,color:#ffffff,stroke:#ffffff,stroke-width:3px
    style SegmentEngine fill:#7c3aed,color:#ffffff,stroke:#ffffff,stroke-width:3px
```

### 1. Contact Mastering Process

**Purpose**: Unifies and manages contact profiles in Customer C360 Data Cloud, replacing the legacy Harte Hanks mastering process.

**Process Workflow**:

```mermaid
sequenceDiagram
    participant Sources as 🔢 Step 1<br/>Multiple Sources<br/>Org62, SFMC, NGMA
    participant Filter as 🔢 Step 2<br/>Profile Filtering<br/>Exclusion Rules
    participant Group as 🔢 Step 3<br/>Email Grouping<br/>Matching Logic
    participant Select as 🔢 Step 4<br/>Primary Selection<br/>Sales Priority Rules
    participant Golden as 🔢 Step 5<br/>Golden Record<br/>Survivorship Rules
    participant DMO as 🔢 Step 6<br/>Master DMO<br/>UUID Assignment
    
    Sources->>Filter: 1️⃣ Ingest contact data from all sources
    Filter->>Filter: 2️⃣ Apply exclusion rules for invalid profiles
    Filter->>Group: 3️⃣ Group profiles with matching emails
    Group->>Select: 4️⃣ Apply sales priority rules for primary contact
    Select->>Golden: 5️⃣ Create Golden Record using survivorship
    Golden->>DMO: 6️⃣ Store in DMO with unique Contact Master ID
    
    Note over Sources,DMO: Result: Single accurate record per individual
```

**Key Features**:
- **Email-based Grouping**: Profiles with matching emails are grouped together
- **Sales Priority Rules**: CRM contacts prioritized over marketing leads
- **Survivorship Logic**: Most complete and recent data survives
- **Unique UUID**: Each individual gets one Contact Master ID

### 2. Lead Mastering Process

**Purpose**: Build unified Lead Master on Org62 leads in Data Cloud, replacing Harte Hanks lead mastering.

**Data Sources & Rules**:

```mermaid
flowchart LR
    subgraph "🔢 Step 1: Data Sources"
        Org62Lead[📋 Org62 Lead<br/>CRM Lead Records]
        EDWLead[🏗️ EDW Lead<br/>Historical Lead Data]
        Org62Contact[👤 Org62 Contact<br/>Converted Leads]
    end
    
    subgraph "🔢 Step 2: Unification Rules"
        DupeHandling[🔄 Duplicate Handling<br/>Email & Company Match]
        SpecialRules[⚡ Special Cases<br/>NULL Field Handling]
        Grouping[📊 Grouping Logic<br/>Lead Relationship Rules]
    end
    
    subgraph "🔢 Step 3: Master Creation"
        MasterStamp[🏷️ Master Lead ID<br/>Stamping Process]
        FinalDLO[📦 Final Lead DLO<br/>Master Records]
        Mapping[🗺️ DLO Mapping<br/>Downstream Integration]
    end
    
    Org62Lead --> DupeHandling
    EDWLead --> DupeHandling
    Org62Contact --> SpecialRules
    
    DupeHandling --> Grouping
    SpecialRules --> Grouping
    Grouping --> MasterStamp
    MasterStamp --> FinalDLO
    FinalDLO --> Mapping
    
    style MasterStamp fill:#dc2626,color:#ffffff
    style FinalDLO fill:#166534,color:#ffffff
```

**Implementation Phases**:
- **Phase 1**: Basic lead unification with email matching
- **Phase 2**: Advanced grouping and survivorship rules
- **Phase 3**: Special case handling and NULL field logic

### 3. Account Mastering Process

**Purpose**: Establish end-to-end architecture for Org62 Account data using Master Account ID from Customer & GTM Data Team.

**Mastering Architecture**:

```mermaid
sequenceDiagram
    participant Org62 as 🔢 Step 1<br/>Org62 Accounts<br/>CRM Account Data
    participant Leadspace as 🔢 Step 2<br/>Leadspace Enrichment<br/>Customer & GTM Team
    participant Snowflake as 🔢 Step 3<br/>Snowflake Views<br/>Master Account Data
    participant DataCloud as 🔢 Step 4<br/>Data Cloud<br/>Account Resolution
    participant MasterDMO as 🔢 Step 5<br/>Master Account DMO<br/>Final Records
    
    Note over Org62,MasterDMO: Weekly Processing Cycle - Fridays
    
    Org62->>Leadspace: 1️⃣ Send new accounts for enrichment
    Leadspace->>Leadspace: 2️⃣ Daily enrichment for new accounts
    Leadspace->>Leadspace: 3️⃣ Yearly enrichment for existing accounts
    Leadspace->>Snowflake: 4️⃣ Update master account views weekly
    Snowflake->>DataCloud: 5️⃣ Ingest mastered data from views
    DataCloud->>MasterDMO: 6️⃣ Create Master Account ID assignments
    
    Note right of MasterDMO: Parent-Child Relationships:<br/>- Own Master Account ID<br/>- Reference Parent Master ID
```

**Key Features**:
- **Leadspace Integration**: Customer & GTM Data Team handles enrichment
- **Weekly Refresh**: Snowflake views updated every Friday
- **Parent-Child Logic**: Accounts reference master or parent master IDs
- **Scalable Architecture**: Replaces Harte Hanks process

### Impact on Segmentation Quality

**Before Mastering**:
```
Customer "John Smith" appears as:
- CRM Contact: john.smith@company.com
- SFMC Subscriber: j.smith@company.com  
- Lead Record: johnsmith@company.com
Segment Result: 3 separate records = Inflated count
```

**After Mastering**:
```
Customer "John Smith" unified as:
- Golden Record: john.smith@company.com (Contact Master ID: UUID-123)
- All variations linked to single identity
Segment Result: 1 accurate record = True count
```

**Segmentation Benefits**:
- **Accurate Counts**: No duplicate individuals in segments
- **Complete Profiles**: Enriched data from multiple sources
- **Better Targeting**: Unified view enables precise segmentation
- **Compliance Ready**: Single consent status per individual

---

## Conclusion

The Segmentation Engine demonstrates **12 out of 16 design patterns (75% coverage)**, showcasing a well-architected microservice built on Salesforce's Scone framework. 

**Key Strengths:**
1. **Clear Separation of Concerns**: Counting vs Materialization strategies
2. **Async Processing**: Non-blocking operations with SQS
3. **Event-Driven Activation**: Kafka events trigger multiple downstream systems
4. **Query Abstraction**: Business logic separated from SQL generation
5. **Scalable Architecture**: Handles millions of records efficiently

**Patterns Not Implemented:**
- **Decorator**: Could add enrichment layers to segment data
- **Composite**: Could support nested segment hierarchies
- **Singleton**: Not applicable (stateless microservice)
- **Flyweight**: Could optimize memory for segment metadata

The Segmentation Engine serves as a critical component in the Data Cloud activation workflow, bridging the gap between data analysis (segments) and marketing execution (campaigns).

---

## Citations & References

### Source Code Repositories
1. **Segmentation Engine Implementation**  
   https://git.soma.salesforce.com/a360/segmentation-engine
   
2. **Scone Microservices Framework**  
   https://git.soma.salesforce.com/services/sfdc-bazel/tree/main/projects/libs/scone

### Design Pattern References
3. **Refactoring.Guru Design Patterns**  
   https://refactoring.guru/design-patterns
   
4. **Gang of Four (GoF) Design Patterns**  
   Gamma, E., Helm, R., Johnson, R., & Vlissides, J. (1994). *Design Patterns: Elements of Reusable Object-Oriented Software*

### Architecture Documentation
5. **Unified Profile Segmentation Documentation**  
   https://confluence.internal.salesforce.com/spaces/UNIFIEDPROFILE/pages/1079183456/Segmentation

6. **Salesforce Data Cloud Segmentation Service Documentation**  
   Internal Salesforce Documentation

7. **Understanding Consent, Subscriptions and Preferences at Salesforce: A Dummies Guide**  
   https://confluence.internal.salesforce.com/spaces/UNIFIEDPROFILE/pages/1078460747/Understanding+Consent+Subscriptions+and+Preferences+at+Salesforce+A+Dummies+Guide

8. **Contact & Account Master - Solution**  
   https://confluence.internal.salesforce.com/spaces/UNIFIEDPROFILE/pages/1078460951/Contact+Account+Master+-+Solution

9. **Scoring & Segmentation-Categorizing- Data Enrichment**  
   https://confluence.internal.salesforce.com/spaces/UNIFIEDPROFILE/pages/1078461534/Scoring+Segmentation-Categorizing-+Data+Enrichment

10. **Modern Architectural Patterns**  
    Industry best practices for microservices and distributed systems

---

**Document Version:** 1.0  
**Created:** November 10, 2025  
**Author:** Architecture Analysis  
**Based on:** Salesforce Data Cloud Segmentation Service Implementation

