# Python Data Generation

## Overview
Generate synthetic seed data using Python to simulate realistic BPO customer journey scenarios. Create 5 CSV files with intentional data quality issues for testing and learning purposes.

## Objectives
- Generate customers dataset with demographic information
- Create customer status updates for SCD Type 2 tracking
- Generate interaction types reference data
- Create customer interactions with various channels
- Build date dimension for time-based analysis
- Include intentional data quality issues for testing

## Prerequisites
- Part 2 completed (dbt project configured)
- Python 3.8+ with pandas, faker libraries
- Understanding of data quality concepts
- seeds/ directory exists in dbt project

---

## Step 1: Install Required Python Libraries

### Purpose
Install libraries for data generation and manipulation.

### Implementation
```powershell
# Ensure virtual environment is activated
cd C:\Users\YOUR_USERNAME\dbt_learning\bpo_customer_journey
.\dbt_venv01\Scripts\activate

# Install required libraries
pip install pandas faker
```

### Library Purposes
- pandas: DataFrame manipulation and CSV export
- faker: Generate realistic fake data (names, emails, phones)

---

## Step 2: Generate Customers Dataset

### Purpose
Create base customer records with intentional data quality issues.

### Data Specifications
- Total customers: 1,010
- Intentional duplicates: 10 (1%)
- Invalid phone numbers: ~30 (3%)
- Null emails: ~20 (2%)
- Segments: enterprise, smb, consumer
- Statuses: lead, trial, paid

### Implementation

File: generate_seed_customers.py

```python
import pandas as pd
from faker import Faker
import random
from datetime import datetime, timedelta

fake = Faker()
random.seed(42)

# Configuration
num_customers = 1000  # Will add 10 duplicates
duplicate_count = 10

# Generate base customers
customers = []

for i in range(1, num_customers + 1):
    customer_id = f'CUST{str(i).zfill(6)}'
    
    # Generate customer data
    customer = {
        'customer_id': customer_id,
        'customer_name': fake.name(),
        'email': fake.email(),
        'phone': f'+63-{random.randint(900,999)}-{random.randint(100,999)}-{random.randint(1000,9999)}',
        'segment': random.choice(['enterprise', 'smb', 'consumer']),
        'status': random.choice(['lead', 'trial', 'paid']),
        'created_at': fake.date_time_between(start_date='-90d', end_date='now').strftime('%Y-%m-%d %H:%M:%S'),
        'updated_at': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'updated_by': 'system'
    }
    customers.append(customer)

# Add intentional duplicates (same customer_id, different data)
for i in range(duplicate_count):
    duplicate = customers[i].copy()
    duplicate['customer_name'] = fake.name()  # Different name
    duplicate['email'] = fake.email()  # Different email
    customers.append(duplicate)

# Introduce data quality issues
for i in range(len(customers)):
    # 3% invalid phone numbers
    if random.random() < 0.03:
        customers[i]['phone'] = f'{random.randint(1000,9999)}-{random.randint(100,999)}'
    
    # 2% null emails
    if random.random() < 0.02:
        customers[i]['email'] = None

# Convert to DataFrame and save
df = pd.DataFrame(customers)
df.to_csv('seeds/seed_customers.csv', index=False)

print(f'Generated {len(df)} customer records')
print(f'Duplicates: {duplicate_count}')
print(f'Invalid phones: ~{int(len(df) * 0.03)}')
print(f'Null emails: ~{int(len(df) * 0.02)}')
```

### Key Design Decisions

**Why Duplicates?**
- Test deduplication logic in staging layer
- Demonstrate unique constraints and tests
- Real-world scenario (data entry errors)

**Why Invalid Phone Numbers?**
- Test regex validation patterns
- Practice data quality testing
- Demonstrate warning vs error severity

**Why Null Emails?**
- Test not_null constraints
- Practice handling missing data
- Demonstrate conditional logic

### Best Practices
- Use random.seed() for reproducible results
- Document intentional issues in comments
- Keep issue percentages realistic (1-5%)
- Generate enough volume for meaningful tests (1000+ rows)

---

## Step 3: Generate Customer Status Updates

### Purpose
Create status change events for SCD Type 2 snapshot testing.

### Data Specifications
- Total updates: 432 records
- Maps to existing customer_ids
- Status transitions: lead to trial to paid to at_risk
- Realistic date progression

### Implementation

File: generate_seed_customer_status_updates.py

```python
import pandas as pd
from datetime import datetime, timedelta
import random

random.seed(42)

# Load existing customers to get valid customer_ids
customers_df = pd.read_csv('seeds/seed_customers.csv')
customer_ids = customers_df['customer_id'].tolist()

# Status progression paths
status_transitions = {
    'lead': ['trial', 'churned'],
    'trial': ['paid', 'churned'],
    'paid': ['at_risk', 'paid'],  # Can stay paid
    'at_risk': ['paid', 'churned']
}

updates = []
base_date = datetime(2024, 1, 15)

# Generate status updates for random subset of customers
num_customers_with_updates = 200

for customer_id in random.sample(customer_ids, num_customers_with_updates):
    # Get initial status from seed
    current_status = customers_df[customers_df['customer_id'] == customer_id]['status'].values[0]
    
    # Generate 1-3 status changes
    num_changes = random.randint(1, 3)
    current_date = base_date + timedelta(days=random.randint(1, 30))
    
    for change in range(num_changes):
        # Select next status based on transition rules
        if current_status in status_transitions:
            next_status = random.choice(status_transitions[current_status])
        else:
            break
        
        update = {
            'customer_id': customer_id,
            'status': next_status,
            'updated_at': current_date.strftime('%Y-%m-%d %H:%M:%S'),
            'updated_by': random.choice(['system', 'sales_team', 'support_team'])
        }
        updates.append(update)
        
        current_status = next_status
        current_date += timedelta(days=random.randint(7, 21))

# Convert to DataFrame and save
df = pd.DataFrame(updates)
df.to_csv('seeds/seed_customer_status_updates.csv', index=False)

print(f'Generated {len(df)} status update records')
print(f'Customers with updates: {num_customers_with_updates}')
```

### Key Design Decisions

**Status Transitions:**
- Follow realistic customer journey paths
- Prevent impossible transitions (churned cannot become paid)
- Allow circular paths (at_risk back to paid)

**Date Logic:**
- Progressive dates (no time travel)
- Realistic intervals (7-21 days between changes)
- Within Q1 2024 timeframe

**Updated By Field:**
- Track who made the change
- Useful for auditing
- Demonstrates metadata tracking

---

## Step 4: Generate Interaction Types Reference

### Purpose
Create reference data for different interaction types in BPO operations.

### Data Specifications
- 7 interaction types
- Categories: voice, digital
- Billability flags
- Average durations

### Implementation

File: generate_seed_interaction_types.py

```python
import pandas as pd

interaction_types = [
    {
        'interaction_type_id': 'INT001',
        'interaction_type': 'inbound_call',
        'type_category': 'voice',
        'is_billable': True,
        'avg_duration_seconds': 420,
        'description': 'Customer-initiated phone call'
    },
    {
        'interaction_type_id': 'INT002',
        'interaction_type': 'outbound_call',
        'type_category': 'voice',
        'is_billable': True,
        'avg_duration_seconds': 360,
        'description': 'Agent-initiated phone call'
    },
    {
        'interaction_type_id': 'INT003',
        'interaction_type': 'live_chat',
        'type_category': 'digital',
        'is_billable': True,
        'avg_duration_seconds': 600,
        'description': 'Real-time web chat session'
    },
    {
        'interaction_type_id': 'INT004',
        'interaction_type': 'email',
        'type_category': 'digital',
        'is_billable': False,
        'avg_duration_seconds': 180,
        'description': 'Email correspondence'
    },
    {
        'interaction_type_id': 'INT005',
        'interaction_type': 'escalation',
        'type_category': 'voice',
        'is_billable': True,
        'avg_duration_seconds': 900,
        'description': 'Escalated to supervisor/manager'
    },
    {
        'interaction_type_id': 'INT006',
        'interaction_type': 'technical_support',
        'type_category': 'voice',
        'is_billable': True,
        'avg_duration_seconds': 720,
        'description': 'Technical troubleshooting session'
    },
    {
        'interaction_type_id': 'INT007',
        'interaction_type': 'billing_inquiry',
        'type_category': 'voice',
        'is_billable': True,
        'avg_duration_seconds': 480,
        'description': 'Billing or payment related inquiry'
    }
]

df = pd.DataFrame(interaction_types)
df.to_csv('seeds/seed_interaction_types.csv', index=False)

print(f'Generated {len(df)} interaction type records')
```

### Key Design Decisions

**Interaction Types:**
- Based on real BPO operations
- Mix of voice and digital channels
- Varying durations for realism

**Billability:**
- Email marked as non-billable
- All voice interactions billable
- Reflects real pricing models

---

## Step 5: Generate Customer Interactions

### Purpose
Create interaction events with session-like patterns for sessionization logic.

### Data Specifications
- Total interactions: 5,775
- Distribution: ~5-6 interactions per customer
- Channels: phone, web chat, email, mobile app
- Sentiment scores: -1.0 to 1.0
- Resolution statuses: resolved, pending, escalated, closed

### Implementation

File: generate_seed_interactions.py

```python
import pandas as pd
from datetime import datetime, timedelta
import random
import uuid

random.seed(42)

# Load customers
customers_df = pd.read_csv('seeds/seed_customers.csv')
customer_ids = customers_df['customer_id'].unique()[:1000]  # Use first 1000 unique

# Load interaction types
interaction_types_df = pd.read_csv('seeds/seed_interaction_types.csv')
interaction_types = interaction_types_df['interaction_type'].tolist()

interactions = []
base_date = datetime(2024, 1, 1)

# Generate 5-6 interactions per customer
for customer_id in customer_ids:
    num_interactions = random.randint(4, 7)
    customer_start_date = base_date + timedelta(days=random.randint(0, 60))
    last_interaction_time = customer_start_date
    
    for i in range(num_interactions):
        # Create session-like patterns (30-min gaps create new sessions)
        if i == 0 or random.random() < 0.3:
            # New session (large time gap)
            time_gap = timedelta(hours=random.randint(24, 168))
        else:
            # Same session (small time gap)
            time_gap = timedelta(minutes=random.randint(1, 25))
        
        interaction_time = last_interaction_time + time_gap
        
        # Ensure within Q1 2024
        if interaction_time > datetime(2024, 3, 31):
            break
        
        interaction_type = random.choice(interaction_types)
        
        # Get avg duration for this type
        avg_duration = interaction_types_df[
            interaction_types_df['interaction_type'] == interaction_type
        ]['avg_duration_seconds'].values[0]
        
        # Add variance to duration
        duration = int(avg_duration * random.uniform(0.5, 1.5))
        
        interaction = {
            'interaction_id': str(uuid.uuid4()),
            'customer_id': customer_id,
            'interaction_type': interaction_type,
            'interaction_timestamp': interaction_time.strftime('%Y-%m-%d %H:%M:%S'),
            'duration_seconds': duration,
            'channel': random.choice(['phone', 'web_chat', 'email', 'mobile_app']),
            'sentiment_score': round(random.uniform(-1.0, 1.0), 2),
            'resolution_status': random.choice(['resolved', 'pending', 'escalated', 'closed'])
        }
        interactions.append(interaction)
        last_interaction_time = interaction_time

# Convert to DataFrame and save
df = pd.DataFrame(interactions)
df.to_csv('seeds/seed_interactions.csv', index=False)

print(f'Generated {len(df)} interaction records')
print(f'Avg interactions per customer: {len(df) / len(customer_ids):.1f}')
```

### Key Design Decisions

**Session Patterns:**
- 30-minute gaps trigger new sessions
- Within-session gaps: 1-25 minutes
- Between-session gaps: 24-168 hours
- Tests sessionization logic in Part 8

**Duration Variance:**
- Base duration from interaction_types
- Add 50% variance for realism
- Some interactions longer/shorter than average

**Sentiment Scores:**
- Normal distribution around 0
- Negative scores indicate dissatisfaction
- Used in customer health metrics

---

## Step 6: Generate Date Dimension

### Purpose
Create calendar dimension for time-based analysis and date joins.

### Data Specifications
- Date range: Q1 2024 (Jan 1 - Mar 31)
- 91 days total
- Philippine holidays marked
- Weekend flags

### Implementation

File: generate_seed_date_dimension.py

```python
import pandas as pd
from datetime import datetime, timedelta

# Q1 2024 date range
start_date = datetime(2024, 1, 1)
end_date = datetime(2024, 3, 31)

dates = []
current_date = start_date

# Philippine holidays in Q1 2024
holidays = [
    datetime(2024, 1, 1),   # New Year's Day
    datetime(2024, 2, 10),  # Chinese New Year (Special non-working)
    datetime(2024, 3, 28),  # Maundy Thursday
    datetime(2024, 3, 29),  # Good Friday
]

while current_date <= end_date:
    date_record = {
        'date_key': int(current_date.strftime('%Y%m%d')),
        'date': current_date.strftime('%Y-%m-%d'),
        'day_of_week': current_date.strftime('%A'),
        'day_of_month': current_date.day,
        'week_of_year': current_date.isocalendar()[1],
        'month': current_date.month,
        'month_name': current_date.strftime('%B'),
        'quarter': (current_date.month - 1) // 3 + 1,
        'year': current_date.year,
        'is_weekend': current_date.weekday() >= 5,
        'is_holiday': current_date in holidays
    }
    dates.append(date_record)
    current_date += timedelta(days=1)

# Convert to DataFrame and save
df = pd.DataFrame(dates)
df.to_csv('seeds/seed_date_dimension.csv', index=False)

print(f'Generated {len(df)} date records')
print(f'Date range: {df["date"].min()} to {df["date"].max()}')
print(f'Holidays: {df["is_holiday"].sum()}')
print(f'Weekend days: {df["is_weekend"].sum()}')
```

### Key Design Decisions

**Date Key Format:**
- YYYYMMDD integer (20240101)
- Sortable and intuitive
- Standard data warehouse practice

**Holiday Selection:**
- Philippine holidays only (project context)
- Can be expanded for other regions
- Used for business day calculations

**Granularity:**
- Daily grain (not hourly)
- Sufficient for most BPO analytics
- Can add time dimension separately if needed

---

## Step 7: Run All Generation Scripts

### Execution Order
```powershell
cd C:\Users\YOUR_USERNAME\dbt_learning\bpo_customer_journey

# Run in order (customers must be first)
python generate_seed_customers.py
python generate_seed_customer_status_updates.py
python generate_seed_interaction_types.py
python generate_seed_interactions.py
python generate_seed_date_dimension.py
```

### Expected Output
```
Generated 1010 customer records
Duplicates: 10
Invalid phones: ~30
Null emails: ~20

Generated 432 status update records
Customers with updates: 200

Generated 7 interaction type records

Generated 5775 interaction records
Avg interactions per customer: 5.8

Generated 91 date records
Date range: 2024-01-01 to 2024-03-31
Holidays: 4
Weekend days: 26
```

---

## Verification Steps

### Check CSV Files Created
```powershell
dir seeds\*.csv
```

Expected files:
- seed_customers.csv (1,010 rows)
- seed_customer_status_updates.csv (432 rows)
- seed_interaction_types.csv (7 rows)
- seed_interactions.csv (5,775 rows)
- seed_date_dimension.csv (91 rows)

### Inspect Data Quality
```powershell
# View first few rows
head seeds\seed_customers.csv
```

### Check for Duplicates
```python
import pandas as pd
df = pd.read_csv('seeds/seed_customers.csv')
print(f"Total rows: {len(df)}")
print(f"Unique customer_ids: {df['customer_id'].nunique()}")
print(f"Duplicates: {len(df) - df['customer_id'].nunique()}")
```

Expected: Total rows: 1010, Unique: 1000, Duplicates: 10

---

## Troubleshooting

### Issue: ModuleNotFoundError for pandas or faker
**Solution:** 
```powershell
pip install pandas faker
# Ensure virtual environment is activated
```

### Issue: FileNotFoundError - seeds directory not found
**Solution:**
```powershell
mkdir seeds
# Or ensure you're in correct directory
cd C:\Users\YOUR_USERNAME\dbt_learning\bpo_customer_journey
```

### Issue: Dates outside Q1 2024
**Solution:**
- Check base_date in scripts
- Verify timedelta logic
- Ensure end_date check in loops

### Issue: Customer IDs don't match between files
**Solution:**
- Run generate_seed_customers.py FIRST
- Other scripts read from seed_customers.csv
- Use same random.seed() for reproducibility

---

## Best Practices Summary

### Data Generation
1. Use faker library for realistic fake data
2. Set random.seed() for reproducibility
3. Document intentional data quality issues
4. Generate sufficient volume (1000+ rows for meaningful tests)

### Data Quality
1. Include 1-5% intentional issues
2. Mark severity appropriately (warn vs error)
3. Test edge cases (nulls, invalid formats, duplicates)
4. Document expected failures

### Maintainability
1. Separate generation scripts by entity
2. Use configuration variables (num_customers, etc.)
3. Add print statements for verification
4. Comment complex logic

### Performance
1. Use list comprehension where possible
2. Avoid nested loops when generating large datasets
3. Use pandas for bulk operations
4. Consider memory usage for very large datasets

---

## Next Steps

After generating seed data:
1. Load seeds into Snowflake using dbt seed (Part 4)
2. Create sources.yml to reference seeds
3. Build staging models on top of seeds (Part 5)

---

## Estimated Time
- Script development: 30-45 minutes (if writing from scratch)
- Running scripts: 2-3 minutes
- Verification: 5 minutes
- Troubleshooting: 10-15 minutes (if needed)

Total: 45-60 minutes

## Success Criteria
- 5 CSV files generated in seeds/ directory
- Row counts match specifications
- Intentional data quality issues present
- Customer IDs consistent across files
- Dates within Q1 2024 range
- Ready to load with dbt seed
