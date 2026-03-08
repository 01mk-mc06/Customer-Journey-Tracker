# Staging Layer

## Overview
Build the staging layer with data cleaning, standardization, and basic transformations on top of seed data.

## Objectives
- Create 5 staging models as views
- Implement data cleaning (lowercase, trim)
- Apply status updates to customers
- Establish naming conventions
- Reference sources properly
- Prepare clean data for intermediate layer

## Prerequisites
- Part 4 completed (seeds loaded, sources.yml configured)
- Understanding of SQL transformations
- Familiarity with dbt ref() and source() functions

---

## Staging Layer Principles

### Purpose of Staging
- Clean and standardize raw data
- Rename columns for consistency
- Cast data types appropriately
- Apply minimal business logic
- One-to-one mapping with source tables (mostly)

### Materialization Strategy
- Use views for staging (lightweight, always fresh)
- No aggregations or complex joins
- Fast to rebuild
- Low storage cost

### Naming Convention
- Prefix: stg_
- Source-aligned names (stg_customers, stg_interactions)
- Lowercase with underscores
- Descriptive and concise

---

## Step 1: Create stg_customers_with_updates

### Purpose
Clean customer data and apply latest status updates from seed_customer_status_updates.

### Implementation

File: models/staging/stg_customers_with_updates.sql

```sql
{{
    config(
        materialized='view',
        tags=['staging', 'daily']
    )
}}

/*
    Staging model for customers with status updates applied
    
    Cleaning:
    - Lowercase and trim text fields
    - Standardize column names
    
    Business Logic:
    - Apply latest status update via LEFT JOIN
    - Use original status if no updates exist
*/

with source_customers as (
    select * from {{ source('raw_seeds', 'seed_customers') }}
),

source_updates as (
    select * from {{ source('raw_seeds', 'seed_customer_status_updates') }}
),

-- Get the latest status update per customer
latest_updates as (
    select
        customer_id,
        status,
        updated_at,
        updated_by,
        row_number() over (
            partition by customer_id 
            order by updated_at desc
        ) as rn
    from source_updates
),

-- Filter to most recent update only
current_updates as (
    select
        customer_id,
        status,
        updated_at,
        updated_by
    from latest_updates
    where rn = 1
),

-- Clean and apply updates
cleaned as (
    select
        c.customer_id,
        lower(trim(c.customer_name)) as customer_name,
        lower(trim(c.email)) as email,
        trim(c.phone) as phone,
        lower(trim(c.segment)) as segment,
        
        -- Use updated status if available, otherwise original
        coalesce(u.status, c.status) as status,
        
        c.created_at,
        
        -- Use update timestamp if available, otherwise original
        coalesce(u.updated_at, c.updated_at) as updated_at,
        coalesce(u.updated_by, c.updated_by) as updated_by,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source_customers c
    left join current_updates u on c.customer_id = u.customer_id
)

select * from cleaned
```

### Design Decisions

**Why LEFT JOIN?**
- Preserves all customers even without status updates
- Uses COALESCE to handle nulls gracefully
- Avoids filtering out customers

**Why ROW_NUMBER()?**
- Handles multiple updates per customer
- Gets latest update by timestamp
- Window function more efficient than subquery

**Why COALESCE?**
- Safe null handling
- Fallback to original values
- Explicit logic visible in code

### Best Practices
- Document CTEs with comments
- Use meaningful CTE names (latest_updates, current_updates)
- Add dbt_loaded_at for audit trail
- Keep transformations simple (save complex logic for intermediate)

---

## Step 2: Create stg_customer_status_updates

### Purpose
Clean status update events.

### Implementation

File: models/staging/stg_customer_status_updates.sql

```sql
{{
    config(
        materialized='view',
        tags=['staging', 'daily']
    )
}}

with source as (
    select * from {{ source('raw_seeds', 'seed_customer_status_updates') }}
),

cleaned as (
    select
        customer_id,
        lower(trim(status)) as status,
        updated_at,
        lower(trim(updated_by)) as updated_by,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source
)

select * from cleaned
```

### Simple Transformations
- Lowercase status and updated_by
- Trim whitespace
- Add metadata column
- No complex logic needed

---

## Step 3: Create stg_interaction_types

### Purpose
Clean interaction type reference data.

### Implementation

File: models/staging/stg_interaction_types.sql

```sql
{{
    config(
        materialized='view',
        tags=['staging', 'reference']
    )
}}

with source as (
    select * from {{ source('raw_seeds', 'seed_interaction_types') }}
),

cleaned as (
    select
        interaction_type_id,
        lower(trim(interaction_type)) as interaction_type,
        lower(trim(type_category)) as type_category,
        is_billable,
        avg_duration_seconds,
        trim(description) as description,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source
)

select * from cleaned
```

### Reference Data Pattern
- Minimal transformations (already clean)
- Standardize text fields
- Preserve IDs and numeric values as-is
- Tag as 'reference' for identification

---

## Step 4: Create stg_interactions

### Purpose
Clean customer interaction events.

### Implementation

File: models/staging/stg_interactions.sql

```sql
{{
    config(
        materialized='view',
        tags=['staging', 'daily']
    )
}}

with source as (
    select * from {{ source('raw_seeds', 'seed_interactions') }}
),

cleaned as (
    select
        interaction_id,
        customer_id,
        lower(trim(interaction_type)) as interaction_type,
        interaction_timestamp,
        duration_seconds,
        lower(trim(channel)) as channel,
        sentiment_score,
        lower(trim(resolution_status)) as resolution_status,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source
)

select * from cleaned
```

### Event Data Pattern
- Preserve IDs and timestamps exactly
- Lowercase categorical fields
- Keep numeric values unchanged
- No filtering or aggregation

---

## Step 5: Create stg_dates

### Purpose
Clean date dimension.

### Implementation

File: models/staging/stg_dates.sql

```sql
{{
    config(
        materialized='view',
        tags=['staging', 'reference']
    )
}}

with source as (
    select * from {{ source('raw_seeds', 'seed_date_dimension') }}
),

cleaned as (
    select
        date_key,
        date,
        lower(trim(day_of_week)) as day_of_week,
        day_of_month,
        week_of_year,
        month,
        lower(trim(month_name)) as month_name,
        quarter,
        year,
        is_weekend,
        is_holiday,
        
        -- Metadata
        current_timestamp() as dbt_loaded_at
        
    from source
)

select * from cleaned
```

### Date Dimension Pattern
- Preserve date_key and date exactly
- Lowercase day/month names
- Keep flags and numbers unchanged
- Reference data (rarely changes)

---

## Step 6: Build All Staging Models

### Purpose
Execute dbt run to create all staging views.

### Implementation
```powershell
# Build all staging models
dbt run --select staging

# Or build specific model
dbt run --select stg_customers_with_updates
```

### Expected Output
```
Running with dbt=1.8.0
Found 5 models

Concurrency: 4 threads

1 of 5 START sql view model staging_staging.stg_customers_with_updates ... [RUN]
2 of 5 START sql view model staging_staging.stg_customer_status_updates  [RUN]
3 of 5 START sql view model staging_staging.stg_dates ................... [RUN]
4 of 5 START sql view model staging_staging.stg_interaction_types ....... [RUN]
1 of 5 OK created sql view model staging_staging.stg_customers_with_updates [SUCCESS 1 in 2.5s]
3 of 5 OK created sql view model staging_staging.stg_dates ............... [SUCCESS 1 in 2.3s]
4 of 5 OK created sql view model staging_staging.stg_interaction_types ... [SUCCESS 1 in 2.4s]
5 of 5 START sql view model staging_staging.stg_interactions ............. [RUN]
2 of 5 OK created sql view model staging_staging.stg_customer_status_updates [SUCCESS 1 in 2.8s]
5 of 5 OK created sql view model staging_staging.stg_interactions ........ [SUCCESS 1 in 2.6s]

Completed successfully
```

### Verify in Snowflake
```sql
use database analytics;
use schema staging_staging;

-- List all staging views
show views in schema analytics.staging_staging;

-- Check row counts
select 'stg_customers_with_updates' as view_name, count(*) as row_count 
from stg_customers_with_updates
union all
select 'stg_customer_status_updates', count(*) from stg_customer_status_updates
union all
select 'stg_interaction_types', count(*) from stg_interaction_types
union all
select 'stg_interactions', count(*) from stg_interactions
union all
select 'stg_dates', count(*) from stg_dates;

-- Sample cleaned data
select * from stg_customers_with_updates limit 5;
```

---

## Step 7: Verify Data Transformations

### Check Lowercase Transformation
```sql
-- Should all be lowercase
select distinct segment from stg_customers_with_updates;
-- Expected: 'enterprise', 'smb', 'consumer'

select distinct status from stg_customers_with_updates;
-- Expected: 'lead', 'trial', 'paid', 'at_risk', 'churned'
```

### Check Status Updates Applied
```sql
-- Compare original vs updated status
select 
    c.customer_id,
    c.status as original_status,
    s.status as current_status,
    case when c.status != s.status then 'UPDATED' else 'UNCHANGED' end as status_change
from raw.staging_seeds.seed_customers c
join analytics.staging_staging.stg_customers_with_updates s 
    on c.customer_id = s.customer_id
where c.status != s.status
limit 10;
```

### Check Null Handling
```sql
-- Check for unexpected nulls
select 
    sum(case when customer_id is null then 1 else 0 end) as null_customer_id,
    sum(case when customer_name is null then 1 else 0 end) as null_customer_name,
    sum(case when segment is null then 1 else 0 end) as null_segment,
    sum(case when status is null then 1 else 0 end) as null_status
from stg_customers_with_updates;
```

---

## Troubleshooting

### Issue: View Not Found
**Problem:** dbt run succeeds but view doesn't exist in Snowflake

**Solution:**
- Check you're using correct database/schema:
  ```sql
  use database analytics;
  use schema staging_staging;
  show views;
  ```
- Verify role has CREATE VIEW permissions
- Check compiled SQL in target/run/ directory

### Issue: FROM Clause Missing
**Problem:** SQL compilation error - missing FROM clause

**Solution:**
- Every SELECT in a CTE needs FROM clause
- Check final CTE has: select * from cte_name

### Issue: Ambiguous Column Reference
**Problem:** Column reference is ambiguous in JOIN

**Solution:**
- Prefix all columns with table alias:
  ```sql
  select 
      c.customer_id,  -- Good
      customer_name   -- Bad (ambiguous if both tables have it)
  from customers c
  join updates u on c.id = u.id
  ```

### Issue: Status Updates Not Applied
**Problem:** All statuses same as original

**Solution:**
- Verify seed_customer_status_updates loaded:
  ```sql
  select count(*) from raw.staging_seeds.seed_customer_status_updates;
  ```
- Check JOIN condition matches customer_id exactly
- Verify ROW_NUMBER() partitioning correct

### Issue: Duplicate Rows in Output
**Problem:** More rows than source table

**Solution:**
- Check for multiple status updates per customer
- Verify ROW_NUMBER() filter (where rn = 1)
- Use DISTINCT if needed (but investigate root cause)

---

## Best Practices Summary

### SQL Style
1. Use CTEs for readability (with source as, cleaned as, final as)
2. One column per line in SELECT
3. Always alias tables in JOINs
4. Comment complex logic
5. Add trailing commas for easy column additions

### dbt Conventions
1. Prefix staging models with stg_
2. Use source() function, never hardcode table names
3. Materialize as views (lightweight)
4. Tag by layer and refresh frequency
5. Add dbt_loaded_at metadata column

### Data Transformations
1. Lowercase categorical text fields
2. Trim all text fields
3. Preserve original data types when possible
4. Handle nulls explicitly (COALESCE)
5. Document any filtering logic

### Testing Strategy
1. Test staging models separately (Part 6)
2. Verify row counts match sources
3. Check for unexpected nulls
4. Validate transformations worked
5. Test downstream dependencies

---

## Common Patterns

### Pattern 1: Basic Cleaning
```sql
with source as (
    select * from {{ source('schema', 'table') }}
),

cleaned as (
    select
        id,
        lower(trim(text_field)) as text_field,
        numeric_field,
        timestamp_field,
        current_timestamp() as dbt_loaded_at
    from source
)

select * from cleaned
```

### Pattern 2: Apply Updates via LEFT JOIN
```sql
with base as (
    select * from {{ source('schema', 'base_table') }}
),

updates as (
    select * from {{ source('schema', 'update_table') }}
    qualify row_number() over (partition by id order by updated_at desc) = 1
),

final as (
    select
        b.id,
        coalesce(u.field, b.field) as field,
        coalesce(u.updated_at, b.updated_at) as updated_at
    from base b
    left join updates u on b.id = u.id
)

select * from final
```

### Pattern 3: Reference Data
```sql
with source as (
    select * from {{ source('schema', 'reference_table') }}
),

cleaned as (
    select
        reference_id,
        lower(trim(reference_name)) as reference_name,
        is_active,
        current_timestamp() as dbt_loaded_at
    from source
)

select * from cleaned
```

---

## Schema Naming Clarification

### Actual Schema Created
With dbt's default behavior:
- Config says: +schema: staging
- Target schema: staging (from profiles.yml)
- Result: staging_staging (target prefix + custom schema)

### Accepted for This Project
- We accept staging_staging naming
- Document in README
- Alternative: use custom generate_schema_name macro (not recommended for learning)

### Future Reference
In production, options:
1. Accept prefixing (most common)
2. Use custom macro to override
3. Set target schema to empty string (not recommended)

---

## Verification Checklist

Before proceeding to Part 6:
- [ ] 5 staging views created in analytics.staging_staging
- [ ] All views query successfully
- [ ] Row counts match seed tables (or explain differences)
- [ ] Text fields lowercase and trimmed
- [ ] Status updates applied to customers
- [ ] No unexpected nulls in critical fields
- [ ] dbt_loaded_at column present on all models
- [ ] source() function used (not hardcoded table names)

---

## Next Steps

After completing staging layer:
1. Create schema.yml with tests (Part 6)
2. Implement data quality checks
3. Document all columns
4. Run comprehensive test suite

---

## Estimated Time
- Model creation: 30-45 minutes
- Building and verification: 15-20 minutes
- Troubleshooting: 10-15 minutes (if needed)
- Total: 60-90 minutes

## Success Criteria
- 5 staging views built successfully
- All transformations applied correctly
- Data clean and standardized
- Ready for testing in Part 6
- Lineage visible in dbt docs
