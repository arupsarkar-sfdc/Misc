# Salesforce Data Cloud Clean Rooms: Technical Architecture Guide

## Executive Summary

Data Clean Rooms enable **privacy-preserving collaboration** between companies (like Nike and Hulu) to analyze audience overlap and activate shared customer segments **without sharing raw customer data**. All analysis happens in a secure, federated environment where only **aggregated, anonymized insights** are returned.

---

## Core Architecture Overview

```mermaid
graph TB
    subgraph "Sell-Side: Nike"
        NIKE_DC[Nike Data Cloud]
        NIKE_SEG["Nike Customer Segments<br/>• Running enthusiasts<br/>• Nike+ app users<br/>• Purchase history"]
        NIKE_CONTACT["Nike Contact Graph<br/>• Hashed emails<br/>• Device IDs<br/>• UID2 tokens"]
    end
    
    subgraph "Clean Room Environment"
        TEMPLATE["Clean Room Template<br/>• Overlap analysis<br/>• Activation rules<br/>• Query constraints"]
        FEDERATED["Federated Query Engine<br/>• No data movement<br/>• Privacy preserving<br/>• Aggregate results only"]
        COLLAB["Clean Room Collaboration<br/>Nike + Hulu overlap<br/>Secure computation"]
    end
    
    subgraph "Buy-Side: Hulu"
        HULU_DC[Hulu Data Cloud]
        HULU_SEG["Hulu Viewer Segments<br/>• Sports content viewers<br/>• Demographics: 18-35<br/>• Streaming behavior"]
        HULU_CONTACT["Hulu Contact Graph<br/>• Hashed emails<br/>• Device IDs<br/>• UID2 tokens"]
    end
    
    subgraph "AppExchange Marketplace"
        TEMPLATES["Pre-built Templates<br/>• Hulu Overlap<br/>• Hulu Activation<br/>• Campaign Performance"]
    end
    
    NIKE_DC --> NIKE_SEG
    NIKE_DC --> NIKE_CONTACT
    HULU_DC --> HULU_SEG
    HULU_DC --> HULU_CONTACT
    
    TEMPLATES --> TEMPLATE
    TEMPLATE --> COLLAB
    NIKE_SEG --> COLLAB
    HULU_SEG --> COLLAB
    NIKE_CONTACT --> FEDERATED
    HULU_CONTACT --> FEDERATED
    FEDERATED --> COLLAB
    
    style NIKE_DC fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style NIKE_SEG fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style NIKE_CONTACT fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style HULU_DC fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px
    style HULU_SEG fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style HULU_CONTACT fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style TEMPLATE fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style FEDERATED fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style COLLAB fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style TEMPLATES fill:#fff3e0,stroke:#f57c00,stroke-width:2px
```

---

## Key Principles

### 1. No Raw Data Movement
- Data **stays in source Data Cloud**
- Only hashed, anonymized identifiers are used
- Federated queries run across both environments

### 2. Privacy-First Design
- **K-anonymity** enforced (minimum 1,000 matches)
- **Differential privacy** applied
- Only **aggregate results** returned

### 3. Use Case: Nike × Hulu Collaboration

**Nike's Goal**: Target Hulu subscribers who match Nike's running enthusiast profile

**Hulu's Goal**: Monetize their audience data while protecting subscriber privacy

**Solution**: Clean Room analyzes overlap without exposing individual user data

---

## Data Structure

### Sell-Side: Nike Data
```mermaid
graph LR
    Provider["Data Provider Hulu<br/>Owns: Audience segments<br/>Provides: Targeting capability"]
    Consumer["Data Consumer Nike<br/>Wants: Audience reach<br/>Consumes: Targeting data"]
    
    Provider -->|Clean Room| Consumer
    
    style Provider fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px
    style Consumer fill:#e3f2fd,stroke:#1976d2,stroke-width:3px

```mermaid
graph LR

    subgraph "Nike DMO Structure"
        SEG_META["Segment Metadata<br/>━━━━━━━━━━━━━━<br/>segmentID: 77233<br/>segmentName: Nike Running<br/>category: Product Affinity"]
        
        SEG_MEMBER["Segment Membership<br/>━━━━━━━━━━━━━━<br/>77233 → hafyernisf21iuwkl<br/>77233 → 4iuwklhafy3joa;i3<br/>77233 → ifnqadflkjo1pojap"]
        
        CONTACT["Contact Graph<br/>━━━━━━━━━━━━━━<br/>hafyernisf21iuwkl → UID2<br/>4iuwklhafy3joa;i3 → RampID<br/>ifnqadflkjo1pojap → DEVICE_IDFA"]
    end
    
    SEG_META --> SEG_MEMBER
    SEG_MEMBER --> CONTACT
    
    style SEG_META fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style SEG_MEMBER fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style CONTACT fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
```

**Sample Nike Data:**

| Nike Customer ID | Hashed Email | Purchase History |
|-----------------|--------------|------------------|
| NC-1001 | f5f1c905... | Running shoes, Nike+ App |
| NC-1002 | d7a5b3f2... | Sports bra, Leggings |
| NC-1003 | a1b2c3d4... | Water bottle |
| NC-1004 | e9f8g7h6... | Training gear |
| NC-1005 | e8e7c6d5... | Running shoes |

### Buy-Side: Hulu Data

```mermaid
graph LR
    subgraph "Hulu DMO Structure"
        H_SEG_META["Segment Metadata<br/>━━━━━━━━━━━━━━<br/>112233: 18-35 Age<br/>445566: Sports Content<br/>556677: Fitness Shows"]
        
        H_SEG_MEMBER["Segment Membership<br/>━━━━━━━━━━━━━━<br/>112233 → iuwklhafyernisf21<br/>445566 → pojapifnqadflkjo1<br/>556677 → joa;i34iuwklhafy3"]
        
        H_CONTACT["Contact Graph<br/>━━━━━━━━━━━━━━<br/>iuwklhafyernisf21 → UID2<br/>pojapifnqadflkjo1 → DEVICE_IDFA<br/>joa;i34iuwklhafy3 → HEM"]
    end
    
    H_SEG_META --> H_SEG_MEMBER
    H_SEG_MEMBER --> H_CONTACT
    
    style H_SEG_META fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H_SEG_MEMBER fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H_CONTACT fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
```

**Sample Hulu Data:**

| Hulu Subscriber ID | Hashed Email | Content Watched |
|-------------------|--------------|-----------------|
| HS-2001 | d7a5b3f2... | Sports documentaries |
| HS-2002 | z4y3x2w1... | Reality TV shows |
| HS-2003 | a1b2c3d4... | Sci-fi films |
| HS-2004 | p1o2i3u4... | Sitcoms |

---

## Matching Process: How It Works

### The Secret: Hashed Email Matching

```mermaid
graph TB
    subgraph "Nike Data"
        N1["NC-1002<br/>hash: d7a5b3f2...<br/>Product: Sports bra"]
        N2["NC-1003<br/>hash: a1b2c3d4...<br/>Product: Water bottle"]
        N3["NC-1001<br/>hash: f5f1c905...<br/>Product: Running shoes"]
    end
    
    subgraph "Clean Room Matching"
        MATCH["Privacy-Preserving Join<br/>━━━━━━━━━━━━━━<br/>ON hashed_email<br/>K-anonymity ≥ 1000<br/>No individual IDs exposed"]
    end
    
    subgraph "Hulu Data"
        H1["HS-2001<br/>hash: d7a5b3f2...<br/>Content: Sports docs"]
        H2["HS-2003<br/>hash: a1b2c3d4...<br/>Content: Sci-fi films"]
        H3["HS-2002<br/>hash: z4y3x2w1...<br/>Content: Reality TV"]
    end
    
    N1 --> MATCH
    N2 --> MATCH
    N3 --> MATCH
    H1 --> MATCH
    H2 --> MATCH
    H3 --> MATCH
    
    MATCH --> RESULT["Match Results<br/>━━━━━━━━━━━━━━<br/>Total Overlap: 2<br/>hash456 ✓<br/>hash789 ✓<br/><br/>Aggregate Only!"]
    
    style N1 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style N2 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style N3 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style H1 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H2 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H3 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style MATCH fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style RESULT fill:#fff3e0,stroke:#f57c00,stroke-width:3px
```

**What Nike Receives:**
- ✅ Total overlap count: **2 matches**
- ✅ Matched hashed IDs: `d7a5b3f2...`, `a1b2c3d4...`
- ✅ Aggregate insights: "Sports documentaries" + "Sci-fi films" viewers
- ❌ **NO** Hulu Subscriber IDs
- ❌ **NO** Individual viewing history
- ❌ **NO** Raw data

---

## Complete Clean Room Process

### End-to-End Sequence Flow

```mermaid
sequenceDiagram
    participant Nike as Nike Data Cloud
    participant AppEx as AppExchange
    participant CR as Clean Room Engine
    participant Hulu as Hulu Data Cloud
    participant Ad as Hulu Ad Platform
    
    rect rgb(230, 240, 255)
    Note over Nike,Ad: Phase 1: Template Discovery
    Nike->>AppEx: Discover Hulu Overlap Template
    AppEx-->>Nike: Template HULU_OVERLAP_V1<br/>Query constraints, privacy rules
    Nike->>CR: Install template & configure
    Hulu->>CR: Accept collaboration invitation
    end
    
    rect rgb(243, 229, 245)
    Note over Nike,Ad: Phase 2: Data Contribution (No Raw Data)
    Nike->>CR: Contribute Segment Metadata<br/>segmentID 77233
    Nike->>CR: Contribute Contact Graph<br/>Hashed: UID2, IDFA, RampID
    Hulu->>CR: Contribute Segment Metadata<br/>Multiple segments
    Hulu->>CR: Contribute Contact Graph<br/>Hashed: UID2, IDFA, HEM
    end
    
    rect rgb(232, 245, 233)
    Note over Nike,Ad: Phase 3: Privacy-Preserving Analysis
    CR->>CR: Execute Federated Query<br/>JOIN on matchKeyID<br/>Apply k≥1000 threshold
    CR->>CR: Generate Overlap Results<br/>Aggregates only
    CR-->>Nike: Overlap Results<br/>Sports: 285K match<br/>18-35: 420K match
    CR-->>Hulu: Overlap Results<br/>Your overlap: 23.8%<br/>Revenue opportunity
    end
    
    rect rgb(255, 243, 224)
    Note over Nike,Ad: Phase 4: Activation
    Nike->>CR: Request activation<br/>Campaign: Running Shoe Launch
    CR->>CR: Execute Activation Query<br/>Generate anonymized tokens<br/>Validate consent
    CR->>Ad: Send Activation Results<br/>Anonymized tokens only
    Ad->>Ad: Create targetable audience<br/>Nike x Hulu Sports Overlap
    end
    
    rect rgb(255, 235, 238)
    Note over Nike,Ad: Phase 5: Campaign Execution
    Nike->>Ad: Launch campaign<br/>Target overlapped audience
    Ad-->>Nike: Performance metrics<br/>Aggregated, no individual data
    end
```

---

## Federated Query Example

### SQL Query in Clean Room

```sql
-- This runs in the secure clean room environment
-- Data NEVER leaves source systems

SELECT
    buyer.segmentID,
    buyer.segmentName,
    buyer.category,
    COUNT(DISTINCT seller.matchKeyID) as matched_population,
    COUNT(DISTINCT buyer.matchKeyID) as total_population
FROM buyer_segments buyer
JOIN seller_segments seller
    ON buyer.matchKeyID = seller.matchKeyID
WHERE 
    buyer.segmentID IN (112233, 445566)  -- Hulu segments
    AND seller.segmentID = 77233          -- Nike segment
GROUP BY 
    buyer.segmentID, 
    buyer.segmentName, 
    buyer.category
HAVING 
    COUNT(DISTINCT seller.matchKeyID) >= 1000  -- Privacy threshold
```

### Query Results (Aggregate Only)

```json
{
  "query_results": {
    "112233": {
      "segment_name": "Sports Content Viewers",
      "category": "Content Affinity",
      "matched_population": 285000,
      "total_population": 1200000,
      "overlap_rate": "23.8%"
    },
    "445566": {
      "segment_name": "18-35 Demographics",
      "category": "Demographics",
      "matched_population": 420000,
      "total_population": 2100000,
      "overlap_rate": "20.0%"
    }
  }
}
```

---

## Complete Technical Architecture

```mermaid
graph TB
    subgraph "Nike Data Cloud - Sell Side"
        N_SEG["Segment Metadata DMO<br/>━━━━━━━━━━━━━━<br/>segmentID: 77233<br/>Nike Running Enthusiasts<br/>category: Product Affinity"]
        N_MEM["Segment Membership DMO<br/>━━━━━━━━━━━━━━<br/>Member IDs linked to segment"]
        N_CONTACT["Contact Graph DMO<br/>━━━━━━━━━━━━━━<br/>memberID → UID2<br/>memberID → RampID<br/>memberID → DEVICE_IDFA"]
    end
    
    subgraph "Hulu Data Cloud - Buy Side"
        H_SEG["Segment Metadata DMO<br/>━━━━━━━━━━━━━━<br/>112233: 18-35 Age<br/>445566: Sports Content<br/>556677: Fitness Shows"]
        H_MEM["Segment Membership DMO<br/>━━━━━━━━━━━━━━<br/>Members per segment"]
        H_CONTACT["Contact Graph DMO<br/>━━━━━━━━━━━━━━<br/>memberID → UID2<br/>memberID → DEVICE_IDFA<br/>memberID → HEM"]
    end
    
    subgraph "Clean Room Environment"
        TEMPLATE["AppExchange Templates<br/>━━━━━━━━━━━━━━<br/>Hulu Overlap Template<br/>Min threshold: 1000<br/>Join on: matchKeyID<br/>Privacy: Aggregate only"]
        
        FED["Federated Query Engine<br/>━━━━━━━━━━━━━━<br/>Privacy-Preserving<br/>Data stays in source<br/>Multi-party computation<br/>Differential privacy<br/>K-anonymity k≥1000"]
        
        OVERLAP["Overlap Results DMO<br/>━━━━━━━━━━━━━━<br/>segmentID: 112233<br/>matched: 285K<br/>total: 1.2M<br/>overlap: 23.8%"]
        
        ACTIVATION["Activation Results DMO<br/>━━━━━━━━━━━━━━<br/>Anonymized tokens<br/>matchKeyID: AB1234CD...<br/>matchKeyID: e3dc577c...<br/>Status: Approved<br/>Consent: Validated"]
    end
    
    subgraph "Hulu Activation"
        AD["Hulu Ad Platform<br/>━━━━━━━━━━━━━━<br/>Receives anonymized IDs<br/>Creates targetable segment<br/>Nike Running Campaign<br/>Expected reach: 285K"]
        
        EXTERNAL["External Platforms<br/>━━━━━━━━━━━━━━<br/>TheTradeDesk<br/>Disney+<br/>Connected TV<br/>All anonymized tokens"]
    end
    
    N_SEG --> FED
    N_MEM --> FED
    N_CONTACT --> FED
    H_SEG --> FED
    H_MEM --> FED
    H_CONTACT --> FED
    TEMPLATE --> FED
    FED --> OVERLAP
    OVERLAP --> ACTIVATION
    ACTIVATION --> AD
    ACTIVATION --> EXTERNAL
    
    style N_SEG fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style N_MEM fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style N_CONTACT fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style H_SEG fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H_MEM fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style H_CONTACT fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style TEMPLATE fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style FED fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style OVERLAP fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style ACTIVATION fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style AD fill:#ffebee,stroke:#c62828,stroke-width:2px
    style EXTERNAL fill:#ffebee,stroke:#c62828,stroke-width:2px
```

---

## Data Model: DMO Relationships

```mermaid
erDiagram
    SEGMENT_METADATA ||--o{ SEGMENT_MEMBERSHIP : contains
    SEGMENT_MEMBERSHIP ||--|| CONTACT_GRAPH : identifies
    CLEAN_ROOM_COLLABORATION ||--o{ OVERLAP_RESULTS : generates
    OVERLAP_RESULTS ||--o{ ACTIVATION_RESULTS : enables
    
    SEGMENT_METADATA {
        string segmentID PK
        string segmentName
        string category
        string parentCategory
        datetime created
        string ownerOrg
    }
    
    SEGMENT_MEMBERSHIP {
        string segmentID FK
        string memberID PK
        datetime addedDate
        string source
    }
    
    CONTACT_GRAPH {
        string memberID FK
        string matchKeyID PK
        string idType
        string hashedValue
        datetime lastUpdated
    }
    
    CLEAN_ROOM_COLLABORATION {
        string collaborationID PK
        string templateID FK
        string sellSideOrg
        string buySideOrg
        string status
        json privacyConstraints
    }
    
    OVERLAP_RESULTS {
        string collaborationID FK
        string segmentID FK
        int matchedPopulation
        int totalPopulation
        float overlapRate
        datetime queryTimestamp
        boolean privacyThresholdMet
    }
    
    ACTIVATION_RESULTS {
        string activationID PK
        string collaborationID FK
        string matchKeyID
        string idType
        string destinationPlatform
        boolean consentValidated
        datetime activatedTimestamp
    }
```

---

## Activation Process: From Overlap to Campaign

### How Anonymized Segments are Activated

```mermaid
flowchart TD
    START([Clean Room Analysis Complete]) --> SEGMENT[Nike Creates Segment<br/>Nike+ App Users who are<br/>Hulu Subscribers]
    
    SEGMENT --> ANON["Segment Contains<br/>Anonymized Hashed IDs<br/>━━━━━━━━━━━━━━<br/>d7a5b3f2...<br/>a1b2c3d4...<br/>e9f8g7h6..."]
    
    ANON --> ACTIVATE["Activation Request<br/>━━━━━━━━━━━━━━<br/>Campaign: Running Shoes<br/>Audience: 285K users<br/>Platform: Hulu Ads"]
    
    ACTIVATE --> TRANSMIT["Server-to-Server<br/>Secure Transmission<br/>━━━━━━━━━━━━━━<br/>Only hashed IDs sent<br/>No PII transmitted"]
    
    TRANSMIT --> HULU_MATCH["Hulu Ad Platform<br/>Matches Hashed IDs<br/>━━━━━━━━━━━━━━<br/>Against subscriber database<br/>Creates targetable segment"]
    
    HULU_MATCH --> SERVE["Ad Serving<br/>━━━━━━━━━━━━━━<br/>When matched user watches Hulu<br/>System serves Nike ad<br/>User never identified to Nike"]
    
    SERVE --> METRICS["Campaign Metrics<br/>━━━━━━━━━━━━━━<br/>Impressions: 1.2M<br/>Clicks: 24K<br/>Conversions: 3.5K<br/>All aggregated, no PII"]
    
    style START fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style SEGMENT fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style ANON fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style ACTIVATE fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    style TRANSMIT fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style HULU_MATCH fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style SERVE fill:#ffebee,stroke:#c62828,stroke-width:2px
    style METRICS fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
```

---

## Privacy Guarantees

### What Nike NEVER Sees

```mermaid
graph TB
    subgraph "Protected Hulu Data"
        NEVER1["❌ Subscriber Names"]
        NEVER2["❌ Email Addresses"]
        NEVER3["❌ Individual Viewing History"]
        NEVER4["❌ Hulu Subscriber IDs"]
        NEVER5["❌ Personal Demographics"]
        NEVER6["❌ Individual-level data"]
    end
    
    subgraph "What Nike RECEIVES"
        YES1["✅ Aggregate overlap count"]
        YES2["✅ Anonymized hashed IDs"]
        YES3["✅ Segment-level insights"]
        YES4["✅ Campaign performance metrics"]
        YES5["✅ Overlap percentages"]
    end
    
    PRIVACY["Privacy Enforcement<br/>━━━━━━━━━━━━━━<br/>K-anonymity ≥ 1000<br/>Differential privacy<br/>No individual exposure<br/>Federated queries only"]
    
    NEVER1 --> PRIVACY
    NEVER2 --> PRIVACY
    NEVER3 --> PRIVACY
    NEVER4 --> PRIVACY
    NEVER5 --> PRIVACY
    NEVER6 --> PRIVACY
    
    PRIVACY --> YES1
    PRIVACY --> YES2
    PRIVACY --> YES3
    PRIVACY --> YES4
    PRIVACY --> YES5
    
    style NEVER1 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style NEVER2 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style NEVER3 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style NEVER4 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style NEVER5 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style NEVER6 fill:#ffebee,stroke:#c62828,stroke-width:2px
    style PRIVACY fill:#e8f5e9,stroke:#388e3c,stroke-width:3px
    style YES1 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style YES2 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style YES3 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style YES4 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style YES5 fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
```

---

## Key Takeaways

### How Clean Rooms Solve the Privacy-Utility Paradox

1. **Privacy Protection**
   - Data never leaves source systems
   - Only hashed identifiers used for matching
   - K-anonymity threshold (≥1000) enforced
   - Individual-level data never exposed

2. **Business Value**
   - Accurate audience overlap measurement
   - Precise targeting without PII exposure
   - Campaign performance tracking
   - Revenue opportunities for data owners

3. **Technical Implementation**
   - Federated queries run across environments
   - DMO (Data Model Object) structure maintains consistency
   - AppExchange templates enable rapid deployment
   - Server-to-server activation maintains security

4. **Real-World Impact**
   - Nike: Target qualified audience efficiently
   - Hulu: Monetize data while protecting subscribers
   - Users: Privacy preserved throughout process
   - Advertisers: Better ROI through precise targeting

---

## Use Cases Beyond Nike × Hulu

### Clean Room Applications

- **Retail × Financial Services**: Target high-value customers
- **Automotive × Insurance**: Identify qualified buyers
- **Healthcare × Pharma**: Patient cohort analysis (HIPAA compliant)
- **Travel × Credit Cards**: Premium travel audience targeting
- **CPG × Retailers**: In-store and online purchase correlation
- **Media × Brands**: Content affinity and product interest overlap

---

## Conclusion

Salesforce Data Cloud Clean Rooms enable **privacy-preserving collaboration** at enterprise scale. Through federated queries, hashed identifiers, and strict privacy thresholds, companies can unlock valuable audience insights and activation opportunities **without compromising individual privacy**.

The Nike × Hulu example demonstrates how two companies can collaborate to create mutual value—Nike gets precise targeting, Hulu monetizes their data, and subscribers' privacy is fully protected throughout the process.