# Lab 04-06-01: Práctica 10: Crear snapshot de VM y restaurarla en otra zona

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Beginner |
| **Bloom Level** | Understand |

## Overview

In this lab, you will create a snapshot of a VM's boot disk and restore it in a different zone. Snapshots are incremental backups that enable disaster recovery, zone migration, and data preservation. You will create a test VM, generate test data, snapshot the boot disk, restore it in another zone, and validate data persistence. This practice demonstrates essential backup and recovery operations in Google Cloud.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create a snapshot of a VM boot disk
- [ ] Restore a disk from a snapshot in a different zone
- [ ] Launch a new VM using the restored disk and validate data persistence
- [ ] Understand snapshot scheduling policies and associated costs

## Prerequisites

### Required Knowledge

- Basic understanding of Google Cloud Compute Engine
- Familiarity with VM instances, disks, and zones/regions
- Basic Linux command-line skills

### Required Access

- Google Cloud Project with billing enabled
- Compute Admin role (roles/compute.admin) or equivalent permissions
- Access to Google Cloud Console and Cloud Shell

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| Local Machine | 4+ vCPU, 8+ GB RAM |
| Internet Connection | ≥10 Mbps stable connection |
| Browser | Modern browser (Chrome/Edge/Firefox) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage Compute Engine resources |
| Google Cloud Shell | Latest | Terminal environment with pre-installed tools |
| Compute Engine API | v1 | Create and manage VMs and disks |

### Initial Setup

```bash
# Set your project ID
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE_SOURCE=us-central1-a
export ZONE_TARGET=us-central1-b
export VM_NAME_SOURCE=snapshot-source-vm
export VM_NAME_TARGET=snapshot-restored-vm
export SNAPSHOT_NAME=boot-disk-snapshot-$(date +%Y%m%d-%H%M%S)

# Verify project is set
echo "Project ID: $PROJECT_ID"
echo "Source Zone: $ZONE_SOURCE"
echo "Target Zone: $ZONE_TARGET"

# Enable Compute Engine API
gcloud services enable compute.googleapis.com
```

## Step-by-Step Instructions

### Step 1: Create Source VM with Test Data

**Objective:** Create a VM instance in the source zone and add test data to validate snapshot restoration.

**Instructions:**

1. Create a VM instance in the source zone:
   
   ```bash
   gcloud compute instances create $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --machine-type=e2-micro \
     --image-family=debian-11 \
     --image-project=debian-cloud \
     --boot-disk-size=10GB \
     --boot-disk-type=pd-standard \
     --tags=http-server
   ```

2. Wait for the VM to be ready (approximately 30 seconds):

   ```bash
   gcloud compute instances describe $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --format="value(status)"
   ```

3. SSH into the VM and create test data:

   ```bash
   gcloud compute ssh $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --command="echo 'Snapshot Test Data - $(date)' | sudo tee /var/www/test-file.txt && sudo mkdir -p /var/www && echo 'index.html content for snapshot validation' | sudo tee /var/www/index.html"
   ```

4. Verify the test data was created:

   ```bash
   gcloud compute ssh $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --command="sudo cat /var/www/test-file.txt && sudo cat /var/www/index.html"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/snapshot-source-vm].
NAME                ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
snapshot-source-vm  us-central1-a  e2-micro                   10.128.0.2   34.xxx.xxx.xxx RUNNING

Snapshot Test Data - [timestamp]
index.html content for snapshot validation
```

**Verification:**

- Confirm VM status is "RUNNING"
- Verify test files exist in /var/www directory

---

### Step 2: Create Snapshot of Boot Disk

**Objective:** Create a snapshot of the source VM's boot disk to preserve its state.

**Instructions:**

1. Get the boot disk name of the source VM:
   
   ```bash
   export BOOT_DISK_NAME=$(gcloud compute instances describe $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --format="value(disks[0].source.basename())")
   
   echo "Boot Disk Name: $BOOT_DISK_NAME"
   ```

2. Stop the VM to ensure consistent snapshot (optional but recommended for production):

   ```bash
   gcloud compute instances stop $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE
   ```

3. Wait for the VM to stop:

   ```bash
   gcloud compute instances describe $VM_NAME_SOURCE \
     --zone=$ZONE_SOURCE \
     --format="value(status)"
   ```

4. Create the snapshot:

   ```bash
   gcloud compute disks snapshot $BOOT_DISK_NAME \
     --zone=$ZONE_SOURCE \
     --snapshot-names=$SNAPSHOT_NAME \
     --description="Snapshot of boot disk for zone migration lab"
   ```

5. Verify snapshot creation:

   ```bash
   gcloud compute snapshots describe $SNAPSHOT_NAME \
     --format="table(name,status,diskSizeGb,storageBytes)"
   ```

**Expected Output:**

```
Boot Disk Name: snapshot-source-vm
Stopping instance(s) snapshot-source-vm...done.
TERMINATED

Creating snapshot(s) boot-disk-snapshot-20240115-143022...done.

NAME                              STATUS  DISK_SIZE_GB  STORAGE_BYTES
boot-disk-snapshot-20240115-143022 READY   10           1234567890
```

**Verification:**

- Snapshot status is "READY"
- Snapshot size matches the boot disk size (10 GB)

---

### Step 3: Create Disk from Snapshot in Target Zone

**Objective:** Restore the snapshot as a new disk in a different zone to demonstrate cross-zone recovery.

**Instructions:**

1. Create a new disk from the snapshot in the target zone:
   
   ```bash
   export RESTORED_DISK_NAME=restored-boot-disk
   
   gcloud compute disks create $RESTORED_DISK_NAME \
     --zone=$ZONE_TARGET \
     --source-snapshot=$SNAPSHOT_NAME \
     --type=pd-standard
   ```

2. Verify the disk was created:

   ```bash
   gcloud compute disks describe $RESTORED_DISK_NAME \
     --zone=$ZONE_TARGET \
     --format="table(name,zone.basename(),sizeGb,status,sourceSnapshot.basename())"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-b/disks/restored-boot-disk].

NAME                ZONE           SIZE_GB  STATUS  SOURCE_SNAPSHOT
restored-boot-disk  us-central1-b  10       READY   boot-disk-snapshot-20240115-143022
```

**Verification:**

- Disk status is "READY"
- Disk is located in the target zone (us-central1-b)
- Source snapshot matches the created snapshot name

---

### Step 4: Create VM Using Restored Disk

**Objective:** Launch a new VM using the restored disk as the boot disk and validate data persistence.

**Instructions:**

1. Create a VM using the restored disk as the boot disk:
   
   ```bash
   gcloud compute instances create $VM_NAME_TARGET \
     --zone=$ZONE_TARGET \
     --machine-type=e2-micro \
     --disk=name=$RESTORED_DISK_NAME,boot=yes,mode=rw
   ```

2. Wait for the VM to be running:

   ```bash
   gcloud compute instances describe $VM_NAME_TARGET \
     --zone=$ZONE_TARGET \
     --format="value(status)"
   ```

3. SSH into the restored VM and verify the test data:

   ```bash
   gcloud compute ssh $VM_NAME_TARGET \
     --zone=$ZONE_TARGET \
     --command="sudo cat /var/www/test-file.txt && echo '---' && sudo cat /var/www/index.html"
   ```

4. Verify the hostname and zone information:

   ```bash
   gcloud compute ssh $VM_NAME_TARGET \
     --zone=$ZONE_TARGET \
     --command="hostname && curl -H 'Metadata-Flavor: Google' http://metadata.google.internal/computeMetadata/v1/instance/zone"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-b/instances/snapshot-restored-vm].

RUNNING

Snapshot Test Data - [original timestamp]
---
index.html content for snapshot validation

snapshot-restored-vm
projects/123456789/zones/us-central1-b
```

**Verification:**

- VM is running in the target zone (us-central1-b)
- Test files contain the original data
- Data persistence is confirmed across zones

---

### Step 5: Configure Snapshot Schedule Policy (Optional)

**Objective:** Create a scheduled snapshot policy to automate regular backups.

**Instructions:**

1. Create a snapshot schedule policy:
   
   ```bash
   gcloud compute resource-policies create snapshot-schedule daily-backup-policy \
     --region=$REGION \
     --max-retention-days=7 \
     --on-source-disk-delete=keep-auto-snapshots \
     --daily-schedule \
     --start-time=02:00 \
     --storage-location=$REGION
   ```

2. Verify the policy was created:

   ```bash
   gcloud compute resource-policies describe daily-backup-policy \
     --region=$REGION
   ```

3. Attach the policy to the restored disk (for demonstration):

   ```bash
   gcloud compute disks add-resource-policies $RESTORED_DISK_NAME \
     --zone=$ZONE_TARGET \
     --resource-policies=daily-backup-policy
   ```

4. Verify the policy attachment:

   ```bash
   gcloud compute disks describe $RESTORED_DISK_NAME \
     --zone=$ZONE_TARGET \
     --format="value(resourcePolicies)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/resourcePolicies/daily-backup-policy].

name: daily-backup-policy
region: us-central1
snapshotSchedulePolicy:
  retentionPolicy:
    maxRetentionDays: 7
    onSourceDiskDelete: KEEP_AUTO_SNAPSHOTS
  schedule:
    dailySchedule:
      startTime: '02:00'
  storageLocations:
  - us-central1

Updated [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-b/disks/restored-boot-disk].

https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/resourcePolicies/daily-backup-policy
```

**Verification:**

- Snapshot schedule policy is created with daily schedule at 02:00
- Policy is attached to the restored disk
- Retention period is set to 7 days

---

### Step 6: Review Snapshot Costs and Best Practices

**Objective:** Understand snapshot pricing, storage optimization, and best practices.

**Instructions:**

1. List all snapshots and their storage usage:
   
   ```bash
   gcloud compute snapshots list \
     --format="table(name,diskSizeGb,storageBytes,creationTimestamp)"
   ```

2. Calculate approximate monthly snapshot costs:

   ```bash
   echo "Snapshot storage pricing (approximate):"
   echo "- Standard snapshot storage: \$0.026 per GB/month"
   echo ""
   echo "Your snapshot details:"
   gcloud compute snapshots describe $SNAPSHOT_NAME \
     --format="value(storageBytes)" | \
     awk '{printf "Storage: %.2f GB\nEstimated monthly cost: $%.2f\n", $1/1024/1024/1024, ($1/1024/1024/1024)*0.026}'
   ```

3. Review snapshot best practices documentation:

   ```bash
   echo "Snapshot Best Practices:"
   echo "1. Use snapshot schedules for automated backups"
   echo "2. Set appropriate retention policies (7-30 days typical)"
   echo "3. Delete unnecessary snapshots to reduce costs"
   echo "4. Snapshots are incremental - only changed blocks are stored"
   echo "5. Consider regional vs. multi-regional storage based on DR needs"
   echo "6. Use VSS snapshots for Windows or stop VMs for consistency"
   ```

**Expected Output:**

```
NAME                              DISK_SIZE_GB  STORAGE_BYTES  CREATION_TIMESTAMP
boot-disk-snapshot-20240115-143022 10           2147483648     2024-01-15T14:30:22.123-08:00

Snapshot storage pricing (approximate):
- Standard snapshot storage: $0.026 per GB/month

Your snapshot details:
Storage: 2.00 GB
Estimated monthly cost: $0.05

Snapshot Best Practices:
1. Use snapshot schedules for automated backups
2. Set appropriate retention policies (7-30 days typical)
3. Delete unnecessary snapshots to reduce costs
4. Snapshots are incremental - only changed blocks are stored
5. Consider regional vs. multi-regional storage based on DR needs
6. Use VSS snapshots for Windows or stop VMs for consistency
```

**Verification:**

- Snapshot storage usage is displayed
- Cost estimation is calculated
- Best practices are reviewed

---

## Validation & Testing

### Success Criteria

- [ ] Source VM created with test data
- [ ] Snapshot created successfully with READY status
- [ ] Disk restored from snapshot in different zone
- [ ] Target VM launched using restored disk
- [ ] Test data persists and is accessible on target VM
- [ ] Snapshot schedule policy created and attached (optional)

### Testing Procedure

1. Verify both VMs are accessible:
   ```bash
   gcloud compute instances list \
     --filter="name:($VM_NAME_SOURCE OR $VM_NAME_TARGET)" \
     --format="table(name,zone,status,networkInterfaces[0].accessConfigs[0].natIP)"
   ```
   **Expected Result:** Both VMs listed with RUNNING or TERMINATED status

2. Confirm snapshot exists and is complete:
   ```bash
   gcloud compute snapshots list --filter="name=$SNAPSHOT_NAME"
   ```
   **Expected Result:** Snapshot with READY status

3. Validate data integrity on restored VM:
   ```bash
   gcloud compute ssh $VM_NAME_TARGET \
     --zone=$ZONE_TARGET \
     --command="ls -lh /var/www/ && md5sum /var/www/test-file.txt"
   ```
   **Expected Result:** Files exist with correct content

4. Verify cross-zone restoration:
   ```bash
   echo "Source Zone: $(gcloud compute instances describe $VM_NAME_SOURCE --zone=$ZONE_SOURCE --format='value(zone.basename())')"
   echo "Target Zone: $(gcloud compute instances describe $VM_NAME_TARGET --zone=$ZONE_TARGET --format='value(zone.basename())')"
   ```
   **Expected Result:** Different zones displayed

## Troubleshooting

### Issue 1: Snapshot Creation Takes Too Long

**Symptoms:**
- Snapshot status remains in "CREATING" state for extended period
- Timeout errors when checking snapshot status

**Cause:**
Large disk size or high I/O activity on the source disk can slow snapshot creation.

**Solution:**
```bash
# Stop the VM to reduce I/O during snapshot creation
gcloud compute instances stop $VM_NAME_SOURCE --zone=$ZONE_SOURCE

# Wait for VM to stop
sleep 30

# Retry snapshot creation
gcloud compute disks snapshot $BOOT_DISK_NAME \
  --zone=$ZONE_SOURCE \
  --snapshot-names=$SNAPSHOT_NAME-retry
```

---

### Issue 2: Cannot Create VM with Restored Disk - "Disk Already Attached" Error

**Symptoms:**
- Error: "The disk resource 'restored-boot-disk' is already being used"
- VM creation fails

**Cause:**
The disk is already attached to another VM or was not properly detached.

**Solution:**
```bash
# List VMs using the disk
gcloud compute instances list \
  --format="table(name,zone,disks[].source.basename())" \
  --filter="disks[].source:$RESTORED_DISK_NAME"

# If attached to another VM, delete that VM first
gcloud compute instances delete $VM_NAME_TARGET \
  --zone=$ZONE_TARGET \
  --quiet

# Wait a moment and retry VM creation
sleep 10
gcloud compute instances create $VM_NAME_TARGET \
  --zone=$ZONE_TARGET \
  --machine-type=e2-micro \
  --disk=name=$RESTORED_DISK_NAME,boot=yes,mode=rw
```

---

### Issue 3: SSH Connection Fails to Restored VM

**Symptoms:**
- SSH connection times out or refuses connection
- Error: "Permission denied (publickey)"

**Cause:**
SSH keys not propagated or firewall rules blocking SSH access.

**Solution:**
```bash
# Ensure firewall rule allows SSH
gcloud compute firewall-rules create allow-ssh-iap \
  --allow=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --network=default

# Reset the VM instance
gcloud compute instances reset $VM_NAME_TARGET \
  --zone=$ZONE_TARGET

# Wait for reset to complete
sleep 30

# Try SSH again with explicit key generation
gcloud compute ssh $VM_NAME_TARGET \
  --zone=$ZONE_TARGET \
  --force-key-file-overwrite
```

---

### Issue 4: Snapshot Policy Not Creating Automatic Snapshots

**Symptoms:**
- No automatic snapshots appear after scheduled time
- Policy attached but not executing

**Cause:**
Policy schedule may not have triggered yet, or time zone confusion.

**Solution:**
```bash
# Verify policy is attached to disk
gcloud compute disks describe $RESTORED_DISK_NAME \
  --zone=$ZONE_TARGET \
  --format="value(resourcePolicies)"

# Check policy details
gcloud compute resource-policies describe daily-backup-policy \
  --region=$REGION

# Note: Automatic snapshots follow UTC time and may take 24 hours for first execution
# List snapshots created by policy (may be empty if policy just created)
gcloud compute snapshots list \
  --filter="sourceDisk:$RESTORED_DISK_NAME AND autoCreated=true"

# For immediate testing, create a manual snapshot instead
gcloud compute disks snapshot $RESTORED_DISK_NAME \
  --zone=$ZONE_TARGET \
  --snapshot-names=manual-test-snapshot
```

---

## Cleanup

```bash
# Delete the target VM
gcloud compute instances delete $VM_NAME_TARGET \
  --zone=$ZONE_TARGET \
  --quiet

# Delete the source VM
gcloud compute instances delete $VM_NAME_SOURCE \
  --zone=$ZONE_SOURCE \
  --quiet

# Detach snapshot policy from disk
gcloud compute disks remove-resource-policies $RESTORED_DISK_NAME \
  --zone=$ZONE_TARGET \
  --resource-policies=daily-backup-policy

# Delete the restored disk
gcloud compute disks delete $RESTORED_DISK_NAME \
  --zone=$ZONE_TARGET \
  --quiet

# Delete the snapshot
gcloud compute snapshots delete $SNAPSHOT_NAME \
  --quiet

# Delete the snapshot schedule policy
gcloud compute resource-policies delete daily-backup-policy \
  --region=$REGION \
  --quiet

# Verify all resources are deleted
echo "Remaining VMs:"
gcloud compute instances list --filter="name:snapshot-"

echo "Remaining Disks:"
gcloud compute disks list --filter="name:restored-boot-disk"

echo "Remaining Snapshots:"
gcloud compute snapshots list --filter="name:boot-disk-snapshot-"
```

> ⚠️ **Warning:** Deleting snapshots is irreversible. Ensure you no longer need the backup before deletion. Snapshots incur storage costs even when source disks are deleted. Always clean up unused snapshots to avoid unnecessary charges.

## Summary

### What You Accomplished

- Created a VM instance with test data in a source zone
- Generated a snapshot of the VM's boot disk
- Restored the snapshot as a new disk in a different zone
- Launched a new VM using the restored disk and validated data persistence
- Created and attached a snapshot schedule policy for automated backups
- Reviewed snapshot costs and best practices

### Key Takeaways

- **Snapshots are incremental**: Only changed blocks are stored after the first full snapshot, reducing storage costs
- **Cross-zone recovery**: Snapshots enable VM migration between zones for disaster recovery or load balancing
- **Consistency matters**: Stopping VMs before snapshots ensures file system consistency, especially for databases
- **Automation reduces risk**: Snapshot schedules automate backups, reducing human error
- **Cost awareness**: Monitor snapshot storage usage and implement retention policies to control costs
- **Regional storage**: Snapshots can be stored regionally or multi-regionally for different DR requirements

### Next Steps

- Explore multi-regional snapshots for enhanced disaster recovery
- Implement snapshot policies across production workloads
- Integrate snapshot creation with CI/CD pipelines for application backups
- Test snapshot restore procedures regularly to validate recovery processes
- Consider using persistent disk cloning for faster disk duplication within the same zone
- Investigate incremental snapshot chains and their impact on restore time

## Additional Resources

- [Google Cloud Snapshots Documentation](https://cloud.google.com/compute/docs/disks/create-snapshots) - Official guide to creating and managing snapshots
- [Snapshot Best Practices](https://cloud.google.com/compute/docs/disks/snapshot-best-practices) - Recommended practices for snapshot management
- [Snapshot Pricing](https://cloud.google.com/compute/disks-image-pricing#disk) - Detailed pricing information for snapshot storage
- [Scheduled Snapshots](https://cloud.google.com/compute/docs/disks/scheduled-snapshots) - Guide to automating snapshot creation
- [Disaster Recovery Planning](https://cloud.google.com/architecture/dr-scenarios-planning-guide) - Architecture guide for DR strategies
- [Persistent Disk Performance](https://cloud.google.com/compute/docs/disks/performance) - Understanding disk types and performance characteristics

---

# Lab 04-07-01: Práctica 11: Crear tablero en Monitoring con métricas de CPU y alertas

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

In this lab, you will implement a comprehensive monitoring and alerting solution using Google Cloud Monitoring. You will create custom dashboards to visualize key performance metrics from Compute Engine instances, configure alerting policies that trigger based on resource utilization thresholds, and set up notification channels to receive alerts. This lab provides hands-on experience with observability practices essential for maintaining production systems and proactively identifying performance issues.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create custom dashboards in Cloud Monitoring with multiple visualization types for CPU, memory, and disk metrics
- [ ] Configure monitoring for Compute Engine instances and analyze performance data
- [ ] Implement alerting policies based on metric thresholds with appropriate conditions
- [ ] Configure notification channels including email for alert delivery
- [ ] Generate synthetic load to trigger alerts and verify the alerting workflow
- [ ] Analyze monitoring data to identify performance trends and potential issues

## Prerequisites

### Required Knowledge

- Understanding of system performance metrics (CPU, memory, disk, network)
- Basic knowledge of monitoring and observability concepts
- Familiarity with threshold-based alerting principles
- Understanding of incident management workflows
- Basic command-line proficiency for SSH and Linux commands

### Required Access

- Active GCP project with billing enabled
- Project Editor role or specific roles: Compute Admin, Monitoring Admin
- At least one running Compute Engine instance from previous labs
- Access to email for receiving alert notifications

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| Internet Connection | Minimum 10 Mbps bandwidth |
| Display Resolution | Minimum 1366x768 |
| Computer RAM | Minimum 8GB |
| Processor | Dual-core or better |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud Console | Latest web version | Primary interface for monitoring configuration |
| Google Chrome or Firefox | Latest stable release | Web browser for accessing GCP Console |
| Google Cloud SDK (gcloud CLI) | 400.0.0+ | Command-line operations and VM access |
| SSH Client | OpenSSH 7.0+ or PuTTY 0.70+ | Connecting to VM instances |

### Initial Setup

```bash
# Verify you have an active GCP project
gcloud config list project

# Set your project ID (replace YOUR_PROJECT_ID with your actual project ID)
export PROJECT_ID=$(gcloud config get-value project)

# Verify you have at least one running VM instance
gcloud compute instances list

# If no instances exist, create a test VM for this lab
gcloud compute instances create monitoring-test-vm \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB \
  --tags=http-server
```

## Step-by-Step Instructions

### Step 1: Access Cloud Monitoring and Verify Metrics Collection

**Objective:** Navigate to Cloud Monitoring and verify that metrics are being collected from your Compute Engine instances.

**Instructions:**

1. In the Google Cloud Console, navigate to **Monitoring** from the navigation menu (or search for "Monitoring" in the search bar).

2. If this is your first time accessing Cloud Monitoring, you may see a welcome screen. Click **Continue** to proceed.

3. Wait for the Monitoring workspace to initialize (this may take 1-2 minutes).

4. From the left sidebar, click on **Metrics Explorer**.

5. In the Metrics Explorer, configure the following:
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **CPU utilization** (instance/cpu/utilization)
   - Click **Apply**

6. You should see a chart displaying CPU utilization for your VM instances over the last hour.

**Expected Output:**

You will see a line chart showing CPU utilization percentage (0-100%) over time for your VM instances. Even idle VMs typically show 1-5% baseline CPU usage.

**Verification:**

- [ ] Cloud Monitoring workspace is accessible
- [ ] Metrics Explorer displays CPU utilization data
- [ ] At least one VM instance appears in the metrics

---

### Step 2: Create a Custom Dashboard

**Objective:** Create a custom dashboard to organize and visualize multiple metrics in a centralized view.

**Instructions:**

1. From the Cloud Monitoring left sidebar, click on **Dashboards**.

2. Click the **+ CREATE DASHBOARD** button at the top.

3. In the "New Dashboard" dialog, enter the name: `VM Performance Dashboard`

4. Click **Confirm** to create the dashboard.

5. You will see an empty dashboard with an option to **+ ADD WIDGET**. Keep this page open for the next step.

**Expected Output:**

An empty dashboard named "VM Performance Dashboard" with options to add charts and widgets.

**Verification:**

- [ ] Dashboard is created and appears in the Dashboards list
- [ ] Dashboard editor is open and ready to add widgets
- [ ] Dashboard name is "VM Performance Dashboard"

---

### Step 3: Add CPU Utilization Chart

**Objective:** Add a line chart displaying CPU utilization for all VM instances.

**Instructions:**

1. On your empty dashboard, click **+ ADD WIDGET** and select **Line**.

2. In the widget configuration panel on the right, configure the following:
   - **Title**: Enter `CPU Utilization`
   - Click **+ ADD METRIC**

3. In the metric selector:
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **CPU utilization** (instance/cpu/utilization)
   - Click **Apply**

4. In the **Filter** section, you can optionally filter by specific instances or leave it to show all instances.

5. Under **Aggregation**:
   - **Aggregator**: Select **mean**
   - **Alignment period**: Select **1 minute**

6. Click **Apply** to add the chart to your dashboard.

**Expected Output:**

A line chart showing CPU utilization over time with the Y-axis representing percentage (0-100%) and X-axis showing time.

**Verification:**

- [ ] CPU Utilization chart is visible on the dashboard
- [ ] Chart displays data for your VM instances
- [ ] Time series line is visible (may be low for idle VMs)

---

### Step 4: Add Memory Utilization Chart

**Objective:** Add a chart to monitor memory usage on VM instances.

**Instructions:**

1. Click **+ ADD WIDGET** again and select **Line**.

2. Configure the widget:
   - **Title**: Enter `Memory Utilization`
   - Click **+ ADD METRIC**

3. In the metric selector:
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Memory utilization** (agent.googleapis.com/memory/percent_used)
   - Click **Apply**

4. If the metric is not available, it means the Cloud Monitoring agent is not installed. We'll address this in the next step.

5. For now, add an alternative metric:
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Disk read bytes** (instance/disk/read_bytes_count)
   - Click **Apply**

6. Configure aggregation:
   - **Aggregator**: Select **sum**
   - **Alignment period**: Select **1 minute**

7. Click **Apply** to add the chart.

**Expected Output:**

A line chart showing disk read activity. If memory metrics are unavailable, this chart will serve as a placeholder until the monitoring agent is installed.

**Verification:**

- [ ] Second chart is added to the dashboard
- [ ] Chart displays disk or memory metrics
- [ ] Chart is properly labeled

---

### Step 5: Install Cloud Monitoring Agent (Optional but Recommended)

**Objective:** Install the Cloud Monitoring agent on your VM to collect detailed system metrics including memory utilization.

**Instructions:**

1. SSH into your VM instance:

   ```bash
   gcloud compute ssh monitoring-test-vm --zone=us-central1-a
   ```

2. Once connected, install the Cloud Monitoring agent:

   ```bash
   curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
   sudo bash add-google-cloud-ops-agent-repo.sh --also-install
   ```

3. Verify the agent is running:

   ```bash
   sudo systemctl status google-cloud-ops-agent
   ```

4. You should see the service status as "active (running)".

5. Exit the SSH session:

   ```bash
   exit
   ```

6. Wait 2-3 minutes for metrics to start appearing in Cloud Monitoring.

7. Return to your dashboard and edit the Memory Utilization chart to use the correct metric:
   - Click the **pencil icon** on the chart to edit
   - Remove the disk metric and add **Memory utilization** (agent.googleapis.com/memory/percent_used)
   - Click **Apply**

**Expected Output:**

```
● google-cloud-ops-agent.service - Google Cloud Ops Agent
   Loaded: loaded (/lib/systemd/system/google-cloud-ops-agent.service; enabled)
   Active: active (running) since [timestamp]
```

**Verification:**

- [ ] Cloud Ops Agent is installed and running
- [ ] Memory utilization metrics appear in Metrics Explorer after 2-3 minutes
- [ ] Dashboard chart shows memory data

---

### Step 6: Add Network Traffic Chart

**Objective:** Add a stacked area chart to visualize network traffic (sent and received bytes).

**Instructions:**

1. Click **+ ADD WIDGET** and select **Stacked Area**.

2. Configure the widget:
   - **Title**: Enter `Network Traffic`

3. Add the first metric:
   - Click **+ ADD METRIC**
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Sent bytes** (instance/network/sent_bytes_count)
   - Click **Apply**

4. Add the second metric:
   - Click **+ ADD METRIC** again
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Received bytes** (instance/network/received_bytes_count)
   - Click **Apply**

5. Configure aggregation for both metrics:
   - **Aggregator**: Select **sum**
   - **Alignment period**: Select **1 minute**

6. Click **Apply** to add the chart.

**Expected Output:**

A stacked area chart showing two colored areas representing sent and received network traffic over time, measured in bytes.

**Verification:**

- [ ] Network Traffic chart is visible on the dashboard
- [ ] Both sent and received metrics are displayed
- [ ] Chart shows stacked visualization

---

### Step 7: Add Disk I/O Chart

**Objective:** Add a chart to monitor disk read and write operations.

**Instructions:**

1. Click **+ ADD WIDGET** and select **Line**.

2. Configure the widget:
   - **Title**: Enter `Disk I/O Operations`

3. Add disk read operations:
   - Click **+ ADD METRIC**
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Disk read operations** (instance/disk/read_ops_count)
   - Click **Apply**

4. Add disk write operations:
   - Click **+ ADD METRIC**
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **Disk write operations** (instance/disk/write_ops_count)
   - Click **Apply**

5. Configure aggregation:
   - **Aggregator**: Select **rate**
   - **Alignment period**: Select **1 minute**

6. Click **Apply** to add the chart.

7. Arrange your dashboard widgets by dragging them to organize the layout effectively.

8. Click **Save** at the top of the dashboard to save all changes.

**Expected Output:**

A line chart showing disk read and write operations per second. The dashboard now contains four charts providing comprehensive VM performance visibility.

**Verification:**

- [ ] Disk I/O chart is added to the dashboard
- [ ] Dashboard contains at least 4 charts
- [ ] Dashboard is saved successfully

---

### Step 8: Configure Email Notification Channel

**Objective:** Set up an email notification channel to receive alerts.

**Instructions:**

1. From the Cloud Monitoring left sidebar, click on **Alerting**.

2. Click on the **EDIT NOTIFICATION CHANNELS** button at the top.

3. Scroll down to the **Email** section.

4. Click **ADD NEW** under Email.

5. Configure the email channel:
   - **Display Name**: Enter `My Alert Email`
   - **Email Address**: Enter your email address
   - **Enabled**: Ensure the checkbox is checked

6. Click **Save**.

7. You will receive a verification email. Check your inbox and click the verification link.

8. Return to the Cloud Console and verify the email channel shows as "Verified".

**Expected Output:**

The notification channel appears in the Email section with a status of "Verified" and your email address displayed.

**Verification:**

- [ ] Email notification channel is created
- [ ] Verification email is received
- [ ] Channel status shows as "Verified"

---

### Step 9: Create CPU Utilization Alert Policy

**Objective:** Create an alerting policy that triggers when CPU utilization exceeds 80% for 5 minutes.

**Instructions:**

1. From the **Alerting** page, click **+ CREATE POLICY** at the top.

2. In the "Select a metric" step, configure:
   - Click **SELECT A METRIC**
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for and select **CPU utilization** (instance/cpu/utilization)
   - Click **Apply**

3. Click **NEXT** to proceed to "Configure alert trigger".

4. Configure the alert condition:
   - **Condition type**: Select **Threshold**
   - **Alert trigger**: Select **Any time series violates**
   - **Threshold position**: Select **Above threshold**
   - **Threshold value**: Enter `80`
   - **Advanced Options** → **Retest window**: Enter `5 minutes`

5. Click **NEXT** to proceed to "Configure notifications and finalize".

6. Configure notifications:
   - Under **Notification Channels**, click the dropdown
   - Select your email notification channel: **My Alert Email**
   - **Incident autoclose duration**: Leave default (7 days)

7. Configure alert details:
   - **Alert name**: Enter `High CPU Utilization Alert`
   - **Documentation**: Enter the following:

   ```
   CPU utilization has exceeded 80% for more than 5 minutes.
   
   Troubleshooting steps:
   1. Check running processes with 'top' or 'htop'
   2. Identify resource-intensive applications
   3. Consider scaling up the instance or optimizing workloads
   4. Review application logs for errors
   ```

8. Click **CREATE POLICY**.

**Expected Output:**

The alert policy is created and appears in the Alerting Policies list with status "Enabled". The policy will monitor all VM instances in the project.

**Verification:**

- [ ] Alert policy "High CPU Utilization Alert" is created
- [ ] Policy status shows as "Enabled"
- [ ] Email notification channel is configured
- [ ] Threshold is set to 80% for 5 minutes

---

### Step 10: Create Memory Utilization Alert Policy (Optional)

**Objective:** Create an additional alert policy for high memory utilization.

**Instructions:**

1. From the **Alerting** page, click **+ CREATE POLICY**.

2. Configure the metric:
   - Click **SELECT A METRIC**
   - **Resource type**: Select **VM Instance**
   - **Metric**: Search for **Memory utilization** (agent.googleapis.com/memory/percent_used)
   - If not available, skip this step (requires monitoring agent)
   - Click **Apply**

3. Click **NEXT**.

4. Configure the condition:
   - **Condition type**: Select **Threshold**
   - **Alert trigger**: Select **Any time series violates**
   - **Threshold position**: Select **Above threshold**
   - **Threshold value**: Enter `85`
   - **Retest window**: Enter `3 minutes`

5. Click **NEXT**.

6. Configure notifications:
   - Select your email notification channel
   - **Alert name**: Enter `High Memory Utilization Alert`
   - **Documentation**: Enter appropriate troubleshooting steps

7. Click **CREATE POLICY**.

**Expected Output:**

A second alert policy is created for memory monitoring.

**Verification:**

- [ ] Memory alert policy is created (if monitoring agent is installed)
- [ ] Both alert policies appear in the Alerting dashboard

---

### Step 11: Generate Synthetic Load to Trigger Alert

**Objective:** Create CPU load on your VM instance to trigger the alert and verify the alerting workflow.

**Instructions:**

1. SSH into your VM instance:

   ```bash
   gcloud compute ssh monitoring-test-vm --zone=us-central1-a
   ```

2. Install the stress-ng tool to generate CPU load:

   ```bash
   sudo apt-get update
   sudo apt-get install -y stress-ng
   ```

3. Generate CPU load for 10 minutes (use all CPU cores):

   ```bash
   stress-ng --cpu 0 --timeout 600s --metrics-brief
   ```

4. Open a new terminal window (keep the stress test running) and monitor the CPU in real-time:

   ```bash
   # In a new terminal
   gcloud compute ssh monitoring-test-vm --zone=us-central1-a
   
   # Run top to see CPU usage
   top
   ```

5. You should see CPU utilization at or near 100%.

6. Leave the stress test running and return to the Cloud Console.

7. Navigate to your **VM Performance Dashboard** and observe the CPU Utilization chart. You should see a spike in CPU usage.

8. Wait approximately 5-7 minutes for the alert condition to be met and the alert to trigger.

**Expected Output:**

```
stress-ng: info:  [12345] dispatching hogs: 2 cpu
stress-ng: info:  [12345] successful run completed in 600.00s
```

The top command will show stress-ng processes consuming 100% CPU.

**Verification:**

- [ ] stress-ng is running and generating CPU load
- [ ] CPU utilization shows 90-100% in the dashboard
- [ ] Alert is expected to trigger within 5-7 minutes

---

### Step 12: Verify Alert Notification and Incident

**Objective:** Confirm that the alert is triggered, an incident is created, and email notification is received.

**Instructions:**

1. While the stress test is running, navigate to **Monitoring** → **Alerting** in the Cloud Console.

2. After approximately 5-7 minutes, you should see an active incident appear under **Incidents**.

3. Click on the incident to view details:
   - Incident start time
   - Alert policy name
   - Current metric value
   - Documentation you added

4. Check your email inbox for the alert notification. The email will contain:
   - Subject: "Google Cloud Monitoring Alert: High CPU Utilization Alert"
   - Incident details
   - Link to view in Cloud Console
   - Documentation text

5. In the Cloud Console incident view, note the **Acknowledge** button.

6. Click **Acknowledge** to acknowledge the incident (this marks it as being handled).

7. Return to your SSH session and stop the stress test by pressing `Ctrl+C`.

8. Wait 5-10 minutes for the CPU to return to normal levels.

9. The incident should automatically resolve once CPU drops below the threshold.

10. Refresh the Alerting page to see the incident status change to "Closed".

**Expected Output:**

- Active incident appears in the Alerting dashboard
- Email notification is received with incident details
- After stopping the stress test, the incident automatically resolves

**Verification:**

- [ ] Incident appears in the Alerting dashboard
- [ ] Email notification is received
- [ ] Incident can be acknowledged
- [ ] Incident resolves automatically when CPU returns to normal

---

### Step 13: Review Incident History and Metrics

**Objective:** Analyze the incident history and review how metrics correlated with the alert.

**Instructions:**

1. Navigate to **Monitoring** → **Alerting**.

2. Click on the **Incidents** tab to view all incidents (both active and closed).

3. Click on your closed "High CPU Utilization Alert" incident.

4. Review the incident timeline:
   - When the incident was opened
   - When it was acknowledged (if you acknowledged it)
   - When it was closed
   - Duration of the incident

5. Scroll down to view the **Metric Threshold Chart** showing CPU utilization during the incident.

6. Navigate to your **VM Performance Dashboard**.

7. Adjust the time range selector to view the period when the stress test was running:
   - Click the time range dropdown (default is "1h")
   - Select **Custom** and choose a range covering your test

8. Observe how all metrics (CPU, network, disk I/O) changed during the high CPU period.

9. Take note of any correlations:
   - Did network traffic change?
   - Did disk I/O increase?
   - How did the system behave overall?

**Expected Output:**

The incident details page shows a complete timeline with the CPU utilization spike clearly visible in the chart. The dashboard shows the correlation between different metrics during the stress test.

**Verification:**

- [ ] Incident history is accessible and detailed
- [ ] Metric charts show the CPU spike
- [ ] Dashboard time range can be adjusted to review historical data
- [ ] Correlations between metrics are observable

---

### Step 14: Create Uptime Check (Optional Advanced Feature)

**Objective:** Configure an uptime check to monitor HTTP endpoint availability.

**Instructions:**

1. From the Cloud Monitoring left sidebar, click on **Uptime checks**.

2. Click **+ CREATE UPTIME CHECK**.

3. Configure the uptime check:
   - **Title**: Enter `VM HTTP Uptime Check`
   - **Protocol**: Select **HTTP**
   - **Resource Type**: Select **Instance**
   - **Applies To**: Select **Single** and choose your VM instance
   - **Path**: Enter `/` (or a specific path if you have a web server running)
   - **Check frequency**: Select **1 minute**

4. Click **CONTINUE**.

5. Configure the response validation (optional):
   - **Response Timeout**: Leave default (10 seconds)
   - You can add response content matching if desired

6. Click **CONTINUE**.

7. Configure alert and notification:
   - **Create an alert**: Check this box
   - **Name**: Enter `VM HTTP Down Alert`
   - Select your email notification channel

8. Click **CONTINUE** and then **CREATE**.

9. Note: If you don't have a web server running on your VM, this check will fail. To set up a simple web server:

   ```bash
   # SSH into your VM
   gcloud compute ssh monitoring-test-vm --zone=us-central1-a
   
   # Install and start Apache
   sudo apt-get install -y apache2
   sudo systemctl start apache2
   sudo systemctl enable apache2
   
   # Verify it's running
   curl localhost
   ```

**Expected Output:**

An uptime check is created and begins monitoring your VM's HTTP endpoint every minute. If the endpoint is accessible, the check shows as "Healthy" with green status.

**Verification:**

- [ ] Uptime check is created and active
- [ ] Check status shows as "Healthy" (if web server is running)
- [ ] Alert policy is created for the uptime check

---

## Validation & Testing

### Success Criteria

- [ ] Custom dashboard "VM Performance Dashboard" contains at least 4 charts (CPU, Memory/Disk, Network, Disk I/O)
- [ ] All charts display real-time metrics from VM instances
- [ ] Email notification channel is configured and verified
- [ ] CPU utilization alert policy is created with 80% threshold and 5-minute window
- [ ] Alert was successfully triggered by synthetic load
- [ ] Email notification was received for the triggered alert
- [ ] Incident appears in the Alerting dashboard and can be acknowledged
- [ ] Incident automatically resolved after CPU returned to normal
- [ ] Incident history is accessible and shows complete timeline

### Testing Procedure

1. Verify dashboard functionality:
   ```bash
   # Navigate to Monitoring → Dashboards → VM Performance Dashboard
   # Confirm all 4 charts are visible and displaying data
   ```
   **Expected Result:** Dashboard loads successfully with all charts showing recent metrics.

2. Test alert triggering:
   ```bash
   # SSH into VM and generate CPU load
   gcloud compute ssh monitoring-test-vm --zone=us-central1-a
   stress-ng --cpu 0 --timeout 300s
   ```
   **Expected Result:** CPU spikes to 90-100%, alert triggers within 5-7 minutes, email is received.

3. Verify notification delivery:
   ```bash
   # Check your email inbox for alerts from noreply@google.com
   # Subject should contain "Google Cloud Monitoring Alert"
   ```
   **Expected Result:** Email received with incident details and documentation.

4. Test incident resolution:
   ```bash
   # Stop the stress test (Ctrl+C)
   # Wait 5-10 minutes
   # Check Monitoring → Alerting → Incidents
   ```
   **Expected Result:** Incident status changes to "Closed" automatically.

5. Verify metrics are being collected continuously:
   ```bash
   # Use gcloud to query recent CPU metrics
   gcloud monitoring time-series list \
     --filter='metric.type="compute.googleapis.com/instance/cpu/utilization"' \
     --format=json \
     | head -20
   ```
   **Expected Result:** JSON output showing recent CPU utilization data points.

## Troubleshooting

### Issue 1: No Metrics Appearing in Dashboard

**Symptoms:**
- Charts show "No data available" or empty graphs
- Metrics Explorer doesn't display any VM instances
- Dashboard appears blank

**Cause:**
The Cloud Monitoring API may not be enabled, or the VM instances are not properly configured with monitoring permissions. Alternatively, there may be a delay in metric ingestion.

**Solution:**
```bash
# Verify the Cloud Monitoring API is enabled
gcloud services enable monitoring.googleapis.com

# Check if VM has monitoring scope
gcloud compute instances describe monitoring-test-vm \
  --zone=us-central1-a \
  --format="get(serviceAccounts[0].scopes)"

# Ensure the monitoring.write scope is present
# If not, stop the VM and update it
gcloud compute instances stop monitoring-test-vm --zone=us-central1-a

gcloud compute instances set-service-account monitoring-test-vm \
  --zone=us-central1-a \
  --scopes=https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/logging.write

gcloud compute instances start monitoring-test-vm --zone=us-central1-a

# Wait 2-3 minutes for metrics to appear
```

---

### Issue 2: Alert Not Triggering Despite High CPU

**Symptoms:**
- CPU utilization shows 90-100% in dashboard
- No incident appears in the Alerting dashboard
- No email notification received
- Alert policy shows as "Enabled"

**Cause:**
The alert condition may not be met due to insufficient duration, the retest window may not have elapsed, or the notification channel is not properly verified.

**Solution:**
```bash
# Verify the alert policy configuration
gcloud alpha monitoring policies list --format=json

# Check notification channel status
gcloud alpha monitoring channels list

# Ensure email is verified - check for "verified: true"

# Verify alert condition duration
# The CPU must remain above threshold for the full retest window (5 minutes)
# Ensure stress-ng runs for at least 10 minutes

# Manually test notification channel
gcloud alpha monitoring channels test [CHANNEL_ID]

# If needed, recreate the alert policy with correct settings
# Navigate to Alerting → Select policy → Edit → Verify all settings
```

---

### Issue 3: Memory Metrics Not Available

**Symptoms:**
- Memory utilization metric (agent.googleapis.com/memory/percent_used) doesn't appear in Metrics Explorer
- Memory chart shows "No data available"
- Only basic instance metrics are visible

**Cause:**
The Cloud Monitoring agent (Ops Agent) is not installed on the VM instance. Basic Compute Engine metrics don't include memory utilization.

**Solution:**
```bash
# SSH into the VM
gcloud compute ssh monitoring-test-vm --zone=us-central1-a

# Install the Ops Agent
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Verify installation
sudo systemctl status google-cloud-ops-agent

# Check agent logs if there are issues
sudo journalctl -u google-cloud-ops-agent -f

# Wait 3-5 minutes for metrics to appear
# Exit SSH and refresh the Metrics Explorer

# Alternative: Use disk metrics as a substitute
# Select instance/disk/read_bytes_count or instance/disk/write_bytes_count
```

---

### Issue 4: Email Verification Not Received

**Symptoms:**
- Email notification channel shows "Unverified" status
- No verification email in inbox
- Cannot complete notification channel setup

**Cause:**
Email may be in spam folder, email address may be incorrect, or there may be a delay in email delivery.

**Solution:**
```bash
# Check spam/junk folder for email from noreply@google.com

# Resend verification email from Cloud Console
# Navigate to Alerting → Edit Notification Channels
# Find your email channel and click "Resend verification"

# If still not received, delete and recreate the channel
gcloud alpha monitoring channels delete [CHANNEL_ID]

# Create new channel via Cloud Console with correct email
# Alternatively, use a different email address

# Verify email channel programmatically
gcloud alpha monitoring channels list --format="table(name,displayName,verificationStatus)"

# If verification continues to fail, use Cloud Console UI instead of CLI
```

---

### Issue 5: Dashboard Not Saving Changes

**Symptoms:**
- Charts are added but disappear after refresh
- "Save" button doesn't respond
- Error message appears when saving

**Cause:**
Browser cache issues, insufficient permissions, or API quota limits may prevent dashboard updates.

**Solution:**
```bash
# Verify you have monitoring.dashboards.create permission
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:YOUR_EMAIL"

# Clear browser cache and cookies for console.cloud.google.com
# Try using an incognito/private browser window

# Check if there are API quota issues
gcloud logging read "resource.type=api AND protoPayload.methodName=monitoring.dashboards.create" \
  --limit 10 --format=json

# If permission issues, add Monitoring Editor role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/monitoring.editor"

# Try creating dashboard via gcloud CLI instead
# Export existing dashboard as JSON template
gcloud monitoring dashboards list --format=json > dashboard_backup.json

# Recreate if necessary
```

---

### Issue 6: Incident Not Resolving Automatically

**Symptoms:**
- CPU has returned to normal levels
- Alert policy shows CPU below threshold
- Incident remains in "Open" state
- No closure notification received

**Cause:**
Autoclose duration may be set too long, or there may be intermittent spikes keeping the incident open.

**Solution:**
```bash
# Check the alert policy's autoclose duration
gcloud alpha monitoring policies describe [POLICY_NAME] --format=json

# Look for "notificationChannels" and "documentation" sections

# Manually close the incident via Cloud Console
# Navigate to Monitoring → Alerting → Incidents
# Click on the incident → Click "Close Incident"

# Verify CPU is consistently below threshold
# Check dashboard for at least 10 minutes of data

# If issues persist, edit the alert policy
# Reduce the autoclose duration to 30 minutes or 1 hour

# Check for flapping (CPU crossing threshold repeatedly)
# Adjust threshold value or retest window to reduce false positives
```

---

## Cleanup

```bash
# Stop the stress test if still running (Ctrl+C in SSH session)

# Delete the alert policies to avoid unnecessary notifications
gcloud alpha monitoring policies list --format="value(name)"
gcloud alpha monitoring policies delete [POLICY_NAME]

# Delete the custom dashboard
gcloud monitoring dashboards list --format="value(name)"
gcloud monitoring dashboards delete [DASHBOARD_NAME]

# Delete notification channels
gcloud alpha monitoring channels list --format="value(name)"
gcloud alpha monitoring channels delete [CHANNEL_ID]

# Delete uptime checks if created
gcloud monitoring uptime list --format="value(name)"
gcloud monitoring uptime delete [UPTIME_CHECK_NAME]

# Uninstall the Ops Agent if desired (optional)
gcloud compute ssh monitoring-test-vm --zone=us-central1-a
sudo apt-get remove google-cloud-ops-agent -y
exit

# Delete the test VM if it was created only for this lab
gcloud compute instances delete monitoring-test-vm \
  --zone=us-central1-a \
  --quiet

# Verify all resources are deleted
gcloud alpha monitoring policies list
gcloud monitoring dashboards list
gcloud alpha monitoring channels list
```

> ⚠️ **Warning:** Deleting alert policies and dashboards is permanent and cannot be undone. Export any configurations you want to keep before deletion. If you're continuing to other labs, you may want to keep the VM instance and monitoring setup for future use. Cloud Monitoring itself does not incur charges, but keeping VM instances running will consume credits.

## Summary

### What You Accomplished

- Created a custom Cloud Monitoring dashboard with multiple visualization types (line charts, stacked area charts)
- Configured monitoring for key VM performance metrics including CPU, memory, network traffic, and disk I/O
- Installed the Cloud Ops Agent to collect detailed system-level metrics
- Set up email notification channels with proper verification
- Created alerting policies with threshold-based conditions and appropriate retest windows
- Generated synthetic CPU load to trigger alerts and validate the alerting workflow
- Received and acknowledged alert notifications via email
- Reviewed incident history and analyzed metric correlations during performance events
- Gained hands-on experience with Google Cloud's observability and incident management capabilities

### Key Takeaways

- Cloud Monitoring provides comprehensive observability for GCP resources without requiring agent installation for basic metrics
- The Cloud Ops Agent is necessary for detailed system metrics like memory utilization and application-specific monitoring
- Effective alerting requires careful threshold selection and appropriate retest windows to avoid alert fatigue
- Custom dashboards enable centralized visibility into system performance and help identify trends and anomalies
- Incident management workflows in Cloud Monitoring support acknowledgment, documentation, and automatic resolution
- Proper notification channel configuration is critical for ensuring alerts reach the right people at the right time
- Monitoring data retention in Cloud Monitoring allows for historical analysis and post-incident reviews
- Uptime checks provide external monitoring for application availability beyond infrastructure metrics

### Next Steps

- Explore advanced monitoring features like log-based metrics and custom metrics from applications
- Implement multi-condition alert policies that combine multiple metrics
- Configure additional notification channels such as Slack, PagerDuty, or webhooks for integration with incident management systems
- Create SLO (Service Level Objective) monitoring for production applications
- Set up monitoring for other GCP services like Cloud SQL, Cloud Storage, and Cloud Functions
- Implement distributed tracing using Cloud Trace for microservices architectures
- Learn Monitoring Query Language (MQL) for advanced metric queries and transformations
- Design monitoring strategies for production environments including on-call rotations and escalation policies

## Additional Resources

- [Google Cloud Monitoring Documentation](https://cloud.google.com/monitoring/docs) - Official documentation covering all monitoring features
- [Cloud Ops Agent Installation Guide](https://cloud.google.com/stackdriver/docs/solutions/agents/ops-agent) - Detailed instructions for agent deployment and configuration
- [Monitoring Query Language (MQL) Reference](https://cloud.google.com/monitoring/mql) - Learn advanced metric queries and transformations
- [Alerting Best Practices](https://cloud.google.com/monitoring/alerts/best-practices) - Google's recommendations for effective alerting
- [Dashboard Samples and Templates](https://cloud.google.com/monitoring/dashboards/samples) - Pre-built dashboard configurations for common use cases
- [SLO Monitoring Guide](https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring) - Implement service level objective tracking
- [Cloud Monitoring Pricing](https://cloud.google.com/stackdriver/pricing) - Understanding costs and free tier limits
- [Notification Channel Integration Guide](https://cloud.google.com/monitoring/support/notification-options) - Configure various notification destinations
- [Site Reliability Engineering Book](https://sre.google/books/) - Google's SRE practices including monitoring and alerting philosophies

---

# Lab 04-08-01: Práctica 12: Analizar logs de error de app en GKE y exportarlos a BigQuery

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

In this lab, you will deploy a containerized application to Google Kubernetes Engine (GKE) that generates error logs, filter and analyze these logs using Cloud Logging's Log Explorer, and configure a log sink to export error logs to BigQuery for SQL-based analysis. This practice demonstrates how to implement centralized logging and observability for containerized workloads, enabling data-driven incident response and application health monitoring.

By exporting logs to BigQuery, you can perform advanced analytics, create dashboards, and derive KPIs such as error rates by service, namespace, or severity—critical capabilities for production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Filter and analyze container logs in GKE using Log Explorer with resource-specific queries
- [ ] Configure a Log Router sink to export logs to a time-partitioned BigQuery dataset
- [ ] Execute SQL queries in BigQuery to extract error KPIs such as error count by time interval, top error messages, and distribution by namespace

## Prerequisites

### Required Knowledge

- Basic understanding of Kubernetes concepts (pods, deployments, namespaces)
- Familiarity with Google Cloud Console navigation
- Basic SQL query syntax
- Understanding of log severity levels (INFO, WARNING, ERROR, etc.)

### Required Access

- Google Cloud Project with billing enabled
- Permissions: `logging.sinks.create`, `logging.sinks.update`, `bigquery.datasets.create`, `bigquery.tables.create`, `container.clusters.get`, `container.pods.create`
- Roles: Logging Admin, BigQuery Data Owner (or BigQuery Admin), Kubernetes Engine Developer

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 4+ vCPU |
| RAM | 8+ GB |
| Network | ≥10 Mbps stable Internet connection |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage GCP resources via CLI |
| kubectl | 1.29.x | Manage GKE cluster and workloads |
| bq CLI | v2 | Query and manage BigQuery datasets |
| Cloud Logging API | v2 | Explore and route logs |
| BigQuery API | v2 | Create datasets and execute queries |
| Kubernetes Engine API | v1 | Manage GKE cluster |

### Initial Setup

```bash
# Set your project ID
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE=us-central1-a
export CLUSTER_NAME=logs-demo-cluster

# Enable required APIs
gcloud services enable container.googleapis.com \
  logging.googleapis.com \
  bigquery.googleapis.com \
  compute.googleapis.com

# Verify APIs are enabled
gcloud services list --enabled | grep -E 'container|logging|bigquery'
```

## Step-by-Step Instructions

### Step 1: Create a GKE Cluster

**Objective:** Provision a small GKE cluster to host the application that will generate error logs.

**Instructions:**

1. Create a GKE cluster with minimal resources for cost efficiency:

   ```bash
   gcloud container clusters create $CLUSTER_NAME \
     --zone=$ZONE \
     --num-nodes=2 \
     --machine-type=e2-medium \
     --enable-cloud-logging \
     --enable-cloud-monitoring \
     --logging=SYSTEM,WORKLOAD
   ```

2. Configure kubectl to use the new cluster:

   ```bash
   gcloud container clusters get-credentials $CLUSTER_NAME --zone=$ZONE
   ```

3. Verify cluster connectivity:

   ```bash
   kubectl cluster-info
   kubectl get nodes
   ```

**Expected Output:**

```
Kubernetes control plane is running at https://...
2 nodes in Ready state
```

**Verification:**

- Confirm that `kubectl get nodes` shows 2 nodes with STATUS "Ready"
- Verify cluster appears in GKE console: Navigation menu → Kubernetes Engine → Clusters

---

### Step 2: Deploy an Application That Generates Error Logs

**Objective:** Deploy a sample application that intentionally generates ERROR-level logs for analysis.

**Instructions:**

1. Create a namespace for the demo application:

   ```bash
   kubectl create namespace error-demo
   ```

2. Create a deployment manifest that runs a container generating error logs:

   ```bash
   cat <<EOF > error-logger-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: error-logger
     namespace: error-demo
     labels:
       app: error-logger
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: error-logger
     template:
       metadata:
         labels:
           app: error-logger
       spec:
         containers:
         - name: logger
           image: busybox:1.36
           command: ["/bin/sh"]
           args:
             - -c
             - |
               while true; do
                 echo "\$(date) INFO: Application running normally"
                 sleep 10
                 echo "\$(date) ERROR: Database connection failed - code 1045" >&2
                 sleep 5
                 echo "\$(date) ERROR: API timeout exceeded - endpoint /v1/users" >&2
                 sleep 15
                 echo "\$(date) ERROR: Authentication failed for user admin" >&2
                 sleep 10
               done
   EOF
   ```

3. Apply the deployment:

   ```bash
   kubectl apply -f error-logger-deployment.yaml
   ```

4. Verify pods are running:

   ```bash
   kubectl get pods -n error-demo
   ```

**Expected Output:**

```
NAME                            READY   STATUS    RESTARTS   AGE
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Verification:**

- Both pods show STATUS "Running"
- Check logs to confirm error generation:
  ```bash
  kubectl logs -n error-demo -l app=error-logger --tail=20
  ```
- You should see both INFO and ERROR messages in the output

---

### Step 3: Filter and Analyze Logs in Log Explorer

**Objective:** Use Cloud Logging's Log Explorer to filter and view only ERROR-level logs from the GKE containers.

**Instructions:**

1. Navigate to Log Explorer:
   - Open Google Cloud Console
   - Go to Navigation menu → Logging → Logs Explorer

2. Enter the following filter query in the query editor:

   ```
   resource.type="k8s_container"
   resource.labels.namespace_name="error-demo"
   severity>=ERROR
   ```

3. Set the time range to "Last 1 hour" in the time selector

4. Click "Run query" to execute the filter

5. Examine the log entries and note the structure:
   - Expand a log entry to view `jsonPayload`, `resource`, `labels`, and `timestamp`
   - Note the `resource.labels.container_name` and `resource.labels.pod_name`

6. Refine the filter to find specific error patterns:

   ```
   resource.type="k8s_container"
   resource.labels.namespace_name="error-demo"
   severity>=ERROR
   textPayload=~"Database connection failed"
   ```

**Expected Output:**

You should see log entries similar to:

```
2024-01-15 10:23:45 ERROR: Database connection failed - code 1045
2024-01-15 10:23:50 ERROR: API timeout exceeded - endpoint /v1/users
```

**Verification:**

- Log entries display with red ERROR severity indicator
- Only logs from the `error-demo` namespace appear
- Each log entry shows the correct pod name and container name

---

### Step 4: Create a BigQuery Dataset for Log Storage

**Objective:** Provision a time-partitioned BigQuery dataset to store exported logs efficiently.

**Instructions:**

1. Set environment variables for the dataset:

   ```bash
   export DATASET_ID=gke_error_logs
   export DATASET_LOCATION=US
   ```

2. Create the BigQuery dataset:

   ```bash
   bq mk --dataset \
     --location=$DATASET_LOCATION \
     --description="GKE error logs exported from Cloud Logging" \
     $PROJECT_ID:$DATASET_ID
   ```

3. Verify dataset creation:

   ```bash
   bq ls --project_id=$PROJECT_ID
   ```

4. View dataset details:

   ```bash
   bq show $PROJECT_ID:$DATASET_ID
   ```

**Expected Output:**

```
Dataset 'project-id:gke_error_logs'
  Last modified                  Description
 ----------------- ----------------------------------------
  15 Jan 10:30:00   GKE error logs exported from Cloud Logging
```

**Verification:**

- Dataset appears in BigQuery console: Navigation menu → BigQuery → SQL workspace
- Dataset location is "US"
- No tables exist yet (will be auto-created by Log Router)

---

### Step 5: Configure Log Router Sink to BigQuery

**Objective:** Create a log sink that routes ERROR-level logs from GKE to the BigQuery dataset.

**Instructions:**

1. Define the sink name and filter:

   ```bash
   export SINK_NAME=gke-errors-to-bigquery
   export SINK_FILTER='resource.type="k8s_container"
   resource.labels.namespace_name="error-demo"
   severity>=ERROR'
   ```

2. Create the log sink:

   ```bash
   gcloud logging sinks create $SINK_NAME \
     bigquery.googleapis.com/projects/$PROJECT_ID/datasets/$DATASET_ID \
     --log-filter="$SINK_FILTER"
   ```

3. Capture the service account created for the sink:

   ```bash
   export SINK_SERVICE_ACCOUNT=$(gcloud logging sinks describe $SINK_NAME --format='value(writerIdentity)')
   echo "Sink service account: $SINK_SERVICE_ACCOUNT"
   ```

4. Grant the sink service account permission to write to BigQuery:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member="$SINK_SERVICE_ACCOUNT" \
     --role="roles/bigquery.dataEditor"
   ```

5. Verify the sink configuration:

   ```bash
   gcloud logging sinks describe $SINK_NAME
   ```

**Expected Output:**

```
createTime: '2024-01-15T10:35:00.000Z'
destination: bigquery.googleapis.com/projects/PROJECT_ID/datasets/gke_error_logs
filter: resource.type="k8s_container" resource.labels.namespace_name="error-demo" severity>=ERROR
name: gke-errors-to-bigquery
writerIdentity: serviceAccount:...
```

**Verification:**

- Sink appears in Logging console: Navigation menu → Logging → Log Router
- Destination points to your BigQuery dataset
- Writer identity is a service account with format `serviceAccount:...`
- IAM binding confirmed:
  ```bash
  gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --filter="bindings.members:$SINK_SERVICE_ACCOUNT"
  ```

---

### Step 6: Generate Additional Error Events

**Objective:** Produce more error log entries to ensure sufficient data flows to BigQuery for analysis.

**Instructions:**

1. Scale the deployment to generate more logs:

   ```bash
   kubectl scale deployment error-logger -n error-demo --replicas=4
   ```

2. Wait for new pods to start:

   ```bash
   kubectl wait --for=condition=ready pod -l app=error-logger -n error-demo --timeout=120s
   ```

3. Verify all pods are running:

   ```bash
   kubectl get pods -n error-demo
   ```

4. Optionally, create a temporary pod that generates burst errors:

   ```bash
   kubectl run error-burst --image=busybox:1.36 -n error-demo --restart=Never -- sh -c "for i in \$(seq 1 50); do echo \"ERROR: Burst error event \$i\" >&2; sleep 1; done"
   ```

5. Wait 3-5 minutes for logs to be ingested and exported to BigQuery

**Expected Output:**

```
NAME                            READY   STATUS    RESTARTS   AGE
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
error-logger-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Verification:**

- Four error-logger pods are running
- Logs continue to appear in Log Explorer with the filter applied
- Wait at least 3 minutes before proceeding to allow log export to complete

---

### Step 7: Verify Log Data in BigQuery

**Objective:** Confirm that error logs have been successfully exported to BigQuery and identify the auto-created table.

**Instructions:**

1. List tables in the BigQuery dataset:

   ```bash
   bq ls $PROJECT_ID:$DATASET_ID
   ```

2. The table name will be auto-generated with format similar to `k8s_container_YYYYMMDD`. Identify the table:

   ```bash
   export TABLE_NAME=$(bq ls $PROJECT_ID:$DATASET_ID | grep k8s_container | awk '{print $1}' | head -n 1)
   echo "Table name: $TABLE_NAME"
   ```

3. View the table schema:

   ```bash
   bq show --schema $PROJECT_ID:$DATASET_ID.$TABLE_NAME
   ```

4. Check row count to confirm data ingestion:

   ```bash
   bq query --use_legacy_sql=false "SELECT COUNT(*) as row_count FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`"
   ```

**Expected Output:**

```
+----------+
| row_count|
+----------+
|       87 |
+----------+
```

**Verification:**

- Table exists in the dataset (visible in BigQuery console)
- Row count is greater than 0
- Schema includes fields like `timestamp`, `severity`, `textPayload`, `resource`, `labels`

---

### Step 8: Execute SQL Queries for Error Analysis

**Objective:** Use SQL queries in BigQuery to derive KPIs and insights from the exported error logs.

**Instructions:**

1. Query 1: Count errors per 5-minute interval:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT
     TIMESTAMP_TRUNC(timestamp, MINUTE, 'UTC') AS time_bucket,
     COUNT(*) AS error_count
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   WHERE severity = 'ERROR'
   GROUP BY time_bucket
   ORDER BY time_bucket DESC
   LIMIT 10
   "
   ```

2. Query 2: Top 5 most frequent error messages:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT
     textPayload AS error_message,
     COUNT(*) AS occurrence_count
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   WHERE severity = 'ERROR'
   GROUP BY error_message
   ORDER BY occurrence_count DESC
   LIMIT 5
   "
   ```

3. Query 3: Error distribution by pod name:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT
     resource.labels.pod_name AS pod_name,
     COUNT(*) AS error_count
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   WHERE severity = 'ERROR'
   GROUP BY pod_name
   ORDER BY error_count DESC
   "
   ```

4. Query 4: Error distribution by namespace and container:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT
     resource.labels.namespace_name AS namespace,
     resource.labels.container_name AS container,
     COUNT(*) AS error_count
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   WHERE severity = 'ERROR'
   GROUP BY namespace, container
   ORDER BY error_count DESC
   "
   ```

5. Save query results for documentation:

   ```bash
   bq query --use_legacy_sql=false --format=prettyjson "
   SELECT
     TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
     severity,
     COUNT(*) AS log_count
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   GROUP BY hour, severity
   ORDER BY hour DESC, severity
   LIMIT 20
   " > query_results.json
   ```

**Expected Output:**

Query 1 output example:
```
+---------------------+-------------+
|     time_bucket     | error_count |
+---------------------+-------------+
| 2024-01-15 10:40:00 |          12 |
| 2024-01-15 10:35:00 |          15 |
| 2024-01-15 10:30:00 |           8 |
+---------------------+-------------+
```

Query 2 output example:
```
+----------------------------------------------------------+------------------+
|                      error_message                       | occurrence_count |
+----------------------------------------------------------+------------------+
| ERROR: Database connection failed - code 1045            |               28 |
| ERROR: API timeout exceeded - endpoint /v1/users         |               25 |
| ERROR: Authentication failed for user admin              |               20 |
+----------------------------------------------------------+------------------+
```

**Verification:**

- All queries execute without errors
- Results show meaningful data grouped by time, message, pod, and namespace
- Error counts align with the number of pods and time elapsed since deployment

---

### Step 9: Document Filter, Schema, and Query Results

**Objective:** Capture configuration details and query outputs for reporting and future reference.

**Instructions:**

1. Document the Log Explorer filter used:

   ```bash
   cat <<EOF > lab_documentation.md
   # GKE Error Logs Analysis - Lab Documentation

   ## Log Explorer Filter
   \`\`\`
   resource.type="k8s_container"
   resource.labels.namespace_name="error-demo"
   severity>=ERROR
   \`\`\`

   ## BigQuery Dataset
   - **Project ID**: $PROJECT_ID
   - **Dataset ID**: $DATASET_ID
   - **Location**: $DATASET_LOCATION
   - **Table Name**: $TABLE_NAME

   ## Log Router Sink
   - **Sink Name**: $SINK_NAME
   - **Destination**: bigquery.googleapis.com/projects/$PROJECT_ID/datasets/$DATASET_ID
   - **Service Account**: $SINK_SERVICE_ACCOUNT

   ## Key Findings
   EOF
   ```

2. Export the table schema to a file:

   ```bash
   bq show --schema --format=prettyjson $PROJECT_ID:$DATASET_ID.$TABLE_NAME > bigquery_schema.json
   ```

3. Create a summary of query results:

   ```bash
   echo "## Query Results Summary" >> lab_documentation.md
   echo "" >> lab_documentation.md
   echo "### Total Error Count" >> lab_documentation.md
   bq query --use_legacy_sql=false --format=csv "SELECT COUNT(*) as total_errors FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\` WHERE severity='ERROR'" >> lab_documentation.md
   echo "" >> lab_documentation.md
   ```

4. View the documentation file:

   ```bash
   cat lab_documentation.md
   ```

**Expected Output:**

A markdown file containing:
- Log filter configuration
- BigQuery dataset and table details
- Sink configuration
- Query results summary

**Verification:**

- `lab_documentation.md` file exists and contains structured information
- `bigquery_schema.json` file contains the complete table schema
- Files can be downloaded or copied for submission/review

---

## Validation & Testing

### Success Criteria

- [ ] GKE cluster is running with 2+ nodes
- [ ] Application pods in `error-demo` namespace are generating ERROR logs
- [ ] Log Explorer filter successfully displays only ERROR-level logs from GKE containers
- [ ] BigQuery dataset `gke_error_logs` exists with at least one table
- [ ] Log Router sink is active and exporting logs to BigQuery
- [ ] BigQuery table contains rows with log data (row count > 0)
- [ ] All four SQL queries execute successfully and return meaningful results
- [ ] Documentation files capture filter, schema, and query outputs

### Testing Procedure

1. Verify end-to-end log flow:

   ```bash
   # Generate a unique error
   kubectl run test-error --image=busybox:1.36 -n error-demo --restart=Never -- sh -c "echo 'ERROR: UNIQUE_TEST_ERROR_12345' >&2; sleep 10"
   ```

2. Wait 2-3 minutes, then search for the unique error in BigQuery:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT timestamp, textPayload
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   WHERE textPayload LIKE '%UNIQUE_TEST_ERROR_12345%'
   ORDER BY timestamp DESC
   LIMIT 1
   "
   ```

   **Expected Result:** The unique error message appears in query results with a recent timestamp

3. Verify sink permissions:

   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:$SINK_SERVICE_ACCOUNT AND bindings.role:roles/bigquery.dataEditor"
   ```

   **Expected Result:** Output shows the sink service account has `roles/bigquery.dataEditor`

4. Confirm logs are still flowing:

   ```bash
   bq query --use_legacy_sql=false "
   SELECT MAX(timestamp) as latest_log
   FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
   "
   ```

   **Expected Result:** Latest log timestamp is within the last 5-10 minutes

---

## Troubleshooting

### Issue 1: No Logs Appearing in BigQuery

**Symptoms:**
- BigQuery dataset exists but contains no tables
- Table exists but has 0 rows
- `bq ls` shows empty dataset after 5+ minutes

**Cause:**
The log sink service account lacks permissions to write to the BigQuery dataset, or the sink filter is too restrictive and matches no logs.

**Solution:**

1. Verify the sink service account has correct permissions:

   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:$SINK_SERVICE_ACCOUNT"
   ```

2. If permissions are missing, re-apply them:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member="$SINK_SERVICE_ACCOUNT" \
     --role="roles/bigquery.dataEditor"
   ```

3. Verify logs match the sink filter in Log Explorer:

   ```bash
   gcloud logging read "$SINK_FILTER" --limit=10 --format=json
   ```

4. If no logs match, adjust the filter or verify pods are generating errors:

   ```bash
   kubectl logs -n error-demo -l app=error-logger --tail=50 | grep ERROR
   ```

5. Force log generation and wait 3-5 minutes:

   ```bash
   kubectl delete pod -n error-demo -l app=error-logger
   kubectl wait --for=condition=ready pod -l app=error-logger -n error-demo --timeout=120s
   ```

---

### Issue 2: Query Fails with "Table Not Found" Error

**Symptoms:**
- BigQuery query returns: `Not found: Table PROJECT_ID:DATASET_ID.TABLE_NAME`
- Table name is not auto-populated in `$TABLE_NAME` variable

**Cause:**
The table has not been created yet by the log sink, or the table name was not correctly captured.

**Solution:**

1. List tables manually and verify existence:

   ```bash
   bq ls $PROJECT_ID:$DATASET_ID
   ```

2. If no table exists, wait 3-5 more minutes for the first log batch to create the table automatically

3. If table exists but variable is empty, manually set the table name:

   ```bash
   export TABLE_NAME=k8s_container_20240115
   echo "Using table: $TABLE_NAME"
   ```

4. Verify table name format (should start with `k8s_container_`):

   ```bash
   bq ls $PROJECT_ID:$DATASET_ID | grep k8s_container
   ```

5. If multiple tables exist (multi-day export), query the most recent:

   ```bash
   export TABLE_NAME=$(bq ls $PROJECT_ID:$DATASET_ID | grep k8s_container | tail -n 1 | awk '{print $1}')
   bq query --use_legacy_sql=false "SELECT COUNT(*) FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`"
   ```

---

### Issue 3: Pods Not Generating Logs or Crashing

**Symptoms:**
- `kubectl get pods -n error-demo` shows CrashLoopBackOff or Error status
- No logs appear in Log Explorer even without severity filter
- `kubectl logs` returns empty output

**Cause:**
The busybox container command may have syntax errors, or the image failed to pull.

**Solution:**

1. Check pod status and events:

   ```bash
   kubectl describe pod -n error-demo -l app=error-logger
   ```

2. If ImagePullBackOff, verify network connectivity or use a different registry:

   ```bash
   kubectl set image deployment/error-logger logger=busybox:latest -n error-demo
   ```

3. If CrashLoopBackOff, check logs for errors:

   ```bash
   kubectl logs -n error-demo -l app=error-logger --previous
   ```

4. Redeploy with a simpler logging command:

   ```bash
   kubectl delete deployment error-logger -n error-demo
   cat <<EOF | kubectl apply -f -
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: error-logger
     namespace: error-demo
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: error-logger
     template:
       metadata:
         labels:
           app: error-logger
       spec:
         containers:
         - name: logger
           image: busybox:1.36
           command: ["sh", "-c"]
           args:
             - while true; do echo "ERROR: Test error message" >&2; sleep 15; done
   EOF
   ```

5. Verify new pods are running and generating logs:

   ```bash
   kubectl wait --for=condition=ready pod -l app=error-logger -n error-demo --timeout=120s
   kubectl logs -n error-demo -l app=error-logger --tail=10
   ```

---

### Issue 4: Log Explorer Filter Returns No Results

**Symptoms:**
- Log Explorer shows "No log entries found" message
- Filter syntax appears correct

**Cause:**
Time range is too narrow, namespace name is incorrect, or logs have not yet been ingested.

**Solution:**

1. Expand the time range to "Last 1 hour" or "Last 6 hours"

2. Verify the namespace name is correct:

   ```bash
   kubectl get namespaces | grep error-demo
   ```

3. Test with a broader filter first:

   ```
   resource.type="k8s_container"
   ```

4. Gradually add filters to narrow results:

   ```
   resource.type="k8s_container"
   resource.labels.namespace_name="error-demo"
   ```

5. Check that Cloud Logging is enabled for the cluster:

   ```bash
   gcloud container clusters describe $CLUSTER_NAME --zone=$ZONE --format="value(loggingService)"
   ```

   Expected output: `logging.googleapis.com/kubernetes`

6. If logging is disabled, update the cluster:

   ```bash
   gcloud container clusters update $CLUSTER_NAME --zone=$ZONE --enable-cloud-logging --logging=SYSTEM,WORKLOAD
   ```

---

## Cleanup

**Objective:** Remove all resources created during the lab to avoid ongoing charges.

```bash
# Delete the Kubernetes namespace and all resources within it
kubectl delete namespace error-demo

# Delete the log sink
gcloud logging sinks delete $SINK_NAME --quiet

# Delete the BigQuery dataset and all tables
bq rm -r -f -d $PROJECT_ID:$DATASET_ID

# Delete the GKE cluster
gcloud container clusters delete $CLUSTER_NAME --zone=$ZONE --quiet

# Remove IAM policy binding (optional, as sink deletion removes the service account)
# gcloud projects remove-iam-policy-binding $PROJECT_ID \
#   --member="$SINK_SERVICE_ACCOUNT" \
#   --role="roles/bigquery.dataEditor"

# Clean up local files
rm -f error-logger-deployment.yaml lab_documentation.md bigquery_schema.json query_results.json
```

> ⚠️ **Warning:** Deleting the GKE cluster will permanently remove all workloads and configurations. Ensure you have documented any important findings before cleanup. BigQuery dataset deletion is immediate and cannot be undone—export any query results you need to retain.

**Verification:**

```bash
# Confirm cluster is deleted
gcloud container clusters list --filter="name=$CLUSTER_NAME"

# Confirm dataset is deleted
bq ls --project_id=$PROJECT_ID | grep $DATASET_ID

# Confirm sink is deleted
gcloud logging sinks list | grep $SINK_NAME
```

All commands should return empty results.

---

## Summary

### What You Accomplished

- Deployed a GKE cluster and containerized application that generates structured error logs
- Filtered and analyzed container logs in Cloud Logging using resource-specific queries and severity filters
- Configured a Log Router sink to export logs to a time-partitioned BigQuery dataset with appropriate IAM permissions
- Executed SQL queries in BigQuery to derive error KPIs including time-series counts, top error messages, and distribution by pod and namespace
- Documented the complete logging pipeline configuration and query results

### Key Takeaways

- **Centralized Logging**: Cloud Logging automatically collects logs from GKE workloads when `--enable-cloud-logging` is enabled, providing a unified view across all containers
- **Log Routing**: Log Router sinks enable real-time export of filtered logs to BigQuery, Cloud Storage, or Pub/Sub for long-term retention and advanced analysis
- **IAM for Sinks**: Log sinks use dedicated service accounts that require explicit permissions (`bigquery.dataEditor`) on destination resources
- **BigQuery for Log Analytics**: Time-partitioned tables in BigQuery enable cost-effective storage and fast SQL queries over large volumes of log data
- **Severity Filtering**: Using `severity>=ERROR` focuses analysis on actionable events, reducing noise and query costs
- **Resource Labels**: GKE logs include rich metadata (namespace, pod, container) that enables granular filtering and grouping in SQL queries

### Next Steps

- Explore creating **scheduled queries** in BigQuery to automatically generate daily error reports
- Set up **Cloud Monitoring alerts** based on log-based metrics (e.g., error rate exceeds threshold)
- Implement **log-based metrics** in Cloud Logging to track error trends over time without querying BigQuery
- Extend the sink filter to include WARNING-level logs and compare error vs. warning ratios
- Integrate BigQuery results with **Data Studio (Looker Studio)** to create visual dashboards for stakeholders
- Investigate **VPC Service Controls** to restrict log export destinations and prevent data exfiltration (covered in later labs)

---

## Additional Resources

- [Cloud Logging Documentation](https://cloud.google.com/logging/docs) - Comprehensive guide to log ingestion, routing, and querying
- [Log Router Overview](https://cloud.google.com/logging/docs/routing/overview) - Detailed explanation of sinks, filters, and destinations
- [BigQuery Logging Tables](https://cloud.google.com/logging/docs/export/bigquery) - Schema reference and best practices for log tables
- [GKE Logging Best Practices](https://cloud.google.com/kubernetes-engine/docs/how-to/logging) - Workload logging configuration and troubleshooting
- [BigQuery Standard SQL Reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax) - SQL syntax for log analysis queries
- [Log-Based Metrics](https://cloud.google.com/logging/docs/logs-based-metrics) - Create metrics from log data for monitoring and alerting
- [Cloud Logging Pricing](https://cloud.google.com/stackdriver/pricing#logging-costs) - Understanding ingestion and storage costs
- [Querying Logs in BigQuery](https://cloud.google.com/logging/docs/export/bigquery#querying_logs) - Sample queries and optimization techniques
