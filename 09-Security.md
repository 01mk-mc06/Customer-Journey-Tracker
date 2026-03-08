# Data Governance and Security

## Overview
Implement enterprise-grade security using Snowflake native features including dynamic data masking, row access policies, and tag-based governance.

## Objectives
- Implement dynamic data masking for PII (email, phone, name)
- Create row access policies for segment filtering
- Apply security policies to all layers
- Test security as different roles
- Tag protected columns
- Document security architecture

## Prerequisites
- Part 8 completed (intermediate layer built)
- Snowflake Enterprise Edition (or Business Critical)
- ACCOUNTADMIN access
- Understanding of RBAC concepts

---

## Security Architecture

### Security Layers

**Layer 1: Role-Based Access Control (RBAC)**
- Created in Part 1
- Controls who can access what
- Foundation for all security

**Layer 2: Dynamic Data Masking**
- Automatically masks PII based on role
- Applied at column level
- Transparent to applications

**Layer 3: Row Access Policies**
- Filters rows based on role
- Applied at table level
- Enforces data segregation

**Layer 4: Object Tagging**
- Metadata for governance
- Classification and discovery
- Audit and compliance

---

## Step 1: Create Dynamic Data Masking Policies

### Purpose
Automatically mask PII columns for unauthorized roles.

### Implementation

Run as ACCOUNTADMIN in Snowflake:

```sql
use role accountadmin;
use database analytics;
use warehouse transforming_wh;

-- Email Masking Policy
-- Masks all but first character and domain
-- Example: john.doe@gmail.com -> j***@gmail.com

create or replace masking policy email_pii_mask as (val string) returns string ->
    case
        when current_role() in ('ACCOUNTADMIN', 'TRANSFORMER_ROLE', 'DATA_ENGINEER_ROLE') 
            then val  -- Full access for authorized roles
        else 
            case 
                when val is null then null
                when val not like '%@%' then '***@invalid.com'
                else 
                    substring(val, 1, 1) || '***@' || split_part(val, '@', 2)
            end
    end
    comment = 'Masks email addresses for unauthorized roles';

-- Phone Masking Policy
-- Keeps country code and last 4 digits
-- Example: +63-917-123-4567 -> +63-***-***-4567

create or replace masking policy phone_pii_mask as (val string) returns string ->
    case
        when current_role() in ('ACCOUNTADMIN', 'TRANSFORMER_ROLE', 'DATA_ENGINEER_ROLE') 
            then val  -- Full access
        else 
            case
                when val is null then null
                when val not like '+%' then '***-***-****'
                else 
                    substring(val, 1, 4) || '-***-***-' || 
                    substring(val, length(val) - 3, 4)
            end
    end
    comment = 'Masks phone numbers keeping country code and last 4 digits';

-- Name Masking Policy
-- Shows first initial and last name
-- Example: John Michael Smith -> J*** Smith

create or replace masking policy name_pii_mask as (val string) returns string ->
    case
        when current_role() in ('ACCOUNTADMIN', 'TRANSFORMER_ROLE', 'DATA_ENGINEER_ROLE') 
            then val  -- Full access
        else 
            case
                when val is null then null
                else substring(val, 1, 1) || '*** ' || split_part(val, ' ', -1)
            end
    end
    comment = 'Masks customer names showing first initial and last name';
```

### Masking Policy Design

**Email Masking:**
- Preserves domain (helps identify issues)
- First character visible (basic verification)
- Invalid emails marked clearly

**Phone Masking:**
- Country code visible (region identification)
- Last 4 digits visible (verification)
- Format preserved

**Name Masking:**
- First initial + last name
- Maintains some usability
- Less restrictive than full mask

### Role-Based Masking

**Full Access:**
- ACCOUNTADMIN (infrastructure)
- TRANSFORMER_ROLE (dbt execution)
- DATA_ENGINEER_ROLE (development)

**Masked Access:**
- ANALYST_ROLE (reporting only)
- External users
- BI tools

---

## Step 2: Apply Masking Policies to Staging Layer

### Purpose
Protect PII in staging layer views.

### Implementation
```sql
-- Apply to staging layer
use schema staging_staging;

-- stg_customers_with_updates
alter view if exists stg_customers_with_updates 
    modify column email set masking policy email_pii_mask;

alter view if exists stg_customers_with_updates 
    modify column phone set masking policy phone_pii_mask;

alter view if exists stg_customers_with_updates 
    modify column customer_name set masking policy name_pii_mask;
```

### Verification
```sql
-- Check policies applied
desc view stg_customers_with_updates;

-- Look for "policy name" column showing masking policy
```

---

## Step 3: Apply Masking Policies to Intermediate Layer

### Purpose
Protect PII in intermediate layer.

### Implementation
```sql
use schema staging_intermediate;

-- int_customers_deduplicated
alter view if exists int_customers_deduplicated 
    modify column email set masking policy email_pii_mask;

alter view if exists int_customers_deduplicated 
    modify column phone set masking policy phone_pii_mask;

alter view if exists int_customers_deduplicated 
    modify column customer_name set masking policy name_pii_mask;

-- int_customer_lifecycle_metrics
alter view if exists int_customer_lifecycle_metrics 
    modify column customer_name set masking policy name_pii_mask;
```

---

## Step 4: Apply Masking Policies to Snapshots

### Purpose
Protect PII in SCD Type 2 snapshot tables.

### Implementation
```sql
use schema snapshots;

-- snap_customer_status_v2
alter table if exists snap_customer_status_v2 
    modify column email set masking policy email_pii_mask;

alter table if exists snap_customer_status_v2 
    modify column phone set masking policy phone_pii_mask;

alter table if exists snap_customer_status_v2 
    modify column customer_name set masking policy name_pii_mask;
```

---

## Step 5: Create Row Access Policy

### Purpose
Filter customer segments based on role (analysts can't see enterprise customers).

### Business Rule
- ANALYST_ROLE: can see SMB and Consumer only
- TRANSFORMER_ROLE, DATA_ENGINEER_ROLE: can see all segments
- ACCOUNTADMIN: can see all segments

### Implementation
```sql
create or replace row access policy customer_segment_access as (segment_value string) returns boolean ->
    case
        -- Full access for admin and engineering roles
        when current_role() in ('ACCOUNTADMIN', 'TRANSFORMER_ROLE', 'DATA_ENGINEER_ROLE') 
            then true
        
        -- Analysts can only see SMB and Consumer (not Enterprise)
        when current_role() = 'ANALYST_ROLE' 
            and segment_value in ('smb', 'consumer') 
            then true
        
        -- Deny all others
        else false
    end
    comment = 'Restricts enterprise customer data access to authorized roles only';
```

### Row Policy Logic

**Returns TRUE:** Row is visible
**Returns FALSE:** Row is filtered out

**Policy Input:** segment_value (column being filtered)
**Policy Output:** boolean (visible or not)

---

## Step 6: Apply Row Access Policies

### Purpose
Apply segment filtering to intermediate and snapshot layers.

### Implementation
```sql
-- Apply to intermediate layer
use schema staging_intermediate;

alter view if exists int_customers_deduplicated 
    add row access policy customer_segment_access on (segment);

alter view if exists int_customer_lifecycle_metrics 
    add row access policy customer_segment_access on (segment);

-- Apply to snapshots
use schema snapshots;

alter table if exists snap_customer_status_v2 
    add row access policy customer_segment_access on (segment);
```

### Important Notes
- Row policies applied at table/view level
- Filter based on column value (segment)
- Cannot be bypassed by query logic
- Transparent to users

---

## Step 7: Test Security as Analyst Role

### Purpose
Verify masking and row filtering work correctly.

### Implementation
```sql
-- Switch to analyst role
use role analyst_role;
use warehouse reporting_wh;
use database analytics;
use schema staging_intermediate;

-- Test 1: Check PII masking
select 
    customer_id,
    customer_name,  -- Should be masked: 'J*** Smith'
    email,          -- Should be masked: 'j***@gmail.com'
    phone,          -- Should be masked: '+63-***-***-4567'
    segment,
    status
from int_customers_deduplicated
limit 5;

-- Test 2: Verify row access policy (should only see smb and consumer)
select 
    segment,
    count(*) as customer_count
from int_customers_deduplicated
group by segment
order by segment;

-- Expected: Only 'consumer' and 'smb' (no 'enterprise')
```

### Expected Results (Analyst)
```
customer_name: M*** Santos
email: m***@gmail.com
phone: +63-***-***-4567
segments visible: consumer, smb only
```

---

## Step 8: Test Security as Transformer Role

### Purpose
Verify authorized roles see unmasked data.

### Implementation
```sql
-- Switch to transformer role
use role transformer_role;
use warehouse transforming_wh;
use database analytics;
use schema staging_intermediate;

-- Test 1: Check unmasked PII
select 
    customer_id,
    customer_name,  -- Should be unmasked: 'Maria Santos'
    email,          -- Should be unmasked: 'maria.santos@gmail.com'
    phone,          -- Should be unmasked: '+63-917-123-4567'
    segment,
    status
from int_customers_deduplicated
limit 5;

-- Test 2: Verify all segments visible
select 
    segment,
    count(*) as customer_count
from int_customers_deduplicated
group by segment
order by segment;

-- Expected: 'consumer', 'enterprise', 'smb' (all segments)
```

### Expected Results (Transformer)
```
customer_name: Maria Santos (unmasked)
email: maria.santos@gmail.com (unmasked)
phone: +63-917-123-4567 (unmasked)
segments visible: consumer, enterprise, smb (all)
```

---

## Step 9: Create Object Tags for Governance

### Purpose
Tag columns with masking policies for governance and discovery.

### Implementation
```sql
use role accountadmin;
use schema staging_intermediate;

-- Create tags
create tag if not exists has_masking_policy
    comment = 'Indicates column has dynamic data masking applied';

create tag if not exists has_row_policy  
    comment = 'Indicates table has row access policy applied';

-- Tag protected columns
alter view int_customers_deduplicated 
    modify column email set tag has_masking_policy = 'email_pii_mask';

alter view int_customers_deduplicated 
    modify column phone set tag has_masking_policy = 'phone_pii_mask';

alter view int_customers_deduplicated 
    modify column customer_name set tag has_masking_policy = 'name_pii_mask';
```

### Tag Uses
- Governance reports
- Data cataloging
- Compliance auditing
- PII discovery

---

## Step 10: Verify Security Implementation

### Check Masking Policies Exist
```sql
use role accountadmin;
use database analytics;

-- List all masking policies
show masking policies in database analytics;

-- Expected: email_pii_mask, phone_pii_mask, name_pii_mask
```

### Check Row Access Policies
```sql
-- List all row access policies
show row access policies in database analytics;

-- Expected: customer_segment_access
```

### Check Policy Assignments
```sql
use schema staging_intermediate;

-- View all policies on a table
desc view int_customers_deduplicated;

-- Look for:
-- - "policy name" column (masking policies)
-- - "policy" metadata (row policies)
```

### Query Policy References
```sql
-- Find all objects using a masking policy
select 
    table_schema,
    table_name,
    column_name,
    policy_name
from analytics.information_schema.policy_references
where policy_name = 'EMAIL_PII_MASK';
```

---

## Troubleshooting

### Issue: Policy Not Applied
**Problem:** Masking policy created but not working

**Solution:**
```sql
-- Verify policy exists
show masking policies in database analytics;

-- Check policy applied to column
desc view int_customers_deduplicated;

-- Reapply if needed
alter view int_customers_deduplicated 
    modify column email set masking policy email_pii_mask;
```

### Issue: Still See Unmasked Data as Analyst
**Problem:** ANALYST_ROLE sees full PII

**Solution:**
```sql
-- Check current role
select current_role();

-- Verify role not in policy exception list
show masking policy email_pii_mask;

-- Test with explicit role switch
use role analyst_role;
select email from int_customers_deduplicated limit 1;
```

### Issue: Row Policy Blocks All Data
**Problem:** No rows visible for any role

**Solution:**
```sql
-- Check policy logic
show row access policy customer_segment_access;

-- Verify segment column has valid values
select distinct segment from int_customers_deduplicated;

-- Test policy logic manually
select 
    segment,
    case
        when current_role() in ('ACCOUNTADMIN', 'TRANSFORMER_ROLE') then true
        when current_role() = 'ANALYST_ROLE' and segment in ('smb', 'consumer') then true
        else false
    end as would_be_visible
from int_customers_deduplicated
limit 10;
```

### Issue: Standard Edition Limitations
**Problem:** Masking policies not supported

**Solution:**
Standard Edition doesn't support masking policies. Options:
1. Upgrade to Enterprise Edition (recommended)
2. Create separate masked views:
   ```sql
   create or replace view int_customers_analyst_view as
   select 
       customer_id,
       substring(customer_name, 1, 1) || '***' as customer_name,
       segment,
       status
   from int_customers_deduplicated;
   
   grant select on view int_customers_analyst_view to role analyst_role;
   ```

### Issue: Can't Remove Policy
**Problem:** Need to change or remove masking policy

**Solution:**
```sql
-- Unset masking policy from column
alter view int_customers_deduplicated 
    modify column email unset masking policy;

-- Then drop or recreate policy
drop masking policy if exists email_pii_mask;

-- Recreate with new logic
create or replace masking policy email_pii_mask as ...
```

---

## Best Practices Summary

### Masking Policy Design
1. White-list authorized roles (explicit is better)
2. Handle NULL values explicitly
3. Preserve format when possible (helps debugging)
4. Balance security with usability

### Row Access Policy Design
1. Keep logic simple (complex = performance issues)
2. Test with all roles
3. Document business rules clearly
4. Consider audit logging needs

### Policy Application
1. Apply to all layers (staging, intermediate, marts, snapshots)
2. Apply early (staging layer)
3. Test immediately after applying
4. Document all policy assignments

### Governance
1. Tag all PII columns
2. Create data classification scheme
3. Regular access reviews
4. Monitor policy usage

---

## Security Comparison Matrix

| Feature | dbt Macros | Snowflake Masking | Row Policies |
|---------|------------|-------------------|--------------|
| Complexity | Simple | Medium | Medium |
| Performance | Good | Excellent | Good |
| Centralized | No | Yes | Yes |
| Auditable | Limited | Yes | Yes |
| Enforceable | No | Yes | Yes |
| Cross-tool | Limited | Yes | Yes |
| Standard Ed | Yes | No | No |
| Enterprise Ed | Yes | Yes | Yes |

### Recommendation
- **Learning/Dev:** dbt macros (Part 9 alternative approach)
- **Production:** Snowflake native (current approach)
- **Enterprise:** Snowflake + row policies + audit logging

---

## Alternative: dbt-Based Masking (Standard Edition)

If Snowflake Enterprise not available, use dbt macros:

### Create Macro
```sql
-- macros/mask_pii.sql
{% macro mask_pii(column_name, mask_type='email') %}
    case
        when '{{ env_var("DBT_ROLE", "analyst") }}' in ('transformer', 'admin')
            then {{ column_name }}
        {% if mask_type == 'email' %}
            when {{ column_name }} is null then null
            else substring({{ column_name }}, 1, 1) || '***@' || split_part({{ column_name }}, '@', 2)
        {% elif mask_type == 'phone' %}
            when {{ column_name }} is null then null
            else substring({{ column_name }}, 1, 4) || '-***-***-' || substring({{ column_name }}, length({{ column_name }}) - 3, 4)
        {% endif %}
    end
{% endmacro %}
```

### Use in Model
```sql
select
    customer_id,
    {{ mask_pii('email', 'email') }} as email,
    {{ mask_pii('phone', 'phone') }} as phone
from customers
```

**Limitations:**
- Not enforced (can be bypassed)
- Requires code changes
- Not centralized

---

## Compliance Considerations

### GDPR (General Data Protection Regulation)
- Right to be forgotten: Use hard delete tracking in snapshots
- Data minimization: Only store necessary PII
- Access controls: Row policies for data segregation

### HIPAA (Health Insurance Portability and Accountability Act)
- PHI protection: Masking policies on health data
- Audit trails: Tag and track all PHI access
- Access controls: Strict role-based permissions

### PCI DSS (Payment Card Industry Data Security Standard)
- Cardholder data masking
- Access logging and monitoring
- Encryption at rest and in transit (Snowflake default)

---

## Verification Checklist

Before proceeding to Part 10:
- [ ] 3 masking policies created (email, phone, name)
- [ ] 1 row access policy created (segment filtering)
- [ ] Policies applied to staging, intermediate, snapshots
- [ ] Tested as ANALYST_ROLE (sees masked data, limited segments)
- [ ] Tested as TRANSFORMER_ROLE (sees full data, all segments)
- [ ] Tags created and applied
- [ ] Security documented

---

## Next Steps

After completing security:
1. Build marts layer with post-hooks for grants (Part 10)
2. Generate dbt documentation
3. Create security documentation for auditors

---

## Estimated Time
- Policy creation: 20-25 minutes
- Policy application: 15-20 minutes
- Testing: 10 minutes
- Troubleshooting: 10-15 minutes (if needed)
- Total: 45-60 minutes

## Success Criteria
- All PII columns masked for analysts
- Row access policies filtering correctly
- Authorized roles see full data
- Security transparent to applications
- Ready for marts layer in Part 10
