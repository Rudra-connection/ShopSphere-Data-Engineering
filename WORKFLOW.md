# ShopSphere – E-Commerce Analytics Pipeline: Workflow & Diagrams

> Open this file in VS Code and press `Ctrl+Shift+V` (Mac: `Cmd+Shift+V`) to preview.
> Install the extension **"Markdown Preview Mermaid Support"** so the diagrams render.

---

## 1. End-to-End Workflow (Big Picture)

```mermaid
flowchart LR
    A[4 Source Systems] --> B[Raw / Landing<br/>7 files as-is]
    B --> C[Profiling<br/>find quality issues]
    C --> D[Cleansing &<br/>Standardisation]
    D --> E[Silver Layer<br/>clean tables]
    E --> F[Dimensional Model<br/>Facts + Dimensions]
    F --> G[Gold Layer]
    G --> H[8 KPIs + Stretch KPI]
    H --> I[Reports / Dashboard]
    D -.-> J[Rejected / Quarantine<br/>records + reasons]
    C -.-> K[Profiling &<br/>Cleansing Report]
    D -.-> K
```

---

## 2. Source Systems → Raw Files

```mermaid
flowchart TB
    subgraph EC[E-Commerce Platform]
        O[orders.csv]
        OI[order_items.csv]
        R[returns.csv]
    end
    subgraph CRM[CRM]
        C[customers.csv]
    end
    subgraph PG[Payment Gateway]
        P[payments.csv]
    end
    subgraph PR[Procurement]
        S[suppliers.json]
        PC[products.csv]
    end
    EC --> RAW[(Bronze / Raw Zone)]
    CRM --> RAW
    PG --> RAW
    PR --> RAW
```

---

## 3. Medallion Architecture (Bronze → Silver → Gold)

```mermaid
flowchart LR
    subgraph BRONZE[Bronze – Raw]
        b1[customers]
        b2[suppliers]
        b3[products]
        b4[orders]
        b5[order_items]
        b6[payments]
        b7[returns]
    end
    subgraph SILVER[Silver – Cleansed]
        s1[clean_customers]
        s2[clean_suppliers]
        s3[clean_products]
        s4[clean_orders]
        s5[clean_order_items]
        s6[clean_payments]
        s7[clean_returns]
        sq[(quarantine<br/>rejected rows)]
    end
    subgraph GOLD[Gold – Analytics]
        d1[dim_customer SCD2]
        d2[dim_product]
        d3[dim_supplier]
        d4[dim_date]
        d5[dim_channel]
        f1[fact_order_items]
        f2[fact_payments]
        f3[fact_returns]
    end
    b1 --> s1 --> d1
    b2 --> s2 --> d3
    b3 --> s3 --> d2
    b4 --> s4
    b5 --> s5 --> f1
    s4 --> f1
    b6 --> s6 --> f2
    b7 --> s7 --> f3
    s1 -.-> sq
    s3 -.-> sq
    s5 -.-> sq
    s6 -.-> sq
```

---

## 4. Cleansing Workflow (applied to every table)

```mermaid
flowchart TD
    A[Load raw file as text] --> B[Standardise column names<br/>snake_case]
    B --> C[Trim spaces + convert<br/>NA / null / - to NULL]
    C --> D{Exact duplicate<br/>rows?}
    D -- yes --> D1[Drop, log count]
    D -- no --> E
    D1 --> E[Fix formats]
    E --> E1[Dates → ISO yyyy-mm-dd]
    E --> E2[Currency / numbers → float]
    E --> E3[Boolean flags → YES / NO]
    E --> E4[Phone → 10 digits]
    E --> E5[Categories → canonical labels]
    E1 & E2 & E3 & E4 & E5 --> F{Duplicate / conflicting<br/>natural key?}
    F -- yes --> F1[Apply WINNER RULE<br/>see diagram 5]
    F -- no --> G
    F1 --> G{Business rule<br/>violations?}
    G -- yes --> G1[Fix if derivable<br/>else quarantine]
    G -- no --> H
    G1 --> H{Foreign key<br/>exists in parent?}
    H -- no --> H1[Flag as orphan<br/>keep with Unknown member -1]
    H -- yes --> I
    H1 --> I[Write Silver table]
    I --> J[Log before/after counts]
```

---

## 5. "Which Record Wins" Rule for Duplicates / Conflicts

```mermaid
flowchart TD
    A[Multiple records<br/>same natural key] --> B{Identical in<br/>all columns?}
    B -- yes --> C[Keep one]
    B -- no --> D{Has a date<br/>or update field?}
    D -- yes --> E[Keep latest record]
    D -- no --> F{One record more<br/>complete?}
    F -- yes --> G[Keep record with<br/>fewest NULLs]
    F -- no --> H[Keep first by<br/>file order + log it]
    I[Same entity,<br/>different keys] --> J[Match on email / phone /<br/>name + address]
    J --> K[Map to one surrogate key<br/>keep xref table]
```

---

## 6. Data-Quality Issues → Fix Map

```mermaid
flowchart LR
    Q1[Duplicates] --> R1[Dedup with winner rule]
    Q2[Missing values] --> R2[Fill, derive, or quarantine<br/>depending on KPI impact]
    Q3[Inconsistent formats] --> R3[Standardise dates, phones,<br/>numbers, flags, labels]
    Q4[Broken foreign keys] --> R4[Orphan flag + Unknown member]
    Q5[Business-rule violations] --> R5[Range checks, date order checks,<br/>recompute line_total]
    Q6[Conflicting records] --> R6[Documented winner rule]
```

---

## 7. Star Schema (Gold Layer)

```mermaid
erDiagram
    DIM_CUSTOMER ||--o{ FACT_ORDER_ITEMS : places
    DIM_PRODUCT  ||--o{ FACT_ORDER_ITEMS : contains
    DIM_SUPPLIER ||--o{ DIM_PRODUCT : supplies
    DIM_DATE     ||--o{ FACT_ORDER_ITEMS : order_date
    DIM_CHANNEL  ||--o{ FACT_ORDER_ITEMS : sold_via
    FACT_ORDER_ITEMS ||--o{ FACT_RETURNS : returned_as
    FACT_ORDER_ITEMS }o--|| FACT_PAYMENTS : paid_by_order
    DIM_DATE     ||--o{ FACT_PAYMENTS : payment_date
    DIM_DATE     ||--o{ FACT_RETURNS : return_date

    DIM_CUSTOMER {
        int customer_key PK
        string customer_id
        string full_name
        string email
        string phone
        string city
        string state
        string segment
        string is_active
        date effective_from
        date effective_to
        string is_current
    }
    DIM_PRODUCT {
        int product_key PK
        string product_id
        string product_name
        string category
        string sub_category
        float unit_price
        float cost_price
        int supplier_key FK
    }
    DIM_SUPPLIER {
        int supplier_key PK
        string supplier_id
        string supplier_name
        string country
        float rating_normalised
        string is_preferred
    }
    DIM_DATE {
        int date_key PK
        date full_date
        int year
        int month
        string month_name
        int quarter
    }
    DIM_CHANNEL {
        int channel_key PK
        string channel_name
    }
    FACT_ORDER_ITEMS {
        int order_item_key PK
        string order_id
        int customer_key FK
        int product_key FK
        int date_key FK
        int channel_key FK
        string order_status
        int quantity
        float unit_price
        float discount_pct
        float line_total
    }
    FACT_PAYMENTS {
        string payment_id PK
        string order_id
        int date_key FK
        float amount
        string payment_status
        string payment_method
    }
    FACT_RETURNS {
        string return_id PK
        int order_item_key FK
        int date_key FK
        string return_reason
        string return_status
        float refund_amount
    }
```

---

## 8. Slowly Changing Dimension – Customer (SCD Type 2)

```mermaid
flowchart TD
    A[Incoming customer row] --> B{customer_id<br/>already in dim?}
    B -- no --> C[Insert new row<br/>is_current = Y<br/>effective_to = 9999-12-31]
    B -- yes --> D{segment or address<br/>changed?}
    D -- no --> E[No action]
    D -- yes --> F[Close old row<br/>effective_to = today<br/>is_current = N]
    F --> G[Insert new row<br/>is_current = Y]
```

Example:

| customer_key | customer_id | segment | effective_from | effective_to | is_current |
|---|---|---|---|---|---|
| 1 | C001 | Silver | 2024-01-10 | 2025-03-31 | N |
| 2 | C001 | Gold | 2025-04-01 | 9999-12-31 | Y |

Facts join to the customer row that was valid on the **order date**.

---

## 9. KPI Dependency Map (Which Tables Feed Which KPI)

```mermaid
flowchart LR
    subgraph Tables
        OI[order_items]
        O[orders]
        P[products]
        R[returns]
        C[customers]
        S[suppliers]
        PAY[payments]
    end
    K1[KPI 1<br/>Net Revenue by Category/Month]
    K2[KPI 2<br/>Top 10 Customers LTV]
    K3[KPI 3<br/>Segment Profile]
    K4[KPI 4<br/>MoM Growth by Channel]
    K5[KPI 5<br/>Return Rate Top/Bottom 10]
    K6[KPI 6<br/>Supplier Scorecard]
    K7[KPI 7<br/>Repeat Purchase Rate]
    K8[KPI 8<br/>Payment Reconciliation]
    K9[Stretch<br/>Margin Risk]

    OI & P & R & O --> K1
    O & OI & R & C --> K2
    C & O & OI --> K3
    O & OI --> K4
    OI & R & P --> K5
    S & P & OI & R --> K6
    C & O --> K7
    PAY & O & OI --> K8
    P & OI --> K9
```

---

## 10. Core KPI Formulas

```mermaid
flowchart TD
    A[line_revenue = quantity x unit_price x 1 - discount_pct/100] --> B[net_revenue = line_revenue - refund_amount]
    B --> K1[KPI 1: SUM by category, month]
    B --> K2[KPI 2: SUM by customer<br/>Completed orders only]
    C[return_rate = returned_qty / ordered_qty<br/>min order volume threshold] --> K5[KPI 5]
    D[supplier_score = w1 x revenue_norm<br/>+ w2 x 1 - return_rate<br/>+ w3 x rating_norm] --> K6[KPI 6]
    E[repeat = customers with 2nd completed order<br/>within 90 days / customers with 1+ completed] --> K7[KPI 7]
```

---

## 11. Payment Reconciliation Logic (KPI 8)

```mermaid
flowchart TD
    A[For each order] --> B[order_total =<br/>SUM of cleaned line_total]
    A --> C[Get payments<br/>for order_id]
    C --> D{Any payment<br/>rows?}
    D -- no --> X1[FLAG: No payment]
    D -- yes --> E{Any with status<br/>Success?}
    E -- no --> X2[FLAG: No successful payment]
    E -- yes --> F{Duplicate or<br/>conflicting rows?}
    F -- yes --> X3[FLAG: Duplicate / conflicting]
    F -- no --> G[paid = SUM successful amounts]
    X3 --> G
    G --> H{paid vs order_total}
    H -- paid < total --> X4[FLAG: Underpayment]
    H -- paid > total --> X5[FLAG: Overpayment]
    H -- equal within tolerance --> OK[Reconciled]
```

---

## 12. Customer Repeat-Purchase Logic (KPI 7)

```mermaid
flowchart LR
    A[Completed orders only] --> B[Find first order date<br/>per customer]
    B --> C{Another completed order<br/>within 90 days?}
    C -- yes --> D[Repeat customer]
    C -- no --> E[Non-repeat]
    D & E --> F[Rate = repeat / all customers<br/>with a completed order]
```

---

## 13. Orphan Handling Decision (Referential Integrity)

```mermaid
flowchart TD
    A[Child row with FK] --> B{FK found in<br/>parent table?}
    B -- yes --> C[Normal load]
    B -- no --> D{Can it be matched<br/>by cleaning the key?<br/>case, spaces, prefix}
    D -- yes --> E[Repair key and load]
    D -- no --> F[Map to Unknown member -1<br/>keep row, flag is_orphan = Y]
    F --> G[Excluded from KPI 2 customer ranking<br/>included in total revenue KPIs]
```

---

## 14. Project Folder Structure

```mermaid
flowchart TD
    ROOT[Ecommerce_Analytics_DE/]
    ROOT --> RAWF[raw/ — 7 source files]
    ROOT --> CODE[pipeline/]
    ROOT --> OUT[output/]
    ROOT --> DOCS[docs/]
    CODE --> P1[01_profile.py]
    CODE --> P2[02_clean_silver.py]
    CODE --> P3[03_build_gold.py]
    CODE --> P4[04_kpis.py or kpis.sql]
    OUT --> O1[silver/ clean tables]
    OUT --> O2[gold/ fact + dim]
    OUT --> O3[kpi_results.xlsx]
    OUT --> O4[quarantine/ rejected rows]
    DOCS --> D1[profiling_cleansing_report.md]
    DOCS --> D2[data_model_ddl.sql]
    DOCS --> D3[README.md]
    DOCS --> D4[ai_usage_log.md]
```

---

## 15. Execution Order (Pipeline Run)

```mermaid
sequenceDiagram
    participant U as You
    participant P as 01_profile.py
    participant S as 02_clean_silver.py
    participant G as 03_build_gold.py
    participant K as 04_kpis
    U->>P: run on raw files
    P-->>U: profiling report (issues list)
    U->>S: run cleansing
    S-->>U: silver tables + quarantine + before/after counts
    U->>G: build model
    G-->>U: dims + facts (SCD2 customer)
    U->>K: run 8 KPIs + stretch
    K-->>U: kpi_results.xlsx
    U->>U: write README + AI usage log
```

---

## 16. AI Usage & Validation Loop (for the AI log deliverable)

```mermaid
flowchart LR
    A[Write prompt] --> B[AI generates code]
    B --> C[Run on data]
    C --> D{Validate}
    D --> D1[Row counts before/after]
    D --> D2[Spot-check sample rows]
    D --> D3[Recompute KPI by hand<br/>in Excel / pandas]
    D1 & D2 & D3 --> E{Correct?}
    E -- yes --> F[Accept + note in log]
    E -- no --> G[Fix or reject<br/>record why]
    G --> A
```

---

## 17. Suggested Time Plan (4–6 hours)

```mermaid
gantt
    title Assessment Time Plan
    dateFormat HH:mm
    axisFormat %H:%M
    section Foundation
    Profiling           :a1, 00:00, 40m
    Cleansing rules     :a2, after a1, 80m
    section Modelling
    Dimensional design  :a3, after a2, 40m
    Pipeline build      :a4, after a3, 55m
    section Output
    KPIs                :a5, after a4, 105m
    Docs + AI log       :a6, after a5, 20m
```
