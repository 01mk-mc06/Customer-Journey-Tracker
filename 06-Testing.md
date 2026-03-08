# Data Quality Testing

## Overview
Implement comprehensive data quality testing framework using dbt tests, including generic tests, dbt_expectations tests, and custom tests.

## Objectives
- Create schema.yml for staging models
- Implement generic tests (unique, not_null, accepted_values, relationships)
- Use dbt_expectations for advanced validations
- Create custom singular test
- Create custom generic test
- Manage test severity (warn vs error)
- Document expected failures

## Prerequisites
- Part 5 completed (staging layer built)
- dbt_utils and dbt_expectations packages installed
- Understanding of data quality concepts

---

## Testing Strategy

### Test Types in dbt

**Generic Tests:**
- Reusable across models
- Built-in: unique, not_null, accepted_values, relationships
- Custom: defined in tests/generic/

**Singular Tests:**
- One-off SQL queries
- Return failing rows
- Stored in tests/ directory

**Test Severity:**
- error: Test failure stops build (default)
- warn: Test failure logged but doesn't stop build

### Testing Philosophy
- Test early and often
- Accept intentional failures (document them)
- Use severity appropriately
- Balance coverage vs maintenance

---

## Step 1: Create schema.yml for Staging Models

### Purpose
Define tests and documentation for all staging models.

### File Location
models/staging/schema.yml

### Implementation - Part 1: Customers

```yaml
version: 2

models:
  - name: stg_customers_with_updates
    description: "Cleaned customer data with latest status updates applied"
    columns:
      - name: customer_id
        description: "Customer unique identifier (10 duplicates intentional for testing)"
        tests:
          - not_null
          - unique:
              config:
                severity: error
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^CUST[0-9]{6}$"
      
      - name: customer_name
        description: "Customer full name (lowercase, trimmed)"
        tests:
          - not_null
      
      - name: email
        description: "Customer email (some nulls intentional, ~2%)"
        tests:
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-z0-9._%+-]+@[a-z0-9.-]+\\.[a-z]{2,}$"
              config:
                severity: warn
      
      - name: phone
        description: "Customer phone in format +63-XXX-XXX-XXXX"
        tests:
          - test_valid_phone_format:
              config:
                severity: warn
      
      - name: segment
        description: "Customer segment"
        tests:
          - not_null
          - accepted_values:
              values: ['enterprise', 'smb', 'consumer']
      
      - name: status
        description: "Current customer status"
        tests:
          - not_null
          - accepted_values:
              values: ['lead', 'trial', 'paid', 'at_risk', 'churned']
      
      - name: created_at
        description: "Customer creation timestamp"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2024-01-01'"
              max_value: "'2024-12-31'"
      
      - name: updated_at
        description: "Last update timestamp"
        tests:
          - not_null

  - name: stg_customer_status_updates
    description: "Customer status change events"
    columns:
      - name: customer_id
        description: "Foreign key to customers"
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers_with_updates')
              field: customer_id
      
      - name: status
        description: "New status value"
        tests:
          - not_null
          - accepted_values:
              values: ['lead', 'trial', 'paid', 'at_risk', 'churned']
      
      - name: updated_at
        description: "Status change timestamp"
        tests:
          - not_null
      
      - name: updated_by
        description: "User who made the change"
        tests:
          - not_null
```

### Implementation - Part 2: Dates and Interaction Types

```yaml
  - name: stg_dates
    description: "Date dimension for Q1 2024"
    columns:
      - name: date_key
        description: "Primary key in YYYYMMDD format"
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
        tests:
          - accepted_values:
              values: ['monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday']
      
      - name: month
        description: "Month number (1-3 for Q1)"
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 1
              max_value: 12
      
      - name: quarter
        description: "Quarter number (1 for Q1)"
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 1
              max_value: 4
      
      - name: year
        description: "Year (2024)"
        tests:
          - accepted_values:
              values: [2024]
      
      - name: is_weekend
        description: "Weekend flag"
        tests:
          - not_null
      
      - name: is_holiday
        description: "Philippine holiday flag"
        tests:
          - not_null

  - name: stg_interaction_types
    description: "Reference data for interaction types"
    columns:
      - name: interaction_type_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      
      - name: interaction_type
        description: "Interaction type name"
        tests:
          - unique
          - not_null
      
      - name: type_category
        description: "Category (voice or digital)"
        tests:
          - accepted_values:
              values: ['voice', 'digital']
      
      - name: is_billable
        description: "Billability flag"
        tests:
          - not_null
      
      - name: avg_duration_seconds
        description: "Average duration in seconds"
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 3600
```

### Implementation - Part 3: Interactions

```yaml
  - name: stg_interactions
    description: "Customer interaction events"
    columns:
      - name: interaction_id
        description: "Primary key (UUID)"
        tests:
          - unique
          - not_null
      
      - name: customer_id
        description: "Foreign key to customers"
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers_with_updates')
              field: customer_id
      
      - name: interaction_type
        description: "Type of interaction"
        tests:
          - not_null
          - relationships:
              to: ref('stg_interaction_types')
              field: interaction_type
      
      - name: interaction_timestamp
        description: "When interaction occurred"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2024-01-01'"
              max_value: "'2024-03-31'"
      
      - name: duration_seconds
        description: "Interaction duration"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 7200
      
      - name: channel
        description: "Channel used"
        tests:
          - not_null
          - accepted_values:
              values: ['phone', 'web_chat', 'email', 'mobile_app']
      
      - name: sentiment_score
        description: "Sentiment score from -1.0 to 1.0"
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: -1.0
              max_value: 1.0
      
      - name: resolution_status
        description: "Resolution status"
        tests:
          - not_null
          - accepted_values:
              values: ['resolved', 'pending', 'escalated', 'closed']
```

---

## Step 2: Create Custom Singular Test

### Purpose
Test that no interactions have future timestamps (beyond Q1 2024).

### File Location
tests/test_no_future_interactions.sql

### Implementation
```sql
/*
    Custom singular test: No future interactions
    
    Purpose: Ensure all interactions are within Q1 2024
    Expected: 0 rows returned (all interactions valid)
*/

select
    interaction_id,
    interaction_timestamp,
    current_date() as today
from {{ ref('stg_interactions') }}
where interaction_timestamp > '2024-03-31'
```

### How Singular Tests Work
- Test passes if query returns 0 rows
- Any rows returned = test failure
- Use SELECT to identify failing records

---

## Step 3: Create Custom Generic Test

### Purpose
Create reusable test for Philippine phone number format validation.

### File Location
tests/generic/test_valid_phone_format.sql

### Implementation
```sql
{% test test_valid_phone_format(model, column_name) %}

/*
    Custom generic test: Valid Philippine phone format
    
    Format: +63-XXX-XXX-XXXX
    Example: +63-917-123-4567
    
    Usage in schema.yml:
    tests:
      - test_valid_phone_format:
          config:
            severity: warn
*/

select
    {{ column_name }} as invalid_phone
from {{ model }}
where {{ column_name }} is not null
  and {{ column_name }} not like '+63-9__-___-____'

{% endtest %}
```

### Alternative: Regex Version (More Strict)
```sql
{% test test_valid_phone_format(model, column_name) %}

select
    {{ column_name }} as invalid_phone
from {{ model }}
where {{ column_name }} is not null
  and not regexp_like(
      {{ column_name }}, 
      '^\\+63-9[0-9]{2}-[0-9]{3}-[0-9]{4}$'
  )

{% endtest %}
```

### Generic Test Structure
```sql
{% test test_name(model, column_name, optional_param=default) %}
    -- Test logic here
    -- Return rows that FAIL the test
{% endtest %}
```

---

## Step 4: Run All Tests

### Purpose
Execute test suite and review results.

### Implementation
```powershell
# Run all tests
dbt test

# Run tests for specific model
dbt test --select stg_customers_with_updates

# Run tests for staging layer
dbt test --select staging
```

### Expected Output
```
Running with dbt=1.8.0
Found 64 tests

Concurrency: 4 threads

1 of 64 START test unique_stg_customers_with_updates_customer_id ......... [RUN]
2 of 64 START test not_null_stg_customers_with_updates_customer_id ....... [RUN]
...
1 of 64 FAIL 10 unique_stg_customers_with_updates_customer_id ............ [FAIL 10 in 2.1s]
2 of 64 PASS not_null_stg_customers_with_updates_customer_id ............. [PASS in 1.9s]
...
60 of 64 PASS test_no_future_interactions ................................ [PASS in 1.8s]
61 of 64 WARN 30 test_valid_phone_format_stg_customers_phone ............. [WARN 30 in 2.0s]
...

Completed with 1 error, 2 warnings
```

### Interpreting Results

**PASS:** Test succeeded (0 failing rows)
**FAIL 10:** Test failed with 10 failing rows
**WARN 30:** Test failed but severity=warn (30 invalid phones)

---

## Step 5: Review Test Failures

### Expected Failures (By Design)

**1. Duplicate customer_ids (FAIL 10)**
- Intentional duplicates for testing
- Severity: error
- Will be handled in intermediate layer (deduplication)

**2. Invalid phone numbers (WARN ~30)**
- Intentional invalid formats
- Severity: warn
- Document as known data quality issue

**3. Null emails (WARN ~20)**
- Intentional nulls
- Severity: warn
- Business decision to allow nulls

### Query Failing Rows
```sql
-- See actual duplicate customer_ids
select 
    customer_id,
    count(*) as duplicate_count
from analytics.staging_staging.stg_customers_with_updates
group by customer_id
having count(*) > 1;

-- See invalid phone numbers
select customer_id, phone
from analytics.staging_staging.stg_customers_with_updates
where phone is not null
  and not regexp_like(phone, '^\\+63-9[0-9]{2}-[0-9]{3}-[0-9]{4}$')
limit 10;
```

---

## Step 6: Test Results Management

### Viewing Test Details
```sql
-- Check target/run/ directory for compiled test SQL
-- Example: target/run/bpo_customer_journey/models/staging/schema.yml/

-- View test in Snowflake query history
use role transformer_role;
show queries in account;
-- Filter by query_tag: 'dbt_customer_journey'
```

### Ignoring Specific Tests
```yaml
# In dbt_project.yml
tests:
  bpo_customer_journey:
    +severity: warn  # All tests warn by default
    
    staging:
      unique_stg_customers_with_updates_customer_id:
        +enabled: false  # Disable specific test
```

### Running Specific Test Types
```powershell
# Run only generic tests
dbt test --select test_type:generic

# Run only singular tests
dbt test --select test_type:singular

# Run only relationship tests
dbt test --select test_name:relationships
```

---

## Troubleshooting

### Issue: Test Fails Unexpectedly
**Problem:** Test expected to pass but fails

**Solution:**
1. Run the test SQL directly in Snowflake
2. Check compiled SQL in target/run/
3. Verify data meets test criteria
4. Check for whitespace or case sensitivity issues

### Issue: Regex Test Doesn't Work
**Problem:** regexp_like() function not found

**Solution:**
- Snowflake function: regexp_like() (Snowflake specific)
- PostgreSQL function: ~ operator
- Use dbt_utils.safe_cast if cross-database
- Check Snowflake documentation for syntax

### Issue: dbt_expectations Test Fails
**Problem:** expect_column_values_to_be_between fails

**Solution:**
- Check min/max values include quotes for strings
- Verify date format: "'2024-01-01'" not "2024-01-01"
- Ensure package installed: dbt deps
- Check package version compatibility

### Issue: Relationship Test Fails
**Problem:** Foreign key relationship test fails

**Solution:**
```sql
-- Find orphaned records
select i.customer_id
from analytics.staging_staging.stg_interactions i
left join analytics.staging_staging.stg_customers_with_updates c
    on i.customer_id = c.customer_id
where c.customer_id is null;
```

### Issue: Too Many Test Failures
**Problem:** Large number of unexpected failures

**Solution:**
1. Run dbt run first to ensure models are current
2. Check data quality in source tables
3. Review transformation logic in staging models
4. Verify test configuration (correct min/max values)

---

## Best Practices Summary

### Test Coverage
1. Test all primary keys (unique, not_null)
2. Test all foreign keys (relationships)
3. Test categorical fields (accepted_values)
4. Test numeric ranges (expect_column_values_to_be_between)
5. Test date ranges and formats

### Test Organization
1. Keep schema.yml close to models (models/staging/schema.yml)
2. Group related tests together
3. Document expected failures
4. Use consistent naming for custom tests

### Severity Management
1. Use error for critical data quality issues
2. Use warn for known/acceptable issues
3. Document all warnings in schema.yml descriptions
4. Review warnings regularly (may become errors later)

### Custom Tests
1. Create generic tests for reusable validations
2. Use singular tests for one-off checks
3. Return failing rows (not boolean)
4. Document test purpose in comments

---

## Common Test Patterns

### Pattern 1: Comprehensive Column Testing
```yaml
- name: column_name
  description: "Column description"
  tests:
    - not_null
    - unique
    - accepted_values:
        values: ['value1', 'value2']
    - dbt_expectations.expect_column_values_to_match_regex:
        regex: "^pattern$"
```

### Pattern 2: Optional Foreign Key
```yaml
- name: optional_fk_id
  description: "Optional foreign key (nulls allowed)"
  tests:
    - relationships:
        to: ref('other_model')
        field: id
        config:
          where: "optional_fk_id is not null"
```

### Pattern 3: Conditional Tests
```yaml
- name: status
  description: "Status field"
  tests:
    - accepted_values:
        values: ['active', 'inactive']
        config:
          where: "record_type = 'customer'"
```

---

## Test Documentation

### In schema.yml
```yaml
- name: customer_id
  description: |
    Customer unique identifier
    
    Known Issues:
    - 10 duplicate records intentional for testing deduplication logic
    - Will be resolved in intermediate layer (int_customers_deduplicated)
    
    Tests:
    - unique (severity: error, expected 10 failures)
    - not_null (should pass)
```

### In README.md
```markdown
## Data Quality

### Expected Test Failures
1. stg_customers.customer_id - 10 duplicates (intentional)
2. stg_customers.phone - ~30 invalid formats (intentional)
3. stg_customers.email - ~20 nulls (intentional)

### Resolution
- Duplicates resolved in int_customers_deduplicated
- Invalid phones flagged but not corrected (business decision)
- Null emails acceptable per business rules
```

---

## Verification Checklist

Before proceeding to Part 7:
- [ ] schema.yml created for all 5 staging models
- [ ] Generic tests configured (unique, not_null, accepted_values, relationships)
- [ ] dbt_expectations tests added (regex, ranges)
- [ ] Custom singular test created (test_no_future_interactions)
- [ ] Custom generic test created (test_valid_phone_format)
- [ ] dbt test executed successfully
- [ ] Expected failures documented
- [ ] Test severity configured appropriately

---

## Next Steps

After completing testing:
1. Create snapshots for SCD Type 2 tracking (Part 7)
2. Build intermediate layer (Part 8)
3. Continue testing at each layer

---

## Estimated Time
- schema.yml creation: 25-35 minutes
- Custom tests: 10-15 minutes
- Running tests: 5 minutes
- Reviewing results: 5-10 minutes
- Troubleshooting: 10-15 minutes (if needed)
- Total: 45-60 minutes

## Success Criteria
- 60+ tests configured
- Tests run successfully with expected failures only
- All critical fields tested
- Custom tests working
- Documentation complete
- Ready for snapshot creation in Part 7
