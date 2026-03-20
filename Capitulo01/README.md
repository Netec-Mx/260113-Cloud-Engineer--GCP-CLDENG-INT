# Lab 01-09-01: Práctica 1: Simular cuentas de facturación con datos ficticios

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Beginner |
| **Bloom Level** | Understand |

## Overview

In this lab, you will explore Google Cloud billing data structures without incurring real costs by simulating billing exports in BigQuery. You will create a dataset and table matching the standard billing export schema, populate it with synthetic cost data, and build SQL queries to analyze spending by project, service, and SKU. Additionally, you will configure budget alerts and create a basic cost visualization dashboard to understand how organizations monitor and control cloud spending.

This hands-on practice provides foundational skills for cost management, enabling you to interpret billing reports and implement proactive budget controls in production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Understand the schema and structure of Google Cloud billing export data in BigQuery
- [ ] Generate and load synthetic billing data to simulate multi-project, multi-service cost scenarios
- [ ] Create budgets and configure alert thresholds without requiring actual billing charges
- [ ] Build SQL queries to analyze costs by project, service, SKU, and labels, and visualize spending trends

## Prerequisites

### Required Knowledge

- Basic understanding of Google Cloud projects and billing accounts
- Familiarity with BigQuery fundamentals (datasets, tables, SQL queries)
- Basic SQL syntax (SELECT, WHERE, GROUP BY, ORDER BY)
- Understanding of CSV file format and data loading concepts

### Required Access

- Billing Account Viewer and Budget Creator roles on a training/lab billing account
- BigQuery Admin role in the practice project
- Project Editor or Owner role to enable APIs
- Access to Cloud Shell or local gcloud CLI installation

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 2 cores minimum |
| RAM | 4 GB minimum |
| Network | Stable Internet connection ≥10 Mbps |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud Project | N/A | Environment with billing enabled |
| Google Cloud Shell | Latest | Managed terminal with pre-installed tools |
| gcloud CLI | 460.0.0+ | Manage GCP resources and APIs |
| bq CLI | v2 | BigQuery command-line tool |
| BigQuery API | v2 | Create datasets, tables, and run queries |
| Cloud Billing Budget API | v1 | Create budgets and alert policies |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Enable required APIs
gcloud services enable bigquery.googleapis.com
gcloud services enable billingbudgets.googleapis.com
gcloud services enable cloudresourcemanager.googleapis.com

# Verify APIs are enabled
gcloud services list --enabled | grep -E "bigquery|billingbudgets"

# Set default region
export REGION="us-central1"
gcloud config set compute/region $REGION
```

## Step-by-Step Instructions

### Step 1: Verify Billing Account Access

**Objective:** Confirm that you have the necessary permissions to view billing data and create budgets.

**Instructions:**

1. List billing accounts accessible to your user:
   
   ```bash
   gcloud billing accounts list
   ```

2. Note your billing account ID and set it as an environment variable:

   ```bash
   export BILLING_ACCOUNT_ID="01ABCD-23EFGH-456789"
   ```

3. Verify your roles on the billing account:

   ```bash
   gcloud billing accounts get-iam-policy $BILLING_ACCOUNT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:user:$(gcloud config get-value account)" \
     --format="table(bindings.role)"
   ```

**Expected Output:**

```
ROLE
roles/billing.viewer
roles/billing.budgetCreator
```

**Verification:**

- You should see at least `roles/billing.viewer` and ideally `roles/billing.budgetCreator` in the output
- If roles are missing, contact your instructor or organization admin to grant them

---

### Step 2: Create BigQuery Dataset and Billing Table

**Objective:** Set up a BigQuery dataset and table with the standard billing export schema to hold simulated cost data.

**Instructions:**

1. Create a new BigQuery dataset for billing simulation:
   
   ```bash
   bq mk --dataset \
     --location=$REGION \
     --description="Simulated billing data for lab practice" \
     $PROJECT_ID:billing_sim
   ```

2. Create the billing export table with standard schema:

   ```bash
   bq mk --table \
     $PROJECT_ID:billing_sim.gcp_billing_export_v1 \
     billing_account_id:STRING,\
     service.id:STRING,\
     service.description:STRING,\
     sku.id:STRING,\
     sku.description:STRING,\
     usage_start_time:TIMESTAMP,\
     usage_end_time:TIMESTAMP,\
     project.id:STRING,\
     project.name:STRING,\
     project.labels:STRING,\
     location.location:STRING,\
     location.country:STRING,\
     cost:FLOAT,\
     currency:STRING,\
     currency_conversion_rate:FLOAT,\
     usage.amount:FLOAT,\
     usage.unit:STRING,\
     credits:FLOAT,\
     invoice.month:STRING
   ```

3. Verify the table was created:

   ```bash
   bq show $PROJECT_ID:billing_sim.gcp_billing_export_v1
   ```

**Expected Output:**

```
Table your-project-id:billing_sim.gcp_billing_export_v1

   Last modified                  Schema                 Total Rows   Total Bytes
 ----------------- ------------------------------------ ------------ -------------
  DD MMM HH:MM:SS   |- billing_account_id: string       0            0
                    |- service.id: string
                    |- service.description: string
                    ...
```

**Verification:**

- The dataset `billing_sim` exists in your project
- The table `gcp_billing_export_v1` contains the billing schema columns
- Total Rows shows 0 (no data loaded yet)

---

### Step 3: Generate Synthetic Billing Data

**Objective:** Create a CSV file with realistic fictitious billing records spanning multiple projects, services, and time periods.

**Instructions:**

1. Create a script to generate synthetic billing data:
   
   ```bash
   cat > generate_billing_data.sh << 'EOF'
   #!/bin/bash
   
   OUTPUT_FILE="billing_data.csv"
   
   # CSV Header
   echo "billing_account_id,service_id,service_description,sku_id,sku_description,usage_start_time,usage_end_time,project_id,project_name,project_labels,location,country,cost,currency,currency_conversion_rate,usage_amount,usage_unit,credits,invoice_month" > $OUTPUT_FILE
   
   # Services and SKUs
   declare -A SERVICES=(
     ["compute"]="Compute Engine"
     ["storage"]="Cloud Storage"
     ["bigquery"]="BigQuery"
     ["network"]="Networking"
   )
   
   declare -A SKUS=(
     ["compute"]="N1 Predefined Instance Core|N1 Predefined Instance Ram"
     ["storage"]="Standard Storage|Storage Nearline"
     ["bigquery"]="Analysis|Streaming Insert"
     ["network"]="Premium Egress|Load Balancing"
   )
   
   PROJECTS=("project-web-prod" "project-data-analytics" "project-dev-test")
   LOCATIONS=("us-central1" "us-east1" "europe-west1")
   BILLING_ACCOUNT="01ABCD-23EFGH-456789"
   
   # Generate 100 records
   for i in {1..100}; do
     # Random service
     SERVICE_KEY=$(echo "${!SERVICES[@]}" | tr ' ' '\n' | shuf -n 1)
     SERVICE_DESC="${SERVICES[$SERVICE_KEY]}"
     SERVICE_ID="services/$SERVICE_KEY"
     
     # Random SKU for that service
     SKU_OPTIONS="${SKUS[$SERVICE_KEY]}"
     SKU_DESC=$(echo "$SKU_OPTIONS" | tr '|' '\n' | shuf -n 1)
     SKU_ID="skus/$(echo $SKU_DESC | tr ' ' '-' | tr '[:upper:]' '[:lower:]')"
     
     # Random project
     PROJECT="${PROJECTS[$((RANDOM % 3))]}"
     
     # Random location
     LOCATION="${LOCATIONS[$((RANDOM % 3))]}"
     
     # Random cost between 0.50 and 500.00
     COST=$(awk -v min=0.5 -v max=500 'BEGIN{srand(); printf "%.2f", min+rand()*(max-min)}')
     
     # Random credits (0 to 20% of cost)
     CREDITS=$(awk -v cost=$COST 'BEGIN{srand(); printf "%.2f", cost * rand() * 0.2}')
     
     # Usage amount
     USAGE=$(awk -v min=1 -v max=1000 'BEGIN{srand(); printf "%.2f", min+rand()*(max-min)}')
     
     # Time (last 30 days)
     DAY_OFFSET=$((RANDOM % 30))
     START_TIME=$(date -u -d "$DAY_OFFSET days ago" +"%Y-%m-%d %H:00:00 UTC")
     END_TIME=$(date -u -d "$DAY_OFFSET days ago" +"%Y-%m-%d %H:59:59 UTC")
     INVOICE_MONTH=$(date -u -d "$DAY_OFFSET days ago" +"%Y%m")
     
     # Labels
     LABELS="env:prod,team:platform"
     
     echo "$BILLING_ACCOUNT,$SERVICE_ID,$SERVICE_DESC,$SKU_ID,$SKU_DESC,$START_TIME,$END_TIME,$PROJECT,$PROJECT,$LABELS,$LOCATION,US,$COST,USD,1.0,$USAGE,hour,$CREDITS,$INVOICE_MONTH" >> $OUTPUT_FILE
   done
   
   echo "Generated $OUTPUT_FILE with 100 records"
   EOF
   
   chmod +x generate_billing_data.sh
   ```

2. Execute the script to generate the CSV:

   ```bash
   ./generate_billing_data.sh
   ```

3. Preview the generated data:

   ```bash
   head -n 5 billing_data.csv
   ```

**Expected Output:**

```
billing_account_id,service_id,service_description,sku_id,sku_description,usage_start_time,usage_end_time,project_id,project_name,project_labels,location,country,cost,currency,currency_conversion_rate,usage_amount,usage_unit,credits,invoice_month
01ABCD-23EFGH-456789,services/compute,Compute Engine,skus/n1-predefined-instance-core,N1 Predefined Instance Core,2024-01-15 10:00:00 UTC,2024-01-15 10:59:59 UTC,project-web-prod,project-web-prod,env:prod;team:platform,us-central1,US,234.56,USD,1.0,512.34,hour,12.34,202401
...
```

**Verification:**

- The file `billing_data.csv` exists in your current directory
- The file contains 101 lines (1 header + 100 data rows)
- Data includes varied projects, services, costs, and timestamps

---

### Step 4: Load Data into BigQuery

**Objective:** Import the synthetic CSV data into the BigQuery billing table for analysis.

**Instructions:**

1. Load the CSV file into BigQuery:
   
   ```bash
   bq load \
     --source_format=CSV \
     --skip_leading_rows=1 \
     --replace \
     $PROJECT_ID:billing_sim.gcp_billing_export_v1 \
     billing_data.csv
   ```

2. Verify the data was loaded:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT COUNT(*) as total_records FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`"
   ```

3. Preview sample records:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       project_id,
       service_description,
       sku_description,
       cost,
       credits,
       usage_start_time
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     LIMIT 5"
   ```

**Expected Output:**

```
+---------------------+----------------+---------------------------+--------+---------+-------------------------+
|     project_id      | service_desc   | sku_description           | cost   | credits | usage_start_time        |
+---------------------+----------------+---------------------------+--------+---------+-------------------------+
| project-web-prod    | Compute Engine | N1 Predefined Instance... | 234.56 |   12.34 | 2024-01-15 10:00:00 UTC |
| project-data-ana... | BigQuery       | Analysis                  | 145.23 |    5.12 | 2024-01-14 15:00:00 UTC |
...
+---------------------+----------------+---------------------------+--------+---------+-------------------------+
```

**Verification:**

- The query returns 100 total records
- Data displays project names, services, costs, and timestamps correctly
- No errors during the load process

---

### Step 5: Analyze Costs with SQL Queries

**Objective:** Build analytical queries to understand spending patterns by project, service, SKU, and time period.

**Instructions:**

1. Calculate total cost by project:
   
   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       project_id,
       ROUND(SUM(cost), 2) as total_cost,
       ROUND(SUM(credits), 2) as total_credits,
       ROUND(SUM(cost - credits), 2) as net_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     GROUP BY project_id
     ORDER BY total_cost DESC"
   ```

2. Identify top 5 most expensive SKUs:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       service_description,
       sku_description,
       COUNT(*) as usage_count,
       ROUND(SUM(cost), 2) as total_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     GROUP BY service_description, sku_description
     ORDER BY total_cost DESC
     LIMIT 5"
   ```

3. Analyze daily spending trend:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       DATE(usage_start_time) as usage_date,
       COUNT(*) as transactions,
       ROUND(SUM(cost), 2) as daily_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     GROUP BY usage_date
     ORDER BY usage_date DESC
     LIMIT 10"
   ```

4. Filter costs by service and location:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       location,
       service_description,
       ROUND(SUM(cost), 2) as total_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     WHERE service_description = 'Compute Engine'
     GROUP BY location, service_description
     ORDER BY total_cost DESC"
   ```

**Expected Output:**

```
# Query 1 - Cost by project
+------------------------+------------+---------------+----------+
|      project_id        | total_cost | total_credits | net_cost |
+------------------------+------------+---------------+----------+
| project-web-prod       |   8234.56  |      456.78   | 7777.78  |
| project-data-analytics |   7123.45  |      389.12   | 6734.33  |
| project-dev-test       |   5432.10  |      234.56   | 5197.54  |
+------------------------+------------+---------------+----------+

# Query 2 - Top SKUs
+------------------+---------------------------+--------------+------------+
| service_desc     | sku_description           | usage_count  | total_cost |
+------------------+---------------------------+--------------+------------+
| Compute Engine   | N1 Predefined Instance... |     28       |  4567.89   |
| BigQuery         | Analysis                  |     25       |  3456.78   |
...
```

**Verification:**

- All queries execute without errors
- Results show aggregated costs with proper rounding
- Data is grouped and sorted as expected

---

### Step 6: Create a Budget and Configure Alerts

**Objective:** Set up a cloud billing budget with threshold alerts to monitor and control spending.

**Instructions:**

1. Navigate to the Billing Budgets page in Cloud Console:
   
   ```bash
   echo "Open this URL in your browser:"
   echo "https://console.cloud.google.com/billing/$BILLING_ACCOUNT_ID/budgets"
   ```

2. In the Cloud Console, click **CREATE BUDGET**.

3. Configure the budget with these settings:
   - **Name:** `Lab Billing Simulation Budget`
   - **Time range:** Monthly
   - **Projects:** Select your lab project
   - **Services:** All services
   - **Credits:** Include credits
   - **Budget type:** Specified amount
   - **Target amount:** `1000` USD

4. Set threshold rules:
   - **50%** of budget (500 USD)
   - **90%** of budget (900 USD)
   - **100%** of budget (1000 USD)

5. Configure notifications:
   - **Email recipients:** Add your email address
   - **Pub/Sub notifications:** (Optional) Skip for this lab

6. Click **FINISH** to create the budget.

7. Verify the budget via CLI:

   ```bash
   gcloud billing budgets list \
     --billing-account=$BILLING_ACCOUNT_ID \
     --format="table(displayName,budgetFilter.projects,amount.specifiedAmount.units)"
   ```

**Expected Output:**

```
DISPLAY_NAME                          PROJECTS                      AMOUNT
Lab Billing Simulation Budget         projects/your-project-id      1000
```

**Verification:**

- The budget appears in the Cloud Console Budgets page
- Email notifications are configured
- Threshold percentages are set at 50%, 90%, and 100%
- The CLI command lists your newly created budget

---

### Step 7: Create a Basic Cost Visualization

**Objective:** Build a simple dashboard or report to visualize spending trends and patterns.

**Instructions:**

1. Create a saved query for cost by service over time:
   
   ```bash
   bq query --use_legacy_sql=false --format=csv \
     "SELECT 
       DATE(usage_start_time) as date,
       service_description,
       ROUND(SUM(cost), 2) as daily_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     GROUP BY date, service_description
     ORDER BY date DESC, daily_cost DESC" > cost_by_service.csv
   ```

2. Open BigQuery in the Cloud Console:

   ```bash
   echo "Open BigQuery Console:"
   echo "https://console.cloud.google.com/bigquery?project=$PROJECT_ID"
   ```

3. In BigQuery Console, run this query and click **EXPLORE DATA** → **Explore with Looker Studio**:

   ```sql
   SELECT 
     DATE(usage_start_time) as date,
     service_description,
     SUM(cost) as total_cost
   FROM `billing_sim.gcp_billing_export_v1`
   GROUP BY date, service_description
   ORDER BY date DESC
   ```

4. In Looker Studio (opens in new tab):
   - Authorize Looker Studio to access BigQuery
   - Create a **Time Series Chart**:
     - **Date Range Dimension:** date
     - **Dimension:** service_description
     - **Metric:** total_cost
   - Create a **Pie Chart**:
     - **Dimension:** service_description
     - **Metric:** SUM(cost)

5. Alternatively, use the CLI to generate a simple text-based report:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       service_description,
       COUNT(*) as transactions,
       ROUND(MIN(cost), 2) as min_cost,
       ROUND(AVG(cost), 2) as avg_cost,
       ROUND(MAX(cost), 2) as max_cost,
       ROUND(SUM(cost), 2) as total_cost
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     GROUP BY service_description
     ORDER BY total_cost DESC"
   ```

**Expected Output:**

```
+------------------+--------------+----------+----------+----------+------------+
| service_desc     | transactions | min_cost | avg_cost | max_cost | total_cost |
+------------------+--------------+----------+----------+----------+------------+
| Compute Engine   |     28       |   12.45  |  163.14  |  498.76  |  4567.89   |
| BigQuery         |     25       |    8.90  |  138.27  |  487.32  |  3456.78   |
| Cloud Storage    |     24       |    5.67  |  125.43  |  465.21  |  3010.32   |
| Networking       |     23       |    3.21  |  112.56  |  445.67  |  2588.88   |
+------------------+--------------+----------+----------+----------+------------+
```

**Verification:**

- The CSV file `cost_by_service.csv` contains daily cost data
- Looker Studio dashboard displays time series and pie charts (if created)
- CLI report shows cost statistics by service
- Data is aggregated and summarized correctly

---

### Step 8: Simulate Budget Alert Conditions

**Objective:** Understand how budget alerts trigger by querying simulated spending against threshold rules.

**Instructions:**

1. Calculate current simulated spending:
   
   ```bash
   bq query --use_legacy_sql=false \
     "SELECT 
       ROUND(SUM(cost), 2) as total_cost,
       ROUND(SUM(cost) / 1000 * 100, 2) as percent_of_budget
     FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`"
   ```

2. Identify which threshold would be triggered:

   ```bash
   bq query --use_legacy_sql=false \
     "WITH spending AS (
       SELECT SUM(cost) as total_cost FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`
     )
     SELECT 
       total_cost,
       CASE 
         WHEN total_cost >= 1000 THEN '100% threshold - Budget exceeded'
         WHEN total_cost >= 900 THEN '90% threshold - Approaching limit'
         WHEN total_cost >= 500 THEN '50% threshold - Half budget used'
         ELSE 'Under 50% - No alert'
       END as alert_status
     FROM spending"
   ```

3. Check your email for budget alert notifications (if real spending exists in your project).

4. Review budget status in Console:

   ```bash
   echo "Check budget status at:"
   echo "https://console.cloud.google.com/billing/$BILLING_ACCOUNT_ID/budgets"
   ```

**Expected Output:**

```
+------------+--------------------+
| total_cost | percent_of_budget  |
+------------+--------------------+
|  13790.11  |      1379.01       |
+------------+--------------------+

+------------+----------------------------------+
| total_cost | alert_status                     |
+------------+----------------------------------+
|  13790.11  | 100% threshold - Budget exceeded |
+------------+----------------------------------+
```

**Verification:**

- Query calculates total simulated cost correctly
- Alert status logic identifies the appropriate threshold
- In a real scenario with actual costs, email notifications would be sent at each threshold
- Budget page in Console shows current vs. target spending (may show $0 if no real costs)

---

## Validation & Testing

### Success Criteria

- [ ] BigQuery dataset `billing_sim` and table `gcp_billing_export_v1` exist with 100 records
- [ ] Synthetic billing data includes multiple projects, services, SKUs, and time periods
- [ ] SQL queries successfully aggregate costs by project, service, SKU, and date
- [ ] A billing budget is created with 50%, 90%, and 100% threshold alerts
- [ ] Cost visualization report or dashboard displays spending trends
- [ ] Budget alert logic correctly identifies threshold conditions based on simulated spending

### Testing Procedure

1. Verify dataset and table structure:
   ```bash
   bq show $PROJECT_ID:billing_sim.gcp_billing_export_v1
   ```
   **Expected Result:** Table schema displays all billing export columns.

2. Confirm record count:
   ```bash
   bq query --use_legacy_sql=false \
     "SELECT COUNT(*) as total FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`"
   ```
   **Expected Result:** `total = 100`

3. Test cost aggregation query:
   ```bash
   bq query --use_legacy_sql=false \
     "SELECT SUM(cost) as total_cost FROM \`$PROJECT_ID.billing_sim.gcp_billing_export_v1\`"
   ```
   **Expected Result:** Returns a numeric sum (e.g., 13790.11).

4. Verify budget exists:
   ```bash
   gcloud billing budgets list --billing-account=$BILLING_ACCOUNT_ID | grep "Lab Billing"
   ```
   **Expected Result:** Budget name appears in the list.

5. Check visualization output:
   ```bash
   cat cost_by_service.csv | head -n 10
   ```
   **Expected Result:** CSV contains date, service, and cost columns with data.

## Troubleshooting

### Issue 1: Permission Denied When Creating Budget

**Symptoms:**
- Error message: `ERROR: (gcloud.billing.budgets.create) User does not have permission to access billing account`
- Unable to create budget in Cloud Console

**Cause:**
Your user account lacks the `roles/billing.budgetCreator` role on the billing account.

**Solution:**
```bash
# Request the billing admin to grant you the role
# (This command must be run by a Billing Account Administrator)
# gcloud billing accounts add-iam-policy-binding $BILLING_ACCOUNT_ID \
#   --member="user:your-email@example.com" \
#   --role="roles/billing.budgetCreator"

# Alternatively, proceed with the lab using only BigQuery queries
# and skip the budget creation step (Steps 6 and 8)
```

---

### Issue 2: BigQuery Load Job Fails with Schema Mismatch

**Symptoms:**
- Error: `Error while reading data, error message: CSV table encountered too many errors, giving up. Rows: 1; errors: 1.`
- Data not loaded into BigQuery table

**Cause:**
The CSV file contains data that doesn't match the table schema, or the delimiter/format is incorrect.

**Solution:**
```bash
# Verify CSV format
head -n 3 billing_data.csv

# Check for correct number of columns (should be 19)
head -n 2 billing_data.csv | tail -n 1 | awk -F',' '{print NF}'

# If column count is wrong, regenerate the CSV
rm billing_data.csv
./generate_billing_data.sh

# Reload with explicit schema autodetection disabled
bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  --replace \
  --allow_jagged_rows \
  --allow_quoted_newlines \
  $PROJECT_ID:billing_sim.gcp_billing_export_v1 \
  billing_data.csv
```

---

### Issue 3: Looker Studio Authorization Fails

**Symptoms:**
- "Access Denied" or "Authorization Required" when trying to connect BigQuery to Looker Studio
- Unable to create visualizations

**Cause:**
Looker Studio requires explicit authorization to access BigQuery data on behalf of your user.

**Solution:**
```bash
# Ensure you have BigQuery Data Viewer role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/bigquery.dataViewer"

# In Looker Studio:
# 1. Click "Authorize" when prompted
# 2. Select your Google account
# 3. Review permissions and click "Allow"
# 4. Retry connecting to BigQuery

# Alternative: Use BigQuery Console charts instead
# Run queries in BigQuery Console and click "Explore Data" > "Chart"
```

---

### Issue 4: Budget Not Appearing in CLI List

**Symptoms:**
- `gcloud billing budgets list` returns empty or doesn't show the newly created budget
- Budget is visible in Cloud Console but not via CLI

**Cause:**
Budget creation may take a few moments to propagate, or the CLI is querying a different billing account.

**Solution:**
```bash
# Wait 30 seconds and retry
sleep 30
gcloud billing budgets list --billing-account=$BILLING_ACCOUNT_ID

# Verify you're using the correct billing account ID
gcloud billing accounts list

# Re-export the correct billing account ID
export BILLING_ACCOUNT_ID="01ABCD-23EFGH-456789"

# List budgets with full details
gcloud billing budgets list \
  --billing-account=$BILLING_ACCOUNT_ID \
  --format=json | jq '.[] | {name: .displayName, amount: .amount}'
```

---

## Cleanup

```bash
# Delete the BigQuery dataset and all tables
bq rm -r -f -d $PROJECT_ID:billing_sim

# Delete the generated CSV file
rm -f billing_data.csv generate_billing_data.sh cost_by_service.csv

# (Optional) Delete the budget if no longer needed
# List budgets to get the budget ID
gcloud billing budgets list --billing-account=$BILLING_ACCOUNT_ID

# Delete budget by name (replace BUDGET_ID with actual ID from list)
# gcloud billing budgets delete BUDGET_ID --billing-account=$BILLING_ACCOUNT_ID

# Verify cleanup
bq ls -d $PROJECT_ID
ls -la billing_data.csv
```

> ⚠️ **Warning:** Deleting the dataset will permanently remove all simulated billing data. If you plan to reference this data later or extend the lab, consider keeping the dataset. Budgets can remain active without incurring charges; they only monitor actual spending.

## Summary

### What You Accomplished

- Created a BigQuery dataset and table matching the Google Cloud billing export schema
- Generated 100 synthetic billing records with realistic cost data across multiple projects and services
- Loaded CSV data into BigQuery and validated the import
- Built SQL queries to analyze spending by project, service, SKU, location, and time
- Configured a billing budget with multi-threshold alerts (50%, 90%, 100%)
- Created cost visualizations using Looker Studio or BigQuery Console charts
- Simulated budget alert conditions by calculating total spending against thresholds

### Key Takeaways

- **Billing Export Structure:** Google Cloud billing data follows a standardized schema with fields for services, SKUs, projects, costs, credits, and usage metadata, enabling detailed cost attribution and analysis.
- **Cost Analysis with SQL:** BigQuery's SQL interface allows flexible aggregation and filtering of billing data to identify top cost drivers, trends, and anomalies.
- **Proactive Budget Management:** Cloud budgets with threshold alerts enable organizations to monitor spending in real time and take corrective action before overruns occur.
- **Visualization for Insights:** Dashboards and reports transform raw billing data into actionable insights, making it easier for stakeholders to understand and optimize cloud costs.
- **Simulated Data for Learning:** Using synthetic data allows you to practice cost management workflows without incurring actual charges or requiring access to production billing data.

### Next Steps

- Extend the simulation by adding more complex label-based filtering (e.g., cost by team, environment, or cost center)
- Integrate budget alerts with Pub/Sub and Cloud Functions to automate responses (e.g., shutting down dev resources when budgets are exceeded)
- Explore Google Cloud's Cost Management tools: Recommender API for cost optimization suggestions, Committed Use Discounts (CUDs), and sustained use discounts
- Practice creating custom billing reports and scheduled queries in BigQuery to automate monthly cost summaries
- Implement real billing export from your organization's billing account and apply the same analysis techniques to production data

## Additional Resources

- [Google Cloud Billing Documentation](https://cloud.google.com/billing/docs) - Official guide to billing accounts, budgets, and cost management
- [BigQuery Billing Export Schema](https://cloud.google.com/billing/docs/how-to/export-data-bigquery-tables#standard-usage-cost-data-schema) - Detailed schema reference for billing export tables
- [Creating and Managing Budgets](https://cloud.google.com/billing/docs/how-to/budgets) - Step-by-step guide to budget configuration and alert setup
- [BigQuery SQL Reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax) - Complete SQL syntax for querying billing and other datasets
- [Looker Studio (formerly Data Studio)](https://support.google.com/looker-studio) - Tutorials for building dashboards and reports
- [Cloud Billing Reports](https://cloud.google.com/billing/docs/how-to/reports) - Using built-in Cloud Console reports for cost analysis
- [FinOps Foundation](https://www.finops.org/) - Best practices and frameworks for cloud financial management
