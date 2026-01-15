# Scotia Offers: Pega Replacement Architecture
## Salesforce Data Cloud, Loyalty Management & Marketing Cloud Solution

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Current State Architecture](#current-state-architecture)
3. [Target State Architecture](#target-state-architecture)
4. [Component Mapping: Current → Target](#component-mapping-current--target)
5. [Data Flow Diagrams](#data-flow-diagrams)
6. [Integration Patterns](#integration-patterns)
7. [Migration Roadmap](#migration-roadmap)
8. [Technical Implementation Details](#technical-implementation-details)
9. [Key Benefits & ROI](#key-benefits--roi)

---

## Executive Summary

### The Challenge
Scotia's current offer management ecosystem consists of **20+ fragmented systems** spanning Azure, GCP, and on-premise infrastructure. The architecture suffers from:
- **30-50 day vendor onboarding cycles**
- **No real-time offer presentation** (critical gap)
- **Batch-dependent data flows** via SFTP
- **Complex reconciliation processes**
- **Vendor dependency challenges**

### The Solution
Replace the Pega-based offer management with a unified **Salesforce stack**:

| Component | Role | Replaces |
|-----------|------|----------|
| **Salesforce Data Cloud** | The Hub & Brain | CDP, Event Exchange Hub, EDW |
| **Loyalty Management** | The Offer Engine | Orion, Constellation, Nova, Promo Code App |
| **Marketing Cloud** | The Orchestration Engine | Homegrown Interface, EDAT |
| **MC Personalization** | Real-Time Inbound Engine | BRL, CMEE, R/T gaps |

### Key Outcomes
- **Reduce vendor onboarding from 30-50 days → 3-5 days**
- **Enable real-time offer decisioning** (closing the R/T gap)
- **Consolidate 20+ systems → 4 integrated platforms**
- **Unified customer 360° view** across all touchpoints

---

## Current State Architecture

### System Inventory

```mermaid
graph TB
    subgraph "Current State: 20+ Fragmented Systems"
        subgraph "Entry Points"
            BNS[("👤 BNS User<br/>Product Setup")]
            VENDOR["Vendor SaaS<br/>⏱️ 30-50 days"]
        end
        
        subgraph "Azure Layer"
            AZURE[("☁️ Azure")]
            BHF7["Product Catalog<br/>& Offer Tracking<br/>(BHF7)"]
        end
        
        subgraph "Ingestion Layer"
            SFTP["📁 SFTP<br/>(Legacy)"]
            BB4J["CBT EDL Ingestion<br/>(BB4J)"]
            B9K3["EDLR<br/>(B9K3)"]
        end
        
        subgraph "GCP Processing"
            GCS["Google Storage"]
            PROMO["Promo Code App"]
            BFB6["Pigeon (BFB6)"]
            BERY["Insight (BERY)"]
            BDMS["Marvel (BDMS)"]
        end
        
        subgraph "Offer Presentation Layer"
            BDQJ["Orion (BDQJ)"]
            BFYL["Constellation (BFYL)"]
            BCCY["Nova (BCCY)"]
            B9XX["BRL (B9XX)"]
            BB8K["CMEE (BB8K)"]
        end
        
        subgraph "CDP & Orchestration"
            CDP["CDP"]
            BHBD["Adaptor (BHBD)"]
            HOMEGROWN["Homegrown Interface<br/>EDAT Orchestration"]
            BFC8["DLP Tokenization<br/>(BFC8)"]
        end
        
        subgraph "Event Processing"
            BF8M["Event Exchange Hub<br/>(BF8M)"]
            BDLQ["Data Power (BDLQ)"]
        end
        
        subgraph "Downstream Systems"
            POSTING["Posting API<br/>(TDS BFI4)"]
            BCJW["EDW (BCJW)"]
            BD8K["AW (BD8K)"]
            BCYN["EP (BCYN)"]
            BGMC["Loyalty Service<br/>(BGMC)"]
            BOND["Bond"]
        end
    end
    
    BNS --> VENDOR
    VENDOR --> BHF7
    BHF7 --> SFTP
    SFTP --> GCS
    GCS --> BFB6
    BFB6 --> BDQJ
    BDQJ --> CDP
    CDP --> BF8M
    BF8M --> POSTING
    BGMC --> BOND
    
    style SFTP fill:#ff6b6b,color:#fff
    style HOMEGROWN fill:#ff6b6b,color:#fff
    style VENDOR fill:#ffd93d,color:#000
```

### Current Data Flow Sequence

```mermaid
sequenceDiagram
    autonumber
    participant BNS as 👤 BNS User
    participant Vendor as Vendor SaaS
    participant SFTP as SFTP (Legacy)
    participant GCP as GCP Storage
    participant Token as DLP Tokenization
    participant Extract as 6G Extract
    participant Offer as Offer Layer<br/>(Orion/Constellation/Nova)
    participant CDP as CDP
    participant EventHub as Event Exchange Hub
    participant Posting as Posting API
    participant Loyalty as Loyalty Service
    participant Bond as Bond
    
    Note over BNS,Vendor: ⏱️ 30-50 days setup time
    BNS->>Vendor: 1. Product setup/offer config
    Vendor->>SFTP: 2. Batch file (tokenized)
    SFTP->>GCP: 3. Upload batch file
    GCP->>Token: 3b. Detokenization
    Token->>Extract: 4. Extract input/offer data
    Extract->>Offer: 5-6. Push to presentation layer
    
    Note over Offer,CDP: ❌ GAP: No R/T connection
    Offer--xCDP: Limited connectivity
    
    CDP->>EventHub: 7. Real-time events
    EventHub->>CDP: 8. Send BNS input events
    CDP->>EventHub: 9. Receive vendor events
    
    par Parallel Processing
        EventHub->>Posting: 10a. Cash bonus/fee rebates
        EventHub->>Loyalty: 10b. Consume events
    end
    
    Loyalty->>Bond: 11. Loyalty fulfillment
```

### Current State Problems

| # | Problem | Impact | Root Cause |
|---|---------|--------|------------|
| 1 | **No R/T Offer Presentation** | Lost revenue, poor CX | Adapter to Orion/Constellation/Nova not connected |
| 2 | **30-50 Day Vendor Onboarding** | Slow time-to-market | Complex manual processes |
| 3 | **Batch-Only Integration** | Stale data, delayed offers | SFTP dependency |
| 4 | **20+ Systems** | High maintenance cost | Organic growth without consolidation |
| 5 | **Manual Reconciliation** | Errors, delays | No unified data model |
| 6 | **No Testing Framework** | Quality issues | Missing testing management process |

---

## Target State Architecture

### Salesforce Stack Overview

```mermaid
graph TB
    subgraph "Target State: Unified Salesforce Platform"
        subgraph "Data Sources"
            CRM["Salesforce CRM<br/>Customer Data"]
            TRANS["Transaction Systems<br/>D2D, Debit Card, Scene"]
            EXTERNAL["External Partners<br/>Bond, Vendors"]
            WEB["Digital Channels<br/>Web, Mobile, ATM"]
        end
        
        subgraph "Salesforce Data Cloud [The Hub & Brain]"
            INGEST["Data Ingestion<br/>• Streaming (Real-time)<br/>• Batch (Scheduled)<br/>• Zero Copy"]
            PROFILE["Unified Profile<br/>• Identity Resolution<br/>• Customer 360°"]
            SEGMENT["Segmentation<br/>• Dynamic Segments<br/>• Calculated Insights"]
            ACTIVATE["Data Actions<br/>• Activations<br/>• Platform Events"]
        end
        
        subgraph "Loyalty Management [The Offer Engine]"
            PROGRAMS["Loyalty Programs<br/>• Points, Tiers, Benefits"]
            OFFERS["Offer Management<br/>• Eligibility Rules<br/>• Redemption Logic"]
            REWARDS["Rewards Catalog<br/>• Cash Bonus<br/>• Fee Rebates<br/>• Fee Waivers"]
            PROMO2["Promotion Engine<br/>• Promo Codes<br/>• Campaign Offers"]
        end
        
        subgraph "Marketing Cloud [The Orchestration Engine]"
            JOURNEY["Journey Builder<br/>• Multi-step Journeys<br/>• Decision Splits"]
            EMAIL["Email Studio"]
            SMS2["Mobile (SMS/Push)"]
            AUTOMATION["Automation Studio<br/>• Scheduled Jobs<br/>• Triggered Sends"]
        end
        
        subgraph "MC Personalization [Real-Time Inbound Engine]"
            DECISION["Real-Time Decisioning<br/>• Next Best Offer<br/>• Personalization"]
            WEB_SDK["Web SDK<br/>• Site Personalization"]
            MOBILE_SDK["Mobile SDK<br/>• In-App Offers"]
            EINSTEIN_DEC["Einstein Decisions<br/>• ML-Powered"]
        end
        
        subgraph "Agentforce [AI Layer]"
            AGENT["Agentforce Agents<br/>• Offer Recommendations<br/>• Customer Service"]
            COPILOT["Einstein Copilot<br/>• Natural Language"]
            ACTIONS["Agent Actions<br/>• Offer Application<br/>• Status Check"]
        end
    end
    
    CRM --> INGEST
    TRANS --> INGEST
    EXTERNAL --> INGEST
    WEB --> INGEST
    
    INGEST --> PROFILE
    PROFILE --> SEGMENT
    SEGMENT --> ACTIVATE
    
    ACTIVATE --> OFFERS
    ACTIVATE --> JOURNEY
    ACTIVATE --> DECISION
    
    OFFERS --> REWARDS
    PROMO2 --> OFFERS
    
    JOURNEY --> EMAIL
    JOURNEY --> SMS2
    
    DECISION --> WEB_SDK
    DECISION --> MOBILE_SDK
    
    PROFILE --> AGENT
    OFFERS --> ACTIONS
    
    style PROFILE fill:#0176d3,color:#fff
    style DECISION fill:#ff6b6b,color:#fff
    style OFFERS fill:#2e844a,color:#fff
    style JOURNEY fill:#9050e9,color:#fff
```

### Target State Data Flow

```mermaid
sequenceDiagram
    autonumber
    participant User as 👤 Customer
    participant Channel as Digital Channel<br/>(Web/Mobile/ATM)
    participant MCP as MC Personalization
    participant DC as Data Cloud
    participant LM as Loyalty Management
    participant MC as Marketing Cloud
    participant Agent as Agentforce
    participant External as External Systems<br/>(Bond, Partners)
    
    Note over User,External: ⚡ Real-Time Flow (< 200ms)
    
    User->>Channel: Visit/Interact
    Channel->>MCP: 1. Capture behavior (Web SDK)
    MCP->>DC: 2. Query unified profile
    DC-->>MCP: 3. Return customer 360° + segments
    MCP->>LM: 4. Get eligible offers
    LM-->>MCP: 5. Return personalized offers
    MCP-->>Channel: 6. Real-time offer display
    
    Note over User,External: 🎯 Offer Acceptance Flow
    
    User->>Channel: Accept offer
    Channel->>LM: 7. Process redemption
    LM->>DC: 8. Update profile (Platform Event)
    
    par Parallel Fulfillment
        LM->>External: 9a. Fulfill reward (Bond)
        DC->>MC: 9b. Trigger confirmation journey
    end
    
    MC->>User: 10. Confirmation (Email/SMS)
    
    Note over User,External: 🤖 AI-Assisted Flow
    
    User->>Agent: "What offers do I have?"
    Agent->>DC: Query profile + offers
    DC-->>Agent: Return context
    Agent->>LM: Get recommendations
    LM-->>Agent: Eligible offers
    Agent-->>User: Personalized response
```

---

## Component Mapping: Current → Target

### Detailed System Migration Map

```mermaid
flowchart LR
    subgraph CURRENT["Current State Systems"]
        direction TB
        C1["CDP"]
        C2["Event Exchange Hub (BF8M)"]
        C3["EDW (BCJW)"]
        C4["Adaptor (BHBD)"]
        C5["Data Power (BDLQ)"]
        C6["ScotiaLive CID (BCJD)"]
        
        C7["Orion (BDQJ)"]
        C8["Constellation (BFYL)"]
        C9["Nova (BCCY)"]
        C10["Promo Code App"]
        C11["Loyalty Service (BGMC)"]
        
        C12["Homegrown Interface/EDAT"]
        C13["CMEE (BB8K)"]
        
        C14["BRL (B9XX)"]
        
        C15["SFTP"]
        C16["Google Storage"]
        C17["Pigeon (BFB6)"]
        C18["Marvel (BDMS)"]
        C19["Insight (BERY)"]
        
        C20["Posting API (TDS BFI4)"]
        C21["6G (BC6L)"]
        C22["CBT EDL (BB4J)"]
    end
    
    subgraph TARGET["Target State: Salesforce"]
        direction TB
        T1["Data Cloud<br/>Unified Profile"]
        T2["Data Cloud<br/>Streaming Ingestion"]
        T3["Data Cloud<br/>Calculated Insights"]
        T4["Data Cloud<br/>Identity Resolution"]
        T5["Data Cloud<br/>Data Actions"]
        
        T6["Loyalty Management<br/>Offer Engine"]
        T7["Loyalty Management<br/>Rewards Catalog"]
        T8["Loyalty Management<br/>Promotion Engine"]
        
        T9["Marketing Cloud<br/>Journey Builder"]
        T10["Marketing Cloud<br/>Automation Studio"]
        
        T11["MC Personalization<br/>Real-Time Decisioning"]
        T12["MC Personalization<br/>Web/Mobile SDK"]
        
        T13["MuleSoft<br/>API Integration"]
        T14["MuleSoft<br/>Event Processing"]
        
        T15["Agentforce<br/>AI Agents"]
    end
    
    C1 --> T1
    C2 --> T2
    C3 --> T3
    C4 --> T4
    C5 --> T5
    C6 --> T4
    
    C7 --> T6
    C8 --> T6
    C9 --> T6
    C10 --> T8
    C11 --> T7
    
    C12 --> T9
    C13 --> T10
    
    C14 --> T11
    
    C15 --> T13
    C16 --> T2
    C17 --> T14
    C18 --> T3
    C19 --> T3
    
    C20 --> T13
    C21 --> T2
    C22 --> T2
    
    style T1 fill:#0176d3,color:#fff
    style T6 fill:#2e844a,color:#fff
    style T9 fill:#9050e9,color:#fff
    style T11 fill:#ff6b6b,color:#fff
```

### Component-by-Component Migration Details

| Current System | Code | Function | Target System | Migration Approach |
|----------------|------|----------|---------------|-------------------|
| **CDP** | - | Central data platform | Data Cloud | Direct replacement with enhanced capabilities |
| **Event Exchange Hub** | BF8M | Real-time events | Data Cloud Streaming | Native streaming ingestion |
| **EDW** | BCJW | Data warehouse | Data Cloud + Tableau | Unified analytics layer |
| **Adaptor** | BHBD | Reconciliation | Data Cloud Identity Resolution | Automated matching |
| **Data Power** | BDLQ | Event tracing | Data Cloud Data Actions | Platform Events |
| **ScotiaLive CID** | BCJD | Customer ID lookup | Data Cloud Identity Resolution | Unified identity |
| **Orion** | BDQJ | Offer presentation | Loyalty Management | Offer engine with rules |
| **Constellation** | BFYL | Offer management | Loyalty Management | Centralized offer mgmt |
| **Nova** | BCCY | Offer delivery | Loyalty Management + MCP | Real-time delivery |
| **Promo Code App** | GCP | Promo codes | Loyalty Management Promotions | Native promo engine |
| **Loyalty Service** | BGMC | Loyalty fulfillment | Loyalty Management Rewards | Rewards catalog |
| **Homegrown Interface** | EDAT | Orchestration | Marketing Cloud Journey Builder | Visual journey design |
| **CMEE** | BB8K | Campaign execution | Marketing Cloud Automation | Automated campaigns |
| **BRL** | B9XX | Business rules | MC Personalization | Real-time decisioning |
| **SFTP** | - | File transfer | MuleSoft / Data Cloud Connectors | API-first integration |
| **Google Storage** | GCP | File storage | Data Cloud Ingestion API | Direct streaming |
| **Pigeon** | BFB6 | Data pipeline | MuleSoft + Data Cloud | Event-driven flow |
| **Marvel** | BDMS | Data management | Data Cloud | Unified data model |
| **Insight** | BERY | Analytics | Data Cloud Calculated Insights | Built-in analytics |
| **Posting API** | TDS BFI4 | Fulfillment | MuleSoft API | API orchestration |
| **6G** | BC6L | Data extraction | Data Cloud Ingestion | Native connectors |
| **CBT EDL** | BB4J | EDL ingestion | Data Cloud Batch Ingestion | Scheduled loads |

---

## Data Flow Diagrams

### 1. Customer Onboarding & Profile Creation

```mermaid
flowchart TD
    subgraph Sources["Data Sources"]
        CRM["CRM<br/>Customer Master"]
        CARDS["Card Systems<br/>Debit/Credit"]
        SCENE["Scene Points<br/>Loyalty Data"]
        DIGITAL["Digital Channels<br/>Web/Mobile Behavior"]
        TRANS["Transaction Systems<br/>D2D, Payments"]
    end
    
    subgraph DataCloud["Salesforce Data Cloud"]
        subgraph Ingestion["Ingestion Layer"]
            STREAM["Streaming API<br/>Real-time Events"]
            BATCH["Batch Ingestion<br/>Scheduled Loads"]
            CONNECT["Connectors<br/>Pre-built Integrations"]
        end
        
        subgraph Processing["Processing Layer"]
            DMO["Data Model Objects<br/>Unified Schema"]
            IDENTITY["Identity Resolution<br/>Match & Merge"]
            GRAPH["Data Graphs<br/>Relationships"]
        end
        
        subgraph Profile["Unified Profile"]
            INDIVIDUAL["Individual Profile<br/>Customer 360°"]
            ACCOUNT["Account Profile<br/>Household View"]
            INSIGHTS["Calculated Insights<br/>• CLV Score<br/>• Offer Propensity<br/>• Churn Risk"]
        end
        
        subgraph Activation["Activation Layer"]
            SEGMENTS["Dynamic Segments<br/>Real-time Membership"]
            ACTIONS["Data Actions<br/>Platform Events"]
        end
    end
    
    CRM --> CONNECT
    CARDS --> BATCH
    SCENE --> STREAM
    DIGITAL --> STREAM
    TRANS --> STREAM
    
    CONNECT --> DMO
    BATCH --> DMO
    STREAM --> DMO
    
    DMO --> IDENTITY
    IDENTITY --> GRAPH
    GRAPH --> INDIVIDUAL
    GRAPH --> ACCOUNT
    
    INDIVIDUAL --> INSIGHTS
    ACCOUNT --> INSIGHTS
    
    INSIGHTS --> SEGMENTS
    SEGMENTS --> ACTIONS
    
    style INDIVIDUAL fill:#0176d3,color:#fff
    style INSIGHTS fill:#2e844a,color:#fff
    style SEGMENTS fill:#9050e9,color:#fff
```

### 2. Offer Eligibility & Decisioning Flow

```mermaid
flowchart TD
    subgraph Trigger["Trigger Events"]
        WEB["Web Visit"]
        MOBILE["Mobile App Open"]
        ATM["ATM Interaction"]
        CALL["Contact Center"]
        TRANS["Transaction Event"]
    end
    
    subgraph MCP["MC Personalization"]
        SDK["Web/Mobile SDK<br/>Behavior Capture"]
        RT_PROFILE["Real-Time Profile<br/>Session Context"]
        DECISION["Einstein Decisioning<br/>ML-Powered"]
    end
    
    subgraph DataCloud["Data Cloud"]
        PROFILE["Unified Profile<br/>Historical Data"]
        SEGMENTS["Segment Membership<br/>• High Value<br/>• At Risk<br/>• New Customer"]
        PROPENSITY["Propensity Scores<br/>• Offer Acceptance<br/>• Product Interest"]
    end
    
    subgraph Loyalty["Loyalty Management"]
        ELIGIBILITY["Eligibility Engine<br/>• Points Balance<br/>• Tier Status<br/>• Redemption History"]
        OFFERS["Offer Catalog<br/>• Cash Bonus<br/>• Fee Rebates<br/>• Fee Waivers<br/>• Partner Offers"]
        RULES["Business Rules<br/>• Frequency Caps<br/>• Exclusions<br/>• Stacking Rules"]
        RANK["Offer Ranking<br/>Priority + ML"]
    end
    
    subgraph Output["Offer Delivery"]
        PERSONALIZED["Personalized Offers<br/>Top 3 Ranked"]
        CHANNEL["Channel Delivery<br/>Web/Mobile/ATM"]
    end
    
    WEB --> SDK
    MOBILE --> SDK
    ATM --> SDK
    CALL --> DECISION
    TRANS --> DECISION
    
    SDK --> RT_PROFILE
    RT_PROFILE --> DECISION
    
    DECISION <--> PROFILE
    PROFILE --> SEGMENTS
    PROFILE --> PROPENSITY
    
    DECISION --> ELIGIBILITY
    SEGMENTS --> ELIGIBILITY
    PROPENSITY --> ELIGIBILITY
    
    ELIGIBILITY --> OFFERS
    OFFERS --> RULES
    RULES --> RANK
    
    RANK --> PERSONALIZED
    PERSONALIZED --> CHANNEL
    
    style DECISION fill:#ff6b6b,color:#fff
    style ELIGIBILITY fill:#2e844a,color:#fff
    style PERSONALIZED fill:#0176d3,color:#fff
```

### 3. Offer Redemption & Fulfillment Flow

```mermaid
flowchart TD
    subgraph CustomerAction["Customer Action"]
        ACCEPT["Customer Accepts Offer"]
        REDEEM["Redemption Request"]
    end
    
    subgraph Loyalty["Loyalty Management"]
        VALIDATE["Validation<br/>• Still Eligible?<br/>• Not Expired?<br/>• Within Limits?"]
        PROCESS["Process Redemption<br/>• Deduct Points<br/>• Apply Benefit"]
        RECORD["Record Transaction<br/>• Redemption Log<br/>• Audit Trail"]
    end
    
    subgraph DataCloud["Data Cloud"]
        UPDATE["Profile Update<br/>Platform Event"]
        RECALC["Recalculate<br/>• Insights<br/>• Segments"]
    end
    
    subgraph Fulfillment["Fulfillment Layer"]
        subgraph Internal["Internal Fulfillment"]
            CASH["Cash Bonus<br/>Credit to Account"]
            FEE["Fee Rebate<br/>Posting API"]
            WAIVER["Fee Waiver<br/>Apply to Account"]
        end
        
        subgraph External["External Fulfillment"]
            BOND["Bond Partner<br/>External Rewards"]
            PARTNER["Partner Systems<br/>Third-Party Offers"]
        end
    end
    
    subgraph Notification["Notification Layer"]
        MC["Marketing Cloud<br/>Journey Trigger"]
        EMAIL["Email Confirmation"]
        SMS["SMS Notification"]
        PUSH["Push Notification"]
        INAPP["In-App Message"]
    end
    
    subgraph Reconciliation["Reconciliation"]
        TRACK["Offer Tracking<br/>Success/Failure"]
        REPORT["PowerBI Reports<br/>Analytics"]
        AUDIT["Audit Trail<br/>Compliance"]
    end
    
    ACCEPT --> VALIDATE
    VALIDATE -->|Valid| PROCESS
    VALIDATE -->|Invalid| REJECT["Reject with Reason"]
    
    PROCESS --> RECORD
    RECORD --> UPDATE
    
    UPDATE --> RECALC
    RECALC --> MC
    
    PROCESS --> CASH
    PROCESS --> FEE
    PROCESS --> WAIVER
    PROCESS --> BOND
    PROCESS --> PARTNER
    
    MC --> EMAIL
    MC --> SMS
    MC --> PUSH
    MC --> INAPP
    
    RECORD --> TRACK
    TRACK --> REPORT
    TRACK --> AUDIT
    
    style PROCESS fill:#2e844a,color:#fff
    style UPDATE fill:#0176d3,color:#fff
    style MC fill:#9050e9,color:#fff
```

### 4. Real-Time Event Processing Architecture

```mermaid
flowchart LR
    subgraph EventSources["Event Sources"]
        D2D["D2D Transactions"]
        DEBIT["Debit Card Events"]
        SCENE["Scene Point Posting"]
        MAINT["Card Maintenance"]
        DIGITAL["Digital Interactions"]
    end
    
    subgraph MuleSoft["MuleSoft Integration"]
        ANYPOINT["Anypoint Platform"]
        TRANSFORM["Data Transformation"]
        ROUTE["Event Routing"]
    end
    
    subgraph DataCloud["Data Cloud Streaming"]
        INGEST["Streaming Ingestion<br/>< 100ms latency"]
        PROCESS["Stream Processing<br/>Real-time Updates"]
        TRIGGER["Event Triggers<br/>Segment Entry/Exit"]
    end
    
    subgraph Actions["Automated Actions"]
        PLATFORM["Platform Events<br/>CRM Updates"]
        WEBHOOK["Webhooks<br/>External Callbacks"]
        FLOW["Flow Triggers<br/>Automation"]
    end
    
    subgraph Downstream["Downstream Systems"]
        LM["Loyalty Management<br/>Points Update"]
        MC["Marketing Cloud<br/>Journey Entry"]
        MCP["MC Personalization<br/>Profile Refresh"]
        AGENT["Agentforce<br/>Context Update"]
    end
    
    D2D --> ANYPOINT
    DEBIT --> ANYPOINT
    SCENE --> ANYPOINT
    MAINT --> ANYPOINT
    DIGITAL --> ANYPOINT
    
    ANYPOINT --> TRANSFORM
    TRANSFORM --> ROUTE
    ROUTE --> INGEST
    
    INGEST --> PROCESS
    PROCESS --> TRIGGER
    
    TRIGGER --> PLATFORM
    TRIGGER --> WEBHOOK
    TRIGGER --> FLOW
    
    PLATFORM --> LM
    PLATFORM --> MC
    PLATFORM --> MCP
    PLATFORM --> AGENT
    
    style INGEST fill:#0176d3,color:#fff
    style TRIGGER fill:#ff6b6b,color:#fff
```

### 5. Agentforce Integration for Offers

```mermaid
flowchart TD
    subgraph Channels["Customer Channels"]
        VOICE["📞 Voice<br/>Contact Center"]
        CHAT["💬 Chat<br/>Website/App"]
        SMS["📱 SMS<br/>Messaging"]
        EMAIL["📧 Email<br/>Responses"]
    end
    
    subgraph Agentforce["Agentforce Platform"]
        AGENT["AI Agent<br/>Offer Specialist"]
        NLU["Natural Language<br/>Understanding"]
        TOPICS["Agent Topics<br/>• Check Offers<br/>• Redeem Offer<br/>• Offer History<br/>• Eligibility"]
        ACTIONS["Agent Actions"]
    end
    
    subgraph DataCloud["Data Cloud Context"]
        PROFILE["Customer Profile<br/>Real-time Fetch"]
        HISTORY["Interaction History<br/>Previous Offers"]
        SEGMENTS["Segment Membership<br/>Eligibility Context"]
    end
    
    subgraph Loyalty["Loyalty Management Actions"]
        CHECK["Check Eligible Offers<br/>→ List Available"]
        APPLY["Apply Offer<br/>→ Process Redemption"]
        STATUS["Check Status<br/>→ Redemption Status"]
        HISTORY2["Get History<br/>→ Past Redemptions"]
    end
    
    subgraph Response["Agent Response"]
        GENERATE["Response Generation<br/>Einstein GPT"]
        PERSONALIZE["Personalization<br/>Customer Context"]
        DELIVER["Multi-Channel Delivery"]
    end
    
    VOICE --> AGENT
    CHAT --> AGENT
    SMS --> AGENT
    EMAIL --> AGENT
    
    AGENT --> NLU
    NLU --> TOPICS
    TOPICS --> ACTIONS
    
    ACTIONS <--> PROFILE
    PROFILE --> HISTORY
    PROFILE --> SEGMENTS
    
    ACTIONS --> CHECK
    ACTIONS --> APPLY
    ACTIONS --> STATUS
    ACTIONS --> HISTORY2
    
    CHECK --> GENERATE
    APPLY --> GENERATE
    STATUS --> GENERATE
    HISTORY2 --> GENERATE
    
    SEGMENTS --> PERSONALIZE
    GENERATE --> PERSONALIZE
    PERSONALIZE --> DELIVER
    
    style AGENT fill:#0176d3,color:#fff
    style ACTIONS fill:#2e844a,color:#fff
    style GENERATE fill:#9050e9,color:#fff
```

---

## Integration Patterns

### 1. Data Ingestion Patterns

```mermaid
flowchart TD
    subgraph Patterns["Data Ingestion Patterns"]
        subgraph Streaming["Pattern 1: Real-Time Streaming"]
            S1["Event Source"]
            S2["MuleSoft Anypoint"]
            S3["Data Cloud Ingestion API"]
            S4["Real-Time Profile Update"]
            S1 --> S2 --> S3 --> S4
        end
        
        subgraph Batch["Pattern 2: Batch Processing"]
            B1["Source System"]
            B2["Scheduled Extract"]
            B3["Data Cloud Batch API"]
            B4["Bulk Profile Update"]
            B1 --> B2 --> B3 --> B4
        end
        
        subgraph ZeroCopy["Pattern 3: Zero Copy"]
            Z1["Data Lake<br/>(Snowflake/Databricks)"]
            Z2["Zero Copy Partner"]
            Z3["Query Federation"]
            Z4["Virtual Access"]
            Z1 --> Z2 --> Z3 --> Z4
        end
        
        subgraph Connector["Pattern 4: Native Connectors"]
            C1["Cloud Apps<br/>(AWS S3, GCS, Azure)"]
            C2["Pre-built Connector"]
            C3["Automated Sync"]
            C4["Unified Data Model"]
            C1 --> C2 --> C3 --> C4
        end
    end
    
    style S4 fill:#ff6b6b,color:#fff
    style B4 fill:#2e844a,color:#fff
    style Z4 fill:#0176d3,color:#fff
    style C4 fill:#9050e9,color:#fff
```

### 2. Integration Architecture

```mermaid
flowchart TB
    subgraph External["External Systems"]
        LEGACY["Legacy Systems<br/>Posting API, EDW"]
        PARTNERS["Partners<br/>Bond, Vendors"]
        CLOUD["Cloud Services<br/>AWS, GCP, Azure"]
    end
    
    subgraph Integration["MuleSoft Integration Layer"]
        ANYPOINT["Anypoint Platform"]
        subgraph APIs["API Layer"]
            EXP["Experience APIs<br/>Channel-specific"]
            PROC["Process APIs<br/>Business Logic"]
            SYS["System APIs<br/>System-specific"]
        end
        subgraph Connectors["Connectors"]
            SF["Salesforce Connector"]
            DB["Database Connectors"]
            FILE["File Connectors"]
            HTTP["HTTP/REST"]
        end
    end
    
    subgraph Salesforce["Salesforce Platform"]
        DC["Data Cloud"]
        LM["Loyalty Management"]
        MC["Marketing Cloud"]
        MCP["MC Personalization"]
        CRM["Sales/Service Cloud"]
    end
    
    LEGACY --> SYS
    PARTNERS --> SYS
    CLOUD --> SYS
    
    SYS --> PROC
    PROC --> EXP
    
    EXP --> SF
    SF --> DC
    SF --> LM
    SF --> MC
    SF --> MCP
    SF --> CRM
    
    style ANYPOINT fill:#00a1e0,color:#fff
    style DC fill:#0176d3,color:#fff
```

### 3. Security & Tokenization Pattern

```mermaid
flowchart TD
    subgraph Inbound["Inbound Data"]
        RAW["Raw PII Data<br/>SSN, Account Numbers"]
    end
    
    subgraph Tokenization["Tokenization Layer"]
        TOKEN["Tokenization Service<br/>Replace PII with Tokens"]
        VAULT["Token Vault<br/>Secure Storage"]
    end
    
    subgraph DataCloud["Data Cloud"]
        STORE["Store Tokenized Data<br/>Safe for Processing"]
        PROCESS["Process & Analyze<br/>Using Tokens"]
    end
    
    subgraph Detokenization["Detokenization (When Needed)"]
        REQUEST["Authorized Request<br/>With Audit"]
        DETOKEN["Detokenization<br/>Retrieve Original"]
        USE["Limited Use<br/>Display/Fulfill"]
    end
    
    RAW --> TOKEN
    TOKEN --> VAULT
    TOKEN --> STORE
    STORE --> PROCESS
    
    PROCESS --> REQUEST
    REQUEST --> VAULT
    VAULT --> DETOKEN
    DETOKEN --> USE
    
    style VAULT fill:#ff6b6b,color:#fff
    style STORE fill:#0176d3,color:#fff
```

---

## Migration Roadmap

### Phase Overview

```mermaid
gantt
    title Scotia Pega Replacement - Migration Roadmap
    dateFormat  YYYY-MM-DD
    
    section Phase 1: Foundation
    Data Cloud Setup & Configuration    :p1a, 2024-01-01, 60d
    Identity Resolution Implementation  :p1b, after p1a, 45d
    Core Data Model & DMOs              :p1c, after p1a, 45d
    MuleSoft Integration Layer          :p1d, 2024-01-15, 75d
    
    section Phase 2: Loyalty & Offers
    Loyalty Management Setup            :p2a, after p1b, 45d
    Offer Catalog Migration             :p2b, after p2a, 30d
    Eligibility Rules Configuration     :p2c, after p2a, 30d
    Promo Code Migration                :p2d, after p2b, 21d
    
    section Phase 3: Marketing & Personalization
    Marketing Cloud Integration         :p3a, after p2b, 45d
    Journey Builder Configuration       :p3b, after p3a, 30d
    MC Personalization Setup            :p3c, after p2c, 45d
    Real-Time Decisioning               :p3d, after p3c, 30d
    
    section Phase 4: Agentforce & AI
    Agentforce Agent Setup              :p4a, after p3b, 30d
    Agent Actions Development           :p4b, after p4a, 21d
    Einstein Integration                :p4c, after p4b, 21d
    
    section Phase 5: Cutover & Optimization
    Parallel Run                        :p5a, after p4c, 30d
    Legacy Decommission                 :p5b, after p5a, 30d
    Optimization & Tuning               :p5c, after p5b, 45d
```

### Phase Details

#### Phase 1: Foundation (Months 1-4)

| Workstream | Activities | Systems Retired |
|------------|------------|-----------------|
| **Data Cloud Setup** | • Org configuration<br/>• Data streams setup<br/>• Ingestion APIs | - |
| **Identity Resolution** | • Match rules<br/>• Merge policies<br/>• CID unification | ScotiaLive CID (BCJD), Adaptor (BHBD) |
| **Data Model** | • DMO design<br/>• Calculated Insights<br/>• Segments | EDW (BCJW), Insight (BERY) |
| **MuleSoft** | • API design<br/>• Connectors<br/>• Event routing | SFTP, Pigeon (BFB6) |

#### Phase 2: Loyalty & Offers (Months 4-7)

| Workstream | Activities | Systems Retired |
|------------|------------|-----------------|
| **Loyalty Setup** | • Programs configuration<br/>• Tiers & benefits<br/>• Points rules | Loyalty Service (BGMC) |
| **Offer Migration** | • Offer catalog<br/>• Eligibility rules<br/>• Stacking rules | Orion (BDQJ), Constellation (BFYL), Nova (BCCY) |
| **Promo Engine** | • Promo codes<br/>• Campaign offers<br/>• Redemption logic | Promo Code App (GCP) |

#### Phase 3: Marketing & Personalization (Months 6-9)

| Workstream | Activities | Systems Retired |
|------------|------------|-----------------|
| **Marketing Cloud** | • Journey design<br/>• Automation rules<br/>• Email/SMS templates | Homegrown Interface (EDAT), CMEE (BB8K) |
| **MC Personalization** | • Real-time decisioning<br/>• Web/Mobile SDK<br/>• Offer ranking | BRL (B9XX) |

#### Phase 4: Agentforce & AI (Months 9-11)

| Workstream | Activities | Systems Retired |
|------------|------------|-----------------|
| **Agentforce** | • Agent topics<br/>• Custom actions<br/>• Channel integration | Manual processes |
| **Einstein AI** | • Propensity models<br/>• Next Best Offer<br/>• Churn prediction | - |

#### Phase 5: Cutover & Optimization (Months 11-14)

| Workstream | Activities | Systems Retired |
|------------|------------|-----------------|
| **Parallel Run** | • Dual processing<br/>• Data validation<br/>• Performance testing | - |
| **Decommission** | • Legacy shutdown<br/>• Data archival<br/>• Final migration | All remaining legacy |
| **Optimization** | • Performance tuning<br/>• ML model training<br/>• Process refinement | - |

---

## Technical Implementation Details

### Data Model Objects (DMOs)

```yaml
# Core DMOs for Scotia Offers

Individual_Profile__dlm:
  description: "Unified customer profile"
  fields:
    - ScotiaCustomerId__c (Primary Key)
    - FirstName__c
    - LastName__c
    - Email__c
    - Phone__c
    - DateOfBirth__c
    - CustomerSince__c
    - Tier__c (Gold, Platinum, etc.)
    - TotalPointsBalance__c
    - LifetimeValue__c
    - ChurnRiskScore__c
    - OfferPropensityScore__c

Account_Profile__dlm:
  description: "Household/account view"
  fields:
    - AccountId__c (Primary Key)
    - AccountType__c
    - AccountStatus__c
    - OpenDate__c
    - ProductsHeld__c
    - HouseholdId__c

Transaction_Event__dlm:
  description: "Transaction events"
  fields:
    - TransactionId__c (Primary Key)
    - ScotiaCustomerId__c (FK)
    - TransactionType__c (D2D, Debit, Scene, etc.)
    - Amount__c
    - Timestamp__c
    - MerchantCategory__c
    - Channel__c

Offer_Interaction__dlm:
  description: "Offer events"
  fields:
    - InteractionId__c (Primary Key)
    - ScotiaCustomerId__c (FK)
    - OfferId__c
    - InteractionType__c (Presented, Clicked, Accepted, Rejected)
    - Timestamp__c
    - Channel__c
    - Outcome__c
```

### Calculated Insights Configuration

```yaml
# Key Calculated Insights

Offer_Propensity_Score:
  type: "Calculated Insight"
  description: "Likelihood to accept an offer"
  formula: |
    WEIGHTED_AVERAGE(
      HistoricalAcceptanceRate * 0.4,
      EngagementScore * 0.3,
      RecencyScore * 0.2,
      FrequencyScore * 0.1
    )
  refresh: "Real-time"
  
Customer_Lifetime_Value:
  type: "Calculated Insight"
  description: "Predicted lifetime value"
  formula: |
    SUM(TransactionValue, Last12Months) * 
    RetentionProbability * 
    AvgCustomerLifespan
  refresh: "Daily"
  
Churn_Risk_Score:
  type: "Calculated Insight"
  description: "Risk of customer churn"
  formula: |
    ML_PREDICTION(
      features: [
        DaysSinceLastTransaction,
        TransactionFrequencyTrend,
        EngagementDecline,
        SupportTicketsSentiment
      ]
    )
  refresh: "Daily"
```

### Segment Definitions

```yaml
# Dynamic Segments

High_Value_Customers:
  criteria:
    - LifetimeValue >= 10000
    - Tier IN ('Platinum', 'Infinite')
  refresh: "Real-time"
  
Offer_Ready_Customers:
  criteria:
    - DaysSinceLastOffer >= 7
    - OfferPropensityScore >= 0.7
    - HasActiveAccount = TRUE
  refresh: "Real-time"
  
At_Risk_Customers:
  criteria:
    - ChurnRiskScore >= 0.6
    - DaysSinceLastTransaction >= 30
  refresh: "Daily"
  
New_Customer_Onboarding:
  criteria:
    - CustomerSince >= TODAY - 30
    - OnboardingComplete = FALSE
  refresh: "Real-time"
```

### Loyalty Management Configuration

```yaml
# Loyalty Program Setup

Program: Scotia_Rewards
  tiers:
    - name: "Basic"
      threshold: 0
      benefits:
        - BaseCashbackRate: 0.5%
        
    - name: "Gold"
      threshold: 5000
      benefits:
        - BaseCashbackRate: 1.0%
        - FeeWaivers: ["Monthly Fee"]
        
    - name: "Platinum"
      threshold: 15000
      benefits:
        - BaseCashbackRate: 1.5%
        - FeeWaivers: ["Monthly Fee", "ATM Fee"]
        - PrioritySupport: true
        
    - name: "Infinite"
      threshold: 50000
      benefits:
        - BaseCashbackRate: 2.0%
        - FeeWaivers: ["All Fees"]
        - PrioritySupport: true
        - DedicatedAdvisor: true

Offer_Types:
  - type: "Cash Bonus"
    fulfillment: "Direct Credit"
    integration: "Posting API"
    
  - type: "Fee Rebate"
    fulfillment: "Account Credit"
    integration: "Posting API"
    
  - type: "Fee Waiver"
    fulfillment: "System Flag"
    integration: "Account System"
    
  - type: "Partner Reward"
    fulfillment: "External"
    integration: "Bond API"
```

### MuleSoft Integration Specifications

```yaml
# API Specifications

Experience_APIs:
  - name: "Offer Experience API"
    path: /v1/offers
    operations:
      - GET /eligible: "Get eligible offers for customer"
      - POST /redeem: "Redeem an offer"
      - GET /history: "Get offer history"
    security: OAuth2, mTLS
    rate_limit: 1000/min
    
Process_APIs:
  - name: "Offer Eligibility API"
    path: /v1/eligibility
    operations:
      - POST /check: "Check offer eligibility"
      - POST /validate: "Validate redemption"
    integration: Loyalty Management
    
  - name: "Fulfillment API"
    path: /v1/fulfillment
    operations:
      - POST /cash-bonus: "Process cash bonus"
      - POST /fee-rebate: "Process fee rebate"
      - POST /partner-reward: "Fulfill partner reward"
    integration: Posting API, Bond
    
System_APIs:
  - name: "Transaction Events API"
    path: /v1/events
    operations:
      - POST /transaction: "Ingest transaction event"
      - POST /card-event: "Ingest card event"
    target: Data Cloud Streaming Ingestion
    
  - name: "Bond Integration API"
    path: /v1/bond
    operations:
      - POST /fulfill: "Fulfill Bond reward"
      - GET /status: "Check fulfillment status"
    target: Bond Partner System
```

### Agentforce Configuration

```yaml
# Agentforce Agent Setup

Agent: Scotia_Offer_Agent
  description: "AI agent for offer inquiries and redemption"
  
  Topics:
    - name: "Check_Offers"
      description: "View available offers"
      trigger_phrases:
        - "What offers do I have?"
        - "Show me my offers"
        - "Any promotions available?"
      actions:
        - GetEligibleOffers
        - FormatOfferResponse
        
    - name: "Redeem_Offer"
      description: "Redeem a specific offer"
      trigger_phrases:
        - "I want to redeem"
        - "Apply this offer"
        - "Use my reward"
      actions:
        - ValidateOffer
        - ProcessRedemption
        - ConfirmRedemption
        
    - name: "Offer_Status"
      description: "Check redemption status"
      trigger_phrases:
        - "Status of my offer"
        - "Did my reward apply?"
        - "Check my redemption"
      actions:
        - GetRedemptionStatus
        - FormatStatusResponse
        
    - name: "Offer_History"
      description: "View past offers and redemptions"
      trigger_phrases:
        - "My offer history"
        - "Past rewards"
        - "Previous redemptions"
      actions:
        - GetOfferHistory
        - FormatHistoryResponse
        
  Actions:
    - name: "GetEligibleOffers"
      type: "Data Cloud Query"
      query: |
        SELECT OfferId, OfferName, OfferValue, ExpiryDate
        FROM Offer_Eligibility__dlm
        WHERE CustomerId = {context.customerId}
        AND Status = 'Active'
        ORDER BY Priority DESC
        LIMIT 5
        
    - name: "ProcessRedemption"
      type: "Flow Invocation"
      flow: "Offer_Redemption_Flow"
      inputs:
        - customerId
        - offerId
        - channel
      outputs:
        - redemptionId
        - status
        - confirmationMessage
```

---

## Key Benefits & ROI

### Quantified Benefits

```mermaid
graph LR
    subgraph Before["Current State Metrics"]
        B1["⏱️ 30-50 days<br/>Vendor Onboarding"]
        B2["❌ No Real-Time<br/>Offer Presentation"]
        B3["📊 20+ Systems<br/>To Maintain"]
        B4["🔄 Manual<br/>Reconciliation"]
        B5["📉 Low Offer<br/>Acceptance Rate"]
    end
    
    subgraph After["Target State Metrics"]
        A1["⚡ 3-5 days<br/>Vendor Onboarding"]
        A2["✅ < 200ms<br/>Real-Time Decisioning"]
        A3["🎯 4 Integrated<br/>Platforms"]
        A4["🤖 Automated<br/>Reconciliation"]
        A5["📈 3x Higher<br/>Acceptance Rate"]
    end
    
    B1 --> A1
    B2 --> A2
    B3 --> A3
    B4 --> A4
    B5 --> A5
    
    style A1 fill:#2e844a,color:#fff
    style A2 fill:#2e844a,color:#fff
    style A3 fill:#2e844a,color:#fff
    style A4 fill:#2e844a,color:#fff
    style A5 fill:#2e844a,color:#fff
```

### ROI Summary

| Metric | Current | Target | Improvement |
|--------|---------|--------|-------------|
| **Vendor Onboarding Time** | 30-50 days | 3-5 days | 90% reduction |
| **Offer Decisioning Latency** | Batch (hours) | < 200ms | Real-time |
| **Systems to Maintain** | 20+ | 4 | 80% reduction |
| **Offer Acceptance Rate** | ~5% | ~15% | 3x increase |
| **Operational Cost** | Baseline | -40% | Significant savings |
| **Time to Market (New Offers)** | Weeks | Days | 5x faster |
| **Customer Satisfaction** | Baseline | +25 NPS | Improved CX |

### Strategic Value

1. **Unified Customer View**
   - Single source of truth for customer data
   - Real-time profile updates
   - 360° view across all touchpoints

2. **AI-Powered Personalization**
   - Einstein-driven offer recommendations
   - Propensity scoring
   - Next Best Action

3. **Agentforce Enablement**
   - Natural language offer inquiries
   - Automated redemption processing
   - Reduced contact center load

4. **Future-Ready Architecture**
   - Composable, API-first design
   - Easy partner integrations
   - Scalable for new channels

---

## Appendix

### A. Glossary

| Term | Definition |
|------|------------|
| **DMO** | Data Model Object - Data Cloud's unified data schema |
| **MCP** | Marketing Cloud Personalization |
| **CDP** | Customer Data Platform (current state) |
| **D2D** | Day-to-Day transactions |
| **Scene** | Scotia's loyalty points program |
| **Bond** | External partner for reward fulfillment |
| **EDAT** | Current orchestration system |
| **CID** | Customer Identifier |

### B. Related Documents

- [Data Cloud Design Patterns](../data-cloud/design-patterns/README_DataCloud_Patterns.md)
- [Agentforce Architecture](../agentforce/agent-graph/README_Agentforce_Graph_Architecture_Flow.md)
- [Integration Patterns](../salesforce-integration-architecture-patterns.md)

### C. Contact

For questions about this architecture:
- **Solution Architect**: [Your Name]
- **Technical Lead**: [Technical Lead]
- **Project Manager**: [PM Name]

---

*Document Version: 1.0*
*Last Updated: January 2025*
*Status: Draft for Review*
