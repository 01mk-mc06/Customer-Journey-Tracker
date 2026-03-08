# Customer Journey Tracker - Documentation Index

## Project Overview
Build a complete customer journey analytics pipeline using dbt Core and Snowflake, implementing staging layers, SCD Type 2 snapshots, sessionization logic, data governance, and analytics-ready marts.

## Technologies
- dbt Core 1.8.0
- Snowflake (Enterprise Edition recommended for full security features)
- Python 3.8+ (pandas, faker)
- SQL

## Project Specifications
- Duration: 10 parts (8-12 hours total)
- Data volume: 7,315 rows across 5 seed tables
- Models: 14 (5 staging, 3 intermediate, 6 marts)
- Snapshots: 2 (SCD Type 2)
- Tests: 70+ data quality tests
- Security: Dynamic data masking, row-level security

---

## Documentation Parts

### Part 1: Infrastructure Setup
**File:** docs_part01_infrastructure.md
**Duration:** 30-45 minutes

**Topics Covered:**
- Snowflake warehouse creation (loading, transforming, reporting)
- Database setup (raw, analytics, audit)
- Role-based access control (4 roles)
- Schema organization
- Audit table creation
- Resource monitors

**Key Deliverables:**
- 3 warehouses configured
- 3 databases with schemas
- 4 roles with proper permissions
- Audit infrastructure ready

**Troubleshooting Included:**
- Permission denied errors
- Database-level privilege issues
- Future grant problems
- Resource monitor configuration

---

### Part 2: dbt Project Configuration
**File:** docs_part02_configuration.md
**Duration:** 45-60 minutes

**Topics Covered:**
- Virtual environment setup
- dbt-snowflake installation
- Project initialization
- profiles.yml configuration
- dbt_project.yml settings
- Package installation (dbt_utils, dbt_expectations, codegen)

**Key Deliverables:**
- dbt project initialized
- Snowflake connection configured
- 3 packages installed
- Project structure established

**Troubleshooting Included:**
- Connection failures
- Schema naming (prefix issues)
- Package version conflicts
- Warehouse auto-suspend

---

### Part 3: Python Data Generation
**File:** docs_part03_data_generation.md
**Duration:** 45-60 minutes

**Topics Covered:**
- Customer data generation (1,010 rows)
- Status updates for SCD Type 2 (432 rows)
- Interaction types reference (7 rows)
- Customer interactions (5,775 rows)
- Date dimension (91 days)
- Intentional data quality issues

**Key Deliverables:**
- 5 CSV files in seeds/ directory
- Realistic fake data with faker
- Intentional duplicates, nulls, invalid formats
- Session patterns for testing

**Troubleshooting Included:**
- Module import errors
- Date range issues
- Customer ID mismatches
- File path problems

---

### Part 4: Seed Loading and Sources
**File:** docs_part04_seed_loading.md
**Duration:** 40-55 minutes

**Topics Covered:**
- dbt seed execution
- Sources.yml configuration
- Source-level testing
- Data validation in Snowflake
- Source freshness checks

**Key Deliverables:**
- 5 seed tables loaded to raw.staging_seeds
- sources.yml with full documentation
- Source tests configured
- Lineage tracking enabled

**Troubleshooting Included:**
- Schema not found
- Permission denied
- CSV encoding errors
- Source function resolution

---

### Part 5: Staging Layer
**File:** docs_part05_staging_layer.md (to be created)
**Duration:** 60-90 minutes

**Topics Covered:**
- Staging model creation (5 models)
- Data cleaning and standardization
- Column renaming and type casting
- Referencing sources with source() function
- Basic transformations

**Key Deliverables:**
- stg_customers_with_updates.sql
- stg_customer_status_updates.sql
- stg_interaction_types.sql
- stg_interactions.sql
- stg_dates.sql

**Patterns Covered:**
- Lowercase/trim standardization
- Date parsing
- Boolean conversions
- LEFT JOIN to apply status updates

---

### Part 6: Data Quality Testing
**File:** docs_part06_testing.md (to be created)
**Duration:** 45-60 minutes

**Topics Covered:**
- schema.yml creation for staging models
- Generic tests (unique, not_null, accepted_values)
- dbt_expectations tests (regex, ranges)
- Custom singular tests
- Custom generic tests
- Test severity (warn vs error)

**Key Deliverables:**
- 60+ tests on staging layer
- Custom test: test_no_future_interactions.sql
- Custom generic: test_valid_phone_format.sql
- Expected failures documented

**Patterns Covered:**
- Testing intentional duplicates (severity: error)
- Regex validation for phone numbers
- Relationship tests between tables
- Date range validations

---

### Part 7: Snapshots (SCD Type 2)
**File:** docs_part07_snapshots.md (to be created)
**Duration:** 30-45 minutes

**Topics Covered:**
- Snapshot strategy (timestamp vs check)
- SCD Type 2 implementation
- dbt snapshot execution
- Point-in-time queries
- Customer journey tracking

**Key Deliverables:**
- snap_customer_status.sql (basic)
- snap_customer_status_v2.sql (with status updates applied)
- Historical tracking with dbt_valid_from/to

**Patterns Covered:**
- timestamp strategy
- updated_at column tracking
- Querying historical states
- Days in status calculations

---

### Part 8: Intermediate Layer
**File:** docs_part08_intermediate_layer.md (to be created)
**Duration:** 90-120 minutes

**Topics Covered:**
- Deduplication logic
- Sessionization with 30-minute timeout
- Customer lifecycle metrics
- Window functions
- Custom macros
- Materialization decisions (view vs table)

**Key Deliverables:**
- int_customers_deduplicated.sql
- int_customer_interactions_sessionized.sql
- int_customer_lifecycle_metrics.sql
- calculate_session_gap.sql macro
- test_sessions.sql (debugging model)

**Patterns Covered:**
- ROW_NUMBER() for deduplication
- LAG() for time gaps
- SUM() OVER for session numbering
- Surrogate key generation
- CTE organization (4-step sessionization)

---

### Part 9: Data Governance and Security
**File:** docs_part09_security.md (to be created)
**Duration:** 45-60 minutes

**Topics Covered:**
- Snowflake dynamic data masking policies
- Row access policies (segment filtering)
- Column-level security
- Object tagging
- Security testing (analyst vs transformer roles)

**Key Deliverables:**
- email_pii_mask policy
- phone_pii_mask policy
- name_pii_mask policy
- customer_segment_access row policy
- Policy assignments on all PII columns

**Patterns Covered:**
- Role-based masking (CASE WHEN current_role())
- Email masking (first char + domain)
- Phone masking (country code + last 4)
- Row filtering by segment

---

### Part 10: Marts Layer
**File:** docs_part10_marts_layer.md (to be created)
**Duration:** 90-120 minutes

**Topics Covered:**
- Star schema design
- Fact table creation
- Dimension table creation
- Type 1 vs Type 2 SCD dimensions
- Surrogate key strategy
- Post-hooks for grants
- dbt docs generation

**Key Deliverables:**
- dim_customers_current.sql (Type 1)
- dim_customers_historical.sql (Type 2)
- dim_dates.sql
- dim_interaction_types.sql
- fct_customer_interactions.sql
- fct_customer_lifetime_value.sql

**Patterns Covered:**
- Surrogate key generation
- Foreign key relationships
- Aggregated fact tables
- Customer health scoring
- Lifecycle duration calculations
- Resolution rate metrics

---

## Project Architecture

### Data Flow
```
CSV Seeds (Python Generated)
    |
    v
dbt seed --> RAW.staging_seeds
    |
    v
Staging Layer --> ANALYTICS.staging_staging (views)
    |
    +---> Snapshots --> ANALYTICS.snapshots (tables, SCD Type 2)
    |
    v
Intermediate Layer --> ANALYTICS.staging_intermediate (views + 1 table)
    |
    v
Marts Layer --> ANALYTICS.staging_marts (tables)
```

### Security Layers
```
ACCOUNTADMIN (setup only)
    |
    +-- DATA_ENGINEER_ROLE (full dev access)
    |       |
    |       +-- TRANSFORMER_ROLE (dbt execution, unmasked data)
    |               |
    |               +-- LOADER_ROLE (load seeds only)
    |
    +-- ANALYST_ROLE (read marts, masked PII, filtered segments)
```

### Model Lineage
```
seed_customers
    |
    +---> stg_customers_with_updates
            |
            +---> snap_customer_status_v2
            |       |
            |       +---> dim_customers_historical
            |
            +---> int_customers_deduplicated
                    |
                    +---> dim_customers_current
                    |
                    +---> fct_customer_lifetime_value

seed_interactions
    |
    +---> stg_interactions
            |
            +---> int_customer_interactions_sessionized
                    |
                    +---> fct_customer_interactions
```

---

## Key Metrics and Counts

### Data Volume
- Total seed rows: 7,315
- Unique customers: 1,000
- Customer duplicates: 10
- Status updates: 432
- Interactions: 5,775
- Date dimension: 91 days

### Models Built
- Staging: 5 views
- Snapshots: 2 tables
- Intermediate: 2 views, 1 table
- Marts: 6 tables
- Total: 16 models

### Tests Implemented
- Source tests: 8
- Staging tests: 45+
- Intermediate tests: 10+
- Marts tests: 40+
- Total: 100+ tests

### Security Policies
- Masking policies: 3 (email, phone, name)
- Row access policies: 1 (segment filter)
- Roles: 4 (loader, transformer, analyst, data_engineer)

---

## Learning Outcomes

### Technical Skills
- dbt project structure and configuration
- Snowflake RBAC and security
- Python data generation with faker
- SQL window functions and CTEs
- SCD Type 2 implementation
- Sessionization logic
- Data quality testing frameworks
- Dynamic data masking
- Star schema modeling

### Best Practices
- Least-privilege access control
- Intentional data quality issues for testing
- Severity management (warn vs error)
- Documentation standards
- Naming conventions
- Materialization strategies
- Schema organization
- Lineage tracking

### Tools and Packages
- dbt_utils (surrogate keys, date functions)
- dbt_expectations (advanced testing)
- codegen (YAML generation)
- Snowflake security features
- Git version control patterns

---

## Project Completion Criteria

- [ ] All 10 parts completed
- [ ] 16 models built and tested
- [ ] 100+ tests passing (with documented expected failures)
- [ ] Security policies applied and tested
- [ ] dbt docs generated and served
- [ ] Verification queries run successfully
- [ ] All troubleshooting documented
- [ ] Project ready for portfolio

---

## Time Estimates

### By Part
- Part 1: 30-45 min
- Part 2: 45-60 min
- Part 3: 45-60 min
- Part 4: 40-55 min
- Part 5: 60-90 min
- Part 6: 45-60 min
- Part 7: 30-45 min
- Part 8: 90-120 min
- Part 9: 45-60 min
- Part 10: 90-120 min

### Total Project Time
- Minimum: 8 hours 30 minutes
- Average: 10 hours 30 minutes
- Maximum: 13 hours 15 minutes

### Recommended Schedule
- Day 1: Parts 1-4 (infrastructure and data)
- Day 2: Parts 5-7 (staging and snapshots)
- Day 3: Parts 8-10 (intermediate, security, marts)

---

## Prerequisites for Starting

### Software
- Python 3.8+ installed
- Snowflake account (trial or paid)
- Code editor (VS Code recommended)
- Git (optional but recommended)

### Knowledge
- Basic SQL (SELECT, JOIN, GROUP BY)
- Basic Python (lists, loops, functions)
- Command line basics
- YAML syntax basics

### Snowflake Access
- Account with ACCOUNTADMIN privileges
- Ability to create warehouses and databases
- Standard, Enterprise, or Business Critical edition

---

## Support and Resources

### Official Documentation
- dbt Docs: docs.getdbt.com
- Snowflake Docs: docs.snowflake.com
- dbt_utils: github.com/dbt-labs/dbt-utils
- dbt_expectations: github.com/calogica/dbt-expectations

### Community
- dbt Slack: getdbt.slack.com
- dbt Discourse: discourse.getdbt.com
- Stack Overflow: tag:dbt

### Project Files
- All documentation: docs_partXX_*.md
- SQL reference: mp3_verification_queries.sql
- Python scripts: generate_seed_*.py
- dbt models: models/staging/, models/intermediate/, models/marts/

---

## Portfolio Presentation Tips

### Highlight These Achievements
1. SCD Type 2 implementation with snapshots
2. Custom sessionization logic (30-min timeout rule)
3. Enterprise security (masking + row policies)
4. Comprehensive testing strategy (100+ tests)
5. Intentional data quality issues for learning
6. Star schema with Type 1 and Type 2 dimensions

### Talking Points for Interviews
1. Design decisions (view vs table materialization)
2. Trade-offs (dbt masking vs Snowflake native)
3. Troubleshooting examples (schema naming, FROM clause bug)
4. Business context (BPO customer journey tracking)
5. Scale considerations (how to handle 10M+ rows)
6. CI/CD readiness (what would be needed for production)

### GitHub Repository Structure
```
bpo_customer_journey/
├── README.md (project overview)
├── docs/ (all documentation files)
├── models/ (dbt models)
├── seeds/ (CSV files)
├── macros/ (custom macros)
├── tests/ (custom tests)
├── snapshots/ (SCD Type 2 logic)
├── scripts/ (Python generation scripts)
└── sql/ (verification queries)
```

---

## Next Steps After Completion

### Deep Dive Topics
1. SCD Type 2 advanced patterns
2. Incremental models and strategies
3. Performance optimization techniques
4. CI/CD with GitHub Actions
5. dbt Cloud vs dbt Core comparison

### Additional Projects
- MP1: Agent Performance Dashboard Pipeline
- MP2: Multi-Site Contact Center Consolidation
- Capstone projects (5 enterprise scenarios)

### Certifications
- dbt Analytics Engineering Certification
- Snowflake SnowPro Core
- Snowflake SnowPro Advanced: Data Engineer

---

## Glossary

**Artifact:** dbt compiled output (target/ directory)
**Ephemeral:** Materialized as CTE, not persisted
**Grain:** Level of detail in a fact table
**Marts:** Analytics-ready tables for BI tools
**Materialization:** How dbt persists models (view, table, incremental)
**SCD:** Slowly Changing Dimension (Type 0, 1, 2, 3)
**Seed:** CSV file loaded via dbt seed command
**Sessionization:** Grouping events into sessions based on time gaps
**Source:** External table referenced via source() function
**Surrogate Key:** Generated unique identifier

---

This documentation set provides complete guidance for building MP3: Customer Journey Tracker from infrastructure setup through final marts deployment.
