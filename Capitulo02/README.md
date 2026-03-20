# Lab 02-07-01: Create VM in Compute Engine and Connect via IAP (Identity-Aware Proxy)

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Beginner |
| **Bloom Level** | Understand |

## Overview

In this lab, you will create a virtual machine in Google Compute Engine without a public IP address and securely connect to it using Identity-Aware Proxy (IAP) TCP Tunneling. This approach eliminates the need to expose SSH ports to the public internet, significantly improving security posture. You will configure VPC firewall rules, enable OS Login for centralized identity management, and verify access controls through IAM roles. This practice demonstrates a zero-trust security model where access is granted based on identity rather than network location.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create a Compute Engine VM instance without a public IP address in a VPC network
- [ ] Configure VPC firewall rules to allow SSH access exclusively through IAP's IP range (35.235.240.0/20)
- [ ] Connect to a private VM using IAP TCP Tunneling from both Google Cloud Console and gcloud CLI
- [ ] Verify identity-based access controls using IAM roles (iap.tunnelResourceAccessor and compute.osLogin)

## Prerequisites

### Required Knowledge

- Basic understanding of Linux command line and SSH protocol
- Familiarity with VPC networking concepts (subnets, firewall rules)
- Understanding of IAM roles and service accounts in Google Cloud
- Basic knowledge of network security principles

### Required Access

- Google Cloud Project with billing enabled
- IAM permissions: Compute Admin (or compute.instanceAdmin.v1), Security Admin (for firewall rules), and Project IAM Admin (to grant roles)
- Access to Google Cloud Console and Cloud Shell

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 2 vCPUs minimum |
| RAM | 8 GB minimum |
| Network | Stable internet connection ≥10 Mbps |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage Compute Engine resources and IAP tunneling |
| Google Cloud Shell | Latest | Pre-configured environment with all tools installed |
| OpenSSH Client | 8.9p1+ | SSH connectivity through IAP tunnel |
| Compute Engine API | v1 | Create and manage VM instances |
| IAP API | v1 | Enable Identity-Aware Proxy for TCP tunneling |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Enable required APIs
gcloud services enable compute.googleapis.com
gcloud services enable iap.googleapis.com

# Set default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Verify current configuration
gcloud config list
```

## Step-by-Step Instructions

### Step 1: Create VPC Firewall Rule for IAP Access

**Objective:** Configure a firewall rule that allows SSH traffic (TCP port 22) exclusively from Google's IAP IP range to instances tagged with `iap-ssh`.

**Instructions:**

1. Create the firewall rule to allow IAP-based SSH access:
   
   ```bash
   gcloud compute firewall-rules create allow-ssh-from-iap \
     --network=default \
     --direction=INGRESS \
     --priority=1000 \
     --action=ALLOW \
     --rules=tcp:22 \
     --source-ranges=35.235.240.0/20 \
     --target-tags=iap-ssh \
     --description="Allow SSH from IAP TCP forwarding range"
   ```

2. Verify the firewall rule was created successfully:

   ```bash
   gcloud compute firewall-rules describe allow-ssh-from-iap
   ```

**Expected Output:**

```
Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/firewalls/allow-ssh-from-iap].
Creating firewall...done.
NAME: allow-ssh-from-iap
NETWORK: default
DIRECTION: INGRESS
PRIORITY: 1000
ALLOW: tcp:22
```

**Verification:**

- Confirm the firewall rule appears in the list: `gcloud compute firewall-rules list --filter="name=allow-ssh-from-iap"`
- Verify the source range is exactly `35.235.240.0/20`
- Confirm target tags include `iap-ssh`

---

### Step 2: Create Compute Engine VM Without Public IP

**Objective:** Launch a Debian 12 VM instance without a public IP address, with OS Login enabled and the appropriate network tag for IAP access.

**Instructions:**

1. Create the VM instance with no external IP:
   
   ```bash
   gcloud compute instances create iap-test-vm \
     --zone=us-central1-a \
     --machine-type=e2-micro \
     --image-family=debian-12 \
     --image-project=debian-cloud \
     --boot-disk-size=10GB \
     --boot-disk-type=pd-standard \
     --tags=iap-ssh \
     --no-address \
     --metadata=enable-oslogin=TRUE \
     --scopes=cloud-platform
   ```

2. Verify the instance was created without an external IP:

   ```bash
   gcloud compute instances describe iap-test-vm \
     --zone=us-central1-a \
     --format="table(name,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/iap-test-vm].
NAME          ZONE           MACHINE_TYPE  INTERNAL_IP  EXTERNAL_IP  STATUS
iap-test-vm   us-central1-a  e2-micro      10.128.0.2                RUNNING
```

**Verification:**

- Confirm `EXTERNAL_IP` column is empty (no public IP assigned)
- Verify the instance has the `iap-ssh` tag: `gcloud compute instances describe iap-test-vm --zone=us-central1-a --format="value(tags.items)"`
- Confirm OS Login is enabled: `gcloud compute instances describe iap-test-vm --zone=us-central1-a --format="value(metadata.items.enable-oslogin)"`

---

### Step 3: Grant IAM Roles for IAP and OS Login

**Objective:** Assign the necessary IAM roles to your user account to enable IAP tunneling and OS Login authentication.

**Instructions:**

1. Get your current authenticated user email:
   
   ```bash
   export USER_EMAIL=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")
   echo "Current user: $USER_EMAIL"
   ```

2. Grant the IAP-secured Tunnel User role:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=user:$USER_EMAIL \
     --role=roles/iap.tunnelResourceAccessor
   ```

3. Grant the Compute OS Login role:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=user:$USER_EMAIL \
     --role=roles/compute.osLogin
   ```

4. Optionally, grant Compute Instance Admin role if not already assigned:

   ```bash
   gcloud projects add-iam-policy-binding $PROJECT_ID \
     --member=user:$USER_EMAIL \
     --role=roles/compute.instanceAdmin.v1
   ```

**Expected Output:**

```
Updated IAM policy for project [PROJECT_ID].
bindings:
- members:
  - user:your-email@example.com
  role: roles/iap.tunnelResourceAccessor
...
```

**Verification:**

- Verify roles were granted: `gcloud projects get-iam-policy $PROJECT_ID --flatten="bindings[].members" --filter="bindings.members:$USER_EMAIL" --format="table(bindings.role)"`
- Confirm both `roles/iap.tunnelResourceAccessor` and `roles/compute.osLogin` appear in the output

---

### Step 4: Connect to VM Using IAP from Cloud Console

**Objective:** Establish an SSH connection to the private VM through the Google Cloud Console using IAP.

**Instructions:**

1. Navigate to the Compute Engine instances page in the Google Cloud Console:
   - Go to **Navigation Menu** > **Compute Engine** > **VM instances**

2. Locate the `iap-test-vm` instance in the list

3. Click the **SSH** button next to the instance name

4. A new browser window will open with an SSH session through IAP

5. Once connected, verify you are logged in:

   ```bash
   whoami
   hostname
   ip addr show
   ```

**Expected Output:**

```
your_username_google_com
iap-test-vm
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
2: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc mq state UP
    inet 10.128.0.2/32 scope global dynamic ens4
```

**Verification:**

- Confirm you see a shell prompt with your username
- Verify the hostname matches `iap-test-vm`
- Confirm only internal IP addresses are visible (no public IP on network interfaces)
- Check connection method: `echo $SSH_CONNECTION` should show IAP tunnel endpoints

---

### Step 5: Connect to VM Using gcloud CLI with IAP Tunneling

**Objective:** Connect to the private VM using the gcloud command-line tool with the `--tunnel-through-iap` flag.

**Instructions:**

1. From Cloud Shell or your local terminal with gcloud installed, connect via IAP:
   
   ```bash
   gcloud compute ssh iap-test-vm \
     --zone=us-central1-a \
     --tunnel-through-iap
   ```

2. If prompted to generate SSH keys, accept the defaults by pressing Enter

3. Once connected, run diagnostic commands:

   ```bash
   curl -s http://metadata.google.internal/computeMetadata/v1/instance/name \
     -H "Metadata-Flavor: Google"
   ```

4. Check that no public IP is assigned to the instance:

   ```bash
   curl -s http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip \
     -H "Metadata-Flavor: Google"
   ```

**Expected Output:**

```
External IP address was not found within instance metadata.
```

**Verification:**

- Confirm successful SSH connection without errors
- Verify metadata query returns `iap-test-vm`
- Confirm no external IP is found in metadata
- Test command execution: `sudo apt update` should work if you have proper OS Login permissions

---

### Step 6: Install and Test Software on the VM

**Objective:** Validate full SSH functionality by installing software and verifying the VM's internal-only network configuration.

**Instructions:**

1. Update package lists and install monitoring tools:
   
   ```bash
   sudo apt update
   sudo apt install -y htop net-tools
   ```

2. Run htop to verify system resources:

   ```bash
   htop
   ```

   Press `q` to quit htop.

3. Check network configuration to confirm no public IP:

   ```bash
   ip addr show
   ifconfig
   ```

4. Verify outbound internet connectivity (through Cloud NAT or default internet gateway if configured):

   ```bash
   ping -c 4 8.8.8.8
   curl -I https://www.google.com
   ```

5. Exit the SSH session:

   ```bash
   exit
   ```

**Expected Output:**

```
Reading package lists... Done
Building dependency tree... Done
htop is already the newest version
net-tools is already the newest version
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

**Verification:**

- Confirm packages install successfully
- Verify `ip addr show` displays only internal IP (10.x.x.x range)
- Confirm outbound connectivity works (ping and curl succeed)
- Ensure you can exit and reconnect via IAP without issues

---

## Validation & Testing

### Success Criteria

- [ ] VM instance `iap-test-vm` is running without a public IP address
- [ ] Firewall rule `allow-ssh-from-iap` permits SSH only from IAP range (35.235.240.0/20)
- [ ] IAM roles `iap.tunnelResourceAccessor` and `compute.osLogin` are assigned to your user account
- [ ] SSH connection succeeds through both Cloud Console and gcloud CLI using IAP
- [ ] Software installation and command execution work normally on the VM
- [ ] No direct internet access to SSH port 22 on the VM (verified by lack of public IP)

### Testing Procedure

1. Test IAP connectivity from gcloud:
   ```bash
   gcloud compute ssh iap-test-vm \
     --zone=us-central1-a \
     --tunnel-through-iap \
     --command="echo 'IAP connection successful'"
   ```
   **Expected Result:** Output displays `IAP connection successful`

2. Verify the VM has no public IP:
   ```bash
   gcloud compute instances describe iap-test-vm \
     --zone=us-central1-a \
     --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
   ```
   **Expected Result:** Empty output (no external IP)

3. Confirm firewall rule targets the correct tag:
   ```bash
   gcloud compute firewall-rules describe allow-ssh-from-iap \
     --format="get(targetTags)"
   ```
   **Expected Result:** `['iap-ssh']`

4. Test IAM role assignment:
   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.role:roles/iap.tunnelResourceAccessor AND bindings.members:user:$USER_EMAIL" \
     --format="value(bindings.role)"
   ```
   **Expected Result:** `roles/iap.tunnelResourceAccessor`

5. Verify OS Login is enabled:
   ```bash
   gcloud compute instances describe iap-test-vm \
     --zone=us-central1-a \
     --format="value(metadata.items[enable-oslogin])"
   ```
   **Expected Result:** `TRUE`

## Troubleshooting

### Issue 1: SSH Connection Fails with "Permission Denied"

**Symptoms:**
- Error message: `Permission denied (publickey)`
- SSH connection terminates immediately after attempting to connect
- Cloud Console SSH button shows authentication errors

**Cause:**
OS Login is enabled but the user account does not have the required `compute.osLogin` role, or SSH keys are not properly propagated.

**Solution:**
```bash
# Re-grant the OS Login role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:$(gcloud auth list --filter=status:ACTIVE --format="value(account)") \
  --role=roles/compute.osLogin

# Wait 60 seconds for IAM propagation
sleep 60

# Retry connection
gcloud compute ssh iap-test-vm \
  --zone=us-central1-a \
  --tunnel-through-iap
```

If the issue persists, verify OS Login is enabled on the instance:
```bash
gcloud compute instances add-metadata iap-test-vm \
  --zone=us-central1-a \
  --metadata=enable-oslogin=TRUE
```

---

### Issue 2: Connection Timeout or "Failed to Connect to Port 22"

**Symptoms:**
- Error: `ssh: connect to host X.X.X.X port 22: Connection timed out`
- IAP tunnel fails to establish
- Console SSH button spins indefinitely without connecting

**Cause:**
Firewall rule is missing, misconfigured, or the VM does not have the correct network tag (`iap-ssh`).

**Solution:**
```bash
# Verify the firewall rule exists and is correctly configured
gcloud compute firewall-rules describe allow-ssh-from-iap

# Check if the VM has the correct tag
gcloud compute instances describe iap-test-vm \
  --zone=us-central1-a \
  --format="value(tags.items)"

# If the tag is missing, add it
gcloud compute instances add-tags iap-test-vm \
  --zone=us-central1-a \
  --tags=iap-ssh

# Verify IAP role is assigned
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/iap.tunnelResourceAccessor" \
  --format="table(bindings.members)"
```

If the firewall rule is missing, recreate it:
```bash
gcloud compute firewall-rules create allow-ssh-from-iap \
  --network=default \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 \
  --target-tags=iap-ssh
```

---

### Issue 3: "IAP Tunnel Resource Accessor Role Required" Error

**Symptoms:**
- Error message: `ERROR: (gcloud.compute.ssh) Could not fetch resource: - '@type': type.googleapis.com/google.rpc.PreconditionFailure`
- Message indicates missing IAP permissions

**Cause:**
The user account does not have the `roles/iap.tunnelResourceAccessor` role at the project level.

**Solution:**
```bash
# Grant the IAP tunnel resource accessor role
export USER_EMAIL=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:$USER_EMAIL \
  --role=roles/iap.tunnelResourceAccessor

# Wait for IAM propagation (typically 60-120 seconds)
echo "Waiting for IAM propagation..."
sleep 90

# Retry the connection
gcloud compute ssh iap-test-vm \
  --zone=us-central1-a \
  --tunnel-through-iap
```

---

### Issue 4: VM Created with Public IP Despite --no-address Flag

**Symptoms:**
- VM shows an external IP address in the console or CLI output
- The `--no-address` flag was used but a public IP was still assigned

**Cause:**
Default network configuration or organizational policies may automatically assign external IPs. Some projects have default settings that override the `--no-address` flag.

**Solution:**
```bash
# Delete the existing VM
gcloud compute instances delete iap-test-vm \
  --zone=us-central1-a \
  --quiet

# Recreate without external IP and explicitly set network tier
gcloud compute instances create iap-test-vm \
  --zone=us-central1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=10GB \
  --tags=iap-ssh \
  --no-address \
  --network-tier=STANDARD \
  --metadata=enable-oslogin=TRUE \
  --scopes=cloud-platform

# Verify no external IP is assigned
gcloud compute instances describe iap-test-vm \
  --zone=us-central1-a \
  --format="get(networkInterfaces[0].accessConfigs)"
```

If the output is empty or `[]`, the VM successfully has no public IP.

---

## Cleanup

```bash
# Stop the VM instance (if you want to preserve it for future labs)
gcloud compute instances stop iap-test-vm \
  --zone=us-central1-a

# OR delete the VM instance completely
gcloud compute instances delete iap-test-vm \
  --zone=us-central1-a \
  --quiet

# Optionally, remove the firewall rule if no longer needed
gcloud compute firewall-rules delete allow-ssh-from-iap --quiet

# Remove IAM role bindings (optional, only if not needed for other labs)
export USER_EMAIL=$(gcloud auth list --filter=status:ACTIVE --format="value(account)")

gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member=user:$USER_EMAIL \
  --role=roles/iap.tunnelResourceAccessor

gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member=user:$USER_EMAIL \
  --role=roles/compute.osLogin
```

> ⚠️ **Warning:** Deleting the VM is irreversible. If you plan to use this VM in subsequent labs, use `stop` instead of `delete`. Stopped VMs still incur disk storage charges but not compute charges. Firewall rules and IAM bindings do not incur charges and can be left in place if needed for future work.

## Summary

### What You Accomplished

- Created a Compute Engine VM instance without a public IP address, enhancing security by eliminating direct internet exposure
- Configured VPC firewall rules to restrict SSH access exclusively through Google's Identity-Aware Proxy (IAP) IP range
- Enabled OS Login for centralized identity management and SSH key management
- Successfully connected to a private VM using IAP TCP Tunneling from both the Cloud Console and gcloud CLI
- Verified identity-based access controls using IAM roles for secure, auditable access

### Key Takeaways

- **Zero-Trust Security:** IAP enables secure access to VMs without exposing SSH ports to the public internet, implementing identity-based access control
- **No Bastion Hosts Required:** IAP eliminates the need for jump boxes or bastion hosts, simplifying architecture and reducing attack surface
- **Centralized Identity Management:** OS Login integrates with Google Cloud IAM, allowing centralized user management and automatic SSH key provisioning
- **Audit and Compliance:** All IAP connections are logged in Cloud Audit Logs, providing full visibility into who accessed which resources and when
- **Defense in Depth:** Combining firewall rules (network layer), IAP (application layer), and IAM (identity layer) creates multiple security boundaries

### Next Steps

- Explore IAP for TCP forwarding to other services (MySQL, RDP, custom applications)
- Implement organization-wide policies to enforce IAP usage for all SSH access
- Configure Cloud Audit Logs to monitor and alert on IAP connection attempts
- Integrate IAP with VPC Service Controls for additional data exfiltration protection
- Practice creating custom IAM roles with minimal privileges for specific VM access patterns

## Additional Resources

- [Identity-Aware Proxy Documentation](https://cloud.google.com/iap/docs) - Official Google Cloud IAP documentation
- [IAP TCP Forwarding Overview](https://cloud.google.com/iap/docs/tcp-forwarding-overview) - Detailed guide on IAP tunneling
- [OS Login Documentation](https://cloud.google.com/compute/docs/oslogin) - Managing SSH access with OS Login
- [VPC Firewall Rules Best Practices](https://cloud.google.com/vpc/docs/firewalls) - Securing network access in Google Cloud
- [IAM Roles for Compute Engine](https://cloud.google.com/compute/docs/access/iam) - Understanding IAM permissions for VM access
- [Cloud Audit Logs for IAP](https://cloud.google.com/iap/docs/audit-log-howto) - Monitoring and auditing IAP access

---

# Lab 02-08-01: Práctica 3: Crear bucket en Cloud Storage con versionado y ciclo de vida

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Beginner |
| **Bloom Level** | Understand |

## Overview

In this lab, you will create a Cloud Storage bucket with Uniform Bucket-Level Access (UBLA), enable object versioning, and configure lifecycle management policies. You will learn how to manage multiple versions of objects, restore previous versions, and automate data lifecycle transitions to optimize storage costs. This practice is essential for implementing data retention strategies and disaster recovery procedures in production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create a regional Cloud Storage bucket with Uniform Bucket-Level Access (UBLA) enabled
- [ ] Enable and manage object versioning to maintain historical copies of files
- [ ] Configure lifecycle policies for automatic object transitions and retention management
- [ ] Restore previous versions of objects and verify version history
- [ ] Apply IAM roles at the bucket level and validate access controls

## Prerequisites

### Required Knowledge

- Basic understanding of Cloud Storage concepts (buckets, objects, storage classes)
- Familiarity with Google Cloud Console and Cloud Shell
- Basic command-line experience with gsutil and gcloud
- Understanding of IAM roles and permissions

### Required Access

- Google Cloud Project with billing enabled
- Storage Admin role (roles/storage.admin) in the project
- Access to Cloud Shell or local environment with Google Cloud SDK installed

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| RAM | 4 GB minimum |
| CPU | 2 vCPU minimum |
| Network | Stable internet connection ≥10 Mbps |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage Cloud Storage resources |
| gsutil | 5.x+ | Upload, download, and manage objects |
| Cloud Shell | Latest | Pre-configured environment with all tools |
| Cloud Storage API | v1 | Bucket and object operations |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Enable Cloud Storage API
gcloud services enable storage-api.googleapis.com

# Set variables for the lab
export REGION="us-central1"
export BUCKET_NAME="${PROJECT_ID}-lab-storage-$(date +%Y%m%d)"
export TEST_USER="student@example.com"

# Verify current user
gcloud auth list

# Display configuration
echo "Project ID: $PROJECT_ID"
echo "Region: $REGION"
echo "Bucket Name: $BUCKET_NAME"
```

## Step-by-Step Instructions

### Step 1: Create a Regional Bucket with UBLA Enabled

**Objective:** Create a Cloud Storage bucket with Uniform Bucket-Level Access to simplify IAM management and enable versioning.

**Instructions:**

1. Create the bucket with UBLA enabled:
   
   ```bash
   gcloud storage buckets create gs://${BUCKET_NAME} \
     --location=${REGION} \
     --uniform-bucket-level-access
   ```

2. Verify the bucket was created with UBLA:

   ```bash
   gcloud storage buckets describe gs://${BUCKET_NAME} \
     --format="yaml(iamConfiguration)"
   ```

3. Alternatively, view bucket details using gsutil:

   ```bash
   gsutil bucketpolicyonly get gs://${BUCKET_NAME}
   ```

**Expected Output:**

```
Creating gs://your-project-id-lab-storage-20240115/...
iamConfiguration:
  bucketPolicyOnly:
    enabled: true
  uniformBucketLevelAccess:
    enabled: true
    lockedTime: '2024-04-15T10:30:00.000Z'
```

**Verification:**

- Confirm `uniformBucketLevelAccess.enabled: true` in the output
- The bucket appears in the Cloud Console under Cloud Storage > Buckets

---

### Step 2: Enable Object Versioning

**Objective:** Enable versioning on the bucket to maintain historical versions of all objects.

**Instructions:**

1. Enable versioning on the bucket:
   
   ```bash
   gcloud storage buckets update gs://${BUCKET_NAME} \
     --versioning
   ```

2. Verify versioning is enabled:

   ```bash
   gsutil versioning get gs://${BUCKET_NAME}
   ```

**Expected Output:**

```
gs://your-project-id-lab-storage-20240115: Enabled
```

**Verification:**

- The output shows "Enabled" for versioning
- In Cloud Console, navigate to the bucket settings and confirm "Object Versioning" is ON

---

### Step 3: Upload an Object and Create Multiple Versions

**Objective:** Upload a test file, modify it, and re-upload to generate multiple versions.

**Instructions:**

1. Create a sample text file:
   
   ```bash
   echo "Version 1: Initial content" > sample.txt
   cat sample.txt
   ```

2. Upload the first version:

   ```bash
   gsutil cp sample.txt gs://${BUCKET_NAME}/
   ```

3. Modify the file and upload version 2:

   ```bash
   echo "Version 2: Updated content with new data" > sample.txt
   gsutil cp sample.txt gs://${BUCKET_NAME}/
   ```

4. Modify again and upload version 3:

   ```bash
   echo "Version 3: Final revision with additional changes" > sample.txt
   gsutil cp sample.txt gs://${BUCKET_NAME}/
   ```

5. List all versions of the object:

   ```bash
   gsutil ls -a gs://${BUCKET_NAME}/sample.txt
   ```

**Expected Output:**

```
gs://your-project-id-lab-storage-20240115/sample.txt#1705318200000000
gs://your-project-id-lab-storage-20240115/sample.txt#1705318245000000
gs://your-project-id-lab-storage-20240115/sample.txt#1705318290000000
```

**Verification:**

- Three versions of sample.txt are listed with different generation numbers (timestamps)
- The most recent version has no "#archived" suffix when listing without `-a`

---

### Step 4: Restore a Previous Version

**Objective:** Restore an older version of the object by copying it to become the current version.

**Instructions:**

1. View the current content:
   
   ```bash
   gsutil cat gs://${BUCKET_NAME}/sample.txt
   ```

2. List all versions and capture the generation ID of version 1 (oldest):

   ```bash
   gsutil ls -a gs://${BUCKET_NAME}/sample.txt
   ```

3. Copy the first version (replace the generation number with your actual oldest generation):

   ```bash
   # Get the oldest generation (first in the list)
   OLDEST_GEN=$(gsutil ls -a gs://${BUCKET_NAME}/sample.txt | head -1)
   echo "Restoring: $OLDEST_GEN"
   
   gsutil cp $OLDEST_GEN gs://${BUCKET_NAME}/sample.txt
   ```

4. Verify the restored content:

   ```bash
   gsutil cat gs://${BUCKET_NAME}/sample.txt
   ```

**Expected Output:**

```
Version 1: Initial content
```

**Verification:**

- The current version now shows "Version 1: Initial content"
- A new generation has been created with the restored content
- Run `gsutil ls -a gs://${BUCKET_NAME}/sample.txt` to see all four versions now exist

---

### Step 5: Configure Lifecycle Management Policies

**Objective:** Create lifecycle rules to automatically delete old versions and transition objects to cost-effective storage classes.

**Instructions:**

1. Create a lifecycle configuration file:
   
   ```bash
   cat > lifecycle.json <<'EOF'
   {
     "lifecycle": {
       "rule": [
         {
           "action": {
             "type": "Delete"
           },
           "condition": {
             "numNewerVersions": 2
           }
         },
         {
           "action": {
             "type": "SetStorageClass",
             "storageClass": "NEARLINE"
           },
           "condition": {
             "age": 30,
             "matchesPrefix": ["archive/"]
           }
         }
       ]
     }
   }
   EOF
   ```

2. Apply the lifecycle configuration to the bucket:

   ```bash
   gsutil lifecycle set lifecycle.json gs://${BUCKET_NAME}
   ```

3. Verify the lifecycle configuration:

   ```bash
   gsutil lifecycle get gs://${BUCKET_NAME}
   ```

**Expected Output:**

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": {
          "type": "Delete"
        },
        "condition": {
          "numNewerVersions": 2
        }
      },
      {
        "action": {
          "storageClass": "NEARLINE",
          "type": "SetStorageClass"
        },
        "condition": {
          "age": 30,
          "matchesPrefix": [
            "archive/"
          ]
        }
      }
    ]
  }
}
```

**Verification:**

- The lifecycle rules are displayed correctly
- In Cloud Console, check Bucket details > Lifecycle to see the configured rules
- Note: Policy execution is not immediate and may take up to 24 hours

---

### Step 6: Configure IAM Access Controls

**Objective:** Grant Storage Object Viewer role to a test user and validate access.

**Instructions:**

1. Grant Storage Object Viewer role at the bucket level:
   
   ```bash
   gcloud storage buckets add-iam-policy-binding gs://${BUCKET_NAME} \
     --member="user:${TEST_USER}" \
     --role="roles/storage.objectViewer"
   ```

2. Verify the IAM policy:

   ```bash
   gcloud storage buckets get-iam-policy gs://${BUCKET_NAME} \
     --format="table(bindings.role, bindings.members)"
   ```

3. List all IAM bindings for the bucket:

   ```bash
   gsutil iam get gs://${BUCKET_NAME}
   ```

**Expected Output:**

```
Updated IAM policy for bucket [your-project-id-lab-storage-20240115].
bindings:
- members:
  - user:student@example.com
  role: roles/storage.objectViewer
```

**Verification:**

- The test user appears in the IAM policy with the Storage Object Viewer role
- If you have access to the test user account, attempt to read the object (should succeed) and write (should fail)

---

### Step 7: Test Access and Validate Configuration

**Objective:** Validate that versioning, lifecycle policies, and IAM controls are working as expected.

**Instructions:**

1. Test reading the current version (should work with objectViewer):
   
   ```bash
   # Simulate read access
   gsutil ls gs://${BUCKET_NAME}/
   gsutil cat gs://${BUCKET_NAME}/sample.txt
   ```

2. Verify version count:

   ```bash
   VERSION_COUNT=$(gsutil ls -a gs://${BUCKET_NAME}/sample.txt | wc -l)
   echo "Total versions: $VERSION_COUNT"
   ```

3. Create a test object in the archive prefix to test lifecycle transition:

   ```bash
   echo "Archive test data" > archive-test.txt
   gsutil cp archive-test.txt gs://${BUCKET_NAME}/archive/test-file.txt
   ```

4. Verify the object storage class:

   ```bash
   gsutil stat gs://${BUCKET_NAME}/archive/test-file.txt | grep "Storage class"
   ```

**Expected Output:**

```
Total versions: 4
Storage class: STANDARD
```

**Verification:**

- Version count matches the number of uploads performed
- Archive object is initially in STANDARD class (will transition to NEARLINE after 30 days)
- All commands execute without permission errors

---

## Validation & Testing

### Success Criteria

- [ ] Bucket created with UBLA enabled in the specified region
- [ ] Object versioning is enabled and multiple versions exist
- [ ] At least 3 versions of sample.txt are present in the bucket
- [ ] Previous version successfully restored as the current version
- [ ] Lifecycle policy configured with delete rule (numNewerVersions > 2) and transition rule (30 days to Nearline)
- [ ] IAM policy grants Storage Object Viewer to test user
- [ ] Test object created under archive/ prefix for lifecycle validation

### Testing Procedure

1. Validate bucket configuration:
   ```bash
   gcloud storage buckets describe gs://${BUCKET_NAME} \
     --format="yaml(location,iamConfiguration.uniformBucketLevelAccess.enabled,versioning.enabled)"
   ```
   **Expected Result:** 
   ```yaml
   iamConfiguration:
     uniformBucketLevelAccess:
       enabled: true
   location: US-CENTRAL1
   versioning:
     enabled: true
   ```

2. Verify version restoration:
   ```bash
   gsutil cat gs://${BUCKET_NAME}/sample.txt
   ```
   **Expected Result:** Content should match version 1: "Version 1: Initial content"

3. Check lifecycle policy:
   ```bash
   gsutil lifecycle get gs://${BUCKET_NAME} | grep -E "(Delete|SetStorageClass|numNewerVersions|age)"
   ```
   **Expected Result:** Both Delete and SetStorageClass actions present

4. Validate IAM binding:
   ```bash
   gcloud storage buckets get-iam-policy gs://${BUCKET_NAME} \
     --flatten="bindings[].members" \
     --filter="bindings.role:roles/storage.objectViewer" \
     --format="value(bindings.members)"
   ```
   **Expected Result:** `user:student@example.com` appears in output

## Troubleshooting

### Issue 1: Bucket Creation Fails with "Name Already Exists"

**Symptoms:**
- Error message: "ServiceException: 409 Bucket <name> already exists"
- Bucket creation command fails

**Cause:**
Bucket names are globally unique across all Google Cloud projects. The chosen name is already in use.

**Solution:**
```bash
# Generate a unique bucket name with timestamp
export BUCKET_NAME="${PROJECT_ID}-lab-storage-$(date +%s)"
echo "New bucket name: $BUCKET_NAME"

# Retry bucket creation
gcloud storage buckets create gs://${BUCKET_NAME} \
  --location=${REGION} \
  --uniform-bucket-level-access
```

---

### Issue 2: Cannot See Multiple Versions of Object

**Symptoms:**
- `gsutil ls gs://${BUCKET_NAME}/sample.txt` shows only one version
- No generation numbers visible

**Cause:**
Versioning may not be enabled, or the `-a` flag (all versions) is missing from the command.

**Solution:**
```bash
# Verify versioning is enabled
gsutil versioning get gs://${BUCKET_NAME}

# If disabled, enable it
gsutil versioning set on gs://${BUCKET_NAME}

# Use -a flag to list all versions
gsutil ls -a gs://${BUCKET_NAME}/sample.txt
```

---

### Issue 3: Lifecycle Policy Not Applied Immediately

**Symptoms:**
- Old versions still exist after applying lifecycle policy
- Objects not transitioning to Nearline storage class

**Cause:**
Lifecycle policies are not applied immediately. Google Cloud processes lifecycle rules asynchronously, typically within 24 hours.

**Solution:**
```bash
# Verify the policy is configured correctly
gsutil lifecycle get gs://${BUCKET_NAME}

# Note: No immediate action needed - this is expected behavior
# For lab purposes, focus on validating the configuration, not the execution
echo "Lifecycle policies are configured correctly and will be applied within 24 hours."
```

---

### Issue 4: Permission Denied When Adding IAM Policy

**Symptoms:**
- Error: "PERMISSION_DENIED: Permission denied on resource"
- Cannot add IAM binding for test user

**Cause:**
Your account lacks the necessary permissions (Storage Admin or Owner role) to modify bucket IAM policies.

**Solution:**
```bash
# Check your current roles
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:$(gcloud config get-value account)" \
  --format="table(bindings.role)"

# If you lack permissions, request Storage Admin role from project owner
# Or use a service account with appropriate permissions
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/storage.admin"
```

---

## Cleanup

```bash
# Remove all objects including all versions
gsutil -m rm -a gs://${BUCKET_NAME}/**

# Delete the bucket
gcloud storage buckets delete gs://${BUCKET_NAME} --quiet

# Remove local test files
rm -f sample.txt archive-test.txt lifecycle.json

# Verify bucket deletion
gcloud storage buckets list | grep $BUCKET_NAME
```

> ⚠️ **Warning:** Deleting a bucket with versioning enabled requires removing all object versions first. Use the `-a` flag with `gsutil rm` to delete all versions. Static IP addresses and buckets left running will incur charges. Always clean up resources after completing the lab.

> 💡 **Note:** If you plan to reuse this bucket for subsequent labs, skip the cleanup step and document the bucket name for future reference.

## Summary

### What You Accomplished

- Created a regional Cloud Storage bucket with Uniform Bucket-Level Access (UBLA) for simplified IAM management
- Enabled object versioning and generated multiple versions of a test file
- Successfully restored a previous version of an object to become the current version
- Configured lifecycle management policies to automatically delete old versions (keeping only 2 newest) and transition objects to Nearline storage after 30 days
- Applied IAM roles at the bucket level and validated access controls for a test user
- Validated all configurations and tested version management operations

### Key Takeaways

- **Uniform Bucket-Level Access (UBLA)** simplifies permissions by managing access at the bucket level using IAM, rather than ACLs on individual objects
- **Object versioning** provides protection against accidental deletion or overwriting, enabling point-in-time recovery
- **Lifecycle policies** automate data management to optimize storage costs by deleting old versions or transitioning objects to cheaper storage classes
- **Version restoration** is achieved by copying a specific generation back to the bucket, creating a new current version
- **IAM roles** at the bucket level apply to all objects, making permission management more scalable and secure
- Lifecycle policy execution is asynchronous and may take up to 24 hours to take effect

### Next Steps

- Explore advanced lifecycle conditions such as `isLive`, `createdBefore`, and `daysSinceCustomTime`
- Implement Object Lifecycle Management in production environments to reduce storage costs
- Configure Cloud Storage notifications to trigger Cloud Functions on object changes
- Integrate Cloud Storage with BigQuery for data analytics workflows
- Practice using customer-managed encryption keys (CMEK) for enhanced data security
- Implement retention policies and bucket locks for compliance requirements
- Continue to Lab 02-08-02 to explore Cloud Storage security and access controls in depth

## Additional Resources

- [Cloud Storage Versioning Documentation](https://cloud.google.com/storage/docs/object-versioning) - Official guide on enabling and managing object versions
- [Object Lifecycle Management](https://cloud.google.com/storage/docs/lifecycle) - Comprehensive documentation on lifecycle policies and conditions
- [Uniform Bucket-Level Access](https://cloud.google.com/storage/docs/uniform-bucket-level-access) - Best practices for IAM-based access control
- [Storage Classes and Pricing](https://cloud.google.com/storage/docs/storage-classes) - Comparison of Standard, Nearline, Coldline, and Archive storage
- [gsutil Tool Documentation](https://cloud.google.com/storage/docs/gsutil) - Complete reference for gsutil commands
- [Cloud Storage IAM Roles](https://cloud.google.com/storage/docs/access-control/iam-roles) - Detailed list of predefined roles and permissions
- [Best Practices for Cloud Storage](https://cloud.google.com/storage/docs/best-practices) - Performance, security, and cost optimization guidelines

---

# Lab 02-09-01: Práctica 4: Crear VPC con 2 subredes, firewall y balanceo de carga interno

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

In this lab, you will design and implement a complete internal networking infrastructure on Google Cloud Platform. You will create a custom VPC network with two regional subnets, deploy backend instances, configure firewall rules following security best practices, and set up an internal TCP/UDP load balancer to distribute traffic across multiple backend servers. This hands-on exercise simulates a real-world scenario where applications require high availability and fault tolerance within a private network environment.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Design and create a custom VPC network with proper CIDR planning and regional distribution
- [ ] Configure two subnets in different regions with appropriate IP address ranges
- [ ] Implement firewall rules following the principle of least privilege for secure network access
- [ ] Deploy and configure an internal TCP/UDP load balancer to distribute traffic between backend instances
- [ ] Verify connectivity and load distribution using test instances and internal IP addresses

## Prerequisites

### Required Knowledge

- Basic understanding of IP addressing and CIDR notation
- Familiarity with Google Cloud Console navigation
- Knowledge of TCP/IP networking fundamentals
- Understanding of client-server architecture
- Completed Module 2.1-2.8 theory content

### Required Access

- Active Google Cloud Platform account with billing enabled or trial credits
- Project Editor role or equivalent permissions (Compute Admin, Network Admin)
- Access to Google Cloud Console and Cloud Shell

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| Internet Connection | Minimum 10 Mbps bandwidth |
| Display | Minimum 1366x768 resolution |
| Computer | Minimum 8GB RAM, dual-core processor |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud Console | Latest web version | Primary interface for managing GCP resources |
| Google Chrome or Firefox | Latest stable release | Web browser for accessing GCP Console |
| Google Cloud SDK (gcloud CLI) | 400.0.0+ | Command-line interface for GCP operations |
| SSH Client | OpenSSH 7.0+ or PuTTY 0.70+ | Secure connection to VM instances |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID=your-project-id
gcloud config set project $PROJECT_ID

# Verify project configuration
gcloud config list project

# Set default compute region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

## Step-by-Step Instructions

### Step 1: Create Custom VPC Network

**Objective:** Create a custom mode VPC network that will host your subnets and resources.

**Instructions:**

1. Open Google Cloud Console and navigate to **VPC network** > **VPC networks**.

2. Click **CREATE VPC NETWORK**.

3. Configure the VPC with the following settings:
   - **Name:** `custom-vpc-lab`
   - **Description:** `Custom VPC for internal load balancer lab`
   - **Subnet creation mode:** Custom

4. Create the VPC using Cloud Shell:

   ```bash
   gcloud compute networks create custom-vpc-lab \
       --subnet-mode=custom \
       --bgp-routing-mode=regional \
       --description="Custom VPC for internal load balancer lab"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/networks/custom-vpc-lab].
NAME              SUBNET_MODE  BGP_ROUTING_MODE  IPV4_RANGE  GATEWAY_IPV4
custom-vpc-lab    CUSTOM       REGIONAL
```

**Verification:**

- Confirm the VPC appears in the VPC networks list
- Verify that subnet mode is set to "Custom"

---

### Step 2: Create Subnet in us-central1 Region

**Objective:** Create the first subnet in the us-central1 region with appropriate CIDR range.

**Instructions:**

1. Create the first subnet for backend instances:

   ```bash
   gcloud compute networks subnets create subnet-us-central1 \
       --network=custom-vpc-lab \
       --region=us-central1 \
       --range=10.1.0.0/24 \
       --description="Subnet for backend instances in us-central1"
   ```

2. Verify subnet creation:

   ```bash
   gcloud compute networks subnets describe subnet-us-central1 \
       --region=us-central1
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/subnetworks/subnet-us-central1].
NAME                  REGION       NETWORK         RANGE
subnet-us-central1    us-central1  custom-vpc-lab  10.1.0.0/24
```

**Verification:**

- Confirm the subnet appears under the custom-vpc-lab network
- Verify the IP range is 10.1.0.0/24
- Check that the region is us-central1

---

### Step 3: Create Subnet in us-east1 Region

**Objective:** Create the second subnet in the us-east1 region with non-overlapping CIDR range.

**Instructions:**

1. Create the second subnet for additional backend instances:

   ```bash
   gcloud compute networks subnets create subnet-us-east1 \
       --network=custom-vpc-lab \
       --region=us-east1 \
       --range=10.2.0.0/24 \
       --description="Subnet for backend instances in us-east1"
   ```

2. List all subnets in the VPC:

   ```bash
   gcloud compute networks subnets list \
       --network=custom-vpc-lab \
       --sort-by=REGION
   ```

**Expected Output:**

```
NAME                  REGION       NETWORK         RANGE
subnet-us-central1    us-central1  custom-vpc-lab  10.1.0.0/24
subnet-us-east1       us-east1     custom-vpc-lab  10.2.0.0/24
```

**Verification:**

- Confirm both subnets exist in different regions
- Verify IP ranges do not overlap (10.1.0.0/24 and 10.2.0.0/24)
- Check that both subnets belong to custom-vpc-lab

---

### Step 4: Create Firewall Rule for SSH Access

**Objective:** Implement firewall rule to allow SSH access to instances for management purposes.

**Instructions:**

1. Create a firewall rule allowing SSH from IAP (Identity-Aware Proxy) range:

   ```bash
   gcloud compute firewall-rules create allow-ssh-iap \
       --network=custom-vpc-lab \
       --direction=INGRESS \
       --priority=1000 \
       --action=ALLOW \
       --rules=tcp:22 \
       --source-ranges=35.235.240.0/20 \
       --target-tags=backend-instance \
       --description="Allow SSH from IAP"
   ```

2. Verify the firewall rule:

   ```bash
   gcloud compute firewall-rules describe allow-ssh-iap
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/firewalls/allow-ssh-iap].
NAME            NETWORK         DIRECTION  PRIORITY  ALLOW   DENY
allow-ssh-iap   custom-vpc-lab  INGRESS    1000      tcp:22
```

**Verification:**

- Confirm the rule targets instances with tag "backend-instance"
- Verify source range is 35.235.240.0/20 (IAP range)
- Check that protocol is TCP port 22

---

### Step 5: Create Firewall Rule for Internal Load Balancer Health Checks

**Objective:** Allow Google Cloud health check probes to reach backend instances.

**Instructions:**

1. Create a firewall rule for health checks:

   ```bash
   gcloud compute firewall-rules create allow-health-check \
       --network=custom-vpc-lab \
       --direction=INGRESS \
       --priority=1000 \
       --action=ALLOW \
       --rules=tcp:80 \
       --source-ranges=35.191.0.0/16,130.211.0.0/22 \
       --target-tags=backend-instance \
       --description="Allow health checks from Google Cloud"
   ```

2. List all firewall rules in the VPC:

   ```bash
   gcloud compute firewall-rules list \
       --filter="network:custom-vpc-lab" \
       --sort-by=PRIORITY
   ```

**Expected Output:**

```
NAME                  NETWORK         DIRECTION  PRIORITY  ALLOW   DENY
allow-health-check    custom-vpc-lab  INGRESS    1000      tcp:80
allow-ssh-iap         custom-vpc-lab  INGRESS    1000      tcp:22
```

**Verification:**

- Confirm source ranges include 35.191.0.0/16 and 130.211.0.0/22
- Verify the rule allows TCP port 80
- Check that target tag is "backend-instance"

---

### Step 6: Create Firewall Rule for Internal Traffic

**Objective:** Allow internal communication between instances within the VPC.

**Instructions:**

1. Create a firewall rule for internal VPC traffic:

   ```bash
   gcloud compute firewall-rules create allow-internal \
       --network=custom-vpc-lab \
       --direction=INGRESS \
       --priority=1000 \
       --action=ALLOW \
       --rules=tcp:0-65535,udp:0-65535,icmp \
       --source-ranges=10.1.0.0/24,10.2.0.0/24 \
       --description="Allow internal traffic between subnets"
   ```

2. Verify all firewall rules are in place:

   ```bash
   gcloud compute firewall-rules list \
       --filter="network:custom-vpc-lab" \
       --format="table(name,direction,sourceRanges.list():label=SRC_RANGES,allowed[].map().firewall_rule().list():label=ALLOW)"
   ```

**Expected Output:**

```
NAME                  DIRECTION  SRC_RANGES                                    ALLOW
allow-health-check    INGRESS    35.191.0.0/16,130.211.0.0/22                 tcp:80
allow-internal        INGRESS    10.1.0.0/24,10.2.0.0/24                      tcp:0-65535,udp:0-65535,icmp
allow-ssh-iap         INGRESS    35.235.240.0/20                              tcp:22
```

**Verification:**

- Confirm internal traffic rule covers both subnet ranges
- Verify all protocols (TCP, UDP, ICMP) are allowed
- Check that three firewall rules exist for the VPC

---

### Step 7: Create Backend Instance in us-central1

**Objective:** Deploy the first backend instance with a simple web server.

**Instructions:**

1. Create the first backend instance:

   ```bash
   gcloud compute instances create backend-instance-1 \
       --zone=us-central1-a \
       --machine-type=e2-micro \
       --subnet=subnet-us-central1 \
       --network-tier=PREMIUM \
       --tags=backend-instance \
       --image-family=debian-11 \
       --image-project=debian-cloud \
       --boot-disk-size=10GB \
       --boot-disk-type=pd-standard \
       --metadata=startup-script='#!/bin/bash
   apt-get update
   apt-get install -y apache2
   echo "<!DOCTYPE html><html><body><h1>Backend Instance 1 - us-central1-a</h1><p>Hostname: $(hostname)</p></body></html>" > /var/www/html/index.html
   systemctl restart apache2'
   ```

2. Verify the instance is running:

   ```bash
   gcloud compute instances describe backend-instance-1 \
       --zone=us-central1-a \
       --format="get(name,status,networkInterfaces[0].networkIP)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/backend-instance-1].
NAME                  ZONE           MACHINE_TYPE  INTERNAL_IP
backend-instance-1    us-central1-a  e2-micro      10.1.0.2
```

**Verification:**

- Confirm instance status is "RUNNING"
- Verify internal IP is within 10.1.0.0/24 range
- Check that network tag "backend-instance" is applied

---

### Step 8: Create Backend Instance in us-central1 (Second Instance)

**Objective:** Deploy a second backend instance in the same region for load distribution.

**Instructions:**

1. Create the second backend instance:

   ```bash
   gcloud compute instances create backend-instance-2 \
       --zone=us-central1-b \
       --machine-type=e2-micro \
       --subnet=subnet-us-central1 \
       --network-tier=PREMIUM \
       --tags=backend-instance \
       --image-family=debian-11 \
       --image-project=debian-cloud \
       --boot-disk-size=10GB \
       --boot-disk-type=pd-standard \
       --metadata=startup-script='#!/bin/bash
   apt-get update
   apt-get install -y apache2
   echo "<!DOCTYPE html><html><body><h1>Backend Instance 2 - us-central1-b</h1><p>Hostname: $(hostname)</p></body></html>" > /var/www/html/index.html
   systemctl restart apache2'
   ```

2. List all backend instances:

   ```bash
   gcloud compute instances list \
       --filter="tags.items=backend-instance" \
       --format="table(name,zone,machineType,networkInterfaces[0].networkIP:label=INTERNAL_IP,status)"
   ```

**Expected Output:**

```
NAME                  ZONE           MACHINE_TYPE  INTERNAL_IP  STATUS
backend-instance-1    us-central1-a  e2-micro      10.1.0.2     RUNNING
backend-instance-2    us-central1-b  e2-micro      10.1.0.3     RUNNING
```

**Verification:**

- Confirm both instances are in RUNNING status
- Verify instances are in different zones within us-central1
- Check both have internal IPs in 10.1.0.0/24 range

---

### Step 9: Create Instance Group for Load Balancer Backend

**Objective:** Create an unmanaged instance group to organize backend instances.

**Instructions:**

1. Create an unmanaged instance group in us-central1-a:

   ```bash
   gcloud compute instance-groups unmanaged create ig-us-central1-a \
       --zone=us-central1-a \
       --description="Instance group for backend-instance-1"
   ```

2. Add backend-instance-1 to the instance group:

   ```bash
   gcloud compute instance-groups unmanaged add-instances ig-us-central1-a \
       --zone=us-central1-a \
       --instances=backend-instance-1
   ```

3. Create an unmanaged instance group in us-central1-b:

   ```bash
   gcloud compute instance-groups unmanaged create ig-us-central1-b \
       --zone=us-central1-b \
       --description="Instance group for backend-instance-2"
   ```

4. Add backend-instance-2 to the instance group:

   ```bash
   gcloud compute instance-groups unmanaged add-instances ig-us-central1-b \
       --zone=us-central1-b \
       --instances=backend-instance-2
   ```

5. Set named port for both instance groups:

   ```bash
   gcloud compute instance-groups unmanaged set-named-ports ig-us-central1-a \
       --zone=us-central1-a \
       --named-ports=http:80
   
   gcloud compute instance-groups unmanaged set-named-ports ig-us-central1-b \
       --zone=us-central1-b \
       --named-ports=http:80
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instanceGroups/ig-us-central1-a].
NAME               LOCATION       SCOPE  NETWORK         MANAGED  INSTANCES
ig-us-central1-a   us-central1-a  zone   custom-vpc-lab  No       1
ig-us-central1-b   us-central1-b  zone   custom-vpc-lab  No       1
```

**Verification:**

- Confirm both instance groups are created
- Verify each group contains one instance
- Check named port "http:80" is configured

---

### Step 10: Create Health Check for Load Balancer

**Objective:** Configure health check to monitor backend instance availability.

**Instructions:**

1. Create a TCP health check for the load balancer:

   ```bash
   gcloud compute health-checks create tcp health-check-tcp-80 \
       --port=80 \
       --check-interval=10s \
       --timeout=5s \
       --healthy-threshold=2 \
       --unhealthy-threshold=3 \
       --description="Health check for internal load balancer"
   ```

2. Verify the health check configuration:

   ```bash
   gcloud compute health-checks describe health-check-tcp-80
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/healthChecks/health-check-tcp-80].
NAME                   PROTOCOL
health-check-tcp-80    TCP
```

**Verification:**

- Confirm health check uses TCP protocol on port 80
- Verify check interval is 10 seconds
- Check timeout is 5 seconds

---

### Step 11: Create Backend Service for Internal Load Balancer

**Objective:** Configure the backend service that defines how traffic is distributed.

**Instructions:**

1. Create a regional backend service:

   ```bash
   gcloud compute backend-services create backend-service-internal \
       --load-balancing-scheme=INTERNAL \
       --protocol=TCP \
       --region=us-central1 \
       --health-checks=health-check-tcp-80 \
       --description="Backend service for internal load balancer"
   ```

2. Add the first instance group to the backend service:

   ```bash
   gcloud compute backend-services add-backend backend-service-internal \
       --region=us-central1 \
       --instance-group=ig-us-central1-a \
       --instance-group-zone=us-central1-a
   ```

3. Add the second instance group to the backend service:

   ```bash
   gcloud compute backend-services add-backend backend-service-internal \
       --region=us-central1 \
       --instance-group=ig-us-central1-b \
       --instance-group-zone=us-central1-b
   ```

4. Verify the backend service configuration:

   ```bash
   gcloud compute backend-services describe backend-service-internal \
       --region=us-central1 \
       --format="get(name,backends[].group)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/backendServices/backend-service-internal].
NAME                        BACKENDS
backend-service-internal    ig-us-central1-a, ig-us-central1-b
```

**Verification:**

- Confirm backend service is regional (us-central1)
- Verify both instance groups are added as backends
- Check load balancing scheme is INTERNAL

---

### Step 12: Create Forwarding Rule for Internal Load Balancer

**Objective:** Create the forwarding rule that provides the load balancer's internal IP address.

**Instructions:**

1. Create the internal forwarding rule:

   ```bash
   gcloud compute forwarding-rules create forwarding-rule-internal-lb \
       --load-balancing-scheme=INTERNAL \
       --region=us-central1 \
       --network=custom-vpc-lab \
       --subnet=subnet-us-central1 \
       --ip-protocol=TCP \
       --ports=80 \
       --backend-service=backend-service-internal \
       --backend-service-region=us-central1
   ```

2. Get the internal IP address of the load balancer:

   ```bash
   gcloud compute forwarding-rules describe forwarding-rule-internal-lb \
       --region=us-central1 \
       --format="get(IPAddress)"
   ```

3. Store the load balancer IP for later use:

   ```bash
   export LB_IP=$(gcloud compute forwarding-rules describe forwarding-rule-internal-lb \
       --region=us-central1 \
       --format="get(IPAddress)")
   echo "Load Balancer IP: $LB_IP"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/forwardingRules/forwarding-rule-internal-lb].
Load Balancer IP: 10.1.0.5
```

**Verification:**

- Confirm the forwarding rule is created
- Verify the IP address is within 10.1.0.0/24 subnet range
- Check that port 80 is configured

---

### Step 13: Create Client Instance for Testing

**Objective:** Deploy a client instance to test the internal load balancer.

**Instructions:**

1. Create a client instance in the same region:

   ```bash
   gcloud compute instances create client-instance \
       --zone=us-central1-a \
       --machine-type=e2-micro \
       --subnet=subnet-us-central1 \
       --network-tier=PREMIUM \
       --tags=client-instance \
       --image-family=debian-11 \
       --image-project=debian-cloud \
       --boot-disk-size=10GB \
       --boot-disk-type=pd-standard
   ```

2. Verify the client instance is running:

   ```bash
   gcloud compute instances describe client-instance \
       --zone=us-central1-a \
       --format="get(name,status,networkInterfaces[0].networkIP)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/client-instance].
NAME              ZONE           MACHINE_TYPE  INTERNAL_IP
client-instance   us-central1-a  e2-micro      10.1.0.4
```

**Verification:**

- Confirm instance status is "RUNNING"
- Verify internal IP is within 10.1.0.0/24 range
- Check the instance is in the same subnet as backend instances

---

### Step 14: Test Load Balancer Connectivity

**Objective:** Verify that the internal load balancer distributes traffic correctly.

**Instructions:**

1. SSH into the client instance:

   ```bash
   gcloud compute ssh client-instance --zone=us-central1-a
   ```

2. Once connected, test the load balancer multiple times:

   ```bash
   # Install curl if not available
   sudo apt-get update && sudo apt-get install -y curl
   
   # Test the load balancer 10 times
   for i in {1..10}; do
     echo "Request $i:"
     curl -s http://10.1.0.5 | grep "<h1>"
     echo ""
     sleep 1
   done
   ```

3. Verify load distribution by checking responses from different backend instances:

   ```bash
   # Count responses from each backend
   for i in {1..20}; do
     curl -s http://10.1.0.5 | grep "<h1>"
   done | sort | uniq -c
   ```

4. Exit the SSH session:

   ```bash
   exit
   ```

**Expected Output:**

```
Request 1:
<h1>Backend Instance 1 - us-central1-a</h1>

Request 2:
<h1>Backend Instance 2 - us-central1-b</h1>

Request 3:
<h1>Backend Instance 1 - us-central1-a</h1>

...

     10 <h1>Backend Instance 1 - us-central1-a</h1>
     10 <h1>Backend Instance 2 - us-central1-b</h1>
```

**Verification:**

- Confirm responses come from both backend instances
- Verify load distribution is approximately balanced
- Check that all requests receive successful HTTP 200 responses

---

### Step 15: Verify Backend Health Status

**Objective:** Confirm that health checks are passing for all backend instances.

**Instructions:**

1. Check the health status of backend instances:

   ```bash
   gcloud compute backend-services get-health backend-service-internal \
       --region=us-central1
   ```

2. View detailed health check results:

   ```bash
   gcloud compute backend-services get-health backend-service-internal \
       --region=us-central1 \
       --format="table(status.healthStatus[].instance.basename(),status.healthStatus[].healthState)"
   ```

**Expected Output:**

```
backend[0]: https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instanceGroups/ig-us-central1-a
  status:
    healthStatus:
    - healthState: HEALTHY
      instance: https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/backend-instance-1
backend[1]: https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-b/instanceGroups/ig-us-central1-b
  status:
    healthStatus:
    - healthState: HEALTHY
      instance: https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-b/instances/backend-instance-2
```

**Verification:**

- Confirm both backend instances show "HEALTHY" status
- Verify health checks are reaching both instance groups
- Check that no instances are in UNHEALTHY state

---

## Validation & Testing

### Success Criteria

- [ ] Custom VPC network "custom-vpc-lab" is created with custom subnet mode
- [ ] Two subnets exist in different regions with non-overlapping CIDR blocks
- [ ] Three firewall rules are configured (SSH, health checks, internal traffic)
- [ ] Two backend instances are running with Apache web servers
- [ ] Internal load balancer is configured with forwarding rule and backend service
- [ ] Both backend instances are healthy according to health checks
- [ ] Client instance can successfully access the load balancer IP
- [ ] Load is distributed between both backend instances
- [ ] Each backend instance returns its unique identification in responses

### Testing Procedure

1. Verify VPC and subnet configuration:

   ```bash
   gcloud compute networks describe custom-vpc-lab
   gcloud compute networks subnets list --network=custom-vpc-lab
   ```
   
   **Expected Result:** VPC exists with two subnets in us-central1 and us-east1

2. Test firewall rules:

   ```bash
   gcloud compute firewall-rules list --filter="network:custom-vpc-lab"
   ```
   
   **Expected Result:** Three firewall rules are active

3. Verify backend instance health:

   ```bash
   gcloud compute backend-services get-health backend-service-internal --region=us-central1
   ```
   
   **Expected Result:** Both instances show HEALTHY status

4. Test load balancer from client instance:

   ```bash
   gcloud compute ssh client-instance --zone=us-central1-a --command="curl -s http://10.1.0.5"
   ```
   
   **Expected Result:** HTML response from one of the backend instances

5. Verify load distribution pattern:

   ```bash
   gcloud compute ssh client-instance --zone=us-central1-a --command="for i in {1..20}; do curl -s http://10.1.0.5 | grep '<h1>'; done | sort | uniq -c"
   ```
   
   **Expected Result:** Requests distributed across both backend instances

## Troubleshooting

### Issue 1: Backend Instances Show UNHEALTHY Status

**Symptoms:**
- Health check fails for one or more backend instances
- Load balancer does not route traffic to unhealthy backends
- `gcloud compute backend-services get-health` shows UNHEALTHY state

**Cause:**
The firewall rule for health checks may not be properly configured, or Apache web server is not running on the backend instances.

**Solution:**

```bash
# Verify health check firewall rule exists
gcloud compute firewall-rules describe allow-health-check

# Check if Apache is running on backend instances
gcloud compute ssh backend-instance-1 --zone=us-central1-a --command="sudo systemctl status apache2"

# Restart Apache if needed
gcloud compute ssh backend-instance-1 --zone=us-central1-a --command="sudo systemctl restart apache2"

# Verify firewall rule allows health check source ranges
gcloud compute firewall-rules update allow-health-check \
    --source-ranges=35.191.0.0/16,130.211.0.0/22

# Wait 2-3 minutes for health checks to update
sleep 180
gcloud compute backend-services get-health backend-service-internal --region=us-central1
```

---

### Issue 2: Cannot Connect to Load Balancer from Client Instance

**Symptoms:**
- `curl` command times out when accessing load balancer IP
- Connection refused or no route to host error
- Client instance cannot reach the load balancer forwarding rule IP

**Cause:**
The internal firewall rule may not allow traffic from the client instance subnet, or the load balancer IP address is incorrect.

**Solution:**

```bash
# Verify the correct load balancer IP
gcloud compute forwarding-rules describe forwarding-rule-internal-lb \
    --region=us-central1 \
    --format="get(IPAddress)"

# Check internal firewall rule
gcloud compute firewall-rules describe allow-internal

# Update firewall rule if needed
gcloud compute firewall-rules update allow-internal \
    --source-ranges=10.1.0.0/24,10.2.0.0/24

# Test connectivity from client instance
gcloud compute ssh client-instance --zone=us-central1-a --command="ping -c 3 10.1.0.5"

# Test TCP connection to port 80
gcloud compute ssh client-instance --zone=us-central1-a --command="nc -zv 10.1.0.5 80"
```

---

### Issue 3: Load Not Distributed Evenly Between Backend Instances

**Symptoms:**
- All requests go to only one backend instance
- Load distribution is heavily skewed (e.g., 95% to one instance)
- One backend consistently receives more traffic

**Cause:**
Session affinity may be configured, or one backend instance may be handling requests faster, causing the load balancer to prefer it.

**Solution:**

```bash
# Check backend service session affinity
gcloud compute backend-services describe backend-service-internal \
    --region=us-central1 \
    --format="get(sessionAffinity)"

# Update session affinity to NONE if needed
gcloud compute backend-services update backend-service-internal \
    --region=us-central1 \
    --session-affinity=NONE

# Verify both backends have equal capacity
gcloud compute backend-services describe backend-service-internal \
    --region=us-central1 \
    --format="get(backends[])"

# Test load distribution again
gcloud compute ssh client-instance --zone=us-central1-a --command="for i in {1..50}; do curl -s http://10.1.0.5 | grep '<h1>'; done | sort | uniq -c"
```

---

### Issue 4: Quota Exceeded Error When Creating Resources

**Symptoms:**
- Error message: "Quota 'CPUS' exceeded. Limit: X in region us-central1"
- Cannot create new VM instances
- Resource creation fails with quota error

**Cause:**
Your GCP project has reached the resource quota limit for the region.

**Solution:**

```bash
# Check current quota usage
gcloud compute project-info describe --project=$PROJECT_ID

# View region-specific quotas
gcloud compute regions describe us-central1

# Use smaller machine types
gcloud compute instances create backend-instance-1 \
    --machine-type=e2-micro  # Smallest available machine type

# Delete unused resources to free quota
gcloud compute instances delete unused-instance --zone=us-central1-a --quiet

# Request quota increase (if needed for production)
# Navigate to: IAM & Admin > Quotas in Cloud Console
```

---

## Cleanup

**Important:** Execute these cleanup steps to avoid ongoing charges for resources created in this lab.

```bash
# Delete the client instance
gcloud compute instances delete client-instance \
    --zone=us-central1-a \
    --quiet

# Delete the forwarding rule (load balancer frontend)
gcloud compute forwarding-rules delete forwarding-rule-internal-lb \
    --region=us-central1 \
    --quiet

# Delete the backend service
gcloud compute backend-services delete backend-service-internal \
    --region=us-central1 \
    --quiet

# Delete the health check
gcloud compute health-checks delete health-check-tcp-80 \
    --quiet

# Delete instance groups
gcloud compute instance-groups unmanaged delete ig-us-central1-a \
    --zone=us-central1-a \
    --quiet

gcloud compute instance-groups unmanaged delete ig-us-central1-b \
    --zone=us-central1-b \
    --quiet

# Delete backend instances
gcloud compute instances delete backend-instance-1 \
    --zone=us-central1-a \
    --quiet

gcloud compute instances delete backend-instance-2 \
    --zone=us-central1-b \
    --quiet

# Delete firewall rules
gcloud compute firewall-rules delete allow-internal \
    --quiet

gcloud compute firewall-rules delete allow-health-check \
    --quiet

gcloud compute firewall-rules delete allow-ssh-iap \
    --quiet

# Delete subnets
gcloud compute networks subnets delete subnet-us-central1 \
    --region=us-central1 \
    --quiet

gcloud compute networks subnets delete subnet-us-east1 \
    --region=us-east1 \
    --quiet

# Delete the VPC network
gcloud compute networks delete custom-vpc-lab \
    --quiet

# Verify all resources are deleted
gcloud compute instances list
gcloud compute networks list
```

> ⚠️ **Warning:** Deleting resources is permanent and cannot be undone. Ensure you have saved any important data or configurations before executing cleanup commands. Resources like VM instances and load balancers incur charges even when idle, so complete cleanup promptly after lab completion.

> 💡 **Tip:** You can verify that all resources have been deleted by checking the Google Cloud Console under **VPC network**, **Compute Engine**, and **Network services** sections.

## Summary

### What You Accomplished

- Created a custom VPC network with two regional subnets using non-overlapping CIDR blocks
- Implemented comprehensive firewall rules following security best practices for SSH, health checks, and internal communication
- Deployed multiple backend instances across different zones with Apache web servers
- Configured an internal TCP/UDP load balancer with health checks and backend services
- Organized instances into unmanaged instance groups for load balancer integration
- Successfully tested load distribution and verified that traffic is balanced across backend instances
- Validated health check functionality and backend instance availability
- Demonstrated internal networking concepts without exposing services to the public internet

### Key Takeaways

- **Custom VPC networks** provide complete control over IP addressing and subnet design, unlike default networks
- **Regional subnets** allow resources in multiple zones to communicate within the same IP range
- **Firewall rules** should follow the principle of least privilege, only allowing necessary traffic sources and destinations
- **Internal load balancers** distribute traffic within a VPC without requiring external IP addresses, improving security
- **Health checks** are critical for ensuring traffic is only routed to healthy backend instances
- **Instance groups** organize instances for load balancer backends and enable scalability
- **Proper cleanup** is essential to avoid unexpected charges from idle resources

### Next Steps

- Proceed to **Lab 02-10-01** to integrate Cloud SQL database with this VPC infrastructure
- Explore **managed instance groups** with autoscaling for production-grade deployments
- Learn about **Cloud NAT** to enable outbound internet access for instances without external IPs
- Study **VPC peering** and **Shared VPC** for multi-project networking architectures
- Investigate **Cloud Load Balancing** options including HTTP(S), SSL Proxy, and TCP Proxy load balancers
- Review **VPC Flow Logs** for network monitoring and troubleshooting
- Consider implementing **Cloud Armor** security policies for DDoS protection

## Additional Resources

- [Google Cloud VPC Documentation](https://cloud.google.com/vpc/docs) - Comprehensive guide to Virtual Private Cloud networking
- [Internal TCP/UDP Load Balancing Overview](https://cloud.google.com/load-balancing/docs/internal) - Detailed documentation on internal load balancers
- [VPC Firewall Rules](https://cloud.google.com/vpc/docs/firewalls) - Best practices for configuring firewall rules
- [Health Check Concepts](https://cloud.google.com/load-balancing/docs/health-checks) - Understanding health check configuration and troubleshooting
- [Google Cloud Networking Best Practices](https://cloud.google.com/architecture/best-practices-vpc-design) - Architecture guidance for VPC design
- [IP Addressing and CIDR Notation](https://cloud.google.com/vpc/docs/vpc#manually_created_subnet_ip_ranges) - Guide to planning IP address ranges
- [Google Cloud Free Tier](https://cloud.google.com/free) - Information about free tier limits and trial credits
- [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference) - Complete command reference for Google Cloud SDK

---

# Lab 02-10-01: Práctica 5: Implementar Cloud SQL y conectarlo con una VM de aplicación

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

In this lab, you will deploy a fully managed Cloud SQL database instance and establish secure connectivity with a Compute Engine application server. You'll configure network settings, implement authentication using Cloud SQL Proxy, create database schemas, and execute queries to verify complete database functionality. This hands-on exercise demonstrates real-world patterns for deploying secure, scalable database solutions in Google Cloud Platform, essential for modern cloud-native applications.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Provision a Cloud SQL instance with appropriate machine type, storage, and high availability configuration
- [ ] Configure network connectivity between Cloud SQL and Compute Engine using authorized networks
- [ ] Implement secure authentication using Cloud SQL Proxy for encrypted connections
- [ ] Create database schemas, tables, and sample data for application testing
- [ ] Deploy an application VM that successfully connects to and queries the Cloud SQL database
- [ ] Verify database connectivity and execute CRUD operations from the application VM

## Prerequisites

### Required Knowledge

- Basic SQL knowledge (SELECT, INSERT, UPDATE, DELETE statements)
- Understanding of relational database concepts (tables, schemas, primary keys)
- Familiarity with VPC networking from previous labs
- Basic Linux command-line skills (navigation, file editing, package management)
- Understanding of database client-server architecture

### Required Access

- Google Cloud Platform account with active billing or free trial credits
- Project Editor role or specific roles: Cloud SQL Admin, Compute Admin, Service Account User
- Completed Lab 02-09-01 (VPC networking fundamentals) or existing VPC network

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| Internet Connection | Minimum 10 Mbps bandwidth |
| Display | Minimum 1366x768 resolution |
| RAM | Minimum 8GB |
| Processor | Dual-core or better |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud Console | Latest web version | Primary interface for managing GCP resources |
| Google Cloud SDK (gcloud CLI) | 400.0.0+ | Command-line interface for GCP operations |
| Google Chrome or Firefox | Latest stable release | Web browser for accessing GCP Console |
| SSH Client | OpenSSH 7.0+ or PuTTY 0.70+ | Secure connection to VM instances |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Set default region and zone
export REGION="us-central1"
export ZONE="us-central1-a"
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

# Enable required APIs
gcloud services enable sqladmin.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable servicenetworking.googleapis.com
```

## Step-by-Step Instructions

### Step 1: Create a Cloud SQL MySQL Instance

**Objective:** Provision a managed MySQL database instance with appropriate configuration for development and testing.

**Instructions:**

1. Create the Cloud SQL instance using gcloud CLI:
   
   ```bash
   gcloud sql instances create lab-mysql-instance \
     --database-version=MYSQL_8_0 \
     --tier=db-f1-micro \
     --region=$REGION \
     --root-password=SecureP@ssw0rd123 \
     --backup-start-time=03:00 \
     --enable-bin-log \
     --storage-type=SSD \
     --storage-size=10GB
   ```

2. Wait for the instance to be created (this may take 5-10 minutes). Monitor the creation status:

   ```bash
   gcloud sql instances describe lab-mysql-instance --format="value(state)"
   ```

3. Retrieve the instance connection name:

   ```bash
   gcloud sql instances describe lab-mysql-instance --format="value(connectionName)"
   ```

**Expected Output:**

```
Creating Cloud SQL instance...done.
Created [https://sqladmin.googleapis.com/sql/v1beta4/projects/PROJECT_ID/instances/lab-mysql-instance].
NAME                  DATABASE_VERSION  LOCATION       TIER         PRIMARY_ADDRESS  PRIVATE_ADDRESS  STATUS
lab-mysql-instance    MYSQL_8_0         us-central1-a  db-f1-micro  34.XX.XXX.XXX    -                RUNNABLE
```

**Verification:**

- Confirm the instance status shows "RUNNABLE"
- Note the connection name format: `PROJECT_ID:REGION:lab-mysql-instance`

---

### Step 2: Create a Database and User

**Objective:** Set up a dedicated database and user account with appropriate privileges for application access.

**Instructions:**

1. Create a new database named `appdb`:
   
   ```bash
   gcloud sql databases create appdb --instance=lab-mysql-instance
   ```

2. Create a database user for application connections:

   ```bash
   gcloud sql users create appuser \
     --instance=lab-mysql-instance \
     --password=AppUser@2024
   ```

3. List databases to verify creation:

   ```bash
   gcloud sql databases list --instance=lab-mysql-instance
   ```

**Expected Output:**

```
Creating Cloud SQL database...done.
Created database [appdb].

Creating Cloud SQL user...done.
Created user [appuser].

NAME
appdb
information_schema
mysql
performance_schema
sys
```

**Verification:**

- Confirm `appdb` appears in the database list
- Verify user `appuser` was created successfully

---

### Step 3: Configure Network Access for Cloud SQL

**Objective:** Enable secure network connectivity by authorizing the application VM's IP address.

**Instructions:**

1. Get your current public IP address (for initial testing):
   
   ```bash
   curl -s https://api.ipify.org
   ```

2. Authorize your current IP address to connect to Cloud SQL:

   ```bash
   MY_IP=$(curl -s https://api.ipify.org)
   gcloud sql instances patch lab-mysql-instance \
     --authorized-networks=$MY_IP/32
   ```

3. Retrieve the Cloud SQL instance's public IP address:

   ```bash
   gcloud sql instances describe lab-mysql-instance --format="value(ipAddresses[0].ipAddress)"
   ```

**Expected Output:**

```
35.XXX.XXX.XXX

Patching Cloud SQL instance...done.
Updated [https://sqladmin.googleapis.com/sql/v1beta4/projects/PROJECT_ID/instances/lab-mysql-instance].
```

**Verification:**

- Note the Cloud SQL public IP address for later use
- Confirm the authorized networks list includes your IP address

---

### Step 4: Create a Compute Engine VM Instance

**Objective:** Deploy an application server VM that will connect to the Cloud SQL database.

**Instructions:**

1. Create a Compute Engine VM instance:
   
   ```bash
   gcloud compute instances create app-server-vm \
     --zone=$ZONE \
     --machine-type=e2-micro \
     --image-family=debian-11 \
     --image-project=debian-cloud \
     --boot-disk-size=10GB \
     --boot-disk-type=pd-standard \
     --tags=app-server \
     --metadata=startup-script='#!/bin/bash
     apt-get update
     apt-get install -y wget default-mysql-client'
   ```

2. Wait for the VM to be created and running:

   ```bash
   gcloud compute instances describe app-server-vm --zone=$ZONE --format="value(status)"
   ```

3. Get the external IP address of the VM:

   ```bash
   gcloud compute instances describe app-server-vm --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/zones/us-central1-a/instances/app-server-vm].
NAME            ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
app-server-vm   us-central1-a  e2-micro                   10.X.X.X     34.XX.XXX.XXX  RUNNING
```

**Verification:**

- Confirm the VM status shows "RUNNING"
- Note the external IP address of the VM

---

### Step 5: Authorize the Application VM to Access Cloud SQL

**Objective:** Add the application VM's IP address to Cloud SQL authorized networks for secure database access.

**Instructions:**

1. Get the application VM's external IP address:
   
   ```bash
   APP_VM_IP=$(gcloud compute instances describe app-server-vm --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
   echo "App VM IP: $APP_VM_IP"
   ```

2. Add the VM's IP to Cloud SQL authorized networks:

   ```bash
   gcloud sql instances patch lab-mysql-instance \
     --authorized-networks=$MY_IP/32,$APP_VM_IP/32
   ```

3. Verify the authorized networks configuration:

   ```bash
   gcloud sql instances describe lab-mysql-instance --format="value(settings.ipConfiguration.authorizedNetworks)"
   ```

**Expected Output:**

```
App VM IP: 34.XX.XXX.XXX

Patching Cloud SQL instance...done.

[{'name': 'my-ip', 'value': '203.XX.XX.XX/32'}, {'name': 'app-vm', 'value': '34.XX.XXX.XXX/32'}]
```

**Verification:**

- Confirm both your IP and the VM's IP are listed in authorized networks
- This allows direct connections from the VM to Cloud SQL

---

### Step 6: Install and Configure Cloud SQL Proxy on the VM

**Objective:** Set up Cloud SQL Proxy for secure, encrypted connections to the database without managing SSL certificates.

**Instructions:**

1. SSH into the application VM:
   
   ```bash
   gcloud compute ssh app-server-vm --zone=$ZONE
   ```

2. Once connected to the VM, download and install Cloud SQL Proxy:

   ```bash
   wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
   chmod +x cloud_sql_proxy
   sudo mv cloud_sql_proxy /usr/local/bin/
   ```

3. Verify Cloud SQL Proxy installation:

   ```bash
   cloud_sql_proxy --version
   ```

4. Get the Cloud SQL instance connection name (run this in Cloud Shell, not the VM):

   ```bash
   # Exit the VM first (type 'exit' or open a new Cloud Shell tab)
   gcloud sql instances describe lab-mysql-instance --format="value(connectionName)"
   ```

5. Return to the VM SSH session and start Cloud SQL Proxy in the background:

   ```bash
   # Replace PROJECT_ID:REGION:INSTANCE_NAME with your actual connection name
   cloud_sql_proxy -instances=PROJECT_ID:us-central1:lab-mysql-instance=tcp:3306 &
   ```

**Expected Output:**

```
Cloud SQL Auth proxy v1.33.2
Listening on 127.0.0.1:3306 for PROJECT_ID:us-central1:lab-mysql-instance
Ready for new connections
```

**Verification:**

- Confirm the proxy is listening on port 3306
- Check the process is running: `ps aux | grep cloud_sql_proxy`

---

### Step 7: Connect to Cloud SQL and Create Database Schema

**Objective:** Establish a database connection and create sample tables for application testing.

**Instructions:**

1. From the VM, connect to MySQL using the Cloud SQL Proxy:
   
   ```bash
   mysql -u appuser -p'AppUser@2024' -h 127.0.0.1 -P 3306 appdb
   ```

2. Once connected to MySQL, create a sample table:

   ```sql
   CREATE TABLE employees (
     id INT AUTO_INCREMENT PRIMARY KEY,
     first_name VARCHAR(50) NOT NULL,
     last_name VARCHAR(50) NOT NULL,
     email VARCHAR(100) UNIQUE NOT NULL,
     department VARCHAR(50),
     hire_date DATE,
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );
   ```

3. Verify the table was created:

   ```sql
   SHOW TABLES;
   DESCRIBE employees;
   ```

4. Insert sample data:

   ```sql
   INSERT INTO employees (first_name, last_name, email, department, hire_date)
   VALUES
     ('John', 'Doe', 'john.doe@example.com', 'Engineering', '2024-01-15'),
     ('Jane', 'Smith', 'jane.smith@example.com', 'Marketing', '2024-02-20'),
     ('Mike', 'Johnson', 'mike.johnson@example.com', 'Sales', '2024-03-10');
   ```

5. Query the data to verify insertion:

   ```sql
   SELECT * FROM employees;
   ```

**Expected Output:**

```
mysql> CREATE TABLE employees ...
Query OK, 0 rows affected (0.12 sec)

mysql> SHOW TABLES;
+------------------+
| Tables_in_appdb  |
+------------------+
| employees        |
+------------------+
1 row in set (0.00 sec)

mysql> INSERT INTO employees ...
Query OK, 3 rows affected (0.05 sec)
Records: 3  Duplicates: 0  Warnings: 0

mysql> SELECT * FROM employees;
+----+------------+-----------+---------------------------+-------------+------------+---------------------+
| id | first_name | last_name | email                     | department  | hire_date  | created_at          |
+----+------------+-----------+---------------------------+-------------+------------+---------------------+
|  1 | John       | Doe       | john.doe@example.com      | Engineering | 2024-01-15 | 2024-XX-XX XX:XX:XX |
|  2 | Jane       | Smith     | jane.smith@example.com    | Marketing   | 2024-02-20 | 2024-XX-XX XX:XX:XX |
|  3 | Mike       | Johnson   | mike.johnson@example.com  | Sales       | 2024-03-10 | 2024-XX-XX XX:XX:XX |
+----+------------+-----------+---------------------------+-------------+------------+---------------------+
3 rows in set (0.00 sec)
```

**Verification:**

- Confirm the table structure matches the CREATE statement
- Verify all three employee records are displayed
- Note the auto-generated `id` and `created_at` values

---

### Step 8: Test CRUD Operations

**Objective:** Execute Create, Read, Update, and Delete operations to verify full database functionality.

**Instructions:**

1. Still in the MySQL session, perform an UPDATE operation:
   
   ```sql
   UPDATE employees
   SET department = 'Senior Engineering'
   WHERE email = 'john.doe@example.com';
   ```

2. Verify the update:

   ```sql
   SELECT * FROM employees WHERE email = 'john.doe@example.com';
   ```

3. Perform a DELETE operation:

   ```sql
   DELETE FROM employees WHERE id = 3;
   ```

4. Verify the deletion:

   ```sql
   SELECT COUNT(*) as total_employees FROM employees;
   SELECT * FROM employees;
   ```

5. Test a complex query with filtering and sorting:

   ```sql
   SELECT first_name, last_name, department, hire_date
   FROM employees
   WHERE hire_date >= '2024-01-01'
   ORDER BY hire_date DESC;
   ```

6. Exit the MySQL client:

   ```sql
   EXIT;
   ```

**Expected Output:**

```
mysql> UPDATE employees ...
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> SELECT * FROM employees WHERE email = 'john.doe@example.com';
+----+------------+-----------+----------------------+---------------------+------------+---------------------+
| id | first_name | last_name | email                | department          | hire_date  | created_at          |
+----+------------+-----------+----------------------+---------------------+------------+---------------------+
|  1 | John       | Doe       | john.doe@example.com | Senior Engineering  | 2024-01-15 | 2024-XX-XX XX:XX:XX |
+----+------------+-----------+----------------------+---------------------+------------+---------------------+

mysql> DELETE FROM employees WHERE id = 3;
Query OK, 1 row affected (0.02 sec)

mysql> SELECT COUNT(*) as total_employees FROM employees;
+------------------+
| total_employees  |
+------------------+
|                2 |
+------------------+
```

**Verification:**

- Confirm John Doe's department was updated to "Senior Engineering"
- Verify only 2 employees remain after deletion
- Ensure the complex query returns results sorted by hire date

---

### Step 9: Create a Simple Python Application

**Objective:** Deploy a basic Python application that connects to Cloud SQL and performs database operations.

**Instructions:**

1. Still connected to the VM, install Python and MySQL connector:
   
   ```bash
   sudo apt-get update
   sudo apt-get install -y python3 python3-pip
   pip3 install mysql-connector-python
   ```

2. Create a Python script to connect to the database:

   ```bash
   cat > db_app.py << 'EOF'
   #!/usr/bin/env python3
   import mysql.connector
   from mysql.connector import Error
   
   def create_connection():
       try:
           connection = mysql.connector.connect(
               host='127.0.0.1',
               port=3306,
               database='appdb',
               user='appuser',
               password='AppUser@2024'
           )
           if connection.is_connected():
               print("Successfully connected to Cloud SQL database")
               return connection
       except Error as e:
           print(f"Error connecting to database: {e}")
           return None
   
   def list_employees(connection):
       try:
           cursor = connection.cursor(dictionary=True)
           cursor.execute("SELECT * FROM employees")
           employees = cursor.fetchall()
           
           print("\n=== Employee List ===")
           for emp in employees:
               print(f"ID: {emp['id']}, Name: {emp['first_name']} {emp['last_name']}, "
                     f"Department: {emp['department']}, Email: {emp['email']}")
           
           cursor.close()
       except Error as e:
           print(f"Error fetching employees: {e}")
   
   def add_employee(connection, first_name, last_name, email, department, hire_date):
       try:
           cursor = connection.cursor()
           query = """INSERT INTO employees (first_name, last_name, email, department, hire_date)
                      VALUES (%s, %s, %s, %s, %s)"""
           values = (first_name, last_name, email, department, hire_date)
           cursor.execute(query, values)
           connection.commit()
           print(f"Successfully added employee: {first_name} {last_name}")
           cursor.close()
       except Error as e:
           print(f"Error adding employee: {e}")
   
   if __name__ == "__main__":
       conn = create_connection()
       if conn:
           list_employees(conn)
           add_employee(conn, "Sarah", "Williams", "sarah.williams@example.com", 
                       "HR", "2024-04-01")
           list_employees(conn)
           conn.close()
           print("\nDatabase connection closed.")
   EOF
   ```

3. Make the script executable and run it:

   ```bash
   chmod +x db_app.py
   python3 db_app.py
   ```

**Expected Output:**

```
Successfully connected to Cloud SQL database

=== Employee List ===
ID: 1, Name: John Doe, Department: Senior Engineering, Email: john.doe@example.com
ID: 2, Name: Jane Smith, Department: Marketing, Email: jane.smith@example.com

Successfully added employee: Sarah Williams

=== Employee List ===
ID: 1, Name: John Doe, Department: Senior Engineering, Email: john.doe@example.com
ID: 2, Name: Jane Smith, Department: Marketing, Email: jane.smith@example.com
ID: 4, Name: Sarah Williams, Department: HR, Email: sarah.williams@example.com

Database connection closed.
```

**Verification:**

- Confirm the Python script successfully connects to Cloud SQL
- Verify the new employee (Sarah Williams) is added to the database
- Ensure the script displays all employees before and after insertion

---

### Step 10: Monitor Cloud SQL Performance

**Objective:** Review database metrics and performance indicators in the Cloud Console.

**Instructions:**

1. Exit the VM SSH session:
   
   ```bash
   exit
   ```

2. In Cloud Console, navigate to **SQL** > **lab-mysql-instance**

3. Click on the **Monitoring** tab to view metrics

4. Use gcloud to retrieve instance operations:

   ```bash
   gcloud sql operations list --instance=lab-mysql-instance --limit=5
   ```

5. Check current database connections:

   ```bash
   gcloud sql instances describe lab-mysql-instance \
     --format="value(settings.ipConfiguration.authorizedNetworks)"
   ```

6. View backup configuration:

   ```bash
   gcloud sql instances describe lab-mysql-instance \
     --format="value(settings.backupConfiguration)"
   ```

**Expected Output:**

```
OPERATION                                    TYPE    START                          END                            ERROR  STATUS
operation-id-xxxxxxxxxxxxx                   UPDATE  2024-XX-XXTXX:XX:XX.XXXZ       2024-XX-XXTXX:XX:XX.XXXZ       -      DONE
operation-id-xxxxxxxxxxxxx                   CREATE  2024-XX-XXTXX:XX:XX.XXXZ       2024-XX-XXTXX:XX:XX.XXXZ       -      DONE

backupRetentionSettings:
  retainedBackups: 7
  retentionUnit: COUNT
enabled: true
startTime: '03:00'
transactionLogRetentionDays: 7
```

**Verification:**

- Confirm operations show successful CREATE and UPDATE operations
- Verify automated backups are enabled with retention policy
- Review connection metrics in the Cloud Console

---

## Validation & Testing

### Success Criteria

- [ ] Cloud SQL instance is created and in RUNNABLE state
- [ ] Database `appdb` and user `appuser` are successfully created
- [ ] Application VM can connect to Cloud SQL using Cloud SQL Proxy
- [ ] Sample table `employees` is created with correct schema
- [ ] All CRUD operations (Create, Read, Update, Delete) execute successfully
- [ ] Python application successfully connects and performs database operations
- [ ] Authorized networks are properly configured for secure access
- [ ] Database metrics are visible in Cloud Console monitoring

### Testing Procedure

1. Verify Cloud SQL instance status:
   ```bash
   gcloud sql instances describe lab-mysql-instance --format="value(state)"
   ```
   **Expected Result:** `RUNNABLE`

2. Test database connectivity from the VM:
   ```bash
   gcloud compute ssh app-server-vm --zone=$ZONE --command="mysql -u appuser -p'AppUser@2024' -h 127.0.0.1 -e 'SELECT COUNT(*) FROM appdb.employees;'"
   ```
   **Expected Result:** Returns a count of employees (should be 3 or more)

3. Verify Cloud SQL Proxy is running on the VM:
   ```bash
   gcloud compute ssh app-server-vm --zone=$ZONE --command="ps aux | grep cloud_sql_proxy | grep -v grep"
   ```
   **Expected Result:** Shows the cloud_sql_proxy process running

4. Test Python application execution:
   ```bash
   gcloud compute ssh app-server-vm --zone=$ZONE --command="python3 /home/$USER/db_app.py"
   ```
   **Expected Result:** Successfully connects and displays employee list

5. Verify backup configuration:
   ```bash
   gcloud sql backups list --instance=lab-mysql-instance
   ```
   **Expected Result:** Shows automated backup schedule

## Troubleshooting

### Issue 1: Cloud SQL Instance Creation Fails

**Symptoms:**
- Error message: "The instance or operation is not in an appropriate state"
- Instance stuck in "PENDING_CREATE" state
- Quota exceeded errors

**Cause:**
Project quota limits reached, API not enabled, or insufficient permissions.

**Solution:**
```bash
# Check if SQL Admin API is enabled
gcloud services list --enabled | grep sqladmin

# Enable the API if not already enabled
gcloud services enable sqladmin.googleapis.com

# Check project quotas
gcloud compute project-info describe --project=$PROJECT_ID

# If quota issue, request increase or delete unused instances
gcloud sql instances list
gcloud sql instances delete OLD_INSTANCE_NAME
```

---

### Issue 2: Cannot Connect to Cloud SQL from VM

**Symptoms:**
- Connection timeout errors
- "ERROR 2003 (HY000): Can't connect to MySQL server"
- Cloud SQL Proxy fails to start

**Cause:**
VM IP not authorized, Cloud SQL Proxy not running, or incorrect connection parameters.

**Solution:**
```bash
# Verify VM IP is in authorized networks
gcloud sql instances describe lab-mysql-instance \
  --format="value(settings.ipConfiguration.authorizedNetworks)"

# Add VM IP if missing
APP_VM_IP=$(gcloud compute instances describe app-server-vm --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
gcloud sql instances patch lab-mysql-instance \
  --authorized-networks=$APP_VM_IP/32

# On the VM, restart Cloud SQL Proxy
pkill cloud_sql_proxy
CONNECTION_NAME=$(gcloud sql instances describe lab-mysql-instance --format="value(connectionName)")
cloud_sql_proxy -instances=$CONNECTION_NAME=tcp:3306 &

# Test connectivity
mysql -u appuser -p'AppUser@2024' -h 127.0.0.1 -e "SELECT 1;"
```

---

### Issue 3: Authentication Failed for Database User

**Symptoms:**
- "ERROR 1045 (28000): Access denied for user 'appuser'@'localhost'"
- Password authentication failures
- User not found errors

**Cause:**
Incorrect password, user not created, or user lacks necessary privileges.

**Solution:**
```bash
# Verify user exists
gcloud sql users list --instance=lab-mysql-instance

# Reset user password
gcloud sql users set-password appuser \
  --instance=lab-mysql-instance \
  --password=AppUser@2024

# Connect as root to grant privileges
gcloud sql connect lab-mysql-instance --user=root

# In MySQL prompt, grant privileges
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
EXIT;
```

---

### Issue 4: Cloud SQL Proxy Not Starting on VM

**Symptoms:**
- "Error parsing service account key" message
- "Failed to create oauth2 client" error
- Proxy exits immediately after starting

**Cause:**
VM service account lacks necessary IAM permissions or Cloud SQL Proxy not properly installed.

**Solution:**
```bash
# Grant Cloud SQL Client role to the VM's service account
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# On the VM, reinstall Cloud SQL Proxy
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy
chmod +x cloud_sql_proxy
sudo mv cloud_sql_proxy /usr/local/bin/

# Start proxy with explicit credentials
CONNECTION_NAME=$(gcloud sql instances describe lab-mysql-instance --format="value(connectionName)")
cloud_sql_proxy -instances=$CONNECTION_NAME=tcp:3306 &

# Verify proxy is listening
netstat -tlnp | grep 3306
```

---

### Issue 5: Python Application Cannot Import mysql.connector

**Symptoms:**
- "ModuleNotFoundError: No module named 'mysql'"
- Import errors when running Python script
- pip installation failures

**Cause:**
MySQL connector library not installed or installed for wrong Python version.

**Solution:**
```bash
# On the VM, ensure pip3 is installed
sudo apt-get update
sudo apt-get install -y python3-pip

# Install MySQL connector for Python 3
pip3 install mysql-connector-python

# Verify installation
python3 -c "import mysql.connector; print(mysql.connector.__version__)"

# If still failing, use system package
sudo apt-get install -y python3-mysql.connector

# Run script with explicit Python 3
python3 db_app.py
```

---

## Cleanup

**Important:** Cloud SQL instances incur charges even when idle. Complete cleanup immediately after finishing the lab to avoid unexpected costs.

```bash
# Stop Cloud SQL Proxy on the VM (if still connected via SSH)
pkill cloud_sql_proxy

# Exit VM SSH session
exit

# Delete the Compute Engine VM instance
gcloud compute instances delete app-server-vm \
  --zone=$ZONE \
  --quiet

# Delete the Cloud SQL instance (THIS CANNOT BE UNDONE)
gcloud sql instances delete lab-mysql-instance --quiet

# Verify deletion
gcloud sql instances list
gcloud compute instances list --filter="name=app-server-vm"
```

> ⚠️ **Warning:** Deleting the Cloud SQL instance is permanent and cannot be undone. All data, backups, and configurations will be lost. Ensure you have exported any data you need to retain before deletion.

> 💰 **Cost Note:** Cloud SQL instances are billed per hour of operation. A db-f1-micro instance costs approximately $0.015/hour. Always delete instances when not in use to minimize costs.

**Additional Cleanup Steps:**

```bash
# Remove authorized networks (if keeping instance for later)
gcloud sql instances patch lab-mysql-instance \
  --clear-authorized-networks

# List and delete any remaining backups
gcloud sql backups list --instance=lab-mysql-instance
# Note: Backups are automatically deleted when instance is deleted

# Remove IAM policy bindings (optional)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud projects remove-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

## Summary

### What You Accomplished

- Successfully provisioned a fully managed Cloud SQL MySQL instance with automated backups and high availability configuration
- Configured secure network connectivity using authorized networks and Cloud SQL Proxy for encrypted connections
- Created a database schema with tables, inserted sample data, and executed all CRUD operations
- Deployed a Compute Engine application server VM and established database connectivity
- Developed and tested a Python application that performs database operations programmatically
- Monitored database performance metrics and verified backup configurations
- Implemented security best practices including strong passwords, authorized networks, and encrypted connections

### Key Takeaways

- **Cloud SQL provides fully managed database services** eliminating the need for manual installation, patching, and backup management
- **Cloud SQL Proxy offers secure connectivity** without managing SSL certificates or opening firewall ports directly to the database
- **Authorized networks control access** by restricting database connections to specific IP addresses or ranges
- **Automated backups and point-in-time recovery** protect data and enable disaster recovery scenarios
- **Integration with Compute Engine** allows seamless application-to-database connectivity within Google Cloud
- **Performance monitoring** through Cloud Console provides visibility into database operations and resource utilization
- **Cost management is critical** as Cloud SQL instances incur charges continuously; always clean up resources after testing

### Next Steps

- Explore Cloud SQL high availability (HA) configuration with failover replicas
- Implement Cloud SQL read replicas for scaling read operations
- Configure VPC peering for private IP connectivity (more secure than public IP)
- Set up Cloud SQL backup and restore procedures for disaster recovery
- Integrate Cloud SQL with Cloud Functions for serverless application architectures
- Implement connection pooling for improved application performance
- Configure Cloud Monitoring alerts for database performance thresholds
- Proceed to Lab 03-08-01: Implementing load balancing with SSL certificates

## Additional Resources

- [Cloud SQL Documentation](https://cloud.google.com/sql/docs) - Official Google Cloud SQL documentation and guides
- [Cloud SQL Proxy Overview](https://cloud.google.com/sql/docs/mysql/sql-proxy) - Detailed information on Cloud SQL Proxy authentication and usage
- [Best Practices for Cloud SQL](https://cloud.google.com/sql/docs/mysql/best-practices) - Performance, security, and operational recommendations
- [Cloud SQL Pricing Calculator](https://cloud.google.com/products/calculator) - Estimate costs for different instance configurations
- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/) - Complete MySQL database documentation
- [Python MySQL Connector Documentation](https://dev.mysql.com/doc/connector-python/en/) - Python database connectivity library reference
- [Cloud SQL High Availability Configuration](https://cloud.google.com/sql/docs/mysql/high-availability) - Setting up HA and failover
- [Cloud SQL Security Best Practices](https://cloud.google.com/sql/docs/mysql/security-best-practices) - Comprehensive security guidelines
- [Connecting from Compute Engine](https://cloud.google.com/sql/docs/mysql/connect-compute-engine) - Detailed connectivity patterns and examples
- [Cloud SQL Backup and Recovery](https://cloud.google.com/sql/docs/mysql/backup-recovery/backing-up) - Backup strategies and restoration procedures
