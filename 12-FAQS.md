# Customer Journey Tracker - FAQ

## Architecture Decisions & Trade-offs

---

### Q: Why did we use Snowflake instead of other data warehouses?

**Answer:**
Snowflake provides enterprise-grade security features (dynamic data masking, row access policies) essential for this project's PII protection requirements. It offers automatic scaling, separation of compute and storage, and excellent integration with dbt. The project demonstrates production-level security patterns that translate directly to enterprise environments.

**Downsides:**
- High cost for small datasets (trial credits required)
- Vendor lock-in with Snowflake-specific SQL features (QUALIFY, regexp_like)
- Over-engineered for 7,315 rows (could run on SQLite)
- Requires internet connection and cloud infrastructure

**Alternatives:**
- **PostgreSQL:** Free, open-source, sufficient performance for this data volume, strong dbt support, but lacks native dynamic data masking and row-level security without custom views
- **DuckDB:** Embedded database, zero infrastructure, exceptional analytical performance, perfect for local development, but no enterprise security features
- **BigQuery:** Serverless, pay-per-query model, good dbt support, different security model (column-level access control vs masking), simpler cost structure
- **Databricks SQL:** Unified analytics platform, strong security, but higher complexity and cost

**When to Use Alternatives:**
- PostgreSQL: Cost-sensitive projects, on-premise requirements, no masking needed
- DuckDB: Local development, prototyping, embedded analytics
- BigQuery: Google Cloud ecosystem, serverless preference, variable workloads

---

### Q: Why dbt Core instead of dbt Cloud?

**Answer:**
Learning and cost optimization. dbt Core is free and provides complete control over the development environment, allowing deeper understanding of dbt mechanics, configuration management, and deployment processes. It demonstrates ability to manage version control, packages, and testing independently.

**Downsides:**
- No built-in scheduler (requires external orchestration like Airflow)
- No web-based IDE or collaboration features
- Local development only (no remote execution)
- Manual documentation hosting (dbt docs serve)
- No CI/CD integration out of the box
- Version management responsibility falls on developer

**Alternatives:**
- **dbt Cloud:** Built-in scheduler, web IDE, integrated CI/CD, automatic documentation hosting, metadata API, job monitoring, but costs $50+/developer/month
- **Hybrid approach:** dbt Core for development, dbt Cloud for production deployment
- **dbt Core + Airflow:** Free orchestration, custom scheduling, requires infrastructure management

**When to Use dbt Cloud:**
- Team collaboration required
- Production deployment with SLAs
- Need integrated CI/CD
- Business users need documentation access
- Budget allows $600+/year per developer

---

### Q: Why use seeds instead of loading from a real source system?

**Answer:**
This is a learning project demonstrating data engineering patterns. Seeds provide controlled, reproducible data with intentional quality issues for comprehensive testing. Real source systems would add complexity (API authentication, schema changes, data availability) without additional learning value for the core dbt and modeling concepts.

**Downsides:**
- Seeds not scalable (dbt recommends files under 1MB)
- Not representative of production data pipelines
- Full refresh only (no incremental loading capability)
- No real-time or near-real-time data
- Doesn't demonstrate source system integration patterns
- Static data doesn't show temporal dynamics

**Alternatives for Production:**
- **Snowflake Snowpipe:** Auto-ingestion from S3/Azure Blob Storage, near-real-time, event-driven, handles large volumes
- **Apache Airflow:** Orchestrated extraction from APIs/databases, custom Python operators, flexible scheduling
- **Fivetran/Stitch/Airbyte:** Managed ELT connectors, pre-built integrations, automatic schema detection, but adds cost
- **dbt source freshness:** Monitor external tables, alert on stale data, integrate with orchestration
- **Singer taps:** Open-source data extraction, community-maintained connectors

**Production Pattern:**
```
Source System (API/Database)
    ↓
Extract Layer (Fivetran/Airflow/Snowpipe)
    ↓
Raw Tables (Snowflake)
    ↓
dbt Transformation (staging → intermediate → marts)
```

---

### Q: Why create intentional data quality issues (duplicates, nulls, invalid formats)?

**Answer:**
Demonstrates testing capabilities and real-world data scenarios. Production data always has quality issues; this project shows ability to detect, document, and handle them appropriately using test severity (warn vs error), custom tests, and data quality frameworks.

**Intentional Issues:**
- 10 duplicate customer_ids (1%)
- ~30 invalid phone numbers (3%)
- ~20 null emails (2%)

**Downsides:**
- May confuse stakeholders if deployed without proper documentation
- Requires clear documentation to prevent false alarms
- Test failures can mask real issues if not managed properly
- Could be misinterpreted as poor data quality rather than intentional

**Alternatives:**
- **Perfectly clean data:** Simpler implementation, all tests pass, but doesn't demonstrate data quality handling skills
- **Real production data:** More realistic issues, but introduces PII/security concerns and unpredictable failures
- **Synthetic issues with flags:** Add is_test_record flag to separate intentional vs real issues
- **Graduated approach:** Start clean, introduce issues incrementally as teaching tool

**Best Practice:**
Document all intentional issues in:
- schema.yml descriptions
- README.md data quality section
- Test configuration comments
- Test severity justification

---

### Q: Why use views for staging/intermediate and tables for marts?

**Answer:**
Balances performance with maintainability. Views are lightweight, always fresh, and easy to change during development. Tables for marts provide fast query performance for end users and BI tools.

**Staging/Intermediate = Views:**
- Pros: No storage cost, always current, fast to rebuild, easy to iterate
- Cons: Query time computation, slower for complex transformations

**Marts = Tables:**
- Pros: Pre-computed, fast queries, indexed, optimized for BI tools
- Cons: Storage cost, requires refresh schedule, stale between runs

**Downsides of Current Approach:**
- Intermediate sessionization is a table (could be view for smaller datasets)
- No incremental models (full refresh every run)
- Views can be slow for complex window functions

**Alternatives:**
- **All tables:** Faster queries, higher storage cost, more maintenance
- **All views:** Lower cost, slower queries, always fresh
- **Incremental models:** Only process new/changed data, efficient for large datasets
- **Materialized views:** Best of both worlds (Snowflake Enterprise feature), auto-refresh on base table changes

**Production Recommendation:**
- Staging: Views (rarely queried directly)
- Intermediate: Mix (views for simple, tables for complex window functions)
- Marts: Tables or incremental (frequently queried by BI tools)

---

### Q: Why 30-minute timeout for sessionization instead of other values?

**Answer:**
Industry standard for web/application analytics. Balances session continuity with realistic user behavior. Too short creates fragmented sessions; too long merges unrelated activities.

**30-Minute Logic:**
- Matches Google Analytics default
- Typical attention span for task completion
- Reasonable break/interruption threshold

**Downsides:**
- Arbitrary threshold not based on actual user behavior analysis
- May not fit all business contexts (phone support vs web browsing)
- Doesn't account for interaction type differences

**Alternatives:**
- **15 minutes:** Tighter sessions, better for high-frequency interactions
- **60 minutes:** Looser sessions, better for research/shopping behavior
- **Adaptive timeout:** Different thresholds by channel (5 min for phone, 30 min for email)
- **Inactivity-based:** Calculate from actual interaction patterns (median gap + 1 standard deviation)
- **Business rule-based:** End session on specific events (purchase, logout, explicit close)

**How to Optimize:**
```sql
-- Analyze actual time gaps
select
    percentile_cont(0.50) within group (order by minutes_gap) as median_gap,
    percentile_cont(0.75) within group (order by minutes_gap) as p75_gap,
    percentile_cont(0.90) within group (order by minutes_gap) as p90_gap
from (
    select 
        datediff('minute', 
            lag(interaction_timestamp) over (partition by customer_id order by interaction_timestamp),
            interaction_timestamp
        ) as minutes_gap
    from stg_interactions
)
where minutes_gap is not null;
```

Use P75 or P90 as threshold for data-driven decision.

---

### Q: Why ROW_NUMBER() for deduplication instead of DISTINCT or GROUP BY?

**Answer:**
ROW_NUMBER() provides control over which duplicate to keep (most recent via ORDER BY updated_at DESC). DISTINCT and GROUP BY don't allow prioritization and require aggregating all columns.

**Downsides:**
- More verbose than DISTINCT
- Requires window function support
- Slightly more complex to understand

**Alternatives:**
- **DISTINCT:** Simpler syntax, but keeps arbitrary duplicate (non-deterministic)
- **GROUP BY with MAX():** Works but requires aggregating every column individually
- **QUALIFY (Snowflake-specific):** Cleaner syntax combining window function and filter
  ```sql
  select * from customers
  qualify row_number() over (partition by customer_id order by updated_at desc) = 1
  ```

**When to Use Alternatives:**
- DISTINCT: When you don't care which duplicate is kept, all duplicates identical
- GROUP BY: When you need aggregations anyway (SUM, COUNT, etc.)
- QUALIFY: When using Snowflake and want cleaner code

---

### Q: Why SCD Type 2 for customers instead of Type 1 or Type 3?

**Answer:**
Type 2 preserves complete history for customer journey analysis, churn prediction, and point-in-time reporting. This project demonstrates historical tracking capabilities essential for analytics.

**Type Comparison:**
- **Type 1:** Overwrites old values, no history, simple but loses insight
- **Type 2:** New row per change, complete history, complex queries
- **Type 3:** Previous + current columns, limited history (last change only)

**Downsides:**
- Higher storage cost (multiple rows per customer)
- More complex queries (need valid_from/valid_to logic)
- Larger tables impact query performance
- Requires careful handling of current vs historical records

**Alternatives:**
- **Type 1 only:** Use for truly static attributes (customer_id, created_at)
- **Type 3:** Good for tracking last status only (previous_status, current_status columns)
- **Hybrid:** Type 1 for most attributes, Type 2 only for critical fields (status)
- **Event sourcing:** Store all events, reconstruct state as needed (more flexible but complex)

**When to Use Alternatives:**
- Type 1: Reference data, attributes that rarely change meaningfully
- Type 3: Only care about last change, limited history sufficient
- No SCD: Completely static dimensions (date, geography)

---

### Q: Why generate surrogate keys with MD5 instead of auto-incrementing integers?

**Answer:**
MD5 hashing creates deterministic surrogate keys that remain consistent across rebuilds. Auto-increment would generate different keys each run, breaking foreign key relationships.

**MD5 Approach:**
```sql
md5(cast(coalesce(cast(customer_id as varchar), '_dbt_utils_surrogate_key_null_') as varchar))
```

**Downsides:**
- Longer keys (32 characters vs 8 bytes for integer)
- Slightly slower joins (string vs integer comparison)
- Hash collisions possible (extremely rare with MD5)
- Not human-readable

**Alternatives:**
- **Auto-increment:** Sequential integers, requires database sequence, breaks on rebuild
- **UUID:** Universally unique, but 36 characters, random (not sortable)
- **Natural keys:** Use business keys directly (customer_id), but exposes sensitive data
- **Composite keys:** Combine multiple columns, complex but no hashing needed
- **Hash functions:** SHA256 (more secure), XXHash (faster), CityHash (shorter)

**Production Recommendation:**
- Use MD5 for reproducibility in dbt (current approach)
- Consider integer surrogate keys if performance critical
- Document key generation logic for troubleshooting

---

### Q: Why separate staging and intermediate layers instead of going directly to marts?

**Answer:**
Follows dbt best practices for separation of concerns. Staging handles data cleaning, intermediate applies business logic, marts serve analytics. This modularity improves maintainability, testing, and reusability.

**Layer Responsibilities:**
- **Staging:** 1:1 with sources, clean/standardize only
- **Intermediate:** Business logic, joins, calculations, sessionization
- **Marts:** Analytics-ready, denormalized, aggregated

**Downsides:**
- More models to maintain
- Additional build time (3 layers vs 1)
- Can feel over-engineered for simple transformations
- Learning curve for new developers

**Alternatives:**
- **Two-tier (staging + marts):** Skip intermediate, put all logic in marts, simpler but less modular
- **One-tier (direct to marts):** Fastest, but mixes concerns, hard to maintain
- **Four-tier (add metrics layer):** Add semantic layer for BI, very structured but complex

**When to Simplify:**
- Small projects (<10 models): Two-tier sufficient
- Simple transformations: No intermediate needed
- Prototype/POC: One-tier acceptable

**Current Approach Benefits:**
- Intermediate models reusable (sessionization used by multiple marts)
- Easy to test each layer independently
- Clear responsibility boundaries
- Follows dbt style guide (industry standard)

---

### Q: Why timestamp strategy for snapshots instead of check strategy?

**Answer:**
Timestamp strategy is more performant (only checks updated_at column) and source data has reliable updated_at tracking. Sufficient for this use case.

**Timestamp Strategy:**
```sql
strategy='timestamp'
updated_at='updated_at'
```
- Pros: Fast, efficient, works with updated_at column
- Cons: Misses changes if updated_at not updated

**Check Strategy:**
```sql
strategy='check'
check_cols='all'  # or specific columns
```
- Pros: Detects any change in specified columns, no updated_at needed
- Cons: Slower (compares all columns), more expensive

**Downsides of Timestamp:**
- Relies on source system maintaining updated_at correctly
- Won't detect changes if updated_at doesn't change
- Can't detect deletes (need invalidate_hard_deletes=True)

**When to Use Check:**
- No updated_at column in source
- Source system doesn't maintain timestamps reliably
- Need to detect specific column changes only
- Audit requirements need comprehensive change detection

**Hybrid Approach:**
- Timestamp for most tables (performance)
- Check for critical tables (accuracy)
- Monitor both for discrepancies

---

### Q: Why not use incremental models for large tables?

**Answer:**
This project uses full-refresh for simplicity and because data volume is small (7,315 rows). Incremental models add complexity that isn't necessary at this scale.

**Full Refresh (Current):**
- Truncate and reload entire table every run
- Simple logic, easy to understand
- No state management needed
- Sufficient for <100K rows

**Incremental (Alternative):**
```sql
{{
    config(
        materialized='incremental',
        unique_key='interaction_id'
    )
}}

select * from source
{% if is_incremental() %}
    where interaction_timestamp > (select max(interaction_timestamp) from {{ this }})
{% endif %}
```

**When to Use Incremental:**
- Tables with millions of rows
- Long build times (>5 minutes)
- Frequent updates to small portion of data
- Cost optimization (compute and storage)

**Downsides of Incremental:**
- Complex logic (state management, unique_key, filters)
- Late-arriving data issues
- Requires full-refresh occasionally (schema changes)
- Harder to debug
- More edge cases to handle

**Production Recommendation:**
- Start with full-refresh
- Monitor build times
- Convert to incremental when:
  - Build time >10 minutes
  - Table >1M rows
  - Only recent data changes

---

### Q: Why not implement CI/CD pipeline for this project?

**Answer:**
This is a learning project demonstrating data engineering and dbt skills. CI/CD adds complexity without additional learning value for core concepts. However, it's a clear next step for production deployment.

**Current Approach:**
- Manual dbt run locally
- Manual testing
- Local development only

**What CI/CD Would Add:**
- Automated testing on pull requests
- Deployment to production on merge
- Scheduled runs (daily/hourly)
- Data quality monitoring
- Slack/email alerts on failures

**Downsides of Not Having CI/CD:**
- Manual deployment (error-prone)
- No automated testing before production
- No scheduled refreshes
- No alert system
- Can't collaborate easily

**Implementation Options:**
- **GitHub Actions:** Free for public repos, YAML configuration, integrates with dbt Cloud
- **GitLab CI/CD:** Similar to GitHub Actions, built-in to GitLab
- **dbt Cloud:** Built-in scheduler and CI/CD, but costs money
- **Airflow:** Full orchestration, but requires infrastructure
- **Dagster:** Modern orchestrator, good dbt integration, steeper learning curve

**Production Setup:**
```yaml
# .github/workflows/dbt_run.yml
name: dbt CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dbt
        run: pip install dbt-snowflake
      - name: Run tests
        run: dbt test
  
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: dbt run --target prod
```

---

### Q: What are the main limitations of this project for production use?

**Answer:**
This is a learning/portfolio project. Production deployment would require:

**Missing Components:**
- Orchestration (Airflow/dbt Cloud)
- CI/CD pipeline
- Monitoring and alerting
- Data quality dashboards
- Incremental models for scale
- Real source system integration
- Multi-environment setup (dev/staging/prod)
- Secrets management (not hardcoded credentials)
- Cost optimization and monitoring
- Disaster recovery plan
- SLA definitions and tracking

**Data Scale Limitations:**
- Only 7,315 rows (production: millions to billions)
- Q1 2024 only (production: years of history)
- Full refresh only (production: incremental)
- No partitioning or clustering

**Security Limitations:**
- Trial Snowflake account (production: enterprise contract)
- Limited access control testing
- No audit logging implementation
- No encryption key management
- No data retention policies

**Required Enhancements for Production:**
1. Implement incremental models
2. Add orchestration (Airflow DAGs or dbt Cloud jobs)
3. Set up CI/CD (GitHub Actions or dbt Cloud)
4. Add monitoring (dbt artifacts, Snowflake query history)
5. Implement alerting (Slack, PagerDuty)
6. Multi-environment configuration
7. Secrets management (AWS Secrets Manager, HashiCorp Vault)
8. Data lineage and cataloging (Atlan, Alation, dbt Cloud)
9. Cost monitoring and optimization
10. Comprehensive documentation for operations team

---

## Summary

This project prioritizes learning and demonstrating core data engineering skills over production-ready complexity. Design decisions favor:
- Clarity over optimization
- Standard patterns over cutting-edge techniques
- Comprehensive examples over minimal implementations
- Documentation over automation

For production deployment, evaluate each decision against your specific:
- Data volume and velocity
- Team size and skills
- Budget constraints
- Security requirements
- SLA commitments
- Compliance needs

The patterns demonstrated here scale to production with incremental enhancements rather than fundamental redesign.
