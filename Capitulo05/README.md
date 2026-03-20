# Lab 05-06-01: Práctica 13: Crear rol personalizado con privilegio mínimo y asignar a un usuario

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Medium |
| **Bloom Level** | Understand |

## Overview

This lab demonstrates the principle of least privilege by creating a custom IAM role with precisely scoped permissions. You will design a role for a support analyst who needs to read logs in Logs Explorer and execute read-only queries in BigQuery without administrative capabilities. This practice is essential for maintaining security boundaries and reducing the attack surface in production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Design a custom role with permissions strictly necessary for a specific use case
- [ ] Create and publish the role at the project level and assign it to a principal (user or service account)
- [ ] Validate the resulting access and adjust permissions in case of denials
- [ ] Apply the principle of least privilege to reduce security risks in IAM configurations

## Prerequisites

### Required Knowledge

- Basic understanding of Google Cloud IAM concepts (roles, permissions, principals)
- Familiarity with Cloud Logging and BigQuery interfaces
- Experience navigating the Google Cloud Console
- Understanding of JSON/YAML syntax for role definitions

### Required Access

- Google Cloud Project with billing enabled
- IAM permissions to create custom roles (`roles/iam.roleAdmin` or `roles/owner`)
- Permissions to create service accounts and grant IAM bindings
- Access to Cloud Logging and BigQuery APIs (already enabled)

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 2 cores minimum |
| RAM | 4 GB minimum |
| Network | 10 Mbps stable Internet connection |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage IAM roles and bindings via CLI |
| Google Cloud Shell | Latest | Pre-configured terminal with gcloud and bq |
| bq CLI | v2 | Validate BigQuery permissions |
| IAM API | v1 | Create custom roles |
| Cloud Logging API | v2 | Validate log access |
| BigQuery API | v2 | Test query execution |

### Initial Setup

```bash
# Set your project ID
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1

# Verify current project
echo "Working in project: $PROJECT_ID"

# Enable required APIs (if not already enabled)
gcloud services enable iam.googleapis.com \
  logging.googleapis.com \
  bigquery.googleapis.com \
  cloudresourcemanager.googleapis.com

# Create a test BigQuery dataset and table if not exists
bq mk --dataset --location=$REGION ${PROJECT_ID}:lab_support_dataset 2>/dev/null || echo "Dataset already exists"

# Create a sample table with synthetic data
bq query --use_legacy_sql=false "
CREATE TABLE IF NOT EXISTS \`${PROJECT_ID}.lab_support_dataset.sample_logs\` (
  timestamp TIMESTAMP,
  severity STRING,
  message STRING
);
INSERT INTO \`${PROJECT_ID}.lab_support_dataset.sample_logs\` (timestamp, severity, message)
VALUES 
  (CURRENT_TIMESTAMP(), 'INFO', 'Application started successfully'),
  (CURRENT_TIMESTAMP(), 'WARNING', 'High memory usage detected'),
  (CURRENT_TIMESTAMP(), 'ERROR', 'Connection timeout to database');
"

echo "Initial setup complete"
```

## Step-by-Step Instructions

### Step 1: Define the Custom Role Specification

**Objective:** Identify the minimum set of permissions required for the support analyst use case.

**Instructions:**

1. Document the use case requirements:
   - **Requirement A:** Read logs in Logs Explorer (no sink management)
   - **Requirement B:** Execute read-only queries in BigQuery on a specific dataset (no schema changes or data deletion)

2. Create a role definition file with the necessary permissions:

   ```bash
   cat > custom-support-analyst-role.yaml <<EOF
   title: "Support Analyst Read-Only"
   description: "Allows reading logs and executing read-only BigQuery queries"
   stage: "GA"
   includedPermissions:
   - logging.logEntries.list
   - logging.logs.list
   - resourcemanager.projects.get
   - bigquery.jobs.create
   - bigquery.datasets.get
   - bigquery.tables.get
   - bigquery.tables.list
   - bigquery.tables.getData
   EOF
   ```

3. Review the permission mapping:

   ```bash
   cat custom-support-analyst-role.yaml
   ```

**Expected Output:**

```
title: "Support Analyst Read-Only"
description: "Allows reading logs and executing read-only BigQuery queries"
stage: "GA"
includedPermissions:
- logging.logEntries.list
- logging.logs.list
- resourcemanager.projects.get
- bigquery.jobs.create
- bigquery.datasets.get
- bigquery.tables.get
- bigquery.tables.list
- bigquery.tables.getData
```

**Verification:**

- Confirm the YAML file contains exactly 8 permissions
- Verify no administrative permissions (e.g., `*.delete`, `*.update`, `*.setIamPolicy`) are included

---

### Step 2: Create the Custom Role at Project Level

**Objective:** Publish the custom role definition to the Google Cloud project.

**Instructions:**

1. Generate a unique role ID using your initials and date:

   ```bash
   export CUSTOM_ROLE_ID="supportAnalystReadOnly_$(date +%Y%m%d)"
   echo "Custom Role ID: $CUSTOM_ROLE_ID"
   ```

2. Create the custom role using gcloud:

   ```bash
   gcloud iam roles create $CUSTOM_ROLE_ID \
     --project=$PROJECT_ID \
     --file=custom-support-analyst-role.yaml
   ```

3. Verify the role was created successfully:

   ```bash
   gcloud iam roles describe $CUSTOM_ROLE_ID --project=$PROJECT_ID
   ```

**Expected Output:**

```
description: Allows reading logs and executing read-only BigQuery queries
etag: BwYXXXXXXXX
includedPermissions:
- bigquery.datasets.get
- bigquery.jobs.create
- bigquery.tables.get
- bigquery.tables.getData
- bigquery.tables.list
- logging.logEntries.list
- logging.logs.list
- resourcemanager.projects.get
name: projects/PROJECT_ID/roles/supportAnalystReadOnly_YYYYMMDD
stage: GA
title: Support Analyst Read-Only
```

**Verification:**

- Confirm the role name includes your project ID and custom role ID
- Verify all 8 permissions are listed
- Check that `stage: GA` indicates the role is available for use

---

### Step 3: Create a Test Service Account

**Objective:** Create a principal to which the custom role will be assigned for validation.

**Instructions:**

1. Create a service account for testing:

   ```bash
   export SA_NAME="support-analyst-test"
   export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
   
   gcloud iam service-accounts create $SA_NAME \
     --display-name="Support Analyst Test Account" \
     --description="Test account for custom role validation"
   ```

2. Verify the service account was created:

   ```bash
   gcloud iam service-accounts describe $SA_EMAIL
   ```

3. Store the service account email for later use:

   ```bash
   echo "Service Account: $SA_EMAIL"
   ```

**Expected Output:**

```
displayName: Support Analyst Test Account
email: support-analyst-test@PROJECT_ID.iam.gserviceaccount.com
name: projects/PROJECT_ID/serviceAccounts/support-analyst-test@PROJECT_ID.iam.gserviceaccount.com
```

**Verification:**

- Confirm the service account email matches the expected format
- Verify the display name and description are correct

---

### Step 4: Assign the Custom Role to the Service Account

**Objective:** Grant the custom role to the test service account at the project level.

**Instructions:**

1. Bind the custom role to the service account:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member="serviceAccount:${SA_EMAIL}" \
     --role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"
   ```

2. Additionally, grant dataset-level access in BigQuery for the specific dataset:

   ```bash
   bq show --format=prettyjson ${PROJECT_ID}:lab_support_dataset > dataset_info.json
   
   # Add the service account with READER role
   cat dataset_info.json | jq '.access += [{"role": "READER", "userByEmail": "'$SA_EMAIL'"}]' > dataset_updated.json
   
   bq update --source dataset_updated.json ${PROJECT_ID}:lab_support_dataset
   ```

3. Verify the IAM policy binding:

   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:serviceAccount:${SA_EMAIL}" \
     --format="table(bindings.role)"
   ```

**Expected Output:**

```
ROLE
projects/PROJECT_ID/roles/supportAnalystReadOnly_YYYYMMDD
```

**Verification:**

- Confirm the custom role appears in the IAM policy for the service account
- Verify the binding is at the project level

---

### Step 5: Validate Logging Permissions

**Objective:** Test that the service account can read logs but cannot manage sinks.

**Instructions:**

1. Impersonate the service account to test logging access:

   ```bash
   gcloud logging read "resource.type=global" \
     --limit=5 \
     --impersonate-service-account=$SA_EMAIL \
     --format="table(timestamp,severity,jsonPayload.message)"
   ```

2. Attempt to list log entries (should succeed):

   ```bash
   gcloud logging logs list \
     --impersonate-service-account=$SA_EMAIL \
     --limit=10
   ```

3. Attempt to create a log sink (should fail with permission denied):

   ```bash
   gcloud logging sinks create test-sink \
     storage.googleapis.com/test-bucket-nonexistent \
     --impersonate-service-account=$SA_EMAIL 2>&1 | grep -i "permission\|denied\|error"
   ```

**Expected Output:**

For log reading (success):
```
TIMESTAMP            SEVERITY  MESSAGE
2024-01-15T10:30:00  INFO      Application started
2024-01-15T10:31:00  WARNING   High CPU usage
```

For sink creation (failure):
```
ERROR: (gcloud.logging.sinks.create) User [support-analyst-test@PROJECT_ID.iam.gserviceaccount.com] does not have permission to access projects instance [PROJECT_ID] (or it may not exist): Permission 'logging.sinks.create' denied
```

**Verification:**

- Confirm log entries are readable
- Verify sink creation fails with a permission error
- Check that the error message mentions `logging.sinks.create` permission

---

### Step 6: Validate BigQuery Permissions

**Objective:** Test that the service account can execute read-only queries but cannot modify data or schemas.

**Instructions:**

1. Execute a SELECT query using the service account:

   ```bash
   bq query \
     --use_legacy_sql=false \
     --impersonation_service_account=$SA_EMAIL \
     "SELECT timestamp, severity, message FROM \`${PROJECT_ID}.lab_support_dataset.sample_logs\` LIMIT 5"
   ```

2. Attempt to insert data (should fail):

   ```bash
   bq query \
     --use_legacy_sql=false \
     --impersonation_service_account=$SA_EMAIL \
     "INSERT INTO \`${PROJECT_ID}.lab_support_dataset.sample_logs\` (timestamp, severity, message) VALUES (CURRENT_TIMESTAMP(), 'INFO', 'Unauthorized insert')" 2>&1 | grep -i "permission\|denied\|error"
   ```

3. Attempt to delete a table (should fail):

   ```bash
   bq rm -f -t ${PROJECT_ID}:lab_support_dataset.sample_logs \
     --impersonation_service_account=$SA_EMAIL 2>&1 | grep -i "permission\|denied\|error"
   ```

4. Attempt to create a new table (should fail):

   ```bash
   bq mk -t ${PROJECT_ID}:lab_support_dataset.unauthorized_table \
     id:INTEGER,name:STRING \
     --impersonation_service_account=$SA_EMAIL 2>&1 | grep -i "permission\|denied\|error"
   ```

**Expected Output:**

For SELECT query (success):
```
+---------------------+----------+-------------------------------------+
|      timestamp      | severity |               message               |
+---------------------+----------+-------------------------------------+
| 2024-01-15 10:00:00 | INFO     | Application started successfully    |
| 2024-01-15 10:05:00 | WARNING  | High memory usage detected          |
| 2024-01-15 10:10:00 | ERROR    | Connection timeout to database      |
+---------------------+----------+-------------------------------------+
```

For INSERT/DELETE/CREATE operations (failure):
```
Error in query string: Access Denied: Table PROJECT_ID:lab_support_dataset.sample_logs: Permission bigquery.tables.updateData denied
```

**Verification:**

- Confirm SELECT queries execute successfully
- Verify INSERT, DELETE, and CREATE operations fail with permission errors
- Check that error messages reference missing write/update/delete permissions

---

### Step 7: Test Permission Boundaries and Hardening

**Objective:** Validate that no unintended permissions exist and refine the role if necessary.

**Instructions:**

1. Test access to a different dataset (should fail):

   ```bash
   # Create a second dataset for boundary testing
   bq mk --dataset --location=$REGION ${PROJECT_ID}:restricted_dataset 2>/dev/null || echo "Dataset exists"
   
   bq query \
     --use_legacy_sql=false \
     --impersonation_service_account=$SA_EMAIL \
     "SELECT 1" 2>&1 | head -5
   ```

2. Verify the service account cannot list all projects:

   ```bash
   gcloud projects list \
     --impersonate-service-account=$SA_EMAIL 2>&1 | grep -i "permission\|denied\|error"
   ```

3. Document any permission denials encountered:

   ```bash
   cat > permission-validation-results.txt <<EOF
   Permission Validation Results
   =============================
   
   SUCCESSFUL OPERATIONS:
   - Read log entries via Logs Explorer
   - List available logs
   - Execute SELECT queries on authorized dataset
   - Get table metadata
   
   DENIED OPERATIONS (Expected):
   - Create/update/delete log sinks
   - Insert/update/delete BigQuery data
   - Create/modify BigQuery tables
   - Access unauthorized datasets
   - List all projects in organization
   
   ROLE HARDENING RECOMMENDATIONS:
   - Current permissions are minimal and appropriate
   - No additional restrictions needed
   - Dataset-level access control enforced via BigQuery ACLs
   EOF
   
   cat permission-validation-results.txt
   ```

**Expected Output:**

```
Permission Validation Results
=============================

SUCCESSFUL OPERATIONS:
- Read log entries via Logs Explorer
- List available logs
- Execute SELECT queries on authorized dataset
- Get table metadata

DENIED OPERATIONS (Expected):
- Create/update/delete log sinks
- Insert/update/delete BigQuery data
- Create/modify BigQuery tables
- Access unauthorized datasets
- List all projects in organization

ROLE HARDENING RECOMMENDATIONS:
- Current permissions are minimal and appropriate
- No additional restrictions needed
- Dataset-level access control enforced via BigQuery ACLs
```

**Verification:**

- Confirm all write/administrative operations are blocked
- Verify read operations succeed only on authorized resources
- Document that the role follows the principle of least privilege

---

### Step 8: (Optional) Assign the Role to a Human User

**Objective:** Demonstrate how to assign the custom role to a real user account for production use.

**Instructions:**

1. Identify a test user email (use your own or a colleague's):

   ```bash
   # Replace with actual user email
   export TEST_USER_EMAIL="analyst@example.com"
   echo "Test user: $TEST_USER_EMAIL"
   ```

2. Grant the custom role to the user:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member="user:${TEST_USER_EMAIL}" \
     --role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"
   ```

3. Grant dataset-level access in BigQuery:

   ```bash
   bq show --format=prettyjson ${PROJECT_ID}:lab_support_dataset > dataset_info_user.json
   
   cat dataset_info_user.json | jq '.access += [{"role": "READER", "userByEmail": "'$TEST_USER_EMAIL'"}]' > dataset_updated_user.json
   
   bq update --source dataset_updated_user.json ${PROJECT_ID}:lab_support_dataset
   ```

4. Instruct the user to test access via the Cloud Console:
   - Navigate to **Logging > Logs Explorer**
   - Execute a query to view recent logs
   - Navigate to **BigQuery > SQL Workspace**
   - Run a SELECT query on `lab_support_dataset.sample_logs`

**Expected Output:**

```
Updated IAM policy for project [PROJECT_ID].
bindings:
- members:
  - user:analyst@example.com
  role: projects/PROJECT_ID/roles/supportAnalystReadOnly_YYYYMMDD
...
```

**Verification:**

- Confirm the user can access Logs Explorer and view logs
- Verify the user can query the authorized BigQuery dataset
- Check that the user cannot perform administrative actions

---

## Validation & Testing

### Success Criteria

- [ ] Custom role created with exactly 8 permissions (no more, no less)
- [ ] Role successfully assigned to a service account at project level
- [ ] Service account can read logs but cannot create/modify sinks
- [ ] Service account can execute SELECT queries on authorized dataset
- [ ] Service account cannot INSERT, UPDATE, DELETE, or modify schemas in BigQuery
- [ ] Service account cannot access unauthorized datasets or projects
- [ ] Permission denials are documented and expected
- [ ] Role follows the principle of least privilege

### Testing Procedure

1. **Comprehensive Permission Test:**

   ```bash
   # Create a test script
   cat > test-custom-role.sh <<'EOF'
   #!/bin/bash
   set -e
   
   PROJECT_ID=$(gcloud config get-value project)
   SA_EMAIL="support-analyst-test@${PROJECT_ID}.iam.gserviceaccount.com"
   
   echo "=== Testing Logging Permissions ==="
   echo "Test 1: Read logs (should succeed)"
   gcloud logging read "resource.type=global" --limit=1 --impersonate-service-account=$SA_EMAIL > /dev/null && echo "✓ PASS" || echo "✗ FAIL"
   
   echo "Test 2: Create sink (should fail)"
   gcloud logging sinks create test-sink storage.googleapis.com/test --impersonate-service-account=$SA_EMAIL 2>&1 | grep -q "Permission.*denied" && echo "✓ PASS (correctly denied)" || echo "✗ FAIL"
   
   echo ""
   echo "=== Testing BigQuery Permissions ==="
   echo "Test 3: SELECT query (should succeed)"
   bq query --use_legacy_sql=false --impersonation_service_account=$SA_EMAIL "SELECT COUNT(*) FROM \`${PROJECT_ID}.lab_support_dataset.sample_logs\`" > /dev/null && echo "✓ PASS" || echo "✗ FAIL"
   
   echo "Test 4: INSERT data (should fail)"
   bq query --use_legacy_sql=false --impersonation_service_account=$SA_EMAIL "INSERT INTO \`${PROJECT_ID}.lab_support_dataset.sample_logs\` (timestamp, severity, message) VALUES (CURRENT_TIMESTAMP(), 'TEST', 'test')" 2>&1 | grep -q "denied\|Denied" && echo "✓ PASS (correctly denied)" || echo "✗ FAIL"
   
   echo ""
   echo "All tests completed."
   EOF
   
   chmod +x test-custom-role.sh
   ./test-custom-role.sh
   ```

   **Expected Result:**
   ```
   === Testing Logging Permissions ===
   Test 1: Read logs (should succeed)
   ✓ PASS
   Test 2: Create sink (should fail)
   ✓ PASS (correctly denied)
   
   === Testing BigQuery Permissions ===
   Test 3: SELECT query (should succeed)
   ✓ PASS
   Test 4: INSERT data (should fail)
   ✓ PASS (correctly denied)
   
   All tests completed.
   ```

2. **Role Audit Check:**

   ```bash
   gcloud iam roles describe $CUSTOM_ROLE_ID --project=$PROJECT_ID --format="value(includedPermissions)" | wc -l
   ```

   **Expected Result:** `8` (exactly 8 permissions)

3. **IAM Policy Verification:**

   ```bash
   gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --filter="bindings.role:${CUSTOM_ROLE_ID}" --format="table(bindings.members)"
   ```

   **Expected Result:** Shows service account (and optionally user) with the custom role

## Troubleshooting

### Issue 1: Custom Role Creation Fails with "Already Exists" Error

**Symptoms:**
- Error message: `ALREADY_EXISTS: Role supportAnalystReadOnly_YYYYMMDD already exists`
- Role creation command returns non-zero exit code

**Cause:**
A role with the same ID already exists in the project, possibly from a previous lab run.

**Solution:**

```bash
# Option A: Delete the existing role and recreate
gcloud iam roles delete $CUSTOM_ROLE_ID --project=$PROJECT_ID
sleep 5
gcloud iam roles create $CUSTOM_ROLE_ID --project=$PROJECT_ID --file=custom-support-analyst-role.yaml

# Option B: Update the existing role instead
gcloud iam roles update $CUSTOM_ROLE_ID --project=$PROJECT_ID --file=custom-support-analyst-role.yaml

# Option C: Use a different role ID with timestamp
export CUSTOM_ROLE_ID="supportAnalystReadOnly_$(date +%Y%m%d_%H%M%S)"
gcloud iam roles create $CUSTOM_ROLE_ID --project=$PROJECT_ID --file=custom-support-analyst-role.yaml
```

---

### Issue 2: Service Account Impersonation Fails

**Symptoms:**
- Error: `Permission 'iam.serviceAccounts.getAccessToken' denied`
- Commands with `--impersonate-service-account` flag fail

**Cause:**
Your user account lacks the `roles/iam.serviceAccountTokenCreator` role on the service account.

**Solution:**

```bash
# Grant yourself permission to impersonate the service account
gcloud iam service-accounts add-iam-policy-binding $SA_EMAIL \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/iam.serviceAccountTokenCreator"

# Verify the binding
gcloud iam service-accounts get-iam-policy $SA_EMAIL

# Retry the impersonation command
gcloud logging read "resource.type=global" --limit=1 --impersonate-service-account=$SA_EMAIL
```

---

### Issue 3: BigQuery Queries Fail Despite Correct IAM Role

**Symptoms:**
- Error: `Access Denied: Dataset PROJECT_ID:lab_support_dataset: Permission bigquery.datasets.get denied`
- SELECT queries fail even though role includes `bigquery.datasets.get`

**Cause:**
BigQuery requires dataset-level access control in addition to project-level IAM permissions.

**Solution:**

```bash
# Verify dataset ACLs
bq show --format=prettyjson ${PROJECT_ID}:lab_support_dataset | jq '.access'

# Add the service account with READER role at dataset level
bq show --format=prettyjson ${PROJECT_ID}:lab_support_dataset > dataset_acl.json

cat dataset_acl.json | jq '.access += [{"role": "READER", "userByEmail": "'$SA_EMAIL'"}]' > dataset_acl_updated.json

bq update --source dataset_acl_updated.json ${PROJECT_ID}:lab_support_dataset

# Verify the update
bq show --format=prettyjson ${PROJECT_ID}:lab_support_dataset | jq '.access[] | select(.userByEmail=="'$SA_EMAIL'")'

# Retry the query
bq query --use_legacy_sql=false --impersonation_service_account=$SA_EMAIL "SELECT * FROM \`${PROJECT_ID}.lab_support_dataset.sample_logs\` LIMIT 1"
```

---

### Issue 4: Logs Explorer Shows "No Logs Available"

**Symptoms:**
- Logs Explorer returns empty results
- Error: "You don't have permission to view logs in this project"

**Cause:**
Either insufficient permissions or no logs have been generated in the project yet.

**Solution:**

```bash
# Generate sample log entries
gcloud logging write test-log "Test log entry for validation" --severity=INFO
gcloud logging write test-log "Another test entry" --severity=WARNING

# Wait a few seconds for logs to propagate
sleep 10

# Verify logs exist
gcloud logging read "logName=projects/${PROJECT_ID}/logs/test-log" --limit=5

# Test with service account impersonation
gcloud logging read "logName=projects/${PROJECT_ID}/logs/test-log" \
  --limit=5 \
  --impersonate-service-account=$SA_EMAIL

# If still failing, check IAM binding
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SA_EMAIL}"
```

---

### Issue 5: Permission Denied When Listing Logs

**Symptoms:**
- Error: `Permission 'logging.logs.list' denied`
- `gcloud logging logs list` fails with the service account

**Cause:**
The custom role may not have been applied correctly, or there's a delay in IAM propagation.

**Solution:**

```bash
# Wait for IAM propagation (can take up to 80 seconds)
echo "Waiting for IAM propagation..."
sleep 80

# Verify the role includes the required permission
gcloud iam roles describe $CUSTOM_ROLE_ID --project=$PROJECT_ID | grep "logging.logs.list"

# Check if the role is bound to the service account
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"

# If missing, re-apply the binding
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"

# Retry after propagation
sleep 30
gcloud logging logs list --impersonate-service-account=$SA_EMAIL --limit=10
```

---

## Cleanup

```bash
# Remove IAM policy binding for service account
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"

# (Optional) Remove binding for human user
# gcloud projects remove-iam-policy-binding $PROJECT_ID \
#   --member="user:${TEST_USER_EMAIL}" \
#   --role="projects/${PROJECT_ID}/roles/${CUSTOM_ROLE_ID}"

# Delete the service account
gcloud iam service-accounts delete $SA_EMAIL --quiet

# Delete the custom role
gcloud iam roles delete $CUSTOM_ROLE_ID --project=$PROJECT_ID

# Delete the test BigQuery dataset and tables
bq rm -r -f -d ${PROJECT_ID}:lab_support_dataset

# Remove temporary files
rm -f custom-support-analyst-role.yaml \
      dataset_info.json \
      dataset_updated.json \
      dataset_info_user.json \
      dataset_updated_user.json \
      dataset_acl.json \
      dataset_acl_updated.json \
      permission-validation-results.txt \
      test-custom-role.sh

# Verify cleanup
echo "Verifying cleanup..."
gcloud iam roles describe $CUSTOM_ROLE_ID --project=$PROJECT_ID 2>&1 | grep -q "NOT_FOUND\|deleted" && echo "✓ Role deleted" || echo "⚠ Role still exists"
gcloud iam service-accounts describe $SA_EMAIL 2>&1 | grep -q "NOT_FOUND\|deleted" && echo "✓ Service account deleted" || echo "⚠ Service account still exists"
bq ls -d ${PROJECT_ID}:lab_support_dataset 2>&1 | grep -q "Not found" && echo "✓ Dataset deleted" || echo "⚠ Dataset still exists"

echo "Cleanup complete"
```

> ⚠️ **Warning:** Deleting a custom role that is currently assigned to principals may disrupt access. Always remove IAM bindings before deleting roles. Deleted roles can be undeleted within 7 days if needed using `gcloud iam roles undelete`.

## Summary

### What You Accomplished

- Designed a custom IAM role with 8 precisely scoped permissions for a support analyst use case
- Created and published the custom role at the project level using gcloud CLI
- Assigned the role to a test service account and validated least-privilege access
- Confirmed the service account can read logs and execute BigQuery SELECT queries
- Verified the service account cannot perform administrative or write operations
- Documented permission boundaries and hardening recommendations
- Applied the principle of least privilege to reduce security risks

### Key Takeaways

- **Principle of Least Privilege:** Granting only the minimum permissions necessary reduces the attack surface and limits potential damage from compromised credentials
- **Custom Roles:** Google Cloud's IAM allows fine-grained control through custom roles, enabling precise permission sets beyond predefined roles
- **Layered Access Control:** BigQuery requires both project-level IAM permissions and dataset-level ACLs for complete access control
- **Permission Testing:** Always validate custom roles by testing both allowed and denied operations to ensure correct scoping
- **IAM Propagation:** Permission changes can take up to 80 seconds to propagate; build wait times into automation scripts
- **Audit and Documentation:** Maintain clear documentation of custom roles, their purpose, and validation results for security audits

### Next Steps

- Explore creating custom roles for other personas (developers, auditors, data engineers)
- Implement IAM Recommender to identify over-permissioned principals and optimize existing roles
- Set up Cloud Asset Inventory to track IAM policy changes and detect privilege escalation
- Practice using IAM Conditions to add time-based or resource-based constraints to role bindings
- Review predefined roles in the IAM documentation to understand Google's recommended permission sets
- Continue to Lab 05-06-02 to explore Workload Identity for GKE and eliminate service account keys

## Additional Resources

- [Google Cloud IAM Custom Roles Documentation](https://cloud.google.com/iam/docs/creating-custom-roles) - Official guide to creating and managing custom roles
- [Understanding IAM Permissions](https://cloud.google.com/iam/docs/permissions-reference) - Complete reference of all available permissions
- [IAM Best Practices](https://cloud.google.com/iam/docs/best-practices) - Google's recommended patterns for secure IAM configuration
- [BigQuery Access Control](https://cloud.google.com/bigquery/docs/access-control) - Understanding dataset-level and table-level permissions
- [Cloud Logging Access Control](https://cloud.google.com/logging/docs/access-control) - Permissions required for log viewing and management
- [IAM Policy Troubleshooter](https://cloud.google.com/iam/docs/troubleshooting-access) - Tool to diagnose why a principal has or lacks access
- [Service Account Impersonation](https://cloud.google.com/iam/docs/impersonating-service-accounts) - How to test permissions without generating keys

---

# Lab 05-07-01: Práctica 14: Configurar Workload Identity en GKE para evitar uso de claves

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

This lab demonstrates how to eliminate the security risk of storing service account keys in containers by implementing Workload Identity in Google Kubernetes Engine (GKE). You will configure seamless authentication between Kubernetes pods and Google Cloud services using identity federation, enabling pods to access Cloud resources without managing static credentials. This approach follows Google Cloud security best practices by leveraging Application Default Credentials (ADC) and the principle of least privilege.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Enable Workload Identity on a GKE cluster and configure node pools to support it
- [ ] Create and bind a Kubernetes Service Account (KSA) with a Google Service Account (GSA) using IAM policy bindings
- [ ] Deploy and validate that a pod can access Google Cloud Storage using ADC without service account keys
- [ ] Verify access control by testing both authorized and unauthorized operations

## Prerequisites

### Required Knowledge

- Basic understanding of Kubernetes concepts (pods, service accounts, namespaces)
- Familiarity with Google Cloud IAM (service accounts, roles, permissions)
- Experience with gcloud CLI and kubectl commands
- Understanding of Cloud Storage bucket operations

### Required Access

- Google Cloud Project with billing enabled
- IAM roles: `roles/container.admin`, `roles/iam.serviceAccountAdmin`, `roles/storage.admin`
- Sufficient quota for GKE cluster (minimum 3 nodes, 4 vCPUs)
- APIs enabled: `container.googleapis.com`, `iam.googleapis.com`, `storage.googleapis.com`

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 4+ vCPUs |
| RAM | 8+ GB |
| Network | Stable internet connection ≥10 Mbps |
| Browser | Chrome, Firefox, or Edge (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage GCP resources and authentication |
| kubectl | 1.29.x | Interact with GKE cluster |
| GKE | 1.24+ | Kubernetes cluster with Workload Identity support |
| Cloud Shell | Latest | Pre-configured environment with all tools |

### Initial Setup

```bash
# Set your project ID (replace with your actual project)
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export CLUSTER_NAME=workload-identity-cluster
export NAMESPACE=demo-wi
export KSA_NAME=demo-ksa
export GSA_NAME=demo-gsa
export BUCKET_NAME=${PROJECT_ID}-wi-demo-bucket

# Verify project is set
echo "Working with project: $PROJECT_ID"

# Enable required APIs
gcloud services enable container.googleapis.com \
    iam.googleapis.com \
    storage.googleapis.com \
    --project=$PROJECT_ID
```

## Step-by-Step Instructions

### Step 1: Create GKE Cluster with Workload Identity Enabled

**Objective:** Provision a GKE cluster with Workload Identity enabled at the cluster and node pool level.

**Instructions:**

1. Create a GKE cluster with Workload Identity enabled:

   ```bash
   gcloud container clusters create $CLUSTER_NAME \
       --region=$REGION \
       --num-nodes=1 \
       --machine-type=e2-medium \
       --workload-pool=${PROJECT_ID}.svc.id.goog \
       --enable-ip-alias \
       --release-channel=regular \
       --project=$PROJECT_ID
   ```

2. Get cluster credentials for kubectl:

   ```bash
   gcloud container clusters get-credentials $CLUSTER_NAME \
       --region=$REGION \
       --project=$PROJECT_ID
   ```

3. Verify Workload Identity is enabled on the cluster:

   ```bash
   gcloud container clusters describe $CLUSTER_NAME \
       --region=$REGION \
       --format="value(workloadIdentityConfig.workloadPool)"
   ```

**Expected Output:**

```
Creating cluster workload-identity-cluster...done.
Created [https://container.googleapis.com/v1/projects/PROJECT_ID/zones/us-central1/clusters/workload-identity-cluster].
Fetching cluster endpoint and auth data.
kubeconfig entry generated for workload-identity-cluster.

[PROJECT_ID].svc.id.goog
```

**Verification:**

- Confirm the output shows `[PROJECT_ID].svc.id.goog`
- Run `kubectl get nodes` to verify cluster connectivity

---

### Step 2: Create Cloud Storage Bucket for Testing

**Objective:** Set up a test bucket that will be used to validate Workload Identity access.

**Instructions:**

1. Create a Cloud Storage bucket:

   ```bash
   gsutil mb -p $PROJECT_ID -c STANDARD -l $REGION gs://$BUCKET_NAME/
   ```

2. Upload a test file to the bucket:

   ```bash
   echo "Workload Identity Test File" > test-file.txt
   gsutil cp test-file.txt gs://$BUCKET_NAME/
   ```

3. Verify the file was uploaded:

   ```bash
   gsutil ls gs://$BUCKET_NAME/
   ```

**Expected Output:**

```
Creating gs://PROJECT_ID-wi-demo-bucket/...
Copying file://test-file.txt [Content-Type=text/plain]...
/ [1 files][   28.0 B/   28.0 B]
Operation completed over 1 objects/28.0 B.

gs://PROJECT_ID-wi-demo-bucket/test-file.txt
```

**Verification:**

- Confirm the bucket exists with `gsutil ls | grep $BUCKET_NAME`
- Verify the test file is present in the bucket

---

### Step 3: Create Google Service Account with Minimal Permissions

**Objective:** Create a GSA with only the permissions needed to read from the test bucket.

**Instructions:**

1. Create the Google Service Account:

   ```bash
   gcloud iam service-accounts create $GSA_NAME \
       --display-name="Demo GSA for Workload Identity" \
       --project=$PROJECT_ID
   ```

2. Grant the GSA read-only access to the bucket:

   ```bash
   gsutil iam ch serviceAccount:${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com:objectViewer \
       gs://$BUCKET_NAME
   ```

3. Verify the IAM binding:

   ```bash
   gsutil iam get gs://$BUCKET_NAME
   ```

**Expected Output:**

```
Created service account [demo-gsa].

{
  "bindings": [
    {
      "members": [
        "serviceAccount:demo-gsa@PROJECT_ID.iam.gserviceaccount.com"
      ],
      "role": "roles/storage.objectViewer"
    }
  ]
}
```

**Verification:**

- Confirm the service account appears in `gcloud iam service-accounts list`
- Verify the objectViewer role is assigned in the bucket IAM policy

---

### Step 4: Create Kubernetes Service Account and Namespace

**Objective:** Set up a Kubernetes namespace and service account that will be bound to the GSA.

**Instructions:**

1. Create the namespace:

   ```bash
   kubectl create namespace $NAMESPACE
   ```

2. Create the Kubernetes Service Account:

   ```bash
   kubectl create serviceaccount $KSA_NAME \
       --namespace=$NAMESPACE
   ```

3. Annotate the KSA with the GSA email:

   ```bash
   kubectl annotate serviceaccount $KSA_NAME \
       --namespace=$NAMESPACE \
       iam.gke.io/gcp-service-account=${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
   ```

4. Verify the annotation:

   ```bash
   kubectl describe serviceaccount $KSA_NAME --namespace=$NAMESPACE
   ```

**Expected Output:**

```
namespace/demo-wi created
serviceaccount/demo-ksa created
serviceaccount/demo-ksa annotated

Name:                demo-ksa
Namespace:           demo-wi
Annotations:         iam.gke.io/gcp-service-account: demo-gsa@PROJECT_ID.iam.gserviceaccount.com
```

**Verification:**

- Confirm the annotation is present with the correct GSA email
- Run `kubectl get sa -n $NAMESPACE` to list service accounts

---

### Step 5: Bind KSA to GSA with IAM Policy

**Objective:** Create the IAM policy binding that allows the KSA to impersonate the GSA.

**Instructions:**

1. Add the `iam.workloadIdentityUser` role binding to the GSA:

   ```bash
   gcloud iam service-accounts add-iam-policy-binding \
       ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
       --role=roles/iam.workloadIdentityUser \
       --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]" \
       --project=$PROJECT_ID
   ```

2. Verify the IAM policy binding:

   ```bash
   gcloud iam service-accounts get-iam-policy \
       ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
       --project=$PROJECT_ID
   ```

**Expected Output:**

```
Updated IAM policy for serviceAccount [demo-gsa@PROJECT_ID.iam.gserviceaccount.com].
bindings:
- members:
  - serviceAccount:PROJECT_ID.svc.id.goog[demo-wi/demo-ksa]
  role: roles/iam.workloadIdentityUser
etag: BwYXXXXXXXX
version: 1
```

**Verification:**

- Confirm the member matches the pattern `serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]`
- Verify the role is `roles/iam.workloadIdentityUser`

---

### Step 6: Deploy Test Pod Using Workload Identity

**Objective:** Deploy a pod that uses the KSA and validate it can access Cloud Storage via ADC.

**Instructions:**

1. Create a pod manifest that uses the KSA:

   ```bash
   cat <<EOF > workload-identity-test-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: workload-identity-test
     namespace: $NAMESPACE
   spec:
     serviceAccountName: $KSA_NAME
     containers:
     - name: gcloud-test
       image: google/cloud-sdk:slim
       command: ["sleep"]
       args: ["3600"]
     nodeSelector:
       iam.gke.io/gke-metadata-server-enabled: "true"
   EOF
   ```

2. Apply the pod manifest:

   ```bash
   kubectl apply -f workload-identity-test-pod.yaml
   ```

3. Wait for the pod to be running:

   ```bash
   kubectl wait --for=condition=ready pod/workload-identity-test \
       --namespace=$NAMESPACE \
       --timeout=60s
   ```

4. Verify the pod is using the correct service account:

   ```bash
   kubectl describe pod workload-identity-test --namespace=$NAMESPACE | grep "Service Account"
   ```

**Expected Output:**

```
pod/workload-identity-test created
pod/workload-identity-test condition met

Service Account:  demo-ksa
```

**Verification:**

- Run `kubectl get pods -n $NAMESPACE` and confirm status is `Running`
- Verify the pod has the correct service account mounted

---

### Step 7: Validate Access Using Application Default Credentials

**Objective:** Confirm the pod can authenticate to GCP and access the authorized bucket without keys.

**Instructions:**

1. Check the active service account identity from within the pod:

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       gcloud auth list
   ```

2. Verify the pod can list objects in the authorized bucket:

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       gsutil ls gs://$BUCKET_NAME/
   ```

3. Attempt to read the test file:

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       gsutil cat gs://$BUCKET_NAME/test-file.txt
   ```

4. Verify no service account key files exist in the pod:

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       find /var -name "*key.json" 2>/dev/null || echo "No key files found (expected)"
   ```

**Expected Output:**

```
                Credentialed Accounts
ACTIVE  ACCOUNT
*       demo-gsa@PROJECT_ID.iam.gserviceaccount.com

gs://PROJECT_ID-wi-demo-bucket/test-file.txt

Workload Identity Test File

No key files found (expected)
```

**Verification:**

- The active account should be the GSA (not a Compute Engine default)
- The pod should successfully list and read the bucket contents
- No JSON key files should be present in the container filesystem

---

### Step 8: Test Access Denial for Unauthorized Operations

**Objective:** Validate that the pod cannot perform operations outside its granted permissions.

**Instructions:**

1. Attempt to write to the bucket (should fail with objectViewer role):

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       bash -c "echo 'unauthorized write' | gsutil cp - gs://$BUCKET_NAME/unauthorized.txt"
   ```

2. Attempt to delete an object (should also fail):

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- \
       gsutil rm gs://$BUCKET_NAME/test-file.txt
   ```

3. Verify the original file still exists:

   ```bash
   gsutil ls gs://$BUCKET_NAME/test-file.txt
   ```

**Expected Output:**

```
Copying from <STDIN>...
AccessDeniedException: 403 demo-gsa@PROJECT_ID.iam.gserviceaccount.com does not have storage.objects.create access to the Google Cloud Storage object.

AccessDeniedException: 403 demo-gsa@PROJECT_ID.iam.gserviceaccount.com does not have storage.objects.delete access to the Google Cloud Storage object.

gs://PROJECT_ID-wi-demo-bucket/test-file.txt
```

**Verification:**

- Both write and delete operations should fail with 403 AccessDenied errors
- The test file should remain intact in the bucket
- Error messages should reference the GSA, confirming identity is properly assumed

---

## Validation & Testing

### Success Criteria

- [ ] GKE cluster has Workload Identity enabled with correct workload pool
- [ ] GSA created with minimal permissions (objectViewer only on test bucket)
- [ ] KSA properly annotated and bound to GSA via IAM policy
- [ ] Pod successfully authenticates as GSA using ADC without key files
- [ ] Pod can read from authorized bucket but cannot write or delete
- [ ] No service account JSON keys present in container filesystem

### Testing Procedure

1. Run comprehensive validation from within the pod:

   ```bash
   kubectl exec -it workload-identity-test --namespace=$NAMESPACE -- bash -c '
   echo "=== Active Identity ==="
   gcloud auth list --filter=status:ACTIVE --format="value(account)"
   
   echo -e "\n=== Metadata Server Test ==="
   curl -H "Metadata-Flavor: Google" \
     "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email"
   
   echo -e "\n\n=== Authorized Read Test ==="
   gsutil cat gs://'$BUCKET_NAME'/test-file.txt
   
   echo -e "\n=== Unauthorized Write Test (should fail) ==="
   echo "test" | gsutil cp - gs://'$BUCKET_NAME'/fail.txt 2>&1 | grep -i "access\|denied" || echo "Write succeeded (unexpected!)"
   '
   ```

   **Expected Result:** Identity should match GSA, read should succeed, write should fail with access denied.

2. Verify IAM bindings are correct:

   ```bash
   gcloud iam service-accounts get-iam-policy \
       ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
       --format=json | grep -A2 workloadIdentityUser
   ```

   **Expected Result:** Should show the KSA principal with workloadIdentityUser role.

3. Confirm no credentials are mounted as secrets:

   ```bash
   kubectl get pod workload-identity-test -n $NAMESPACE -o jsonpath='{.spec.volumes[*].secret}' | \
       grep -i "key\|credential" || echo "No credential secrets found (expected)"
   ```

   **Expected Result:** No secrets containing keys or credentials.

## Troubleshooting

### Issue 1: Pod Cannot Authenticate (401 or Default Compute Engine Identity)

**Symptoms:**
- Pod shows Compute Engine default service account instead of GSA
- Authentication errors: "Could not load the default credentials"
- `gcloud auth list` shows no active account or wrong account

**Cause:**
Workload Identity metadata server not available to the pod, or node pool not configured correctly.

**Solution:**

```bash
# Verify node pool has GKE metadata server enabled
gcloud container node-pools describe default-pool \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --format="value(config.workloadMetadataConfig.mode)"

# Should output "GKE_METADATA"
# If not, update the node pool:
gcloud container node-pools update default-pool \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --workload-metadata=GKE_METADATA

# Recreate the pod after node pool update
kubectl delete pod workload-identity-test -n $NAMESPACE
kubectl apply -f workload-identity-test-pod.yaml
```

---

### Issue 2: AccessDeniedException When Reading Bucket

**Symptoms:**
- Pod authenticates as correct GSA but gets 403 errors on bucket access
- Error: "does not have storage.objects.get access"

**Cause:**
IAM binding missing or incorrect on the bucket, or workloadIdentityUser binding not properly set.

**Solution:**

```bash
# Re-verify IAM bindings on the GSA
gcloud iam service-accounts get-iam-policy \
    ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com

# If workloadIdentityUser binding is missing, re-add it:
gcloud iam service-accounts add-iam-policy-binding \
    ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]"

# Re-verify bucket IAM policy
gsutil iam get gs://$BUCKET_NAME

# If GSA missing from bucket policy, re-add:
gsutil iam ch serviceAccount:${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com:objectViewer \
    gs://$BUCKET_NAME

# Wait 60 seconds for IAM propagation, then test again
sleep 60
kubectl exec -it workload-identity-test -n $NAMESPACE -- gsutil ls gs://$BUCKET_NAME/
```

---

### Issue 3: KSA Annotation Not Applied or Incorrect

**Symptoms:**
- Pod does not assume GSA identity
- `kubectl describe sa` shows missing or incorrect annotation

**Cause:**
Annotation command failed silently or typo in GSA email.

**Solution:**

```bash
# Verify current annotations
kubectl get serviceaccount $KSA_NAME -n $NAMESPACE -o yaml | grep annotations -A2

# Remove incorrect annotation if present
kubectl annotate serviceaccount $KSA_NAME -n $NAMESPACE \
    iam.gke.io/gcp-service-account-

# Re-apply correct annotation
kubectl annotate serviceaccount $KSA_NAME -n $NAMESPACE \
    iam.gke.io/gcp-service-account=${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com

# Recreate pod to pick up annotation
kubectl delete pod workload-identity-test -n $NAMESPACE
kubectl apply -f workload-identity-test-pod.yaml
```

---

### Issue 4: Workload Identity Not Enabled on Cluster

**Symptoms:**
- Cluster describe shows empty workloadIdentityConfig
- Pods cannot access metadata server for Workload Identity

**Cause:**
Cluster created without `--workload-pool` flag.

**Solution:**

```bash
# Check if Workload Identity is enabled
gcloud container clusters describe $CLUSTER_NAME \
    --region=$REGION \
    --format="value(workloadIdentityConfig.workloadPool)"

# If empty, enable Workload Identity on existing cluster:
gcloud container clusters update $CLUSTER_NAME \
    --region=$REGION \
    --workload-pool=${PROJECT_ID}.svc.id.goog

# Update node pool metadata configuration
gcloud container node-pools update default-pool \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --workload-metadata=GKE_METADATA

# This may take 5-10 minutes; monitor with:
gcloud container operations list --filter="operationType:UPDATE_CLUSTER"
```

---

## Cleanup

```bash
# Delete the test pod
kubectl delete pod workload-identity-test --namespace=$NAMESPACE

# Delete the Kubernetes service account and namespace
kubectl delete serviceaccount $KSA_NAME --namespace=$NAMESPACE
kubectl delete namespace $NAMESPACE

# Remove IAM binding from GSA
gcloud iam service-accounts remove-iam-policy-binding \
    ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --role=roles/iam.workloadIdentityUser \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]" \
    --project=$PROJECT_ID

# Delete the Google Service Account
gcloud iam service-accounts delete \
    ${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
    --project=$PROJECT_ID \
    --quiet

# Delete the test bucket and contents
gsutil -m rm -r gs://$BUCKET_NAME/

# Delete the GKE cluster
gcloud container clusters delete $CLUSTER_NAME \
    --region=$REGION \
    --project=$PROJECT_ID \
    --quiet

# Remove local test file
rm -f test-file.txt workload-identity-test-pod.yaml

echo "Cleanup complete. All resources deleted."
```

> ⚠️ **Warning:** Deleting the GKE cluster will remove all workloads and configurations. Ensure you have saved any important data or configurations before proceeding. Cluster deletion may take 5-10 minutes. Verify all resources are deleted to avoid unexpected charges.

## Summary

### What You Accomplished

- Enabled and configured Workload Identity on a GKE cluster to eliminate the need for service account keys
- Created a secure identity binding between Kubernetes Service Accounts and Google Service Accounts using IAM policies
- Validated that containerized applications can authenticate to Google Cloud services using Application Default Credentials (ADC)
- Implemented and tested the principle of least privilege by granting minimal permissions and verifying access controls
- Demonstrated a keyless authentication pattern that significantly reduces the risk of credential theft and exfiltration

### Key Takeaways

- **Workload Identity eliminates static credentials:** By federating Kubernetes identities with Google Cloud IAM, you remove the need to manage, rotate, or secure JSON key files in containers.
- **IAM bindings are bidirectional:** Both the KSA annotation and the GSA policy binding are required for Workload Identity to function.
- **ADC provides transparent authentication:** Applications using Google Cloud client libraries automatically use Workload Identity without code changes.
- **Least privilege is enforceable:** You can grant fine-grained permissions to GSAs and validate that pods cannot exceed their authorized scope.
- **Metadata server is critical:** The GKE metadata server (`iam.gke.io/gke-metadata-server-enabled`) must be available on nodes for Workload Identity to work.

### Next Steps

- Implement Workload Identity across all production GKE workloads to eliminate key-based authentication
- Explore cross-project Workload Identity for multi-tenant or multi-project architectures
- Integrate Workload Identity with other Google Cloud services (BigQuery, Pub/Sub, Secret Manager)
- Automate Workload Identity configuration using Terraform or Config Connector
- Review and audit service account permissions regularly using IAM recommender and Policy Analyzer
- Proceed to Lab 05-08-01 to implement VPC Service Controls for data exfiltration prevention

## Additional Resources

- [Google Cloud Workload Identity Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) - Official guide and best practices
- [Workload Identity Federation Concepts](https://cloud.google.com/iam/docs/workload-identity-federation) - Deep dive into identity federation architecture
- [GKE Security Hardening Guide](https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster) - Comprehensive security recommendations for GKE
- [Application Default Credentials (ADC)](https://cloud.google.com/docs/authentication/application-default-credentials) - How client libraries authenticate automatically
- [IAM Best Practices for GKE](https://cloud.google.com/iam/docs/best-practices-for-using-workload-identity) - Patterns for secure service account management
- [Troubleshooting Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#troubleshooting) - Common issues and solutions

---

# Lab 05-08-01: Práctica 15: Proteger datos sensibles con VPC Service Controls y validar acceso

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Advanced |
| **Bloom Level** | Understand |

## Overview

In this lab, you will implement VPC Service Controls to protect sensitive data in Google Cloud managed services. You will create an access level and service perimeter using Access Context Manager, restrict access to BigQuery and Cloud Storage resources, and validate that unauthorized access is blocked while authorized access from trusted networks is permitted. This hands-on practice demonstrates a critical security control for preventing data exfiltration in production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create a VPC Service Controls perimeter and include a project with sensitive data
- [ ] Apply access controls over managed services (BigQuery and Cloud Storage)
- [ ] Validate blocks from untrusted sources and permitted access from trusted networks

## Prerequisites

### Required Knowledge

- Understanding of Google Cloud IAM and resource hierarchy (Organization, Folder, Project)
- Basic familiarity with BigQuery and Cloud Storage
- Knowledge of VPC networking and Private Google Access
- Experience with gcloud CLI and Cloud Shell

### Required Access

- Access to a Google Cloud Organization with billing enabled
- `Access Context Manager Admin` role at the organization level
- `Project Owner` or `Editor` role on the target project
- Permissions to create VPC networks and Compute Engine instances
- BigQuery and Cloud Storage APIs enabled in the project

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 4+ vCPU |
| RAM | 8+ GB |
| Network | Stable Internet connection (≥10 Mbps) |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage Google Cloud resources |
| gsutil | 5.x+ | Validate Cloud Storage access |
| bq CLI | v2 | Validate BigQuery access |
| Access Context Manager API | v1 | Create perimeters and access levels |
| BigQuery API | v2 | Protected service for testing |
| Cloud Storage API | v1 | Protected service for testing |

### Initial Setup

```bash
# Set environment variables
export PROJECT_ID=$(gcloud config get-value project)
export ORGANIZATION_ID=$(gcloud projects describe $PROJECT_ID --format="value(parent.id)")
export REGION=us-central1
export ZONE=us-central1-a
export PERIMETER_NAME="sensitive_data_perimeter"
export ACCESS_LEVEL_NAME="trusted_network_access"

# Enable required APIs
gcloud services enable accesscontextmanager.googleapis.com \
  bigquery.googleapis.com \
  storage.googleapis.com \
  compute.googleapis.com \
  servicenetworking.googleapis.com

# Verify organization access
gcloud organizations describe $ORGANIZATION_ID
```

## Step-by-Step Instructions

### Step 1: Create Test Resources (BigQuery and Cloud Storage)

**Objective:** Set up test data resources that will be protected by VPC Service Controls.

**Instructions:**

1. Create a Cloud Storage bucket with test data:
   
   ```bash
   export BUCKET_NAME="${PROJECT_ID}-sensitive-data-$(date +%s)"
   gsutil mb -p $PROJECT_ID -l $REGION gs://$BUCKET_NAME/
   echo "Confidential Data - Employee Records" > test-sensitive-file.txt
   gsutil cp test-sensitive-file.txt gs://$BUCKET_NAME/
   ```

2. Create a BigQuery dataset and table with sample data:

   ```bash
   bq mk --location=$REGION --dataset ${PROJECT_ID}:sensitive_dataset
   
   bq query --use_legacy_sql=false "
   CREATE OR REPLACE TABLE \`${PROJECT_ID}.sensitive_dataset.employee_data\` AS
   SELECT 
     'EMP001' AS employee_id,
     'John Doe' AS name,
     'john.doe@company.com' AS email,
     '123-45-6789' AS ssn
   UNION ALL
   SELECT 'EMP002', 'Jane Smith', 'jane.smith@company.com', '987-65-4321'
   "
   ```

3. Verify resources are accessible before applying controls:

   ```bash
   gsutil ls gs://$BUCKET_NAME/
   bq ls ${PROJECT_ID}:sensitive_dataset
   ```

**Expected Output:**

```
gs://[PROJECT_ID]-sensitive-data-[TIMESTAMP]/test-sensitive-file.txt
               tableId                Type    Labels   Time Partitioning   Clustered Fields  
 ---------------------------------- ------- -------- ------------------- ------------------ 
  employee_data                      TABLE
```

**Verification:**

- Confirm the bucket exists and contains the test file
- Confirm the BigQuery dataset and table are created successfully

---

### Step 2: Create an Access Level for Trusted Networks

**Objective:** Define an access level that specifies which networks or identities are considered trusted.

**Instructions:**

1. Get your current Cloud Shell IP address (for testing purposes):
   
   ```bash
   export CLOUD_SHELL_IP=$(curl -s ifconfig.me)
   echo "Cloud Shell IP: $CLOUD_SHELL_IP"
   ```

2. Create an access level policy YAML file:

   ```bash
   cat > access-level.yaml <<EOF
   name: accessPolicies/\${POLICY_NAME}/accessLevels/${ACCESS_LEVEL_NAME}
   title: Trusted Network Access
   basic:
     conditions:
     - ipSubnetworks:
       - ${CLOUD_SHELL_IP}/32
       - 10.128.0.0/20
   EOF
   ```

3. Get or create the Access Context Manager policy:

   ```bash
   # Check if policy exists
   export POLICY_NAME=$(gcloud access-context-manager policies list \
     --organization=$ORGANIZATION_ID \
     --format="value(name)" | head -n 1)
   
   # If no policy exists, create one
   if [ -z "$POLICY_NAME" ]; then
     gcloud access-context-manager policies create \
       --organization=$ORGANIZATION_ID \
       --title="VPC-SC-Policy"
     export POLICY_NAME=$(gcloud access-context-manager policies list \
       --organization=$ORGANIZATION_ID \
       --format="value(name)" | head -n 1)
   fi
   
   echo "Policy Name: $POLICY_NAME"
   ```

4. Create the access level:

   ```bash
   gcloud access-context-manager levels create ${ACCESS_LEVEL_NAME} \
     --policy=$POLICY_NAME \
     --title="Trusted Network Access" \
     --basic-level-spec=access-level.yaml \
     --combine-function=AND
   ```

**Expected Output:**

```
Created access level [trusted_network_access].
name: accessPolicies/[POLICY_NUMBER]/accessLevels/trusted_network_access
title: Trusted Network Access
```

**Verification:**

- List access levels to confirm creation:
  ```bash
  gcloud access-context-manager levels list --policy=$POLICY_NAME
  ```

---

### Step 3: Create a VPC Service Controls Perimeter

**Objective:** Create a service perimeter that protects BigQuery and Cloud Storage services within your project.

**Instructions:**

1. Get the project number (required for perimeter creation):
   
   ```bash
   export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
   echo "Project Number: $PROJECT_NUMBER"
   ```

2. Create the service perimeter with restricted services:

   ```bash
   gcloud access-context-manager perimeters create ${PERIMETER_NAME} \
     --policy=$POLICY_NAME \
     --title="Sensitive Data Perimeter" \
     --resources=projects/${PROJECT_NUMBER} \
     --restricted-services=bigquery.googleapis.com,storage.googleapis.com \
     --access-levels=accessPolicies/${POLICY_NAME}/accessLevels/${ACCESS_LEVEL_NAME} \
     --perimeter-type=regular
   ```

3. Verify the perimeter was created:

   ```bash
   gcloud access-context-manager perimeters describe ${PERIMETER_NAME} \
     --policy=$POLICY_NAME
   ```

**Expected Output:**

```
name: accessPolicies/[POLICY_NUMBER]/servicePerimeters/sensitive_data_perimeter
title: Sensitive Data Perimeter
status:
  resources:
  - projects/[PROJECT_NUMBER]
  restrictedServices:
  - bigquery.googleapis.com
  - storage.googleapis.com
  accessLevels:
  - accessPolicies/[POLICY_NUMBER]/accessLevels/trusted_network_access
```

**Verification:**

- Confirm the perimeter includes your project
- Verify BigQuery and Cloud Storage are in the restricted services list
- Check that the access level is properly associated

---

### Step 4: Test Access Blocking from Untrusted Source

**Objective:** Validate that VPC Service Controls blocks access attempts from sources not matching the access level.

**Instructions:**

1. Wait 2-3 minutes for the perimeter to propagate:
   
   ```bash
   echo "Waiting for perimeter propagation..."
   sleep 120
   ```

2. Update the access level to exclude your current IP (simulate untrusted source):

   ```bash
   cat > access-level-restricted.yaml <<EOF
   name: accessPolicies/\${POLICY_NAME}/accessLevels/${ACCESS_LEVEL_NAME}
   title: Trusted Network Access
   basic:
     conditions:
     - ipSubnetworks:
       - 10.128.0.0/20
   EOF
   
   gcloud access-context-manager levels update ${ACCESS_LEVEL_NAME} \
     --policy=$POLICY_NAME \
     --basic-level-spec=access-level-restricted.yaml
   ```

3. Wait for the update to propagate:

   ```bash
   sleep 60
   ```

4. Attempt to access Cloud Storage (should be blocked):

   ```bash
   gsutil ls gs://$BUCKET_NAME/ 2>&1 | tee storage-blocked.log
   ```

5. Attempt to query BigQuery (should be blocked):

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT * FROM \`${PROJECT_ID}.sensitive_dataset.employee_data\` LIMIT 1" \
     2>&1 | tee bigquery-blocked.log
   ```

**Expected Output:**

```
AccessDeniedException: 403 Request blocked by VPC Service Controls
OR
Request is prohibited by organization's policy. vpcServiceControlsUniqueIdentifier: [IDENTIFIER]
```

**Verification:**

- Confirm error messages contain "VPC Service Controls" or "organization's policy"
- Save error logs for documentation purposes
- The specific error message confirms the perimeter is active and blocking access

---

### Step 5: Create Trusted VM with Private Google Access

**Objective:** Set up a VM within an authorized VPC network that can access protected resources.

**Instructions:**

1. Create a VPC network with Private Google Access enabled:
   
   ```bash
   gcloud compute networks create trusted-vpc \
     --subnet-mode=custom \
     --bgp-routing-mode=regional
   
   gcloud compute networks subnets create trusted-subnet \
     --network=trusted-vpc \
     --region=$REGION \
     --range=10.128.0.0/20 \
     --enable-private-ip-google-access
   ```

2. Create a firewall rule to allow SSH via IAP:

   ```bash
   gcloud compute firewall-rules create allow-iap-ssh \
     --network=trusted-vpc \
     --allow=tcp:22 \
     --source-ranges=35.235.240.0/20 \
     --direction=INGRESS
   ```

3. Create a VM instance in the trusted network:

   ```bash
   gcloud compute instances create trusted-vm \
     --zone=$ZONE \
     --machine-type=e2-medium \
     --subnet=trusted-subnet \
     --no-address \
     --scopes=cloud-platform \
     --metadata=enable-oslogin=TRUE
   ```

4. Update the access level to include the trusted VPC subnet:

   ```bash
   cat > access-level-with-vpc.yaml <<EOF
   name: accessPolicies/\${POLICY_NAME}/accessLevels/${ACCESS_LEVEL_NAME}
   title: Trusted Network Access
   basic:
     conditions:
     - ipSubnetworks:
       - 10.128.0.0/20
       - ${CLOUD_SHELL_IP}/32
   EOF
   
   gcloud access-context-manager levels update ${ACCESS_LEVEL_NAME} \
     --policy=$POLICY_NAME \
     --basic-level-spec=access-level-with-vpc.yaml
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/[PROJECT_ID]/zones/us-central1-a/instances/trusted-vm].
NAME        ZONE           MACHINE_TYPE  INTERNAL_IP  EXTERNAL_IP  STATUS
trusted-vm  us-central1-a  e2-medium     10.128.0.2                RUNNING
```

**Verification:**

- Verify the VM is running with no external IP
- Confirm Private Google Access is enabled on the subnet
- Wait 60 seconds for access level update to propagate

---

### Step 6: Validate Authorized Access from Trusted VM

**Objective:** Confirm that access from the trusted VM is permitted by VPC Service Controls.

**Instructions:**

1. Wait for access level changes to propagate:
   
   ```bash
   echo "Waiting for access level update..."
   sleep 90
   ```

2. SSH into the trusted VM using IAP tunnel:

   ```bash
   gcloud compute ssh trusted-vm \
     --zone=$ZONE \
     --tunnel-through-iap
   ```

3. Inside the trusted VM, test Cloud Storage access:

   ```bash
   gsutil ls gs://${BUCKET_NAME}/
   gsutil cat gs://${BUCKET_NAME}/test-sensitive-file.txt
   ```

4. Test BigQuery access from the trusted VM:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT * FROM \`${PROJECT_ID}.sensitive_dataset.employee_data\`"
   ```

5. Exit the VM:

   ```bash
   exit
   ```

**Expected Output:**

From Cloud Storage:
```
gs://[BUCKET_NAME]/test-sensitive-file.txt
Confidential Data - Employee Records
```

From BigQuery:
```
+-------------+-------------+---------------------------+-------------+
| employee_id |    name     |           email           |     ssn     |
+-------------+-------------+---------------------------+-------------+
| EMP001      | John Doe    | john.doe@company.com      | 123-45-6789 |
| EMP002      | Jane Smith  | jane.smith@company.com    | 987-65-4321 |
+-------------+-------------+---------------------------+-------------+
```

**Verification:**

- Confirm successful access to both Cloud Storage and BigQuery
- Verify data is retrieved without VPC Service Controls errors
- Document the successful authorized access

---

### Step 7: Test Access from Cloud Shell (Now Authorized)

**Objective:** Verify that Cloud Shell can access resources after being included in the access level.

**Instructions:**

1. From Cloud Shell, attempt to access Cloud Storage:
   
   ```bash
   gsutil ls gs://$BUCKET_NAME/
   ```

2. Query BigQuery from Cloud Shell:

   ```bash
   bq query --use_legacy_sql=false \
     "SELECT COUNT(*) as employee_count FROM \`${PROJECT_ID}.sensitive_dataset.employee_data\`"
   ```

**Expected Output:**

```
gs://[BUCKET_NAME]/test-sensitive-file.txt

+----------------+
| employee_count |
+----------------+
|              2 |
+----------------+
```

**Verification:**

- Confirm Cloud Shell IP is in the access level
- Verify successful access to protected resources
- Compare with earlier blocked attempts

---

## Validation & Testing

### Success Criteria

- [ ] VPC Service Controls perimeter created successfully
- [ ] Access level defined with trusted network ranges
- [ ] Access blocked when source is not in access level (error logs captured)
- [ ] Access permitted from trusted VM with Private Google Access
- [ ] Access permitted from Cloud Shell after IP inclusion
- [ ] BigQuery and Cloud Storage both protected and validated

### Testing Procedure

1. Verify perimeter configuration:
   ```bash
   gcloud access-context-manager perimeters describe ${PERIMETER_NAME} \
     --policy=$POLICY_NAME \
     --format="yaml"
   ```
   **Expected Result:** Perimeter shows project, restricted services, and access levels.

2. Confirm access level settings:
   ```bash
   gcloud access-context-manager levels describe ${ACCESS_LEVEL_NAME} \
     --policy=$POLICY_NAME \
     --format="yaml"
   ```
   **Expected Result:** IP subnets include trusted VPC range and Cloud Shell IP.

3. Review blocked access logs:
   ```bash
   cat storage-blocked.log bigquery-blocked.log
   ```
   **Expected Result:** Logs contain VPC Service Controls error messages.

4. Test egress from trusted VM:
   ```bash
   gcloud compute ssh trusted-vm --zone=$ZONE --tunnel-through-iap --command="gsutil ls gs://${BUCKET_NAME}/"
   ```
   **Expected Result:** Successful listing of bucket contents.

## Troubleshooting

### Issue 1: "Access Context Manager API not enabled" Error

**Symptoms:**
- Error when creating access levels or perimeters
- Message: "API [accesscontextmanager.googleapis.com] not enabled"

**Cause:**
The Access Context Manager API is not enabled for the organization or project.

**Solution:**
```bash
# Enable the API
gcloud services enable accesscontextmanager.googleapis.com

# Verify API is enabled
gcloud services list --enabled | grep accesscontextmanager

# Wait 60 seconds and retry the command
sleep 60
```

---

### Issue 2: "Permission Denied" When Creating Perimeter

**Symptoms:**
- Error: "User does not have permission to access organization"
- Cannot create or modify access levels or perimeters

**Cause:**
User lacks the `Access Context Manager Admin` role at the organization level.

**Solution:**
```bash
# Check current organization-level IAM bindings
gcloud organizations get-iam-policy $ORGANIZATION_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$(gcloud config get-value account)"

# Request organization admin to grant the role
echo "Request the following role from your organization administrator:"
echo "gcloud organizations add-iam-policy-binding $ORGANIZATION_ID \\"
echo "  --member='user:$(gcloud config get-value account)' \\"
echo "  --role='roles/accesscontextmanager.policyAdmin'"
```

---

### Issue 3: Access Still Works After Creating Perimeter

**Symptoms:**
- VPC Service Controls perimeter created but access is not blocked
- No "Request blocked" errors when accessing protected services

**Cause:**
Perimeter changes can take 2-5 minutes to propagate across Google's infrastructure.

**Solution:**
```bash
# Wait for propagation
echo "Waiting for perimeter propagation (3 minutes)..."
sleep 180

# Verify perimeter status
gcloud access-context-manager perimeters describe ${PERIMETER_NAME} \
  --policy=$POLICY_NAME

# Clear any cached credentials
gcloud auth application-default revoke
gcloud auth application-default login

# Retry access test
gsutil ls gs://$BUCKET_NAME/
```

---

### Issue 4: Trusted VM Cannot Access Protected Services

**Symptoms:**
- VM in trusted VPC still receives "Request blocked" errors
- Private Google Access is enabled but access fails

**Cause:**
The VM's subnet IP range is not included in the access level, or Private Google Access is not properly configured.

**Solution:**
```bash
# Verify Private Google Access is enabled
gcloud compute networks subnets describe trusted-subnet \
  --region=$REGION \
  --format="get(privateIpGoogleAccess)"

# Enable if needed
gcloud compute networks subnets update trusted-subnet \
  --region=$REGION \
  --enable-private-ip-google-access

# Verify access level includes the subnet
gcloud access-context-manager levels describe ${ACCESS_LEVEL_NAME} \
  --policy=$POLICY_NAME \
  --format="yaml" | grep -A 5 ipSubnetworks

# Update access level if subnet is missing
cat > access-level-fix.yaml <<EOF
name: accessPolicies/\${POLICY_NAME}/accessLevels/${ACCESS_LEVEL_NAME}
title: Trusted Network Access
basic:
  conditions:
  - ipSubnetworks:
    - 10.128.0.0/20
    - ${CLOUD_SHELL_IP}/32
EOF

gcloud access-context-manager levels update ${ACCESS_LEVEL_NAME} \
  --policy=$POLICY_NAME \
  --basic-level-spec=access-level-fix.yaml

# Wait and retry
sleep 90
```

---

### Issue 5: "VPC Service Controls Unique Identifier" Error

**Symptoms:**
- Error message includes `vpcServiceControlsUniqueIdentifier`
- Access denied with organization policy reference

**Cause:**
This is the expected behavior when VPC Service Controls blocks a request. The unique identifier helps track the blocked request.

**Solution:**
```bash
# This is SUCCESS - the perimeter is working correctly
# Document the error for validation purposes
echo "VPC Service Controls is active and blocking unauthorized access."

# To allow access, ensure the source is in an authorized network:
# 1. Add IP to access level, OR
# 2. Access from a VM in the trusted VPC subnet

# View the perimeter to confirm configuration
gcloud access-context-manager perimeters describe ${PERIMETER_NAME} \
  --policy=$POLICY_NAME
```

---

## Cleanup

```bash
# Delete the VM instance
gcloud compute instances delete trusted-vm \
  --zone=$ZONE \
  --quiet

# Delete firewall rules
gcloud compute firewall-rules delete allow-iap-ssh \
  --quiet

# Delete VPC network and subnet
gcloud compute networks subnets delete trusted-subnet \
  --region=$REGION \
  --quiet

gcloud compute networks delete trusted-vpc \
  --quiet

# Delete BigQuery dataset
bq rm -r -f -d ${PROJECT_ID}:sensitive_dataset

# Delete Cloud Storage bucket
gsutil rm -r gs://$BUCKET_NAME/

# Delete VPC Service Controls perimeter
gcloud access-context-manager perimeters delete ${PERIMETER_NAME} \
  --policy=$POLICY_NAME \
  --quiet

# Delete access level
gcloud access-context-manager levels delete ${ACCESS_LEVEL_NAME} \
  --policy=$POLICY_NAME \
  --quiet

# Clean up local files
rm -f test-sensitive-file.txt access-level*.yaml storage-blocked.log bigquery-blocked.log

echo "Cleanup completed successfully."
```

> ⚠️ **Warning:** VPC Service Controls policies are organization-level resources. Only delete the perimeter and access level you created for this lab. Do not delete the Access Context Manager policy if it's shared with other projects or teams. Verify with your organization administrator before cleanup.

## Summary

### What You Accomplished

- Created a VPC Service Controls perimeter protecting BigQuery and Cloud Storage
- Defined an access level specifying trusted network IP ranges
- Validated that unauthorized access attempts are blocked with specific error messages
- Configured a trusted VM with Private Google Access for authorized data access
- Demonstrated data exfiltration prevention using Google Cloud security controls

### Key Takeaways

- VPC Service Controls provides a security perimeter around Google Cloud managed services, preventing data exfiltration even if IAM permissions are compromised
- Access levels define conditions (IP ranges, device policies, identity) that determine trusted sources
- Private Google Access enables VMs without external IPs to access Google APIs while staying within the security perimeter
- Perimeter changes require propagation time (2-5 minutes) before taking effect
- VPC Service Controls is a critical defense-in-depth layer for protecting sensitive data in regulated industries

### Next Steps

- Explore VPC Service Controls bridge perimeters for secure cross-project data sharing
- Implement ingress and egress rules for fine-grained API call control
- Integrate VPC Service Controls with Access Context Manager device policies for zero-trust security
- Configure dry-run mode to test perimeter impact before enforcement
- Review Cloud Audit Logs for VPC Service Controls violations and access patterns

## Additional Resources

- [VPC Service Controls Overview](https://cloud.google.com/vpc-service-controls/docs/overview) - Official documentation and architecture
- [Access Context Manager](https://cloud.google.com/access-context-manager/docs) - Creating access levels and policies
- [Private Google Access](https://cloud.google.com/vpc/docs/private-google-access) - Configuring private access to Google APIs
- [VPC Service Controls Best Practices](https://cloud.google.com/vpc-service-controls/docs/best-practices) - Security recommendations
- [Troubleshooting VPC Service Controls](https://cloud.google.com/vpc-service-controls/docs/troubleshooting) - Common issues and solutions
- [Data Exfiltration Prevention](https://cloud.google.com/architecture/exfiltration-prevention-toolkit) - Comprehensive security architecture guide
