# Snapshots (SCD Type 2)

## Overview
Implement Slowly Changing Dimension Type 2 (SCD Type 2) using dbt snapshots to track customer status changes over time.

## Objectives
- Understand SCD Type 2 concepts
- Create snapshot with timestamp strategy
- Track customer status history
- Query point-in-time data
- Calculate days in status
- Analyze customer journey transitions

## Prerequisites
- Part 6 completed (staging layer tested)
- Understanding of SCD concepts
- Familiarity with window functions

---

## SCD Type 2 Concepts

### What is SCD Type 2?

Slowly Changing Dimension Type 2 tracks historical changes by creating new rows for each change rather than updating existing rows.

### SCD Types Comparison

**Type 0:** Never change (static reference data)
- Example: Product categories

**Type 1:** Overwrite old value
- Example: Current customer address
- Pros: Simple, low storage
- Cons: Lose history

**Type 2:** Add new row for each change
- Example: Customer status over time
- Pros: Full history preserved
- Cons: Higher storage, more complex queries

**Type 3:** Add new column for change
- Example: previous_status, current_status
- Pros: Simple queries, limited history
- Cons: Only tracks last change

### SCD Type 2 Metadata Columns

dbt snapshot adds these columns:
- **dbt_valid_from:** When this version became active
- **dbt_valid_to:** When this version expired (NULL = current)
- **dbt_updated_at:** When dbt last checked this record
- **dbt_scd_id:** Unique identifier for each version

---

## Step 1: Create Basic Snapshot

### Purpose
Track customer status changes using timestamp strategy.

### File Location
snapshots/snap_customer_status.sql

### Implementation
```sql
{% snapshot snap_customer_status %}

{{
    config(
        target_database='analytics',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at',
        tags=['snapshots', 'scd_type_2']
    )
}}

select
    customer_id,
    customer_name,
    email,
    phone,
    segment,
    status,
    created_at,
    updated_at
    
from {{ source('raw_seeds', 'seed_customers') }}

{% endsnapshot %}
```

### Configuration Explained

**target_database:** Where snapshot table is created (analytics)

**target_schema:** Schema for snapshots (snapshots)

**unique_key:** Column(s) that identify a unique entity (customer_id)
- Can be single column or list: ['col1', 'col2']
- dbt tracks changes per unique_key

**strategy:** How to detect changes
- timestamp: Uses updated_at column
- check: Compares all columns (or specified columns)

**updated_at:** Column containing update timestamp (for timestamp strategy)

**tags:** For selective execution and organization

### Timestamp Strategy Details

How it works:
1. First run: Creates initial snapshot (all rows)
2. Subsequent runs: Compares updated_at in source vs snapshot
3. If updated_at changed: Closes old row (sets dbt_valid_to), creates new row
4. If unchanged: Updates dbt_updated_at only (no new row)

---

## Step 2: Create Snapshot with Status Updates Applied

### Purpose
Snapshot customers with status updates already applied from Part 5.

### File Location
snapshots/snap_customer_status_v2.sql

### Implementation
```sql
{% snapshot snap_customer_status_v2 %}

{{
    config(
        target_database='analytics',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at',
        invalidate_hard_deletes=False,
        tags=['snapshots', 'scd_type_2', 'customers']
    )
}}

/*
    Customer snapshot with status updates applied
    
    Purpose: Track customer status changes over time
    Source: stg_customers_with_updates (includes latest status)
    
    SCD Type 2 Features:
    - dbt_valid_from: When this status became active
    - dbt_valid_to: When this status expired (NULL = current)
    - Full history of all status transitions
*/

select
    customer_id,
    customer_name,
    email,
    phone,
    segment,
    status,
    created_at,
    updated_at,
    updated_by
    
from {{ ref('stg_customers_with_updates') }}

{% endsnapshot %}
```

### Key Difference from snap_customer_status

**snap_customer_status:** 
- Source: seed_customers (raw data)
- Won't capture status updates from seed_customer_status_updates

**snap_customer_status_v2:** 
- Source: stg_customers_with_updates (includes status updates)
- Captures all status transitions
- Better reflects actual customer journey

---

## Step 3: Run Snapshots

### Purpose
Execute dbt snapshot to create historical tracking tables.

### Implementation
```powershell
# Run all snapshots
dbt snapshot

# Or run specific snapshot
dbt snapshot --select snap_customer_status_v2
```

### Expected Output - First Run
```
Running with dbt=1.8.0
Found 2 snapshots

Concurrency: 4 threads

1 of 2 START snapshot snapshots.snap_customer_status ..................... [RUN]
2 of 2 START snapshot snapshots.snap_customer_status_v2 ................. [RUN]
1 of 2 OK created snapshot snapshots.snap_customer_status ............... [SUCCESS 1 in 3.2s]
2 of 2 OK created snapshot snapshots.snap_customer_status_v2 ............ [SUCCESS 1 in 3.5s]

Completed successfully
```

First run creates initial snapshot:
- All records have dbt_valid_to = NULL (all current)
- dbt_valid_from = current timestamp
- Total rows = number of unique customers

### Expected Output - Subsequent Runs
```
1 of 2 OK snapshot snapshots.snap_customer_status_v2 .................... [SUCCESS 1 in 2.8s]
```

Subsequent runs:
- Update dbt_updated_at for unchanged records
- Create new rows for changed records (closes old rows)
- Maintains full history

---

## Step 4: Verify Snapshot Data

### Check Snapshot Tables Created
```sql
use database analytics;
use schema snapshots;

-- List snapshot tables
show tables in schema analytics.snapshots;

-- Should see:
-- snap_customer_status
-- snap_customer_status_v2
```

### Check Row Counts
```sql
-- Count total records vs current records
select 
    count(*) as total_records,
    count(distinct customer_id) as unique_customers,
    count(case when dbt_valid_to is null then 1 end) as current_records,
    count(case when dbt_valid_to is not null then 1 end) as historical_records
from snap_customer_status_v2;

-- First run expected:
-- total_records: 1000 (unique customers)
-- unique_customers: 1000
-- current_records: 1000 (all current)
-- historical_records: 0 (no history yet)
```

### View SCD Metadata Columns
```sql
select 
    customer_id,
    status,
    dbt_valid_from,
    dbt_valid_to,
    dbt_updated_at,
    dbt_scd_id
from snap_customer_status_v2
limit 10;
```

### Find Customers with Status Changes
```sql
-- Customers with multiple versions (status changes)
select 
    customer_id,
    count(*) as num_versions
from snap_customer_status_v2
group by customer_id
having count(*) > 1
order by num_versions desc
limit 10;

-- First run: 0 results (no history yet)
-- After updates: Shows customers with status changes
```

---

## Step 5: Simulate Status Changes

### Purpose
Update source data to test snapshot functionality.

### Update Seed Data
```sql
use role accountadmin;
use database raw;
use schema staging_seeds;

-- Update 5 customers to new status
update seed_customers
set 
    status = 'paid',
    updated_at = current_timestamp()
where customer_id in ('CUST000001', 'CUST000002', 'CUST000003', 'CUST000004', 'CUST000005')
  and status = 'trial';

-- Verify changes
select customer_id, status, updated_at
from seed_customers
where customer_id in ('CUST000001', 'CUST000002', 'CUST000003', 'CUST000004', 'CUST000005');
```

### Rebuild Staging Model
```powershell
# Rebuild staging to pick up changes
dbt run --select stg_customers_with_updates
```

### Run Snapshot Again
```powershell
# Run snapshot to capture changes
dbt snapshot --select snap_customer_status_v2
```

### Verify History Created
```sql
-- Should now see historical records
select 
    customer_id,
    status,
    dbt_valid_from,
    dbt_valid_to,
    case when dbt_valid_to is null then 'CURRENT' else 'HISTORICAL' end as record_type
from snap_customer_status_v2
where customer_id in ('CUST000001', 'CUST000002', 'CUST000003')
order by customer_id, dbt_valid_from;

-- Expected for CUST000001:
-- Row 1: status='trial', dbt_valid_to=(timestamp), HISTORICAL
-- Row 2: status='paid', dbt_valid_to=NULL, CURRENT
```

---

## Step 6: Query Historical Data

### Point-in-Time Query
```sql
/*
    Who was in 'trial' status on February 1, 2024?
    
    Logic: Find records where:
    - status = 'trial'
    - dbt_valid_from <= '2024-02-01'
    - dbt_valid_to > '2024-02-01' OR dbt_valid_to IS NULL
*/

select 
    customer_id,
    customer_name,
    status,
    dbt_valid_from,
    dbt_valid_to
from snap_customer_status_v2
where status = 'trial'
  and dbt_valid_from <= '2024-02-01'
  and (dbt_valid_to > '2024-02-01' or dbt_valid_to is null)
limit 10;
```

### Customer Journey View
```sql
/*
    View complete journey for a customer with status changes
*/

select 
    customer_id,
    customer_name,
    status,
    dbt_valid_from,
    dbt_valid_to,
    datediff('day', dbt_valid_from, coalesce(dbt_valid_to, current_timestamp())) as days_in_status,
    case when dbt_valid_to is null then 'CURRENT' else 'HISTORICAL' end as record_type
from snap_customer_status_v2
where customer_id = (
    -- Get a customer with the most status changes
    select customer_id 
    from snap_customer_status_v2 
    group by customer_id 
    having count(*) > 1 
    limit 1
)
order by dbt_valid_from;
```

### Days in Each Status
```sql
/*
    Calculate total days in each status per customer
*/

select
    customer_id,
    status,
    sum(datediff('day', dbt_valid_from, coalesce(dbt_valid_to, current_timestamp()))) as total_days_in_status
from snap_customer_status_v2
group by customer_id, status
order by customer_id, status;
```

### Current vs Historical Counts
```sql
/*
    Compare current vs historical record counts by status
*/

select
    status,
    count(*) as total_records,
    count(case when dbt_valid_to is null then 1 end) as current_count,
    count(case when dbt_valid_to is not null then 1 end) as historical_count
from snap_customer_status_v2
group by status
order by status;
```

---

## Snapshot Strategies Comparison

### Timestamp Strategy
```sql
{% snapshot snap_timestamp_example %}
{{
    config(
        unique_key='id',
        strategy='timestamp',
        updated_at='updated_at'
    )
}}
select * from {{ source('schema', 'table') }}
{% endsnapshot %}
```

**Pros:**
- Fast (only checks updated_at column)
- Reliable if source has good updated_at tracking
- Works well with large tables

**Cons:**
- Requires updated_at column in source
- Misses changes if updated_at not updated
- Can't detect deletions (unless using invalidate_hard_deletes)

### Check Strategy
```sql
{% snapshot snap_check_example %}
{{
    config(
        unique_key='id',
        strategy='check',
        check_cols=['col1', 'col2', 'col3']
    )
}}
select * from {{ source('schema', 'table') }}
{% endsnapshot %}
```

**Pros:**
- Detects any change in specified columns
- No need for updated_at column
- More reliable for detecting all changes

**Cons:**
- Slower (compares all check_cols)
- Can be expensive on large tables
- More complex to maintain

### Check Strategy - All Columns
```sql
{{
    config(
        unique_key='id',
        strategy='check',
        check_cols='all'
    )
}}
```

Checks all columns for changes (slowest but most comprehensive).

---

## Troubleshooting

### Issue: Snapshot Table Not Created
**Problem:** dbt snapshot succeeds but table doesn't exist

**Solution:**
```sql
-- Check if table exists
show tables in schema analytics.snapshots;

-- Verify permissions
use role transformer_role;
show grants on schema analytics.snapshots;

-- Manually create if needed
use database analytics;
create schema if not exists snapshots;
```

### Issue: All Rows Have NULL dbt_valid_to
**Problem:** No historical records, all rows current

**Cause:** Source data hasn't changed between snapshot runs

**Solution:**
- Update source data (as shown in Step 5)
- Rebuild staging model
- Run snapshot again
- Verify updated_at changed in source

### Issue: Too Many Historical Rows
**Problem:** New row created every snapshot run

**Cause:** updated_at changes every time even when data doesn't

**Solution:**
- Use check strategy instead of timestamp
- Or fix source to only update updated_at when data actually changes

### Issue: Snapshot Doesn't Detect Changes
**Problem:** Data changed but snapshot didn't create new row

**Cause:** unique_key might be wrong or updated_at not updated

**Solution:**
```sql
-- Verify unique_key is actually unique
select unique_key, count(*)
from source_table
group by unique_key
having count(*) > 1;

-- Check if updated_at changed
select old.updated_at as old_updated_at,
       new.updated_at as new_updated_at
from snapshot_table old
join source_table new on old.customer_id = new.customer_id
where old.dbt_valid_to is null;
```

### Issue: Hard Deletes Not Tracked
**Problem:** Deleted source records still show as current in snapshot

**Solution:**
```sql
-- Enable hard delete tracking
{{
    config(
        invalidate_hard_deletes=True
    )
}}
```

When enabled:
- Deleted source records get dbt_valid_to = current_timestamp
- Marks them as no longer current

---

## Best Practices Summary

### Snapshot Configuration
1. Use timestamp strategy when source has reliable updated_at
2. Use check strategy when no updated_at or need comprehensive change detection
3. Always specify unique_key carefully (must be unique per entity)
4. Enable invalidate_hard_deletes if deletions are important

### Snapshot Execution
1. Run snapshots on schedule (daily, hourly) based on business needs
2. Never delete snapshot tables (lose history)
3. Don't manually edit snapshot tables
4. Run snapshot after loading new source data

### Query Patterns
1. Always filter on dbt_valid_to for point-in-time queries
2. Use NULL check for current records: dbt_valid_to IS NULL
3. Calculate days_in_status with COALESCE for current records
4. Join snapshots carefully (specify valid_from/valid_to conditions)

### Performance
1. Add indexes on unique_key and dbt_valid_to (Snowflake clustering)
2. Partition large snapshot tables by dbt_valid_from
3. Consider incremental snapshots for very large tables
4. Monitor snapshot table growth

---

## Common Patterns

### Pattern 1: Current Records Only
```sql
select *
from snap_customer_status_v2
where dbt_valid_to is null;
```

### Pattern 2: Historical Records Only
```sql
select *
from snap_customer_status_v2
where dbt_valid_to is not null;
```

### Pattern 3: Point-in-Time Query
```sql
select *
from snap_customer_status_v2
where dbt_valid_from <= '2024-02-01'
  and (dbt_valid_to > '2024-02-01' or dbt_valid_to is null);
```

### Pattern 4: Latest Change Per Customer
```sql
select
    customer_id,
    status,
    dbt_valid_from as last_change_date
from snap_customer_status_v2
where dbt_valid_to is null;
```

### Pattern 5: Status Transition Analysis
```sql
select
    prev.status as previous_status,
    curr.status as current_status,
    count(*) as transition_count
from snap_customer_status_v2 prev
join snap_customer_status_v2 curr
    on prev.customer_id = curr.customer_id
    and prev.dbt_valid_to = curr.dbt_valid_from
group by prev.status, curr.status
order by transition_count desc;
```

---

## SCD Type 2 Use Cases

### Customer Lifecycle Analysis
- Track progression through sales funnel
- Calculate average time in each status
- Identify drop-off points

### Churn Prediction
- Analyze patterns before churn
- Time in 'at_risk' status before churn
- Win-back success rates

### Historical Reporting
- Month-end status snapshots
- Point-in-time revenue calculations
- Historical segment distributions

### Compliance and Auditing
- Track all changes for regulatory compliance
- Answer "what did we know when" questions
- Reconstruct historical states

---

## Verification Checklist

Before proceeding to Part 8:
- [ ] 2 snapshot files created in snapshots/ directory
- [ ] dbt snapshot executes successfully
- [ ] Snapshot tables created in analytics.snapshots schema
- [ ] SCD metadata columns present (dbt_valid_from, dbt_valid_to, etc.)
- [ ] Can query current records (dbt_valid_to IS NULL)
- [ ] Can query historical records
- [ ] Point-in-time queries work correctly
- [ ] Understand timestamp vs check strategy

---

## Next Steps

After completing snapshots:
1. Build intermediate layer with sessionization (Part 8)
2. Use snapshot data in marts layer (Part 10)
3. Create Type 2 dimension from snapshot

---

## Estimated Time
- Snapshot creation: 15-20 minutes
- Testing and verification: 10-15 minutes
- Simulating changes: 5 minutes
- Querying historical data: 10 minutes
- Total: 30-45 minutes

## Success Criteria
- 2 snapshots created and running
- SCD Type 2 metadata columns present
- Historical tracking working
- Point-in-time queries functional
- Ready for intermediate layer in Part 8
