# Part 4: Seed Loading and Sources Configuration

## Overview
Load CSV seed data into Snowflake using dbt seed command and configure sources.yml to reference the loaded tables.

## Objectives
- Load 5 CSV files into Snowflake RAW database
- Configure sources.yml for source data referencing
- Run source freshness checks
- Validate data loaded correctly
- Test source-level data quality

## Prerequisites
- Part 3 completed (5 CSV files generated in seeds/)
- dbt project configured
- Snowflake connection active
- Virtual environment activated

---

## Step 1: Review Seed Configuration

### Purpose
Verify seed configuration in dbt_project.yml before loading.

### Configuration Check
```yaml
seeds:
  bpo_customer_journey:
    +schema: seeds
    +database: raw
    +tags: ['seeds']
```

### What This Does
- database: raw - Seeds load to RAW database (not ANALYTICS)
- schema: seeds - Creates raw.seeds schema
- Note: dbt prepends target schema (dev_seeds or staging_seeds)

### Actual Schema Created
With default behavior: raw.staging_seeds

If this is undesired, create custom macro:
```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

For this project, we accept staging_seeds schema.

---

## Step 2: Load Seeds into Snowflake

### Purpose
Execute dbt seed to load CSV files into Snowflake tables.

### Implementation
```powershell
# Ensure in correct directory
cd C:\Users\YOUR_USERNAME\dbt_learning\bpo_customer_journey

# Load all seeds
dbt seed
```

### Expected Output
```
Running with dbt=1.8.0
Found 5 seeds

Concurrency: 4 threads

1 of 5 START seed file raw.staging_seeds.seed_customers .................. [RUN]
2 of 5 START seed file raw.staging_seeds.seed_customer_status_updates ... [RUN]
3 of 5 START seed file raw.staging_seeds.seed_date_dimension ............. [RUN]
4 of 5 START seed file raw.staging_seeds.seed_interaction_types .......... [RUN]
1 of 5 OK loaded seed file staging_seeds.seed_customers .................. [INSERT 1010 in 3.2s]
3 of 5 OK loaded seed file staging_seeds.seed_date_dimension ............. [INSERT 91 in 3.1s]
4 of 5 OK loaded seed file staging_seeds.seed_interaction_types .......... [INSERT 7 in 3.2s]
5 of 5 START seed file raw.staging_seeds.seed_interactions ............... [RUN]
2 of 5 OK loaded seed file staging_seeds.seed_customer_status_updates ... [INSERT 432 in 4.1s]
5 of 5 OK loaded seed file staging_seeds.seed_interactions ............... [INSERT 5775 in 5.3s]

Completed successfully
```

### What Happens
1. dbt reads CSV files from seeds/ directory
2. Creates tables in raw.staging_seeds schema
3. Truncates and loads data (full refresh by default)
4. Runs in parallel (4 threads)
5. Reports row counts and execution time

### Best Practices
- Run dbt seed with --full-refresh when data changes
- Use --select flag for specific seeds: dbt seed --select seed_customers
- Monitor execution time (large seeds may need warehouse size increase)

---

## Step 3: Verify Seeds in Snowflake

### Purpose
Confirm data loaded correctly in Snowflake.

### Verification Queries
```sql
use role transformer_role;
use warehouse transforming_wh;
use database raw;
use schema staging_seeds;

-- List all seed tables
show tables in schema raw.staging_seeds;

-- Check row counts
select 'seed_customers' as table_name, count(*) as row_count from seed_customers
union all
select 'seed_customer_status_updates', count(*) from seed_customer_status_updates
union all
select 'seed_interaction_types', count(*) from seed_interaction_types
union all
select 'seed_interactions', count(*) from seed_interactions
union all
select 'seed_date_dimension', count(*) from seed_date_dimension;

-- Sample data from each table
select * from seed_customers limit 5;
select * from seed_interactions limit 5;
```

### Expected Row Counts
- seed_customers: 1,010
- seed_customer_status_updates: 432
- seed_interaction_types: 7
- seed_interactions: 5,775
- seed_date_dimension: 91

---

## Step 4: Create sources.yml

### Purpose
Define source metadata to reference seed tables in staging models.

### File Location
models/staging/sources.yml

### Implementation
```yaml
version: 2

sources:
  - name: raw_seeds
    description: "Seed data loaded from CSV files into RAW database"
    database: raw
    schema: staging_seeds  # Actual schema after dbt prefixing
    
    tables:
      - name: seed_customers
        description: "Customer master data with intentional duplicates for testing"
        columns:
          - name: customer_id
            description: "Primary key - customer identifier (10 duplicates intentional)"
            tests:
              - not_null
          
          - name: customer_name
            description: "Customer full name"
          
          - name: email
            description: "Customer email (some nulls intentional)"
          
          - name: phone
            description: "Customer phone in format +63-XXX-XXX-XXXX (some invalid intentional)"
          
          - name: segment
            description: "Customer segment (enterprise, smb, consumer)"
          
          - name: status
            description: "Current customer status (lead, trial, paid, at_risk, churned)"
          
          - name: created_at
            description: "Customer creation timestamp"
          
          - name: updated_at
            description: "Last update timestamp"
          
          - name: updated_by
            description: "User who last updated the record"
      
      - name: seed_customer_status_updates
        description: "Customer status change events for SCD Type 2 tracking"
        columns:
          - name: customer_id
            description: "Foreign key to seed_customers"
            tests:
              - not_null
          
          - name: status
            description: "New status value"
            tests:
              - not_null
          
          - name: updated_at
            description: "Timestamp of status change"
            tests:
              - not_null
          
          - name: updated_by
            description: "User who made the status change"
            tests:
              - not_null
      
      - name: seed_interaction_types
        description: "Reference data for interaction types in BPO operations"
        columns:
          - name: interaction_type_id
            description: "Primary key - interaction type identifier"
            tests:
              - unique
              - not_null
          
          - name: interaction_type
            description: "Interaction type name"
          
          - name: type_category
            description: "Category (voice, digital)"
          
          - name: is_billable
            description: "Whether this interaction type is billable"
          
          - name: avg_duration_seconds
            description: "Average expected duration in seconds"
          
          - name: description
            description: "Interaction type description"
      
      - name: seed_interactions
        description: "Customer interaction events across multiple channels"
        columns:
          - name: interaction_id
            description: "Primary key - UUID for each interaction"
            tests:
              - unique
              - not_null
          
          - name: customer_id
            description: "Foreign key to seed_customers"
            tests:
              - not_null
          
          - name: interaction_type
            description: "Type of interaction (foreign key to seed_interaction_types)"
          
          - name: interaction_timestamp
            description: "When the interaction occurred"
          
          - name: duration_seconds
            description: "Interaction duration in seconds"
          
          - name: channel
            description: "Channel used (phone, web_chat, email, mobile_app)"
          
          - name: sentiment_score
            description: "Sentiment score from -1.0 to 1.0"
          
          - name: resolution_status
            description: "Resolution status (resolved, pending, escalated, closed)"
      
      - name: seed_date_dimension
        description: "Date dimension for Q1 2024 (Jan 1 - Mar 31)"
        columns:
          - name: date_key
            description: "Primary key - YYYYMMDD format"
            tests:
              - unique
              - not_null
          
          - name: date
            description: "Actual date value"
            tests:
              - unique
              - not_null
          
          - name: day_of_week
            description: "Day name (Monday, Tuesday, etc.)"
          
          - name: day_of_month
            description: "Day number (1-31)"
          
          - name: week_of_year
            description: "ISO week number"
          
          - name: month
            description: "Month number (1-12)"
          
          - name: month_name
            description: "Month name (January, February, etc.)"
          
          - name: quarter
            description: "Quarter number (1-4)"
          
          - name: year
            description: "Year (2024)"
          
          - name: is_weekend
            description: "Boolean flag for Saturday/Sunday"
          
          - name: is_holiday
            description: "Boolean flag for Philippine holidays"
```

### Best Practices
- Document all columns even if no tests
- Use descriptive descriptions
- Add tests at source level for critical fields
- Note intentional data quality issues in descriptions

---

## Step 5: Test Source Data Quality

### Purpose
Run source-level tests to validate data quality.

### Implementation
```powershell
# Test all sources
dbt test --select source:raw_seeds
```

### Expected Results
Most tests pass except intentional failures:
- FAIL: unique test on seed_customers.customer_id (10 duplicates)
- PASS: All other tests

### Test Output Interpretation
```
1 of 8 START test source_unique_raw_seeds_seed_customers_customer_id ... [RUN]
1 of 8 FAIL 10 source_unique_raw_seeds_seed_customers_customer_id ..... [FAIL 10 in 2.1s]
2 of 8 START test source_not_null_raw_seeds_seed_customers_customer_id  [RUN]
2 of 8 PASS source_not_null_raw_seeds_seed_customers_customer_id ...... [PASS in 1.8s]
...
```

FAIL 10 means: 10 records failed the test (expected - our duplicates)

---

## Step 6: Configure Source Freshness (Optional)

### Purpose
Monitor when source data was last updated.

### Implementation
Add to sources.yml:
```yaml
sources:
  - name: raw_seeds
    description: "Seed data loaded from CSV files"
    database: raw
    schema: staging_seeds
    
    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    
    loaded_at_field: updated_at  # Column to check for freshness
    
    tables:
      # ... table definitions
```

### Run Freshness Check
```powershell
dbt source freshness
```

### Use Cases
- Automated data pipeline monitoring
- Alert on stale data
- SLA compliance tracking

Note: Less relevant for static seed data, more useful for external sources.

---

## Step 7: Use Sources in Staging Models

### Purpose
Reference source tables using source() function instead of hardcoded names.

### Correct Usage
```sql
-- models/staging/stg_customers.sql
{{ config(materialized='view') }}

with source as (
    select * from {{ source('raw_seeds', 'seed_customers') }}
),

cleaned as (
    select
        customer_id,
        lower(trim(customer_name)) as customer_name,
        lower(trim(email)) as email,
        -- ... more transformations
    from source
)

select * from cleaned
```

### Why Use source() Function
1. dbt tracks lineage (visible in docs)
2. Source freshness checks work
3. Easy to switch between environments (dev/prod)
4. Better error messages if source missing

### Incorrect Usage (Don't Do This)
```sql
-- Bad: hardcoded table reference
select * from raw.staging_seeds.seed_customers

-- Bad: direct reference without source()
select * from seed_customers
```

---

## Troubleshooting

### Issue: Schema Not Found
**Problem:** dbt seed fails with "schema does not exist"

**Solution:**
```sql
-- Manually create schema if needed
use database raw;
create schema if not exists staging_seeds;

-- Re-run dbt seed
dbt seed
```

### Issue: Permission Denied
**Problem:** transformer_role cannot create tables in raw database

**Solution:**
```sql
use role accountadmin;

-- Grant necessary permissions
grant usage on database raw to role transformer_role;
grant create table on schema raw.staging_seeds to role transformer_role;
grant all privileges on all tables in schema raw.staging_seeds to role transformer_role;
```

### Issue: CSV Encoding Errors
**Problem:** Special characters in CSV cause errors

**Solution:**
1. Save CSV files with UTF-8 encoding
2. Or add to dbt_project.yml:
```yaml
seeds:
  bpo_customer_journey:
    +quote_columns: true
```

### Issue: Date Format Errors
**Problem:** Dates not parsing correctly

**Solution:**
Specify column types in dbt_project.yml:
```yaml
seeds:
  bpo_customer_journey:
    seed_customers:
      +column_types:
        created_at: timestamp
        updated_at: timestamp
```

### Issue: Source Not Found in Lineage
**Problem:** source() function doesn't resolve

**Solution:**
1. Verify sources.yml in models/staging/ directory
2. Check source name and table name match exactly (case-sensitive)
3. Run dbt parse to refresh project state
4. Check schema name matches actual Snowflake schema

---

## Best Practices Summary

### Seed Management
1. Keep seeds small (under 10,000 rows)
2. Use for reference data and test data only
3. Version control CSV files
4. Document seed purpose in dbt_project.yml

### Source Configuration
1. Always use source() function in models
2. Document all columns even without tests
3. Add tests for critical fields
4. Note intentional data quality issues

### Testing Strategy
1. Test sources for basic data quality
2. Accept intentional failures (document them)
3. Use severity: warn for non-critical issues
4. Test relationships between sources

### Performance
1. Use --select for targeted seed loading
2. Monitor warehouse usage during large seed loads
3. Consider increasing warehouse size for 100K+ row seeds
4. Use --full-refresh sparingly (truncates tables)

---

## Common Patterns

### Loading Specific Seeds
```powershell
# Load one seed
dbt seed --select seed_customers

# Load multiple seeds
dbt seed --select seed_customers seed_interactions

# Load by path
dbt seed --select seeds/seed_*.csv
```

### Updating Seeds
```powershell
# Full refresh (truncate and reload)
dbt seed --full-refresh

# Incremental load not supported for seeds
# Always use full refresh
```

### Testing Seeds
```powershell
# Test all sources
dbt test --select source:raw_seeds

# Test specific source table
dbt test --select source:raw_seeds.seed_customers

# Run models and tests together
dbt build --select source:raw_seeds
```

---

## Verification Checklist

Before proceeding to Part 5:
- [ ] dbt seed completed successfully
- [ ] All 5 tables visible in Snowflake raw.staging_seeds schema
- [ ] Row counts match expected values
- [ ] sources.yml created in models/staging/
- [ ] Source tests configured and run
- [ ] Intentional test failures documented
- [ ] source() function works in test models

---

## Next Steps

After loading seeds and configuring sources:
1. Build staging models on top of sources (Part 5)
2. Implement data cleaning and standardization
3. Add comprehensive testing
4. Document transformations

---

## Estimated Time
- Seed loading: 5-10 minutes
- sources.yml creation: 15-20 minutes
- Source testing: 5 minutes
- Verification: 5 minutes
- Troubleshooting: 10-15 minutes (if needed)

Total: 40-55 minutes

## Success Criteria
- All seeds loaded to raw.staging_seeds
- Row counts match generated data
- sources.yml properly configured
- Source tests run (expected failures documented)
- Ready to build staging layer
