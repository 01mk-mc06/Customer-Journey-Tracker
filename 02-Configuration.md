# Part 2: dbt Project Configuration

## Overview
Configure dbt Core project with Snowflake connection, install required packages, and establish project structure.

## Objectives
- Install dbt-snowflake adapter
- Configure profiles.yml for Snowflake connection
- Initialize dbt project structure
- Install dbt packages (dbt_utils, dbt_expectations, codegen)
- Configure dbt_project.yml with project settings
- Verify connection and configuration

## Prerequisites
- Part 1 completed (Snowflake infrastructure set up)
- Python 3.8+ installed
- Virtual environment created
- Snowflake credentials available

---

## Step 1: Set Up Python Virtual Environment

### Purpose
Isolate dbt dependencies from system Python packages.

### Implementation
```powershell
# Create virtual environment
cd C:\Users\YOUR_USERNAME\dbt_learning
python -m venv dbt_venv01

# Activate virtual environment
.\dbt_venv01\Scripts\activate

# Verify activation (prompt should show (dbt_venv01))
```

### Best Practices
- Use dedicated virtual environment per project
- Name includes dbt version for clarity
- Document activation steps in project README

---

## Step 2: Install dbt-snowflake

### Purpose
Install dbt Core with Snowflake adapter.

### Implementation
```powershell
# Install dbt-snowflake (includes dbt-core)
pip install dbt-snowflake==1.8.0

# Verify installation
dbt --version
```

### Expected Output
```
installed version: 1.8.0
   latest version: 1.8.0
   
Up to date!

Plugins:
  - snowflake: 1.8.0
```

### Troubleshooting
**Issue:** pip command not found
- Solution: Ensure Python is in PATH, or use full path to pip

**Issue:** Permission denied during installation
- Solution: Use --user flag: pip install --user dbt-snowflake

**Issue:** Version conflicts
- Solution: Use fresh virtual environment, avoid mixing dbt versions

---

## Step 3: Initialize dbt Project

### Purpose
Create dbt project directory structure.

### Implementation
```powershell
# Create project directory
mkdir bpo_customer_journey
cd bpo_customer_journey

# Initialize dbt project
dbt init bpo_customer_journey
```

### Project Structure Created
```
bpo_customer_journey/
├── models/
│   └── example/
├── tests/
├── macros/
├── snapshots/
├── analyses/
├── seeds/
├── dbt_project.yml
└── README.md
```

### Best Practices
- Use descriptive project names (business domain, not generic)
- Delete example/ folder after reviewing
- Keep project root clean (configuration files only)

---

## Step 4: Configure profiles.yml

### Purpose
Store Snowflake connection credentials securely.

### File Location
Windows: C:\Users\YOUR_USERNAME\.dbt\profiles.yml
Linux/Mac: ~/.dbt/profiles.yml

### Implementation
```yaml
bpo_customer_journey:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_LOCATOR
      user: YOUR_USERNAME
      password: YOUR_PASSWORD
      role: transformer_role
      warehouse: transforming_wh
      database: analytics
      schema: staging
      threads: 4
      client_session_keep_alive: false
      query_tag: dbt_customer_journey
```

### Configuration Details

**account:** 
- Format: account_locator.region (e.g., abc12345.us-east-1)
- Find via: SELECT CURRENT_ACCOUNT() in Snowflake

**role:** 
- Use transformer_role (created in Part 1)
- Must have permissions to create tables/views in analytics database

**warehouse:** 
- Use transforming_wh
- Must be sized appropriately (SMALL for this project)

**database:** 
- Use analytics as default
- dbt creates objects here unless overridden

**schema:** 
- Use staging as default target schema
- dbt will prefix with target name (e.g., dev_staging)

**threads:** 
- Number of parallel model executions
- 4 is good for development
- Increase for production (8-16)

**query_tag:** 
- Helps identify dbt queries in Snowflake query history
- Useful for debugging and cost attribution

### Security Best Practices

**Option 1: Environment Variables (Recommended)**
```yaml
bpo_customer_journey:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: transformer_role
      warehouse: transforming_wh
      database: analytics
      schema: staging
      threads: 4
```

Set environment variables:
```powershell
setx SNOWFLAKE_ACCOUNT "abc12345.us-east-1"
setx SNOWFLAKE_USER "your_username"
setx SNOWFLAKE_PASSWORD "your_password"
```

**Option 2: Key-Pair Authentication (Production)**
```yaml
bpo_customer_journey:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: YOUR_ACCOUNT_LOCATOR
      user: YOUR_USERNAME
      authenticator: externalbrowser  # SSO
      # OR
      private_key_path: /path/to/rsa_key.p8
      private_key_passphrase: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PASSPHRASE') }}"
      role: transformer_role
      warehouse: transforming_wh
      database: analytics
      schema: staging
      threads: 4
```

### Troubleshooting profiles.yml

**Issue:** Profile not found
- Check file location: dir C:\Users\YOUR_USERNAME\.dbt\profiles.yml
- Check indentation (YAML is whitespace-sensitive)
- Profile name must match project name in dbt_project.yml

**Issue:** Connection fails
- Verify Snowflake credentials manually via web UI
- Check account locator format (include region)
- Verify role has required permissions
- Check warehouse is running (auto_resume enabled)

**Issue:** SSL/TLS errors
- Add to profile: insecure_mode: false
- Update Snowflake connector: pip install --upgrade snowflake-connector-python

---

## Step 5: Test Connection

### Purpose
Verify dbt can connect to Snowflake with configured credentials.

### Implementation
```powershell
cd C:\Users\YOUR_USERNAME\dbt_learning\bpo_customer_journey
dbt debug
```

### Expected Output
```
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
  - git [OK found]

Connection:
  account: abc12345.us-east-1
  user: YOUR_USERNAME
  database: analytics
  warehouse: transforming_wh
  role: transformer_role
  schema: staging
  Connection test: [OK connection ok]

All checks passed!
```

### Troubleshooting dbt debug

**Issue:** profiles.yml not found
- Create .dbt directory: mkdir C:\Users\YOUR_USERNAME\.dbt
- Verify file name (no typos): profiles.yml not profile.yml

**Issue:** Connection test failed - incorrect username or password
- Verify credentials in Snowflake web UI
- Check for extra spaces in YAML
- Try hardcoding password temporarily to isolate env var issues

**Issue:** Database does not exist
- Verify database name matches (case-sensitive in quotes)
- Check role has USAGE on database

**Issue:** Role does not exist
- Verify role name matches exactly
- Check role is granted to user: show grants to user YOUR_USERNAME;

---

## Step 6: Configure dbt_project.yml

### Purpose
Define project-level settings, model configurations, and naming conventions.

### Implementation
```yaml
name: 'bpo_customer_journey'
version: '1.0.0'
config-version: 2

profile: 'bpo_customer_journey'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  bpo_customer_journey:
    +materialized: view
    
    staging:
      +materialized: view
      +tags: ['staging']
      +schema: staging
      
    intermediate:
      +materialized: view
      +tags: ['intermediate']
      +schema: intermediate
      
    marts:
      +materialized: table
      +tags: ['marts']
      +schema: marts

seeds:
  bpo_customer_journey:
    +schema: seeds
    +database: raw
    +tags: ['seeds']

snapshots:
  bpo_customer_journey:
    +target_schema: snapshots
    +target_database: analytics
    +tags: ['snapshots']
```

### Configuration Explanation

**name:** Project identifier (must match directory name)

**profile:** Must match profile name in profiles.yml

**model-paths:** Where dbt looks for model SQL files

**materialized:** 
- view: Stored as Snowflake view (fast to build, slower to query)
- table: Stored as Snowflake table (slower to build, fast to query)
- ephemeral: CTE in dependent models (not materialized)

**tags:** Used for selective execution (dbt run --select tag:staging)

**schema:** 
- staging models → analytics.staging_staging
- Note: dbt prepends target schema to custom schema
- To override: Use generate_schema_name macro

**seeds configuration:**
- database: raw (separate from analytics)
- schema: seeds (keep seeds isolated)

### Best Practices

**Materialization Strategy:**
- Staging: views (lightweight, low storage)
- Intermediate: views (unless complex aggregations)
- Marts: tables (fast query performance)

**Schema Organization:**
- Use custom schemas for logical separation
- Document schema naming in README
- Accept dbt's schema prefixing for simplicity

**Tags:**
- Tag by layer (staging, intermediate, marts)
- Tag by domain (customers, interactions, finance)
- Tag by refresh frequency (daily, hourly, weekly)

---

## Step 7: Install dbt Packages

### Purpose
Extend dbt functionality with community packages.

### Packages to Install

**dbt_utils:** 
- Utility macros (generate_surrogate_key, date_spine, etc.)
- Cross-database compatibility helpers

**dbt_expectations:** 
- Advanced data quality tests
- Great Expectations integration

**codegen:** 
- Generate boilerplate YAML and SQL
- Speeds up development

### Implementation

Create packages.yml:
```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1
    
  - package: calogica/dbt_expectations
    version: 0.10.3
    
  - package: dbt-labs/codegen
    version: 0.12.1
```

Install packages:
```powershell
dbt deps
```

### Expected Output
```
Installing dbt-labs/dbt_utils@1.1.1
Installing calogica/dbt_expectations@0.10.3
Installing dbt-labs/codegen@0.12.1

Updates available for packages: []

Up to date!
```

### Package Installation Details

Packages installed to: dbt_packages/ directory
- Do NOT commit dbt_packages/ to git
- Add to .gitignore
- Run dbt deps after clone

### Troubleshooting dbt deps

**Issue:** Package version not found
- Check package hub for available versions: hub.getdbt.com
- Try latest version or one major version back

**Issue:** Dependency conflicts
- Check compatibility matrix on package README
- Pin specific versions known to work together

**Issue:** Network errors
- Check firewall/proxy settings
- Try manual download from GitHub

---

## Step 8: Verify Setup

### Run Test Models
```powershell
# Run example models (if not deleted)
dbt run --select example

# Expected: 2 models built successfully
```

### Delete Example Models
```powershell
# Remove example directory
rm -r models/example

# Update dbt_project.yml (remove example section if present)
```

### Verify Package Installation
```powershell
# List installed packages
dir dbt_packages

# Should see: dbt_expectations, dbt_utils, codegen
```

---

## Project Structure After Setup

```
bpo_customer_journey/
├── .dbt/                    (created by dbt)
├── analyses/                (ad-hoc queries)
├── dbt_packages/            (installed packages - not in git)
├── macros/                  (custom Jinja macros)
├── models/                  (SQL transformations)
│   ├── staging/            (to be created in Part 5)
│   ├── intermediate/       (to be created in Part 8)
│   └── marts/              (to be created in Part 10)
├── seeds/                   (CSV reference data)
├── snapshots/               (SCD Type 2 logic)
├── target/                  (compiled SQL - not in git)
├── tests/                   (custom data tests)
├── .gitignore
├── dbt_project.yml         (project config)
├── packages.yml            (package dependencies)
└── README.md
```

---

## Best Practices Summary

### Configuration Management
1. Use environment variables for credentials
2. Never commit profiles.yml to git
3. Document all custom configurations
4. Version control dbt_project.yml and packages.yml

### Project Organization
1. Follow dbt standard directory structure
2. Use consistent naming (lowercase, underscores)
3. Tag models by layer and domain
4. Document folder purposes in README

### Package Management
1. Pin package versions for reproducibility
2. Review package release notes before upgrading
3. Test after package updates
4. Document custom package configurations

### Performance
1. Set appropriate thread count (4-8 for dev)
2. Use query_tag for cost tracking
3. Enable client_session_keep_alive for long runs
4. Monitor warehouse usage and adjust size

---

## Common Issues and Solutions

### Schema Naming (Custom Schema Prefix)
**Issue:** Models create schemas like staging_staging instead of staging

**Cause:** dbt prepends target schema to custom schema by default

**Solutions:**

Option 1: Accept the naming (recommended for learning)
- schemas: staging_staging, staging_intermediate, staging_marts
- No code changes needed
- Document in README

Option 2: Override with custom macro
```sql
-- macros/get_custom_schema.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

### Warehouse Auto-Suspend
**Issue:** Warehouse suspends during dbt runs

**Solution:** 
- Increase auto_suspend to 300 seconds
- Enable auto_resume (should already be enabled)
- Consider larger warehouse for faster execution

### Connection Timeouts
**Issue:** Connection lost during long-running models

**Solution:**
- Enable client_session_keep_alive in profiles.yml
- Break large models into smaller incremental models
- Optimize SQL to reduce execution time

---

## Verification Checklist

Before proceeding to Part 3:
- [ ] Virtual environment activated
- [ ] dbt-snowflake installed (dbt --version shows 1.8.0)
- [ ] profiles.yml configured with correct credentials
- [ ] dbt debug passes all checks
- [ ] dbt_project.yml configured with schemas and materializations
- [ ] packages.yml created with 3 packages
- [ ] dbt deps completed successfully
- [ ] dbt_packages/ directory contains installed packages
- [ ] Example models deleted
- [ ] .gitignore includes target/ and dbt_packages/

---

## Next Steps

After completing project configuration:
1. Generate seed data with Python scripts (Part 3)
2. Load seeds into Snowflake (Part 4)
3. Build staging layer models (Part 5)

---

## Estimated Time
- Virtual environment setup: 5 minutes
- dbt installation: 5 minutes
- Project initialization: 5 minutes
- Configuration: 15-20 minutes
- Package installation: 5 minutes
- Verification and troubleshooting: 10-15 minutes

Total: 45-60 minutes

## Success Criteria
- dbt debug shows "All checks passed!"
- dbt deps completes without errors
- Project structure follows dbt conventions
- Connection to Snowflake successful
- Ready to load seed data
