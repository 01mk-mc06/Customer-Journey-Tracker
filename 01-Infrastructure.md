# Infrastructure Setup

## Overview
Set up Snowflake infrastructure with proper role-based access control (RBAC), warehouses, databases, and schemas for the dbt project.

## Objectives
- Create Snowflake warehouses for different workloads
- Establish databases with proper separation of concerns
- Implement RBAC with 4 distinct roles
- Configure resource monitors for cost control
- Set up audit infrastructure

## Prerequisites
- Snowflake account (Standard, Enterprise, or Business Critical edition)
- ACCOUNTADMIN access
- Basic understanding of Snowflake architecture

---

## Step 1: Create Warehouses

### Purpose
Separate compute resources by workload type to optimize performance and cost.

### Implementation
```sql
use role accountadmin;

-- Loading warehouse (XSMALL for seed loading)
create warehouse if not exists loading_wh
  with warehouse_size = 'XSMALL'
  auto_suspend = 60
  auto_resume = true
  initially_suspended = true
  comment = 'Warehouse for data loading operations';

-- Transformation warehouse (SMALL for dbt models)
create warehouse if not exists transforming_wh
  with warehouse_size = 'SMALL'
  auto_suspend = 60
  auto_resume = true
  initially_suspended = true
  comment = 'Warehouse for dbt transformations';

-- Reporting warehouse (XSMALL for analyst queries)
create warehouse if not exists reporting_wh
  with warehouse_size = 'XSMALL'
  auto_suspend = 300
  auto_resume = true
  initially_suspended = true
  comment = 'Warehouse for analyst queries';
```

### Best Practices
- Use XSMALL for development and testing
- Set auto_suspend to 60 seconds to minimize costs
- Use initially_suspended = true to avoid immediate charges
- Separate warehouses prevent workload interference

---

## Step 2: Create Databases

### Purpose
Organize data by processing stage following medallion architecture principles.

### Implementation
```sql
-- Raw database (bronze layer)
create database if not exists raw
  comment = 'Database for raw source data and seeds';

-- Analytics database (silver/gold layers)
create database if not exists analytics
  comment = 'Database for transformed analytics models';

-- Audit database (governance)
create database if not exists audit
  comment = 'Database for audit logs and metadata tracking';
```

### Schema Structure
```sql
-- RAW schemas
use database raw;
create schema if not exists seeds comment = 'CSV seed data loaded via dbt';
create schema if not exists sources comment = 'External source data';

-- ANALYTICS schemas
use database analytics;
create schema if not exists staging comment = 'Staging layer - cleaned raw data';
create schema if not exists intermediate comment = 'Intermediate layer - business logic';
create schema if not exists marts comment = 'Marts layer - analytics-ready tables';
create schema if not exists snapshots comment = 'SCD Type 2 snapshots';
create schema if not exists utils comment = 'Utility tables and metadata';

-- AUDIT schemas
use database audit;
create schema if not exists dbt_runs comment = 'dbt run metadata and logs';
create schema if not exists data_quality comment = 'Test results and DQ metrics';
```

### Best Practices
- Follow consistent naming conventions (lowercase, underscores)
- Use comments to document purpose
- Separate concerns by database (raw vs analytics vs audit)
- Plan for future growth (sources, utils schemas)

---

## Step 3: Create Roles

### Purpose
Implement least-privilege access control following security best practices.

### Implementation
```sql
-- Loader role (limited to loading data)
create role if not exists loader_role
  comment = 'Role for loading raw data and seeds';

-- Transformer role (dbt execution)
create role if not exists transformer_role
  comment = 'Role for running dbt transformations';

-- Analyst role (read-only access to marts)
create role if not exists analyst_role
  comment = 'Role for analysts querying marts';

-- Data Engineer role (full development access)
create role if not exists data_engineer_role
  comment = 'Role for data engineers - full dev access';
```

### Role Hierarchy
```
ACCOUNTADMIN (not used for day-to-day)
    |
    +-- DATA_ENGINEER_ROLE (full dev access)
    |
    +-- TRANSFORMER_ROLE (dbt execution)
    |
    +-- LOADER_ROLE (load data only)
    |
    +-- ANALYST_ROLE (read marts only)
```

---

## Step 4: Grant Warehouse Permissions

### Purpose
Control which roles can use which warehouses.

### Implementation
```sql
-- Loader role
grant usage on warehouse loading_wh to role loader_role;

-- Transformer role
grant usage on warehouse transforming_wh to role transformer_role;

-- Analyst role
grant usage on warehouse reporting_wh to role analyst_role;

-- Data engineer role (all warehouses)
grant usage on warehouse loading_wh to role data_engineer_role;
grant usage on warehouse transforming_wh to role data_engineer_role;
grant usage on warehouse reporting_wh to role data_engineer_role;
```

### Best Practices
- Grant minimum required warehouse access per role
- Prevent analysts from using expensive transformation warehouses
- Allow data engineers full access for troubleshooting

---

## Step 5: Grant Database and Schema Permissions

### Loader Role Grants
```sql
-- Can write to RAW database only
grant usage on database raw to role loader_role;
grant usage, create table on schema raw.seeds to role loader_role;
grant usage, create table on schema raw.sources to role loader_role;
```

### Transformer Role Grants
```sql
-- Read access to RAW
grant usage on database raw to role transformer_role;
grant usage on all schemas in database raw to role transformer_role;
grant select on all tables in database raw to role transformer_role;
grant select on future tables in database raw to role transformer_role;

-- Read/write access to ANALYTICS
grant usage on database analytics to role transformer_role;
grant create schema on database analytics to role transformer_role;
grant usage on all schemas in database analytics to role transformer_role;
grant create table on all schemas in database analytics to role transformer_role;
grant create view on all schemas in database analytics to role transformer_role;
grant all privileges on all tables in database analytics to role transformer_role;
grant all privileges on all views in database analytics to role transformer_role;
grant all privileges on future tables in database analytics to role transformer_role;
grant all privileges on future views in database analytics to role transformer_role;

-- Write access to AUDIT
grant usage on database audit to role transformer_role;
grant usage on all schemas in database audit to role transformer_role;
grant create table on all schemas in database audit to role transformer_role;
grant select, insert on all tables in database audit to role transformer_role;
```

### Analyst Role Grants
```sql
-- Read MARTS schema only
grant usage on database analytics to role analyst_role;
grant usage on schema analytics.marts to role analyst_role;
grant select on all tables in schema analytics.marts to role analyst_role;
grant select on all views in schema analytics.marts to role analyst_role;
grant select on future tables in schema analytics.marts to role analyst_role;
grant select on future views in schema analytics.marts to role analyst_role;
```

### Critical Notes
- Use FUTURE grants to automatically apply permissions to new objects
- Avoid granting ALL PRIVILEGES unless necessary
- Separate read and write access by role

---

## Step 6: Assign Roles to Users

### Implementation
```sql
-- Replace YOUR_USERNAME with actual Snowflake username
grant role loader_role to user YOUR_USERNAME;
grant role transformer_role to user YOUR_USERNAME;
grant role analyst_role to user YOUR_USERNAME;
grant role data_engineer_role to user YOUR_USERNAME;

-- Set default role
alter user YOUR_USERNAME set default_role = transformer_role;
```

### Best Practices
- Set transformer_role as default for dbt development
- Users can switch roles as needed: use role analyst_role;
- Document which role to use for which tasks

---

## Step 7: Create Audit Tables

### Purpose
Track dbt execution metadata and data quality test results.

### Implementation
```sql
use database audit;
use schema dbt_runs;

-- Table to track dbt model execution times
create table if not exists model_run_stats (
    run_id varchar(36),
    model_name varchar(255),
    execution_start_time timestamp_ntz,
    execution_end_time timestamp_ntz,
    execution_time_seconds number(10,2),
    rows_affected number,
    status varchar(50),
    created_at timestamp_ntz default current_timestamp()
);

grant select, insert on table audit.dbt_runs.model_run_stats to role transformer_role;

use schema data_quality;

-- Table to track test failures
create table if not exists test_results (
    test_id varchar(36),
    test_name varchar(255),
    model_name varchar(255),
    test_type varchar(50),
    test_status varchar(20),
    failure_count number,
    test_timestamp timestamp_ntz default current_timestamp()
);

grant select, insert on table audit.data_quality.test_results to role transformer_role;
```

### Use Cases
- Monitor model execution performance
- Track test failures over time
- Identify slow-running models
- Create alerting based on test results

---

## Step 8: Set Up Resource Monitors (Optional)

### Purpose
Control costs by limiting warehouse credit consumption.

### Implementation
```sql
create resource monitor if not exists dev_monitor
  with credit_quota = 10  -- 10 credits per month
  frequency = monthly
  start_timestamp = immediately
  triggers
    on 75 percent do notify
    on 100 percent do suspend
    on 110 percent do suspend_immediate;

-- Assign to warehouses
alter warehouse loading_wh set resource_monitor = dev_monitor;
alter warehouse transforming_wh set resource_monitor = dev_monitor;
alter warehouse reporting_wh set resource_monitor = dev_monitor;
```

### Best Practices
- Set quota based on budget (1 credit approximately 2-3 USD)
- Use notify at 75% to get early warning
- Use suspend at 100% to prevent overages
- Monitor actual usage and adjust quota as needed

---

## Verification Queries

### Check Warehouses
```sql
show warehouses;
```

### Check Databases and Schemas
```sql
show databases;
use database analytics;
show schemas;
```

### Check Roles
```sql
show roles;
```

### Verify Grants for Transformer Role
```sql
show grants to role transformer_role;
```

### Test Connection as Transformer
```sql
use role transformer_role;
use warehouse transforming_wh;
use database analytics;
select current_role(), current_warehouse(), current_database();
```

---

## Troubleshooting

### Issue: Permission Denied Errors
**Problem:** User cannot access databases or warehouses

**Solution:**
1. Verify roles are granted to user:
   ```sql
   show grants to user YOUR_USERNAME;
   ```
2. Re-run grant statements
3. Switch to correct role:
   ```sql
   use role transformer_role;
   ```

### Issue: Database-Level CREATE VIEW Not Allowed
**Problem:** Error - "invalid privilege CREATE VIEW on database"

**Solution:**
- CREATE VIEW is a schema-level privilege, not database-level
- Correct syntax:
  ```sql
  grant create view on all schemas in database analytics to role transformer_role;
  ```

### Issue: Future Grants Not Working
**Problem:** New tables created don't have permissions

**Solution:**
- Verify future grants are in place:
  ```sql
  show future grants in database analytics;
  ```
- Re-apply future grants if missing

### Issue: Resource Monitor Suspends Warehouse During Development
**Problem:** Warehouse suspends unexpectedly

**Solution:**
- Temporarily disable resource monitor:
  ```sql
  alter warehouse transforming_wh set resource_monitor = null;
  ```
- Or increase quota for development period

---

## Improvements and Optimizations

### Multi-Environment Setup
For production deployments, create separate environments:
```sql
-- Development environment
create database if not exists dev_analytics;
create warehouse if not exists dev_transforming_wh with warehouse_size = 'XSMALL';

-- Production environment  
create database if not exists prod_analytics;
create warehouse if not exists prod_transforming_wh with warehouse_size = 'MEDIUM';
```

### Network Policies
Restrict access by IP address:
```sql
create network policy if not exists office_only
  allowed_ip_list = ('203.0.113.0/24');  -- Replace with your IP range
  
alter user YOUR_USERNAME set network_policy = office_only;
```

### Session Policies
Set default warehouse and role:
```sql
alter user YOUR_USERNAME set 
  default_warehouse = transforming_wh
  default_role = transformer_role
  default_namespace = analytics.staging;
```

---

## Best Practices Summary

1. **Security**
   - Follow least-privilege principle
   - Never use ACCOUNTADMIN for day-to-day operations
   - Separate roles by function
   - Use future grants to maintain security on new objects

2. **Cost Control**
   - Use smallest warehouse size that meets performance needs
   - Set aggressive auto_suspend times (60 seconds)
   - Use resource monitors in all environments
   - Monitor warehouse usage weekly

3. **Organization**
   - Consistent naming conventions (lowercase, underscores)
   - Comment all objects for documentation
   - Separate databases by data stage (raw, analytics, audit)
   - Create schemas for logical groupings

4. **Maintainability**
   - Document all custom configurations
   - Version control all SQL scripts
   - Create runbooks for common tasks
   - Regular access reviews (quarterly)

---

## Next Steps

After completing infrastructure setup:
1. Install dbt and configure profiles.yml (Part 2)
2. Verify connection with dbt debug
3. Begin building dbt project structure

---

## Estimated Time
- Initial setup: 20-30 minutes
- Verification: 10 minutes
- Troubleshooting (if needed): 10-20 minutes

## Success Criteria
- All warehouses created and accessible
- All databases and schemas exist
- All roles have appropriate permissions
- Test connection succeeds as transformer_role
- dbt debug passes (in Part 2)
