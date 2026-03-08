# Marts Layer and Project Completion

## Overview
Build the final analytics-ready marts layer with fact and dimension tables following star schema design, generate documentation, and complete the project.

## Objectives
- Create 4 dimension tables (Type 1 and Type 2)
- Create 2 fact tables
- Implement post-hooks for analyst grants
- Generate surrogate keys
- Calculate business metrics
- Generate dbt documentation
- Run full test suite
- Complete project verification

## Prerequisites
- Parts 1-9 completed
- Understanding of star schema design
- Familiarity with Type 1 vs Type 2 dimensions

---

## Star Schema Design

### What is Star Schema?

Central fact table(s) surrounded by dimension tables.

**Benefits:**
- Simple to understand
- Fast query performance
- BI tool friendly
- Denormalized for analytics

### Our Schema

```
dim_customers_current (Type 1)
dim_customers_historical (Type 2)
dim_dates
dim_interaction_types
        |
        v
fct_customer_interactions  (Grain: 1 row per interaction)
fct_customer_lifetime_value (Grain: 1 row per customer)
```

### Fact vs Dimension

**Dimension Tables:**
- Descriptive attributes
- Relatively static
- Who, what, where, when
- Examples: customers, products, dates

**Fact Tables:**
- Measurable events
- Large and growing
- How many, how much
- Examples: sales, interactions, transactions

---

## Step 1: Create dim_customers_current (Type 1 SCD)

### Purpose
Current state dimension showing latest customer information only.

### File Location
models/marts/dim_customers_current.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'dimensions', 'customers'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Current Customer Dimension (Type 1 SCD)
    
    Purpose: Latest snapshot of each customer with current status
    Grain: One row per customer (current state only)
    
    Security: PII columns protected by Snowflake masking policies
    Access: ANALYST_ROLE granted via post-hook
*/

with current_customers as (
    select * from {{ ref('int_customers_deduplicated') }}
),

final as (
    select
        -- Surrogate key
        {{ dbt_utils.generate_surrogate_key(['customer_id']) }} as customer_key,
        
        -- Natural key
        customer_id,
        
        -- Attributes (PII masked by Snowflake policies)
        customer_name,
        email,
        phone,
        segment,
        status,
        
        -- Timestamps
        created_at,
        updated_at,
        updated_by,
        
        -- Flags
        was_duplicate,
        num_duplicates_removed,
        
        -- Metadata
        current_timestamp() as _loaded_at,
        'TYPE_1_CURRENT_ONLY' as scd_type
        
    from current_customers
)

select * from final
```

### Key Design Elements

**Surrogate Key (customer_key):**
- MD5 hash of customer_id
- Consistent across builds
- Used in fact table foreign keys

**Type 1 Characteristics:**
- One row per customer
- Always current state
- No history tracking
- Overwrites on change

**Post-Hook:**
- Automatically grants SELECT to analysts
- Runs after model build
- Maintains access control

---

## Step 2: Create dim_customers_historical (Type 2 SCD)

### Purpose
Historical dimension tracking all customer status changes over time.

### File Location
models/marts/dim_customers_historical.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'dimensions', 'customers', 'scd_type_2'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Historical Customer Dimension (Type 2 SCD)
    
    Purpose: Complete history of customer status changes over time
    Source: snap_customer_status_v2
    Grain: One row per customer per status change
    
    Use Cases:
    - Point-in-time analysis
    - Customer journey tracking
    - Cohort analysis by acquisition date
*/

with snapshot_data as (
    select * from {{ ref('snap_customer_status_v2') }}
),

final as (
    select
        -- Surrogate keys
        {{ dbt_utils.generate_surrogate_key(['customer_id', 'dbt_scd_id']) }} as customer_history_key,
        {{ dbt_utils.generate_surrogate_key(['customer_id']) }} as customer_key,
        
        -- Natural key
        customer_id,
        
        -- Attributes (PII masked by Snowflake policies)
        customer_name,
        email,
        phone,
        segment,
        status,
        
        -- Timestamps
        created_at,
        updated_at,
        
        -- SCD Type 2 metadata
        dbt_valid_from,
        dbt_valid_to,
        dbt_updated_at,
        dbt_scd_id,
        
        -- Flags
        case when dbt_valid_to is null then true else false end as is_current,
        
        -- Days in this status
        datediff('day', dbt_valid_from, coalesce(dbt_valid_to, current_timestamp())) as days_in_status,
        
        -- Metadata
        current_timestamp() as _loaded_at,
        'TYPE_2_HISTORICAL' as scd_type
        
    from snapshot_data
)

select * from final
```

### Type 2 Characteristics
- Multiple rows per customer (one per status change)
- dbt_valid_from/to track periods
- is_current flag for latest record
- Full history preserved

---

## Step 3: Create dim_dates

### Purpose
Date dimension for time-based analysis.

### File Location
models/marts/dim_dates.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'dimensions', 'reference'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Date Dimension
    
    Purpose: Calendar dimension for time-based analysis
    Grain: One row per day (Q1 2024)
    Coverage: January 1 - March 31, 2024
*/

with dates as (
    select * from {{ ref('stg_dates') }}
),

final as (
    select
        -- Surrogate key
        date_key,
        
        -- Date attributes
        date,
        day_of_week,
        day_of_month,
        week_of_year,
        month,
        month_name,
        quarter,
        year,
        
        -- Flags
        is_weekend,
        is_holiday,
        
        -- Derived attributes
        case when is_weekend or is_holiday then false else true end as is_business_day,
        
        concat(year, '-Q', quarter) as year_quarter,
        concat(year, '-', lpad(month::string, 2, '0')) as year_month,
        
        -- Metadata
        current_timestamp() as _loaded_at
        
    from dates
)

select * from final
```

### Date Dimension Benefits
- Consistent date logic across reports
- Easy filtering by business vs weekend
- Quarter/month aggregations simplified
- Holiday tracking built-in

---

## Step 4: Create dim_interaction_types

### Purpose
Reference dimension for interaction type attributes.

### File Location
models/marts/dim_interaction_types.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'dimensions', 'reference'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Interaction Type Dimension
    
    Purpose: Reference data for interaction types
    Grain: One row per interaction type
    Source: stg_interaction_types
*/

with interaction_types as (
    select 
        interaction_type_id,
        interaction_type,
        type_category,
        is_billable,
        avg_duration_seconds,
        description
    from {{ ref('stg_interaction_types') }}
),

final as (
    select
        -- Natural key
        interaction_type_id,
        
        -- Attributes
        interaction_type,
        type_category,
        is_billable,
        avg_duration_seconds,
        description,
        
        -- Derived
        case 
            when type_category = 'voice' then 'Voice Channel'
            when type_category = 'digital' then 'Digital Channel'
        end as channel_group,
        
        -- Metadata
        current_timestamp() as _loaded_at
        
    from interaction_types
)

select * from final
```

---

## Step 5: Create fct_customer_interactions

### Purpose
Granular fact table with one row per customer interaction.

### File Location
models/marts/fct_customer_interactions.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'facts', 'interactions'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Customer Interactions Fact Table
    
    Purpose: Granular interaction-level facts with session context
    Grain: One row per interaction
    
    Key Metrics:
    - Duration, sentiment, resolution status
    - Session membership
    - Time-based analysis via date_key
*/

with interactions as (
    select * from {{ ref('int_customer_interactions_sessionized') }}
),

customers as (
    select * from {{ ref('dim_customers_current') }}
),

dates as (
    select * from {{ ref('dim_dates') }}
),

final as (
    select
        -- Surrogate keys
        {{ dbt_utils.generate_surrogate_key(['i.interaction_id']) }} as interaction_key,
        c.customer_key,
        to_number(to_char(i.interaction_timestamp, 'YYYYMMDD')) as date_key,
        i.session_id,
        
        -- Natural keys
        i.interaction_id,
        i.customer_id,
        
        -- Degenerate dimensions
        i.interaction_type,
        i.channel,
        i.resolution_status,
        
        -- Timestamps
        i.interaction_timestamp,
        
        -- Measures
        i.duration_seconds,
        i.sentiment_score,
        
        -- Session context
        i.session_sequence,
        i.minutes_since_last_interaction,
        i.is_new_session,
        
        -- Derived measures
        i.duration_seconds / 60.0 as duration_minutes,
        case when i.sentiment_score >= 0 then 1 else 0 end as is_positive_sentiment,
        case when i.resolution_status = 'resolved' then 1 else 0 end as is_resolved,
        
        -- Metadata
        current_timestamp() as _loaded_at
        
    from interactions i
    left join customers c on i.customer_id = c.customer_id
    left join dates d on to_number(to_char(i.interaction_timestamp, 'YYYYMMDD')) = d.date_key
)

select * from final
```

### Fact Table Design

**Grain:** One row per interaction

**Keys:**
- interaction_key (surrogate, primary key)
- customer_key (foreign key to dim_customers_current)
- date_key (foreign key to dim_dates)
- session_id (session identifier)

**Measures:**
- duration_seconds (additive)
- sentiment_score (semi-additive, average)
- Flags: is_positive_sentiment, is_resolved

**Degenerate Dimensions:**
- Attributes stored in fact (interaction_type, channel, resolution_status)
- Too granular for separate dimension
- Low cardinality

---

## Step 6: Create fct_customer_lifetime_value

### Purpose
Aggregated fact table with customer-level metrics.

### File Location
models/marts/fct_customer_lifetime_value.sql

### Implementation
```sql
{{
    config(
        materialized='table',
        tags=['marts', 'facts', 'customers', 'clv'],
        post_hook=[
            "grant select on {{ this }} to role analyst_role"
        ]
    )
}}

/*
    Customer Lifetime Value Fact Table
    
    Purpose: Aggregated customer-level metrics for CLV analysis
    Grain: One row per customer
    
    Metrics:
    - Total interactions, sessions
    - Avg sentiment, duration
    - Days as customer, days in each status
    - Lifecycle health indicators
*/

with customers as (
    select * from {{ ref('dim_customers_current') }}
),

interactions as (
    select * from {{ ref('fct_customer_interactions') }}
),

lifecycle as (
    select * from {{ ref('int_customer_lifecycle_metrics') }}
),

-- Aggregate interaction metrics per customer
customer_interaction_metrics as (
    select
        customer_id,
        count(distinct interaction_id) as total_interactions,
        count(distinct session_id) as total_sessions,
        avg(duration_seconds) as avg_interaction_duration_seconds,
        avg(sentiment_score) as avg_sentiment_score,
        sum(case when is_resolved = 1 then 1 else 0 end) as total_resolved,
        sum(case when is_positive_sentiment = 1 then 1 else 0 end) as total_positive_sentiment,
        min(interaction_timestamp) as first_interaction_date,
        max(interaction_timestamp) as last_interaction_date
    from interactions
    group by customer_id
),

-- Get current lifecycle status
current_lifecycle as (
    select
        customer_id,
        status as current_status,
        lifecycle_stage as current_lifecycle_stage,
        days_in_status as days_in_current_status
    from lifecycle
    where is_current_status = true
    qualify row_number() over (partition by customer_id order by dbt_valid_from desc) = 1
),

-- Calculate total days in each status
status_durations as (
    select
        customer_id,
        sum(case when status = 'lead' then days_in_status else 0 end) as total_days_as_lead,
        sum(case when status = 'trial' then days_in_status else 0 end) as total_days_as_trial,
        sum(case when status = 'paid' then days_in_status else 0 end) as total_days_as_paid,
        sum(case when status = 'at_risk' then days_in_status else 0 end) as total_days_at_risk,
        sum(days_in_status) as total_days_as_customer
    from lifecycle
    group by customer_id
),

final as (
    select
        -- Surrogate key
        c.customer_key,
        
        -- Natural key
        c.customer_id,
        
        -- Customer attributes
        c.segment,
        cl.current_status,
        cl.current_lifecycle_stage,
        
        -- Interaction metrics
        coalesce(im.total_interactions, 0) as total_interactions,
        coalesce(im.total_sessions, 0) as total_sessions,
        im.avg_interaction_duration_seconds,
        im.avg_sentiment_score,
        im.total_resolved,
        im.total_positive_sentiment,
        
        -- Engagement metrics
        case 
            when im.total_interactions > 0 
            then im.total_resolved::float / im.total_interactions 
            else null 
        end as resolution_rate,
        
        case 
            when im.total_interactions > 0 
            then im.total_positive_sentiment::float / im.total_interactions 
            else null 
        end as positive_sentiment_rate,
        
        case 
            when im.total_interactions > 0 
            then im.total_interactions::float / im.total_sessions 
            else null 
        end as avg_interactions_per_session,
        
        -- Lifecycle duration metrics
        sd.total_days_as_lead,
        sd.total_days_as_trial,
        sd.total_days_as_paid,
        sd.total_days_at_risk,
        sd.total_days_as_customer,
        cl.days_in_current_status,
        
        -- Timestamps
        c.created_at as customer_since,
        im.first_interaction_date,
        im.last_interaction_date,
        datediff('day', im.last_interaction_date, current_timestamp()) as days_since_last_interaction,
        
        -- Health indicators
        case
            when cl.current_status = 'churned' then 'churned'
            when cl.current_status = 'at_risk' then 'at_risk'
            when cl.current_status = 'paid' and im.avg_sentiment_score >= 0.5 then 'healthy'
            when cl.current_status = 'paid' and im.avg_sentiment_score < 0.5 then 'needs_attention'
            when cl.current_status = 'trial' then 'onboarding'
            when cl.current_status = 'lead' then 'prospecting'
            else 'unknown'
        end as customer_health_status,
        
        -- Metadata
        current_timestamp() as _loaded_at
        
    from customers c
    left join customer_interaction_metrics im on c.customer_id = im.customer_id
    left join current_lifecycle cl on c.customer_id = cl.customer_id
    left join status_durations sd on c.customer_id = sd.customer_id
)

select * from final
```

### Customer Health Scoring Logic

**Churned:** Status = churned
**At Risk:** Status = at_risk
**Healthy:** Paid + positive sentiment (>= 0.5)
**Needs Attention:** Paid + negative sentiment (< 0.5)
**Onboarding:** Status = trial
**Prospecting:** Status = lead

---

## Step 7: Build All Marts

### Purpose
Execute dbt run to create all mart tables.

### Implementation
```powershell
# Build all marts
dbt run --select marts

# Or build individually
dbt run --select dim_customers_current
dbt run --select dim_customers_historical
dbt run --select dim_dates
dbt run --select dim_interaction_types
dbt run --select fct_customer_interactions
dbt run --select fct_customer_lifetime_value
```

### Expected Output
```
Running with dbt=1.8.0
Found 6 models

1 of 6 START sql table model staging_marts.dim_dates .................. [RUN]
2 of 6 START sql table model staging_marts.dim_interaction_types ...... [RUN]
3 of 6 START sql table model staging_marts.dim_customers_current ....... [RUN]
1 of 6 OK created sql table model staging_marts.dim_dates ............. [SUCCESS 1 in 3.2s]
4 of 6 START sql table model staging_marts.dim_customers_historical .... [RUN]
2 of 6 OK created sql table model staging_marts.dim_interaction_types . [SUCCESS 1 in 3.1s]
5 of 6 START sql table model staging_marts.fct_customer_interactions ... [RUN]
3 of 6 OK created sql table model staging_marts.dim_customers_current .. [SUCCESS 1 in 3.5s]
6 of 6 START sql table model staging_marts.fct_customer_lifetime_value  [RUN]
4 of 6 OK created sql table model staging_marts.dim_customers_historical [SUCCESS 1 in 4.1s]
5 of 6 OK created sql table model staging_marts.fct_customer_interactions [SUCCESS 1 in 4.8s]
6 of 6 OK created sql table model staging_marts.fct_customer_lifetime_value [SUCCESS 1 in 5.2s]

Completed successfully
```

---

## Step 8: Verify Marts in Snowflake

### Check Tables Created
```sql
use database analytics;
use schema staging_marts;

-- List all mart tables
show tables in schema analytics.staging_marts;

-- Verify row counts
select 'dim_customers_current' as table_name, count(*) as rows from dim_customers_current
union all
select 'dim_customers_historical', count(*) from dim_customers_historical
union all
select 'dim_dates', count(*) from dim_dates
union all
select 'dim_interaction_types', count(*) from dim_interaction_types
union all
select 'fct_customer_interactions', count(*) from fct_customer_interactions
union all
select 'fct_customer_lifetime_value', count(*) from fct_customer_lifetime_value;
```

### Verify Foreign Keys
```sql
-- Check for orphaned interactions (customer_key doesn't exist)
select count(*) as orphaned_interactions
from fct_customer_interactions f
left join dim_customers_current d on f.customer_key = d.customer_key
where d.customer_key is null;

-- Expected: 0
```

### Sample Data
```sql
-- Sample CLV data
select 
    customer_id,
    segment,
    current_status,
    total_interactions,
    total_sessions,
    avg_sentiment_score,
    resolution_rate,
    customer_health_status
from fct_customer_lifetime_value
limit 10;
```

---

## Step 9: Run Full Test Suite

### Purpose
Validate all models with comprehensive testing.

### Implementation
```powershell
# Run all tests (sources, staging, intermediate, marts)
dbt test

# Or test marts only
dbt test --select marts
```

### Expected Results
- Most tests pass
- Documented failures (10 duplicate customer_ids)
- Warnings for intentional data quality issues

---

## Step 10: Generate Documentation

### Purpose
Create interactive documentation with lineage DAG.

### Implementation
```powershell
# Generate documentation
dbt docs generate

# Serve documentation locally
dbt docs serve
```

### Expected Behavior
- Opens browser at http://localhost:8080
- Interactive DAG showing full lineage
- All model documentation visible
- Tests documented
- Exposures listed

### Documentation Features
- Click models to see details
- View compiled SQL
- See test results
- Explore lineage graph
- Search across project

---

## Step 11: Run Full Project Build

### Purpose
Execute complete build from scratch.

### Implementation
```powershell
# Full rebuild
dbt clean
dbt deps
dbt seed --full-refresh
dbt snapshot
dbt run
dbt test

# Or use dbt build (does everything in dependency order)
dbt build
```

### Build Order (dbt build)
1. Load seeds
2. Test sources
3. Build staging models
4. Test staging models
5. Run snapshots
6. Build intermediate models
7. Test intermediate models
8. Build marts
9. Test marts

---

## Project Completion Verification

### Checklist
- [ ] All 6 mart tables created
- [ ] Row counts match expectations
- [ ] Foreign key relationships valid
- [ ] Post-hooks executed (analyst grants)
- [ ] Tests pass (with documented exceptions)
- [ ] Documentation generated
- [ ] Lineage DAG complete
- [ ] Security policies working

### Final Metrics
```
Total models: 16
- Seeds: 5
- Staging: 5
- Snapshots: 2
- Intermediate: 4
- Marts: 6

Total tests: 100+
Total rows processed: 7,315
Unique customers: 1,000
```

---

## Troubleshooting

### Issue: Post-Hook Fails
**Problem:** Grant statement in post-hook fails

**Solution:**
```sql
-- Run grant manually
use role transformer_role;
grant select on analytics.staging_marts.dim_customers_current to role analyst_role;

-- Verify grant
use role analyst_role;
select * from analytics.staging_marts.dim_customers_current limit 1;
```

### Issue: Surrogate Key Mismatches
**Problem:** customer_key in fact doesn't match dimension

**Solution:**
```sql
-- Verify surrogate key generation is consistent
select 
    customer_id,
    {{ dbt_utils.generate_surrogate_key(['customer_id']) }} as key1,
    {{ dbt_utils.generate_surrogate_key(['customer_id']) }} as key2
from dim_customers_current
limit 5;

-- key1 and key2 should match
```

### Issue: Missing FROM Clause
**Problem:** SQL compilation error

**Solution:**
- Every CTE needs FROM clause
- Check final CTE has: select * from cte_name
- Verify all CTEs are referenced

---

## Best Practices Summary

### Dimension Design
1. Use surrogate keys (MD5 hashes)
2. Preserve natural keys
3. Add SCD metadata (_loaded_at, scd_type)
4. Grant access via post-hooks

### Fact Design
1. Define grain clearly (one row per...)
2. Use foreign keys to dimensions
3. Separate additive vs semi-additive measures
4. Add derived metrics

### Star Schema
1. Denormalize for analytics
2. Keep fact tables narrow (avoid wide tables)
3. Create multiple fact tables at different grains
4. Use conformed dimensions

### Documentation
1. Document grain in model description
2. Explain all business metrics
3. Note any filters or exclusions
4. Link to business glossary

---

