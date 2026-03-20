# Lab 03-07-01: Práctica 6: Crear grupo administrado con autoescalado y health checks

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Medium |
| **Bloom Level** | Understand |

## Overview

This lab guides you through creating a regional Managed Instance Group (MIG) with automated scaling and health checks in Google Compute Engine. You will deploy a fleet of web servers that automatically scale based on CPU utilization and self-heal when instances become unhealthy. This practice demonstrates production-grade infrastructure patterns for high availability and elasticity, essential for running resilient applications in the cloud.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create an instance template with a startup script that deploys a web application
- [ ] Deploy a regional Managed Instance Group with autoscaling and autohealing policies
- [ ] Configure HTTP health checks to monitor instance availability
- [ ] Simulate load to trigger automatic scaling and observe the behavior
- [ ] Validate autohealing by forcing instance failures

## Prerequisites

### Required Knowledge

- Understanding of Google Compute Engine instances and their lifecycle
- Basic knowledge of Linux commands and package management
- Familiarity with HTTP protocols and web servers (Nginx)
- Concepts of autoscaling and load balancing fundamentals

### Required Access

- Google Cloud Project with billing enabled
- IAM role: Compute Admin (roles/compute.admin) or equivalent permissions
- APIs enabled: Compute Engine API

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| RAM | 8 GB minimum |
| CPU | 2 cores minimum |
| Network | 10 Mbps stable Internet connection |
| Browser | Chrome, Firefox, or Edge (current version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage Compute Engine resources |
| Google Cloud Shell | Latest | Terminal environment with pre-installed tools |
| curl | 7.81+ | Test HTTP endpoints |
| Apache Bench (ab) | 2.3+ | Generate HTTP load for autoscaling tests |

### Initial Setup

```bash
# Set your project ID
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1
export ZONE1=us-central1-a
export ZONE2=us-central1-b

# Enable required APIs
gcloud services enable compute.googleapis.com

# Verify project configuration
echo "Project: $PROJECT_ID"
echo "Region: $REGION"
```

## Step-by-Step Instructions

### Step 1: Create the Instance Template with Startup Script

**Objective:** Build a reusable instance template that automatically installs and configures Nginx to serve instance metadata.

**Instructions:**

1. Create a startup script file that will install Nginx and serve instance metadata:

   ```bash
   cat > startup-script.sh << 'EOF'
   #!/bin/bash
   
   # Update package lists
   apt-get update
   
   # Install Nginx
   apt-get install -y nginx
   
   # Get instance metadata
   INSTANCE_NAME=$(curl -H "Metadata-Flavor: Google" \
     http://metadata.google.internal/computeMetadata/v1/instance/name)
   INSTANCE_ZONE=$(curl -H "Metadata-Flavor: Google" \
     http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d'/' -f4)
   INSTANCE_ID=$(curl -H "Metadata-Flavor: Google" \
     http://metadata.google.internal/computeMetadata/v1/instance/id)
   
   # Create custom index page
   cat > /var/www/html/index.html << HTML
   <!DOCTYPE html>
   <html>
   <head>
       <title>MIG Instance</title>
       <style>
           body { font-family: Arial, sans-serif; margin: 40px; background: #f0f0f0; }
           .container { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
           h1 { color: #4285f4; }
           .metadata { background: #e8f0fe; padding: 10px; border-radius: 4px; margin: 10px 0; }
       </style>
   </head>
   <body>
       <div class="container">
           <h1>Managed Instance Group - Auto-scaled Instance</h1>
           <div class="metadata">
               <p><strong>Instance Name:</strong> $INSTANCE_NAME</p>
               <p><strong>Zone:</strong> $INSTANCE_ZONE</p>
               <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
               <p><strong>Timestamp:</strong> $(date)</p>
           </div>
           <p>This instance is part of a regional MIG with autoscaling and autohealing.</p>
       </div>
   </body>
   </html>
   HTML
   
   # Restart Nginx to apply changes
   systemctl restart nginx
   
   # Enable Nginx to start on boot
   systemctl enable nginx
   EOF
   ```

2. Create the instance template using the startup script:

   ```bash
   gcloud compute instance-templates create web-server-template \
     --machine-type=e2-medium \
     --image-family=debian-12 \
     --image-project=debian-cloud \
     --boot-disk-size=10GB \
     --boot-disk-type=pd-standard \
     --tags=http-server,health-check \
     --metadata-from-file=startup-script=startup-script.sh \
     --region=$REGION
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/instanceTemplates/web-server-template].
NAME                   MACHINE_TYPE  PREEMPTIBLE  CREATION_TIMESTAMP
web-server-template    e2-medium                  2024-01-XX...
```

**Verification:**

- Confirm the template exists:
  ```bash
  gcloud compute instance-templates describe web-server-template --format="table(name,properties.machineType)"
  ```

---

### Step 2: Create a Firewall Rule for HTTP Traffic

**Objective:** Allow incoming HTTP traffic and health check probes to reach the instances.

**Instructions:**

1. Create a firewall rule to allow HTTP traffic from all sources:

   ```bash
   gcloud compute firewall-rules create allow-http-mig \
     --allow=tcp:80 \
     --source-ranges=0.0.0.0/0 \
     --target-tags=http-server \
     --description="Allow HTTP traffic to MIG instances"
   ```

2. Create a firewall rule to allow health check traffic from Google Cloud health check ranges:

   ```bash
   gcloud compute firewall-rules create allow-health-check \
     --allow=tcp:80 \
     --source-ranges=130.211.0.0/22,35.191.0.0/16 \
     --target-tags=health-check \
     --description="Allow health check probes from Google Cloud"
   ```

**Expected Output:**

```
Creating firewall...done.
NAME              NETWORK  DIRECTION  PRIORITY  ALLOW   DENY  DISABLED
allow-http-mig    default  INGRESS    1000      tcp:80        False
allow-health-check default INGRESS    1000      tcp:80        False
```

**Verification:**

- List the firewall rules:
  ```bash
  gcloud compute firewall-rules list --filter="name~'allow-(http-mig|health-check)'" --format="table(name,allowed[].map().firewall_rule().list())"
  ```

---

### Step 3: Create an HTTP Health Check

**Objective:** Define a health check that monitors the availability of instances by testing the root path.

**Instructions:**

1. Create an HTTP health check that polls the root path every 10 seconds:

   ```bash
   gcloud compute health-checks create http web-health-check \
     --port=80 \
     --request-path=/ \
     --check-interval=10s \
     --timeout=5s \
     --healthy-threshold=2 \
     --unhealthy-threshold=3 \
     --description="Health check for web server MIG"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/healthChecks/web-health-check].
NAME               PROTOCOL
web-health-check   HTTP
```

**Verification:**

- Describe the health check configuration:
  ```bash
  gcloud compute health-checks describe web-health-check --format="table(name,httpHealthCheck.port,httpHealthCheck.requestPath,checkIntervalSec)"
  ```

---

### Step 4: Create the Regional Managed Instance Group

**Objective:** Deploy a regional MIG that spans multiple zones for high availability with autoscaling and autohealing enabled.

**Instructions:**

1. Create the regional Managed Instance Group:

   ```bash
   gcloud compute instance-groups managed create web-server-mig \
     --base-instance-name=web-server \
     --template=web-server-template \
     --size=1 \
     --zones=$ZONE1,$ZONE2 \
     --health-check=web-health-check \
     --initial-delay=120 \
     --region=$REGION
   ```

2. Configure autoscaling based on CPU utilization:

   ```bash
   gcloud compute instance-groups managed set-autoscaling web-server-mig \
     --region=$REGION \
     --min-num-replicas=1 \
     --max-num-replicas=4 \
     --target-cpu-utilization=0.60 \
     --cool-down-period=60
   ```

3. Set the autohealing policy:

   ```bash
   gcloud compute instance-groups managed update web-server-mig \
     --region=$REGION \
     --health-check=web-health-check \
     --initial-delay=120
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/instanceGroupManagers/web-server-mig].
NAME             LOCATION     SCOPE   BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE    AUTOSCALED
web-server-mig   us-central1  region  web-server          0     1            web-server-template  yes
```

**Verification:**

- Check the MIG status and wait for instances to be created:
  ```bash
  gcloud compute instance-groups managed describe web-server-mig --region=$REGION --format="table(name,baseInstanceName,targetSize,instanceTemplate)"
  ```

- List instances in the group:
  ```bash
  gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION
  ```

---

### Step 5: Verify Initial Deployment and Access

**Objective:** Confirm that instances are healthy and serving content correctly.

**Instructions:**

1. Wait for instances to become healthy (this may take 2-3 minutes):

   ```bash
   # Check instance status
   watch -n 10 gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION
   ```
   
   Press `Ctrl+C` when status shows `RUNNING` and `HEALTHY`.

2. Get the external IP address of one instance:

   ```bash
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig \
     --region=$REGION \
     --format="value(instance.basename())" \
     --limit=1)
   
   INSTANCE_IP=$(gcloud compute instances describe $INSTANCE_NAME \
     --zone=$ZONE1 \
     --format="value(networkInterfaces[0].accessConfigs[0].natIP)" 2>/dev/null)
   
   # If not in ZONE1, try ZONE2
   if [ -z "$INSTANCE_IP" ]; then
     INSTANCE_IP=$(gcloud compute instances describe $INSTANCE_NAME \
       --zone=$ZONE2 \
       --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
   fi
   
   echo "Instance IP: $INSTANCE_IP"
   ```

3. Test the web server:

   ```bash
   curl http://$INSTANCE_IP
   ```

**Expected Output:**

```html
<!DOCTYPE html>
<html>
<head>
    <title>MIG Instance</title>
...
    <p><strong>Instance Name:</strong> web-server-xxxx</p>
    <p><strong>Zone:</strong> us-central1-a</p>
...
```

**Verification:**

- Confirm the page displays instance metadata including name, zone, and ID
- Verify that the health check is passing:
  ```bash
  gcloud compute instance-groups managed describe web-server-mig --region=$REGION --format="table(autoHealingPolicies,currentActions)"
  ```

---

### Step 6: Simulate CPU Load to Trigger Autoscaling

**Objective:** Generate sustained CPU load to exceed the 60% threshold and observe automatic scaling.

**Instructions:**

1. SSH into one of the running instances:

   ```bash
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig \
     --region=$REGION \
     --format="value(instance.basename())" \
     --limit=1)
   
   INSTANCE_ZONE=$(gcloud compute instances list --filter="name=$INSTANCE_NAME" --format="value(zone)")
   
   gcloud compute ssh $INSTANCE_NAME --zone=$INSTANCE_ZONE
   ```

2. Inside the instance, install stress-ng and generate CPU load:

   ```bash
   # Install stress-ng
   sudo apt-get update && sudo apt-get install -y stress-ng
   
   # Generate CPU stress (use 80% of available CPU)
   stress-ng --cpu 2 --cpu-load 80 --timeout 300s &
   
   # Exit the SSH session
   exit
   ```

3. From Cloud Shell, monitor the autoscaling behavior:

   ```bash
   # Watch the MIG size change (run for 3-5 minutes)
   watch -n 15 'gcloud compute instance-groups managed describe web-server-mig --region=$REGION --format="value(targetSize)" && echo "---" && gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="table(instance.basename(),instance.scope().segment(0),status)"'
   ```

4. Alternatively, view autoscaler decisions in the logs:

   ```bash
   gcloud logging read "resource.type=gce_autoscaler AND resource.labels.instance_group_manager_name=web-server-mig" \
     --limit=10 \
     --format="table(timestamp,jsonPayload.decision,jsonPayload.currentSize,jsonPayload.targetSize)"
   ```

**Expected Output:**

After 2-3 minutes, you should see the target size increase:

```
TARGET_SIZE
1
---
INSTANCE         ZONE           STATUS
web-server-xxxx  us-central1-a  RUNNING

[After scaling]
TARGET_SIZE
3
---
INSTANCE         ZONE           STATUS
web-server-xxxx  us-central1-a  RUNNING
web-server-yyyy  us-central1-b  PROVISIONING
web-server-zzzz  us-central1-a  PROVISIONING
```

**Verification:**

- Confirm that `targetSize` increases from 1 to 2-4 instances
- Verify new instances are distributed across both zones
- Check autoscaler events:
  ```bash
  gcloud compute instance-groups managed describe web-server-mig --region=$REGION --format="table(status.autoscaler)"
  ```

---

### Step 7: Test Autohealing by Simulating Instance Failure

**Objective:** Validate that the MIG automatically recreates instances that fail health checks.

**Instructions:**

1. Select a running instance and stop Nginx to simulate a failure:

   ```bash
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig \
     --region=$REGION \
     --format="value(instance.basename())" \
     --limit=1)
   
   INSTANCE_ZONE=$(gcloud compute instances list --filter="name=$INSTANCE_NAME" --format="value(zone)")
   
   echo "Stopping Nginx on $INSTANCE_NAME in $INSTANCE_ZONE"
   
   gcloud compute ssh $INSTANCE_NAME --zone=$INSTANCE_ZONE --command="sudo systemctl stop nginx"
   ```

2. Monitor the health status and autohealing action:

   ```bash
   # Watch for the instance to be marked unhealthy and recreated
   watch -n 10 'gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="table(instance.basename(),status,currentAction,instanceHealth[0].healthState)"'
   ```

3. After 30-60 seconds, you should see the instance marked as `UNHEALTHY` and then `RECREATING`.

**Expected Output:**

```
INSTANCE         STATUS   CURRENT_ACTION  HEALTH_STATE
web-server-xxxx  RUNNING  NONE            HEALTHY
web-server-yyyy  RUNNING  RECREATING      UNHEALTHY
web-server-zzzz  RUNNING  NONE            HEALTHY

[After recreation]
INSTANCE         STATUS   CURRENT_ACTION  HEALTH_STATE
web-server-xxxx  RUNNING  NONE            HEALTHY
web-server-aaaa  RUNNING  NONE            HEALTHY
web-server-zzzz  RUNNING  NONE            HEALTHY
```

**Verification:**

- Confirm the unhealthy instance is automatically deleted and replaced
- Verify the new instance passes health checks
- Check autohealing events:
  ```bash
  gcloud logging read "resource.type=gce_instance_group_manager AND resource.labels.instance_group_manager_name=web-server-mig AND jsonPayload.event_type=AUTOHEALING" \
    --limit=5 \
    --format="table(timestamp,jsonPayload.event_subtype,jsonPayload.instance_name)"
  ```

---

### Step 8: Observe Scale-Down After Load Decreases

**Objective:** Confirm that the MIG automatically scales down when CPU utilization drops below the target.

**Instructions:**

1. Wait for the stress test to complete (5 minutes from when it was started) or manually stop it:

   ```bash
   # If stress is still running, SSH and kill it
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig \
     --region=$REGION \
     --format="value(instance.basename())" \
     --limit=1)
   
   INSTANCE_ZONE=$(gcloud compute instances list --filter="name=$INSTANCE_NAME" --format="value(zone)")
   
   gcloud compute ssh $INSTANCE_NAME --zone=$INSTANCE_ZONE --command="sudo pkill stress-ng"
   ```

2. Monitor the MIG size for scale-down (this may take 5-10 minutes due to cooldown period):

   ```bash
   watch -n 30 'gcloud compute instance-groups managed describe web-server-mig --region=$REGION --format="value(targetSize)" && echo "Instances:" && gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="value(instance.basename())"'
   ```

**Expected Output:**

```
TARGET_SIZE
3

[After cooldown and scale-down]
TARGET_SIZE
1
```

**Verification:**

- Confirm that the MIG gradually reduces instances back to the minimum (1)
- Verify that scale-down respects the cooldown period (60 seconds)

---

## Validation & Testing

### Success Criteria

- [ ] Instance template created successfully with startup script
- [ ] Regional MIG deployed across two zones with initial size of 1
- [ ] HTTP health check configured and instances passing health checks
- [ ] Autoscaling policy active with min=1, max=4, target CPU=60%
- [ ] Instances automatically scale up when CPU exceeds 60%
- [ ] Autohealing recreates instances that fail health checks
- [ ] MIG scales down when load decreases below target threshold

### Testing Procedure

1. Verify the final state of the MIG:

   ```bash
   gcloud compute instance-groups managed describe web-server-mig \
     --region=$REGION \
     --format="table(name,region,targetSize,instanceTemplate,autoHealingPolicies.healthCheck.basename())"
   ```
   
   **Expected Result:** Table showing MIG with targetSize, template, and health check configured.

2. Verify autoscaling configuration:

   ```bash
   gcloud compute instance-groups managed describe web-server-mig \
     --region=$REGION \
     --format="yaml(autoscaler)"
   ```
   
   **Expected Result:** YAML output showing autoscalingPolicy with minNumReplicas=1, maxNumReplicas=4, cpuUtilization.utilizationTarget=0.6.

3. Test web server accessibility from all instances:

   ```bash
   for INSTANCE in $(gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="value(instance.basename())"); do
     ZONE=$(gcloud compute instances list --filter="name=$INSTANCE" --format="value(zone)")
     IP=$(gcloud compute instances describe $INSTANCE --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
     echo "Testing $INSTANCE at $IP:"
     curl -s http://$IP | grep "Instance Name" || echo "FAILED"
     echo "---"
   done
   ```
   
   **Expected Result:** Each instance returns HTML containing its instance name.

4. Verify distribution across zones:

   ```bash
   gcloud compute instance-groups managed list-instances web-server-mig \
     --region=$REGION \
     --format="table(instance.basename(),instance.scope().segment(0))"
   ```
   
   **Expected Result:** Instances distributed across us-central1-a and us-central1-b.

## Troubleshooting

### Issue 1: Instances Not Becoming Healthy

**Symptoms:**
- Instances stuck in `UNHEALTHY` state
- MIG continuously recreates instances
- Health check failures in logs

**Cause:**
The initial delay may be too short for the startup script to complete, or firewall rules are blocking health check probes.

**Solution:**

1. Verify firewall rules allow health check traffic:
   ```bash
   gcloud compute firewall-rules describe allow-health-check
   ```

2. Increase the initial delay to allow more time for startup:
   ```bash
   gcloud compute instance-groups managed update web-server-mig \
     --region=$REGION \
     --initial-delay=180
   ```

3. Check instance logs for startup script errors:
   ```bash
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="value(instance.basename())" --limit=1)
   INSTANCE_ZONE=$(gcloud compute instances list --filter="name=$INSTANCE_NAME" --format="value(zone)")
   
   gcloud compute instances get-serial-port-output $INSTANCE_NAME --zone=$INSTANCE_ZONE | grep -A 10 "startup-script"
   ```

4. Manually test the health check endpoint:
   ```bash
   INSTANCE_IP=$(gcloud compute instances describe $INSTANCE_NAME --zone=$INSTANCE_ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
   curl -I http://$INSTANCE_IP/
   ```

---

### Issue 2: Autoscaling Not Triggering

**Symptoms:**
- CPU load is high but MIG size remains at 1
- No new instances are created despite sustained load
- Autoscaler shows no activity in logs

**Cause:**
Autoscaling may not be properly configured, the cooldown period may be preventing scaling, or CPU metrics are not being collected.

**Solution:**

1. Verify autoscaling is enabled and configured:
   ```bash
   gcloud compute instance-groups managed describe web-server-mig \
     --region=$REGION \
     --format="yaml(autoscaler)"
   ```

2. Check if autoscaler is receiving CPU metrics:
   ```bash
   gcloud logging read "resource.type=gce_autoscaler AND resource.labels.instance_group_manager_name=web-server-mig" \
     --limit=5 \
     --format="table(timestamp,jsonPayload.currentSize,jsonPayload.targetSize,jsonPayload.decision)"
   ```

3. Ensure stress is actually generating CPU load:
   ```bash
   INSTANCE_NAME=$(gcloud compute instance-groups managed list-instances web-server-mig --region=$REGION --format="value(instance.basename())" --limit=1)
   INSTANCE_ZONE=$(gcloud compute instances list --filter="name=$INSTANCE_NAME" --format="value(zone)")
   
   gcloud compute ssh $INSTANCE_NAME --zone=$INSTANCE_ZONE --command="top -bn1 | grep 'Cpu(s)'"
   ```

4. If autoscaling is not working, recreate the autoscaler with explicit settings:
   ```bash
   gcloud compute instance-groups managed stop-autoscaling web-server-mig --region=$REGION
   
   gcloud compute instance-groups managed set-autoscaling web-server-mig \
     --region=$REGION \
     --min-num-replicas=1 \
     --max-num-replicas=4 \
     --target-cpu-utilization=0.60 \
     --cool-down-period=60 \
     --mode=on
   ```

---

### Issue 3: Instances in Wrong Zones

**Symptoms:**
- All instances created in a single zone
- Regional distribution not working
- Zone specified instead of region in commands

**Cause:**
The MIG may have been created as zonal instead of regional, or zone distribution settings are incorrect.

**Solution:**

1. Verify the MIG is regional:
   ```bash
   gcloud compute instance-groups managed describe web-server-mig \
     --region=$REGION \
     --format="value(region)"
   ```

2. If the MIG is zonal, delete it and recreate as regional:
   ```bash
   # Delete zonal MIG (if it exists)
   gcloud compute instance-groups managed delete web-server-mig --zone=$ZONE1 --quiet
   
   # Recreate as regional
   gcloud compute instance-groups managed create web-server-mig \
     --base-instance-name=web-server \
     --template=web-server-template \
     --size=1 \
     --zones=$ZONE1,$ZONE2 \
     --health-check=web-health-check \
     --initial-delay=120 \
     --region=$REGION
   ```

3. Verify zone distribution policy:
   ```bash
   gcloud compute instance-groups managed describe web-server-mig \
     --region=$REGION \
     --format="table(distributionPolicy)"
   ```

---

### Issue 4: Cannot SSH to Instances

**Symptoms:**
- SSH connection times out
- Permission denied errors
- Firewall or IAM issues

**Cause:**
Firewall rules may be blocking SSH, or IAM permissions for OS Login or metadata-based SSH keys are missing.

**Solution:**

1. Create firewall rule to allow SSH from IAP:
   ```bash
   gcloud compute firewall-rules create allow-ssh-iap \
     --allow=tcp:22 \
     --source-ranges=35.235.240.0/20 \
     --description="Allow SSH via Identity-Aware Proxy"
   ```

2. Use `--tunnel-through-iap` flag:
   ```bash
   gcloud compute ssh $INSTANCE_NAME --zone=$INSTANCE_ZONE --tunnel-through-iap
   ```

3. Verify IAM permissions:
   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:user:$(gcloud config get-value account)"
   ```

4. If issues persist, use serial console:
   ```bash
   gcloud compute instances get-serial-port-output $INSTANCE_NAME --zone=$INSTANCE_ZONE
   ```

---

## Cleanup

To avoid ongoing charges, clean up all resources created in this lab:

```bash
# Delete the Managed Instance Group (this will delete all instances)
gcloud compute instance-groups managed delete web-server-mig \
  --region=$REGION \
  --quiet

# Delete the instance template
gcloud compute instance-templates delete web-server-template --quiet

# Delete the health check
gcloud compute health-checks delete web-health-check --quiet

# Delete firewall rules
gcloud compute firewall-rules delete allow-http-mig --quiet
gcloud compute firewall-rules delete allow-health-check --quiet

# Optional: Delete the startup script file
rm -f startup-script.sh

# Verify cleanup
echo "Remaining MIGs:"
gcloud compute instance-groups managed list --filter="name~web-server"
echo "Remaining templates:"
gcloud compute instance-templates list --filter="name~web-server"
```

> ⚠️ **Warning:** Deleting the MIG will immediately terminate all running instances. Ensure you have saved any necessary data or logs before proceeding. If you plan to use this MIG in subsequent labs, consider scaling it down to 1 instance instead of deleting it entirely.

**To scale down instead of delete:**

```bash
# Scale to minimum size
gcloud compute instance-groups managed set-autoscaling web-server-mig \
  --region=$REGION \
  --min-num-replicas=1 \
  --max-num-replicas=1

# Wait for scale-down
gcloud compute instance-groups managed wait-until web-server-mig \
  --region=$REGION \
  --stable
```

## Summary

### What You Accomplished

- Created a reusable instance template with an automated startup script that deploys Nginx and serves instance metadata
- Deployed a regional Managed Instance Group spanning two zones for high availability
- Configured HTTP health checks to continuously monitor instance health
- Implemented autoscaling policies that dynamically adjust capacity based on CPU utilization (60% target, 1-4 instances)
- Enabled autohealing to automatically replace unhealthy instances
- Validated autoscaling by simulating CPU load and observing automatic scale-up and scale-down
- Tested autohealing by forcing an instance failure and confirming automatic recreation

### Key Takeaways

- **Regional MIGs** provide higher availability than zonal MIGs by distributing instances across multiple zones, protecting against zone-level failures
- **Autoscaling** enables cost efficiency by scaling resources to match demand, with configurable thresholds, cooldown periods, and limits
- **Health checks** are essential for autohealing; proper configuration of check intervals, thresholds, and initial delays prevents premature instance recreation
- **Startup scripts** enable infrastructure-as-code patterns for instance configuration, ensuring consistency across all instances in a MIG
- **Cooldown periods** prevent thrashing by allowing time for new instances to stabilize and metrics to reflect actual load before further scaling decisions
- **Autohealing** improves reliability by automatically detecting and replacing failed instances without manual intervention

### Next Steps

- **Lab 03-07-02:** Integrate this MIG with a regional HTTP(S) load balancer for global traffic distribution
- **Lab 03-08-01:** Implement custom monitoring and alerting for MIG scaling events and instance health
- **Lab 04-01-01:** Migrate this workload to Google Kubernetes Engine (GKE) for container-based orchestration
- Explore **preemptible instances** in MIGs to reduce costs for fault-tolerant workloads
- Implement **rolling updates** to deploy new instance template versions with zero downtime
- Configure **stateful MIGs** for workloads requiring persistent disks or IP addresses

## Additional Resources

- [Managed Instance Groups Overview](https://cloud.google.com/compute/docs/instance-groups) - Official documentation on MIGs, their features, and use cases
- [Autoscaling Groups of Instances](https://cloud.google.com/compute/docs/autoscaler) - Detailed guide on autoscaling policies, metrics, and best practices
- [Health Checking for Instance Groups](https://cloud.google.com/compute/docs/instance-groups/autohealing-instances-in-migs) - Configuration and troubleshooting for autohealing
- [Creating Instance Templates](https://cloud.google.com/compute/docs/instance-templates) - Best practices for template design and startup scripts
- [Regional vs. Zonal MIGs](https://cloud.google.com/compute/docs/instance-groups/distributing-instances-with-regional-instance-groups) - Comparison and architecture guidance
- [Best Practices for Compute Engine](https://cloud.google.com/compute/docs/best-practices) - Performance, security, and cost optimization recommendations

---

# Lab 03-08-01: Configure Global Load Balancer with Managed SSL Certificate

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Advanced |
| **Bloom Level** | Understand |

## Overview

This lab guides you through implementing a production-grade global HTTP(S) load balancer with Google-managed SSL certificates. You will deploy web applications across multiple regions, configure intelligent traffic routing, and enable Cloud CDN for optimized content delivery. This configuration represents enterprise-level infrastructure that ensures high availability, low latency, and secure connections for users worldwide.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Deploy a global HTTP(S) load balancer to distribute traffic across multiple regions
- [ ] Configure backend services with managed instance groups in multiple geographic locations
- [ ] Implement Google-managed SSL certificates for secure HTTPS connections
- [ ] Configure URL maps and host rules for intelligent traffic routing
- [ ] Set up Cloud CDN integration for improved content delivery performance
- [ ] Verify SSL certificate provisioning and test global load balancing functionality

## Prerequisites

### Required Knowledge

- Understanding of HTTP/HTTPS protocols and SSL/TLS concepts
- Knowledge of DNS and domain name management
- Familiarity with load balancing concepts from Lab 02-09-01
- Experience with Compute Engine and managed instance groups
- Basic understanding of CDN concepts
- Understanding of global vs regional resources in GCP

### Required Access

- Active Google Cloud Platform account with billing enabled
- Project Editor role or equivalent permissions (Compute Admin, Network Admin)
- Access to a domain name with ability to modify DNS records (or use provided test domain)
- At least $50 in available credits (Cloud SQL and Load Balancers incur charges)

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
| Google Chrome/Firefox | Latest stable release | Accessing GCP Console and Cloud Shell |
| Google Cloud SDK | 400.0.0+ | Command-line interface for GCP operations |
| OpenSSH/PuTTY | 7.0+/0.70+ | Connecting to VM instances |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Set default compute region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Enable required APIs
gcloud services enable compute.googleapis.com
gcloud services enable certificatemanager.googleapis.com

# Verify current configuration
gcloud config list
```

## Step-by-Step Instructions

### Step 1: Create Instance Template for Web Servers

**Objective:** Create a reusable instance template that deploys a simple web application identifying its region and instance name.

**Instructions:**

1. Create a startup script that will run on each instance:

   ```bash
   cat > startup-script.sh << 'EOF'
   #!/bin/bash
   apt-get update
   apt-get install -y apache2
   
   # Get instance metadata
   INSTANCE_NAME=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name)
   ZONE=$(curl -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone | cut -d'/' -f4)
   REGION=$(echo $ZONE | sed 's/-[a-z]$//')
   
   # Create custom index page
   cat > /var/www/html/index.html << HTML
   <!DOCTYPE html>
   <html>
   <head>
       <title>Global Load Balancer Demo</title>
       <style>
           body { font-family: Arial, sans-serif; margin: 40px; background: #f0f0f0; }
           .container { background: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
           .info { background: #e3f2fd; padding: 15px; margin: 10px 0; border-radius: 5px; }
           h1 { color: #1976d2; }
       </style>
   </head>
   <body>
       <div class="container">
           <h1>🌍 Global Load Balancer Demo</h1>
           <div class="info">
               <strong>Instance:</strong> $INSTANCE_NAME<br>
               <strong>Region:</strong> $REGION<br>
               <strong>Zone:</strong> $ZONE<br>
               <strong>Server Time:</strong> $(date)
           </div>
           <p>This request was served by a backend in <strong>$REGION</strong></p>
       </div>
   </body>
   </html>
   HTML
   
   # Enable Apache
   systemctl restart apache2
   systemctl enable apache2
   EOF
   ```

2. Create the instance template:

   ```bash
   gcloud compute instance-templates create global-lb-template \
       --machine-type=e2-micro \
       --network-interface=network=default,network-tier=PREMIUM \
       --metadata-from-file=startup-script=startup-script.sh \
       --tags=http-server,https-server \
       --scopes=https://www.googleapis.com/auth/cloud-platform
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/instanceTemplates/global-lb-template].
NAME                  MACHINE_TYPE  PREEMPTIBLE  CREATION_TIMESTAMP
global-lb-template    e2-micro                   2024-01-15T10:30:00.000-07:00
```

**Verification:**

- Confirm the template exists:
  ```bash
  gcloud compute instance-templates describe global-lb-template
  ```

---

### Step 2: Create Managed Instance Groups in Multiple Regions

**Objective:** Deploy managed instance groups in two different regions (US and Europe) for global distribution.

**Instructions:**

1. Create managed instance group in us-central1:

   ```bash
   gcloud compute instance-groups managed create us-mig \
       --template=global-lb-template \
       --size=2 \
       --region=us-central1 \
       --health-check=http-health-check \
       --initial-delay=60
   ```

2. Create HTTP health check (if not already exists):

   ```bash
   gcloud compute health-checks create http http-health-check \
       --port=80 \
       --request-path=/ \
       --check-interval=10s \
       --timeout=5s \
       --unhealthy-threshold=3 \
       --healthy-threshold=2
   ```

3. Set the health check for us-mig:

   ```bash
   gcloud compute instance-groups managed set-autohealing us-mig \
       --region=us-central1 \
       --health-check=http-health-check \
       --initial-delay=60
   ```

4. Create managed instance group in europe-west1:

   ```bash
   gcloud compute instance-groups managed create eu-mig \
       --template=global-lb-template \
       --size=2 \
       --region=europe-west1 \
       --health-check=http-health-check \
       --initial-delay=60
   ```

5. Configure named ports for both instance groups:

   ```bash
   gcloud compute instance-groups managed set-named-ports us-mig \
       --region=us-central1 \
       --named-ports=http:80
   
   gcloud compute instance-groups managed set-named-ports eu-mig \
       --region=europe-west1 \
       --named-ports=http:80
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/us-central1/instanceGroupManagers/us-mig].
NAME    LOCATION      SCOPE   BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE   AUTOSCALED
us-mig  us-central1   region  us-mig              0     2            global-lb-template  no

Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/regions/europe-west1/instanceGroupManagers/eu-mig].
NAME    LOCATION       SCOPE   BASE_INSTANCE_NAME  SIZE  TARGET_SIZE  INSTANCE_TEMPLATE   AUTOSCALED
eu-mig  europe-west1   region  eu-mig              0     2            global-lb-template  no
```

**Verification:**

- Wait for instances to be created (2-3 minutes):
  ```bash
  gcloud compute instance-groups managed list-instances us-mig --region=us-central1
  gcloud compute instance-groups managed list-instances eu-mig --region=europe-west1
  ```

---

### Step 3: Configure Firewall Rules

**Objective:** Create firewall rules to allow HTTP/HTTPS traffic and health check probes from Google's load balancer.

**Instructions:**

1. Create firewall rule for HTTP/HTTPS traffic:

   ```bash
   gcloud compute firewall-rules create allow-http-https \
       --allow=tcp:80,tcp:443 \
       --source-ranges=0.0.0.0/0 \
       --target-tags=http-server,https-server \
       --description="Allow HTTP and HTTPS traffic"
   ```

2. Create firewall rule for health checks:

   ```bash
   gcloud compute firewall-rules create allow-health-check \
       --allow=tcp:80 \
       --source-ranges=35.191.0.0/16,130.211.0.0/22 \
       --target-tags=http-server \
       --description="Allow health checks from Google Cloud Load Balancer"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/firewalls/allow-http-https].
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/firewalls/allow-health-check].
NAME                NETWORK  DIRECTION  PRIORITY  ALLOW          DENY  DISABLED
allow-http-https    default  INGRESS    1000      tcp:80,tcp:443       False
allow-health-check  default  INGRESS    1000      tcp:80               False
```

**Verification:**

- List firewall rules:
  ```bash
  gcloud compute firewall-rules list --filter="name~'allow-http'"
  ```

---

### Step 4: Reserve Static IP Address

**Objective:** Reserve a global static IP address for the load balancer.

**Instructions:**

1. Reserve a global static IP address:

   ```bash
   gcloud compute addresses create global-lb-ip \
       --ip-version=IPV4 \
       --global
   ```

2. Get the reserved IP address:

   ```bash
   export LB_IP=$(gcloud compute addresses describe global-lb-ip --global --format="get(address)")
   echo "Reserved IP: $LB_IP"
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/addresses/global-lb-ip].
Reserved IP: 34.120.45.67
```

**Verification:**

- Verify the IP address:
  ```bash
  gcloud compute addresses describe global-lb-ip --global
  ```

---

### Step 5: Create Backend Service

**Objective:** Configure backend service with health checks and backends from both regions.

**Instructions:**

1. Create backend service:

   ```bash
   gcloud compute backend-services create global-backend-service \
       --protocol=HTTP \
       --health-checks=http-health-check \
       --global \
       --enable-cdn \
       --cache-mode=CACHE_ALL_STATIC \
       --default-ttl=3600 \
       --client-ttl=3600 \
       --max-ttl=86400
   ```

2. Add US backend to the service:

   ```bash
   gcloud compute backend-services add-backend global-backend-service \
       --instance-group=us-mig \
       --instance-group-region=us-central1 \
       --balancing-mode=UTILIZATION \
       --max-utilization=0.8 \
       --capacity-scaler=1.0 \
       --global
   ```

3. Add EU backend to the service:

   ```bash
   gcloud compute backend-services add-backend global-backend-service \
       --instance-group=eu-mig \
       --instance-group-region=europe-west1 \
       --balancing-mode=UTILIZATION \
       --max-utilization=0.8 \
       --capacity-scaler=1.0 \
       --global
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/backendServices/global-backend-service].
NAME                      BACKENDS  PROTOCOL
global-backend-service              HTTP

Updated [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/backendServices/global-backend-service].
```

**Verification:**

- Describe the backend service:
  ```bash
  gcloud compute backend-services describe global-backend-service --global
  ```

---

### Step 6: Create URL Map

**Objective:** Configure URL map to route traffic to the backend service.

**Instructions:**

1. Create URL map:

   ```bash
   gcloud compute url-maps create global-lb-url-map \
       --default-service=global-backend-service
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/urlMaps/global-lb-url-map].
NAME                  DEFAULT_SERVICE
global-lb-url-map     global-backend-service
```

**Verification:**

- Verify URL map configuration:
  ```bash
  gcloud compute url-maps describe global-lb-url-map
  ```

---

### Step 7: Configure Google-Managed SSL Certificate

**Objective:** Create a Google-managed SSL certificate for your domain.

**Instructions:**

1. **IMPORTANT:** You need a domain name for this step. Replace `your-domain.com` with your actual domain:

   ```bash
   export DOMAIN="your-domain.com"
   
   gcloud compute ssl-certificates create global-ssl-cert \
       --domains=$DOMAIN \
       --global
   ```

2. **Alternative for testing:** If you don't have a domain, create a self-signed certificate:

   ```bash
   # Generate self-signed certificate (for testing only)
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
       -keyout private.key \
       -out certificate.crt \
       -subj "/C=US/ST=State/L=City/O=Organization/CN=test.example.com"
   
   # Create SSL certificate resource
   gcloud compute ssl-certificates create global-ssl-cert-self-signed \
       --certificate=certificate.crt \
       --private-key=private.key \
       --global
   ```

3. Check certificate status:

   ```bash
   gcloud compute ssl-certificates describe global-ssl-cert --global
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/sslCertificates/global-ssl-cert].
NAME              TYPE     CREATION_TIMESTAMP             EXPIRE_TIME  MANAGED_STATUS
global-ssl-cert   MANAGED  2024-01-15T10:45:00.000-07:00               PROVISIONING
```

> ⚠️ **Note:** Google-managed SSL certificates can take 15-60 minutes to provision. The certificate must be attached to a load balancer and the domain must resolve to the load balancer's IP address.

**Verification:**

- Monitor certificate provisioning:
  ```bash
  watch -n 30 gcloud compute ssl-certificates describe global-ssl-cert --global
  ```

---

### Step 8: Create HTTPS Target Proxy

**Objective:** Create target HTTPS proxy to handle SSL termination.

**Instructions:**

1. Create HTTPS target proxy:

   ```bash
   gcloud compute target-https-proxies create global-https-proxy \
       --url-map=global-lb-url-map \
       --ssl-certificates=global-ssl-cert
   ```

2. Create HTTP target proxy for redirect (optional but recommended):

   ```bash
   gcloud compute target-http-proxies create global-http-proxy \
       --url-map=global-lb-url-map
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/targetHttpsProxies/global-https-proxy].
NAME                 SSL_CERTIFICATES  URL_MAP
global-https-proxy   global-ssl-cert   global-lb-url-map
```

**Verification:**

- Verify proxy configuration:
  ```bash
  gcloud compute target-https-proxies describe global-https-proxy
  ```

---

### Step 9: Create Global Forwarding Rules

**Objective:** Create forwarding rules to route incoming traffic to the target proxies.

**Instructions:**

1. Create HTTPS forwarding rule:

   ```bash
   gcloud compute forwarding-rules create global-https-forwarding-rule \
       --address=global-lb-ip \
       --global \
       --target-https-proxy=global-https-proxy \
       --ports=443
   ```

2. Create HTTP forwarding rule (for redirect):

   ```bash
   gcloud compute forwarding-rules create global-http-forwarding-rule \
       --address=global-lb-ip \
       --global \
       --target-http-proxy=global-http-proxy \
       --ports=80
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/forwardingRules/global-https-forwarding-rule].
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/forwardingRules/global-http-forwarding-rule].
```

**Verification:**

- List all forwarding rules:
  ```bash
  gcloud compute forwarding-rules list --global
  ```

---

### Step 10: Configure DNS Records

**Objective:** Point your domain to the load balancer's IP address.

**Instructions:**

1. Get the load balancer IP:

   ```bash
   echo "Configure your DNS A record to point to: $LB_IP"
   ```

2. In your DNS provider's console (e.g., Cloud DNS, GoDaddy, Namecheap):
   - Create an A record for your domain
   - Set the value to the IP address from step 1
   - Set TTL to 300 seconds (5 minutes) for testing

3. If using Cloud DNS, create the record:

   ```bash
   # Create DNS zone (if not exists)
   gcloud dns managed-zones create global-lb-zone \
       --dns-name="$DOMAIN." \
       --description="Zone for global load balancer"
   
   # Start transaction
   gcloud dns record-sets transaction start --zone=global-lb-zone
   
   # Add A record
   gcloud dns record-sets transaction add $LB_IP \
       --name="$DOMAIN." \
       --ttl=300 \
       --type=A \
       --zone=global-lb-zone
   
   # Execute transaction
   gcloud dns record-sets transaction execute --zone=global-lb-zone
   ```

**Expected Output:**

```
Created [https://dns.googleapis.com/dns/v1/projects/PROJECT_ID/managedZones/global-lb-zone].
Transaction started [transaction.yaml].
Record addition appended to transaction at [transaction.yaml].
Executed transaction [transaction.yaml] for managed-zone [global-lb-zone].
```

**Verification:**

- Verify DNS propagation:
  ```bash
  nslookup $DOMAIN
  dig $DOMAIN +short
  ```

---

### Step 11: Enable Cloud Armor (Optional Security Layer)

**Objective:** Add basic security policies using Cloud Armor.

**Instructions:**

1. Create a Cloud Armor security policy:

   ```bash
   gcloud compute security-policies create global-armor-policy \
       --description="Basic security policy for global load balancer"
   ```

2. Add a rule to block traffic from a specific country (example: block traffic from XX):

   ```bash
   gcloud compute security-policies rules create 1000 \
       --security-policy=global-armor-policy \
       --expression="origin.region_code == 'CN'" \
       --action=deny-403 \
       --description="Block traffic from specific region"
   ```

3. Add rate limiting rule:

   ```bash
   gcloud compute security-policies rules create 2000 \
       --security-policy=global-armor-policy \
       --expression="true" \
       --action=rate-based-ban \
       --rate-limit-threshold-count=100 \
       --rate-limit-threshold-interval-sec=60 \
       --ban-duration-sec=600 \
       --conform-action=allow \
       --exceed-action=deny-429 \
       --description="Rate limit: 100 requests per minute"
   ```

4. Attach the policy to the backend service:

   ```bash
   gcloud compute backend-services update global-backend-service \
       --security-policy=global-armor-policy \
       --global
   ```

**Expected Output:**

```
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/securityPolicies/global-armor-policy].
Created [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/securityPolicies/global-armor-policy/rules/1000].
Updated [https://www.googleapis.com/compute/v1/projects/PROJECT_ID/global/backendServices/global-backend-service].
```

**Verification:**

- View security policy:
  ```bash
  gcloud compute security-policies describe global-armor-policy
  ```

---

### Step 12: Test the Global Load Balancer

**Objective:** Verify that the load balancer is working correctly with SSL and distributing traffic globally.

**Instructions:**

1. Wait for SSL certificate to be provisioned (check status):

   ```bash
   gcloud compute ssl-certificates describe global-ssl-cert --global --format="get(managed.status)"
   ```

   Wait until status shows `ACTIVE` (may take 15-60 minutes).

2. Test HTTP access (should work immediately):

   ```bash
   curl -I http://$LB_IP
   ```

3. Test HTTPS access (after SSL certificate is active):

   ```bash
   curl -I https://$DOMAIN
   ```

4. View the full response to see which region serves your request:

   ```bash
   curl https://$DOMAIN
   ```

5. Test from multiple locations using online tools:
   - Visit https://www.whatsmydns.net/ and check your domain
   - Use https://tools.keycdn.com/performance to test from multiple locations

6. Monitor backend health:

   ```bash
   gcloud compute backend-services get-health global-backend-service --global
   ```

**Expected Output:**

```
HTTP/1.1 200 OK
Content-Type: text/html
Server: Apache/2.4.52 (Debian)

<!DOCTYPE html>
<html>
<head><title>Global Load Balancer Demo</title></head>
<body>
  <h1>🌍 Global Load Balancer Demo</h1>
  <div class="info">
    <strong>Instance:</strong> us-mig-abcd<br>
    <strong>Region:</strong> us-central1<br>
    <strong>Zone:</strong> us-central1-a<br>
  </div>
</body>
</html>
```

**Verification:**

- Confirm multiple requests are distributed across instances:
  ```bash
  for i in {1..10}; do curl -s https://$DOMAIN | grep "Instance:"; done
  ```

---

## Validation & Testing

### Success Criteria

- [ ] Managed instance groups are running in at least two regions with healthy instances
- [ ] Backend service shows all backends as HEALTHY
- [ ] SSL certificate status is ACTIVE
- [ ] HTTPS requests to the domain return 200 OK status
- [ ] Responses show different instances/regions serving requests
- [ ] Cloud CDN is enabled and caching static content
- [ ] Cloud Armor policies are applied and active
- [ ] DNS resolves correctly to the load balancer IP

### Testing Procedure

1. Verify all backend instances are healthy:

   ```bash
   gcloud compute backend-services get-health global-backend-service --global
   ```
   
   **Expected Result:** All instances show `healthState: HEALTHY`

2. Test SSL certificate:

   ```bash
   echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null | openssl x509 -noout -dates
   ```
   
   **Expected Result:** Valid certificate with future expiration date

3. Test Cloud CDN caching:

   ```bash
   curl -I https://$DOMAIN/index.html
   ```
   
   **Expected Result:** Response headers include `X-Cache-Status` or `Age` headers

4. Test geographic distribution:

   ```bash
   # Run from different locations or use VPN
   for i in {1..20}; do 
     curl -s https://$DOMAIN | grep -E "(Instance|Region):" 
     sleep 1
   done
   ```
   
   **Expected Result:** Requests distributed across multiple instances and regions

5. Load test the infrastructure:

   ```bash
   # Install Apache Bench if not available
   sudo apt-get install -y apache2-utils
   
   # Run load test
   ab -n 1000 -c 10 https://$DOMAIN/
   ```
   
   **Expected Result:** No failed requests, consistent response times

6. Verify Cloud Armor is active:

   ```bash
   gcloud compute security-policies describe global-armor-policy
   ```
   
   **Expected Result:** Policy shows attached to backend service

## Troubleshooting

### Issue 1: SSL Certificate Stuck in PROVISIONING State

**Symptoms:**
- Certificate status remains `PROVISIONING` or `FAILED_NOT_VISIBLE` for extended period
- HTTPS access returns SSL certificate errors
- Certificate shows domain authorization challenges failing

**Cause:**
Google-managed SSL certificates require the domain to resolve to the load balancer IP and serve HTTP traffic on port 80 before they can be provisioned. DNS propagation delays or incorrect DNS configuration prevent validation.

**Solution:**

```bash
# Verify DNS is pointing to correct IP
nslookup $DOMAIN

# Ensure HTTP forwarding rule exists
gcloud compute forwarding-rules describe global-http-forwarding-rule --global

# Check certificate status details
gcloud compute ssl-certificates describe global-ssl-cert --global

# If stuck, delete and recreate after confirming DNS
gcloud compute ssl-certificates delete global-ssl-cert --global --quiet
gcloud compute ssl-certificates create global-ssl-cert --domains=$DOMAIN --global

# Alternative: Use self-signed certificate temporarily
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout private.key -out certificate.crt \
    -subj "/CN=$DOMAIN"

gcloud compute ssl-certificates create temp-ssl-cert \
    --certificate=certificate.crt \
    --private-key=private.key \
    --global

gcloud compute target-https-proxies update global-https-proxy \
    --ssl-certificates=temp-ssl-cert
```

---

### Issue 2: Backend Instances Showing as UNHEALTHY

**Symptoms:**
- `gcloud compute backend-services get-health` shows instances as UNHEALTHY
- Load balancer returns 502 Bad Gateway errors
- No traffic reaches backend instances

**Cause:**
Health check probes cannot reach instances due to firewall rules, instances not serving HTTP on port 80, or application startup delays.

**Solution:**

```bash
# Verify firewall rules allow health checks
gcloud compute firewall-rules describe allow-health-check

# Check if instances are running
gcloud compute instance-groups managed list-instances us-mig --region=us-central1
gcloud compute instance-groups managed list-instances eu-mig --region=europe-west1

# SSH into an instance to verify Apache is running
gcloud compute ssh $(gcloud compute instances list --filter="name~'us-mig'" --limit=1 --format="value(name)") --zone=us-central1-a

# On the instance:
sudo systemctl status apache2
curl localhost

# Exit and test health check manually
curl http://[INSTANCE_EXTERNAL_IP]/

# Recreate instances if needed
gcloud compute instance-groups managed recreate-instances us-mig \
    --instances=us-mig-xxxx \
    --region=us-central1

# Increase initial delay if instances need more startup time
gcloud compute instance-groups managed update us-mig \
    --region=us-central1 \
    --initial-delay=120
```

---

### Issue 3: 404 Not Found or 502 Bad Gateway Errors

**Symptoms:**
- Accessing the load balancer IP or domain returns 404 or 502 errors
- Backend instances are healthy but traffic doesn't reach them
- Load balancer configuration appears correct

**Cause:**
URL map misconfiguration, backend service not properly linked, or named ports not set correctly on instance groups.

**Solution:**

```bash
# Verify URL map configuration
gcloud compute url-maps describe global-lb-url-map

# Check backend service configuration
gcloud compute backend-services describe global-backend-service --global

# Verify named ports on instance groups
gcloud compute instance-groups managed describe us-mig --region=us-central1 | grep namedPorts
gcloud compute instance-groups managed describe eu-mig --region=europe-west1 | grep namedPorts

# Set named ports if missing
gcloud compute instance-groups managed set-named-ports us-mig \
    --region=us-central1 \
    --named-ports=http:80

gcloud compute instance-groups managed set-named-ports eu-mig \
    --region=europe-west1 \
    --named-ports=http:80

# Update backend service if needed
gcloud compute backend-services update global-backend-service \
    --port-name=http \
    --protocol=HTTP \
    --global

# Check forwarding rules point to correct proxy
gcloud compute forwarding-rules describe global-https-forwarding-rule --global
```

---

### Issue 4: Cloud CDN Not Caching Content

**Symptoms:**
- Every request shows as cache MISS
- No `X-Cache-Status` headers in responses
- Performance not improved for static content

**Cause:**
Cache-Control headers from origin prevent caching, or CDN configuration doesn't match content type.

**Solution:**

```bash
# Verify CDN is enabled on backend service
gcloud compute backend-services describe global-backend-service --global | grep enableCDN

# Update backend service CDN settings
gcloud compute backend-services update global-backend-service \
    --enable-cdn \
    --cache-mode=CACHE_ALL_STATIC \
    --global

# SSH into instance and add cache headers to Apache
gcloud compute ssh $(gcloud compute instances list --filter="name~'us-mig'" --limit=1 --format="value(name)") --zone=us-central1-a

# On the instance, edit Apache config:
sudo bash -c 'cat >> /etc/apache2/sites-enabled/000-default.conf << EOF
<FilesMatch "\.(html|css|js|jpg|png|gif)$">
    Header set Cache-Control "public, max-age=3600"
</FilesMatch>
EOF'

sudo a2enmod headers
sudo systemctl restart apache2

# Test cache headers
curl -I https://$DOMAIN/index.html | grep -i cache

# Invalidate CDN cache if needed
gcloud compute url-maps invalidate-cdn-cache global-lb-url-map \
    --path="/*"
```

---

### Issue 5: High Latency from Certain Regions

**Symptoms:**
- Users in specific geographic regions experience slow response times
- Some backends receive disproportionate traffic
- Latency varies significantly between requests

**Cause:**
Insufficient backend capacity in certain regions, or traffic routing not optimized for user location.

**Solution:**

```bash
# Add more backends in underserved regions
gcloud compute instance-groups managed create asia-mig \
    --template=global-lb-template \
    --size=2 \
    --region=asia-east1 \
    --health-check=http-health-check \
    --initial-delay=60

gcloud compute instance-groups managed set-named-ports asia-mig \
    --region=asia-east1 \
    --named-ports=http:80

# Add new backend to service
gcloud compute backend-services add-backend global-backend-service \
    --instance-group=asia-mig \
    --instance-group-region=asia-east1 \
    --balancing-mode=UTILIZATION \
    --max-utilization=0.8 \
    --global

# Enable connection draining for smoother failover
gcloud compute backend-services update global-backend-service \
    --connection-draining-timeout=30 \
    --global

# Adjust backend capacity
gcloud compute backend-services update-backend global-backend-service \
    --instance-group=us-mig \
    --instance-group-region=us-central1 \
    --capacity-scaler=1.5 \
    --global

# Scale instance groups if needed
gcloud compute instance-groups managed resize us-mig \
    --region=us-central1 \
    --size=4
```

---

## Cleanup

> ⚠️ **Warning:** Load balancers and Cloud SQL instances incur charges even when idle. Complete cleanup immediately after finishing the lab to avoid unexpected costs.

```bash
# Delete forwarding rules
gcloud compute forwarding-rules delete global-https-forwarding-rule --global --quiet
gcloud compute forwarding-rules delete global-http-forwarding-rule --global --quiet

# Delete target proxies
gcloud compute target-https-proxies delete global-https-proxy --quiet
gcloud compute target-http-proxies delete global-http-proxy --quiet

# Delete URL map
gcloud compute url-maps delete global-lb-url-map --quiet

# Delete backend service
gcloud compute backend-services delete global-backend-service --global --quiet

# Delete SSL certificate
gcloud compute ssl-certificates delete global-ssl-cert --global --quiet

# Delete security policy (if created)
gcloud compute security-policies delete global-armor-policy --quiet

# Delete managed instance groups
gcloud compute instance-groups managed delete us-mig --region=us-central1 --quiet
gcloud compute instance-groups managed delete eu-mig --region=europe-west1 --quiet

# Delete instance template
gcloud compute instance-templates delete global-lb-template --quiet

# Delete health check
gcloud compute health-checks delete http-health-check --quiet

# Delete firewall rules
gcloud compute firewall-rules delete allow-http-https --quiet
gcloud compute firewall-rules delete allow-health-check --quiet

# Release static IP address
gcloud compute addresses delete global-lb-ip --global --quiet

# Delete DNS records (if using Cloud DNS)
gcloud dns record-sets transaction start --zone=global-lb-zone
gcloud dns record-sets transaction remove $LB_IP \
    --name="$DOMAIN." \
    --ttl=300 \
    --type=A \
    --zone=global-lb-zone
gcloud dns record-sets transaction execute --zone=global-lb-zone
gcloud dns managed-zones delete global-lb-zone --quiet

# Clean up local files
rm -f startup-script.sh private.key certificate.crt

# Verify cleanup
echo "Verifying cleanup..."
gcloud compute forwarding-rules list --global
gcloud compute backend-services list --global
gcloud compute instance-groups managed list
```

**Estimated Cost Savings:** Completing cleanup immediately saves approximately $50-100/month in load balancer and instance costs.

## Summary

### What You Accomplished

- Deployed a production-grade global HTTP(S) load balancer across multiple regions
- Configured managed instance groups with auto-healing in US and Europe
- Implemented Google-managed SSL certificates for secure HTTPS connections
- Set up Cloud CDN for improved content delivery and reduced latency
- Applied Cloud Armor security policies for DDoS protection and rate limiting
- Configured intelligent traffic routing with URL maps and health checks
- Tested global load distribution and verified SSL/TLS functionality

### Key Takeaways

- **Global Load Balancing:** Google Cloud's global load balancer operates at Layer 7 (HTTP/HTTPS) and automatically routes users to the nearest healthy backend, reducing latency and improving user experience
- **Managed SSL Certificates:** Google-managed certificates automatically renew and require domain ownership verification through DNS and HTTP challenge
- **Cloud CDN Integration:** Enabling CDN at the backend service level caches static content at Google's edge locations, reducing origin server load by 60-90%
- **Health Checks:** Proper health check configuration is critical; unhealthy backends are automatically removed from rotation within seconds
- **Security Layers:** Cloud Armor provides enterprise-grade DDoS protection, rate limiting, and geographic restrictions at the edge before traffic reaches backends
- **Cost Considerations:** Global load balancers charge for forwarding rules ($0.025/hour), data processed ($0.008-0.010/GB), and backend instance costs
- **DNS Propagation:** SSL certificate provisioning requires DNS to be properly configured and propagated globally (15-60 minutes)

### Next Steps

- Explore advanced URL routing patterns with path matchers and host rules
- Implement custom Cloud Armor rules for application-specific security
- Configure Cloud Monitoring alerts for backend health and latency thresholds
- Set up Cloud Logging to analyze traffic patterns and troubleshoot issues
- Experiment with traffic splitting for A/B testing and canary deployments
- Integrate with Cloud CDN signed URLs for private content delivery
- Configure backend buckets for serving static content directly from Cloud Storage
- Implement Identity-Aware Proxy (IAP) for authenticated access to applications

## Additional Resources

- [Google Cloud Load Balancing Documentation](https://cloud.google.com/load-balancing/docs) - Official comprehensive guide to all load balancer types
- [SSL Certificates Overview](https://cloud.google.com/load-balancing/docs/ssl-certificates) - Detailed information on managed and self-managed certificates
- [Cloud CDN Best Practices](https://cloud.google.com/cdn/docs/best-practices) - Optimization techniques for content delivery
- [Cloud Armor Security Policies](https://cloud.google.com/armor/docs/security-policy-concepts) - Advanced security configuration options
- [Load Balancer Troubleshooting](https://cloud.google.com/load-balancing/docs/https/troubleshooting-ext-https-lbs) - Common issues and solutions
- [Backend Service Configuration](https://cloud.google.com/load-balancing/docs/backend-service) - Detailed backend configuration options
- [Health Check Concepts](https://cloud.google.com/load-balancing/docs/health-check-concepts) - Understanding health check mechanics
- [Global vs Regional Load Balancing](https://cloud.google.com/load-balancing/docs/choosing-load-balancer) - Choosing the right load balancer type

---

# Lab 03-09-01: Práctica 8: Desplegar app contenedorizada en GKE y exponer con Ingress

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Intermediate |
| **Bloom Level** | Understand |

## Overview

In this lab, you will deploy a containerized application on Google Kubernetes Engine (GKE) and expose it to the internet using an Ingress resource. You will create a GKE Autopilot cluster, deploy a multi-replica application, configure a Kubernetes Service, and provision a Layer 7 HTTP(S) Load Balancer through Ingress. This practice demonstrates how GKE integrates with Google Cloud's native load balancing infrastructure to provide highly available, scalable web applications with automatic health checking and traffic distribution.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create and configure a GKE Autopilot cluster and obtain cluster credentials
- [ ] Deploy a containerized application using Kubernetes Deployments with multiple replicas
- [ ] Expose the application internally using a Kubernetes Service (ClusterIP/NodePort)
- [ ] Configure an Ingress resource to provision an external HTTP(S) Load Balancer
- [ ] Verify Layer 7 load balancing, health checks, and traffic distribution across pods
- [ ] Scale the deployment and observe automatic backend updates in the load balancer

## Prerequisites

### Required Knowledge

- Basic understanding of containerization and Docker concepts
- Familiarity with Kubernetes objects (Pods, Deployments, Services, Ingress)
- Basic knowledge of HTTP protocols and load balancing concepts
- Experience with Google Cloud Console and Cloud Shell

### Required Access

- Google Cloud Project with billing enabled
- IAM role: **Kubernetes Engine Admin** (roles/container.admin)
- IAM role: **Compute Network Admin** (roles/compute.networkAdmin) for Ingress/LB provisioning
- Kubernetes Engine API enabled
- Compute Engine API enabled

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 4+ vCPU |
| RAM | 8+ GB |
| Network | Stable connection ≥10 Mbps |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Manage GKE clusters and authentication |
| kubectl | 1.29.x | Deploy and manage Kubernetes resources |
| curl | 7.81+ | Test HTTP endpoints |
| Google Cloud Shell | Latest | Pre-configured environment with all tools |

### Initial Setup

```bash
# Set your project ID (replace with your actual project ID)
export PROJECT_ID="your-project-id"
gcloud config set project $PROJECT_ID

# Set your preferred region and zone
export REGION="us-central1"
export ZONE="us-central1-a"
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

# Enable required APIs
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com

# Verify your current configuration
gcloud config list
```

## Step-by-Step Instructions

### Step 1: Create a GKE Autopilot Cluster

**Objective:** Provision a managed GKE Autopilot cluster that automatically handles node provisioning and scaling.

**Instructions:**

1. Create the GKE Autopilot cluster:
   
   ```bash
   gcloud container clusters create-auto my-gke-cluster \
     --region=$REGION \
     --project=$PROJECT_ID
   ```

   > **Note:** Autopilot cluster creation typically takes 3-5 minutes. Autopilot mode manages nodes automatically, optimizing for cost and operational efficiency.

2. Verify the cluster is running:

   ```bash
   gcloud container clusters list --region=$REGION
   ```

3. Get cluster credentials to configure kubectl:

   ```bash
   gcloud container clusters get-credentials my-gke-cluster \
     --region=$REGION \
     --project=$PROJECT_ID
   ```

**Expected Output:**

```
Fetching cluster endpoint and auth data.
kubeconfig entry generated for my-gke-cluster.
```

**Verification:**

- Confirm kubectl is connected to the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

You should see cluster endpoint information and at least one node in Ready status.

---

### Step 2: Deploy a Containerized Application

**Objective:** Create a Kubernetes Deployment with three replicas of a sample web application.

**Instructions:**

1. Create a deployment manifest file:
   
   ```bash
   cat > deployment.yaml <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: hello-app
     labels:
       app: hello
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: hello
     template:
       metadata:
         labels:
           app: hello
       spec:
         containers:
         - name: hello-app
           image: gcr.io/google-samples/hello-app:1.0
           ports:
           - containerPort: 8080
           resources:
             requests:
               cpu: 100m
               memory: 128Mi
             limits:
               cpu: 200m
               memory: 256Mi
   EOF
   ```

2. Apply the deployment:

   ```bash
   kubectl apply -f deployment.yaml
   ```

3. Monitor deployment rollout:

   ```bash
   kubectl rollout status deployment/hello-app
   ```

4. Verify pods are running:

   ```bash
   kubectl get pods -l app=hello
   ```

**Expected Output:**

```
NAME                         READY   STATUS    RESTARTS   AGE
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Verification:**

- All three pods should show STATUS as "Running" and READY as "1/1"
- Check deployment details:

```bash
kubectl describe deployment hello-app
```

---

### Step 3: Create a Kubernetes Service

**Objective:** Expose the application internally within the cluster using a Service of type NodePort.

**Instructions:**

1. Create a service manifest:
   
   ```bash
   cat > service.yaml <<EOF
   apiVersion: v1
   kind: Service
   metadata:
     name: hello-service
     labels:
       app: hello
   spec:
     type: NodePort
     selector:
       app: hello
     ports:
     - protocol: TCP
       port: 80
       targetPort: 8080
   EOF
   ```

2. Apply the service:

   ```bash
   kubectl apply -f service.yaml
   ```

3. Verify the service was created:

   ```bash
   kubectl get service hello-service
   ```

4. Check service endpoints to confirm pod association:

   ```bash
   kubectl get endpoints hello-service
   ```

**Expected Output:**

```
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-service   NodePort   10.xx.xxx.xxx   <none>        80:xxxxx/TCP   10s
```

**Verification:**

- The service should have a CLUSTER-IP assigned
- Endpoints should list three IP addresses (one for each pod):

```bash
kubectl describe service hello-service
```

Look for the "Endpoints" field showing three pod IPs.

---

### Step 4: Create an Ingress Resource

**Objective:** Provision an external HTTP(S) Load Balancer by creating an Ingress resource that routes traffic to the Service.

**Instructions:**

1. Create an ingress manifest:
   
   ```bash
   cat > ingress.yaml <<EOF
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: hello-ingress
     annotations:
       kubernetes.io/ingress.class: "gce"
       kubernetes.io/ingress.global-static-ip-name: ""
   spec:
     defaultBackend:
       service:
         name: hello-service
         port:
           number: 80
   EOF
   ```

2. Apply the ingress:

   ```bash
   kubectl apply -f ingress.yaml
   ```

3. Monitor ingress provisioning (this may take 5-10 minutes):

   ```bash
   kubectl get ingress hello-ingress --watch
   ```

   Press `Ctrl+C` once you see an EXTERNAL-IP address assigned.

4. Get the external IP address:

   ```bash
   kubectl get ingress hello-ingress
   ```

**Expected Output:**

```
NAME            CLASS    HOSTS   ADDRESS          PORTS   AGE
hello-ingress   <none>   *       34.xxx.xxx.xxx   80      8m
```

**Verification:**

- Wait until the ADDRESS column shows an external IP (not `<pending>`)
- Verify the load balancer backend health in Cloud Console:

```bash
# Get the load balancer details
kubectl describe ingress hello-ingress
```

Look for annotations showing the backend service and health check details.

---

### Step 5: Test External Access and Load Balancing

**Objective:** Verify that the application is accessible via the external IP and that traffic is distributed across all three pods.

**Instructions:**

1. Store the external IP in a variable:
   
   ```bash
   export INGRESS_IP=$(kubectl get ingress hello-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "Ingress IP: $INGRESS_IP"
   ```

2. Test HTTP access using curl:

   ```bash
   curl http://$INGRESS_IP
   ```

3. Make multiple requests to observe load balancing across pods:

   ```bash
   for i in {1..10}; do
     curl -s http://$INGRESS_IP | grep "Hello, world!"
     sleep 1
   done
   ```

4. Check which pod is responding (the hostname will vary):

   ```bash
   for i in {1..6}; do
     echo "Request $i:"
     curl -s http://$INGRESS_IP
     echo ""
   done
   ```

5. Open the external IP in a web browser:

   ```bash
   echo "Open this URL in your browser: http://$INGRESS_IP"
   ```

**Expected Output:**

```
Hello, world!
Version: 1.0.0
Hostname: hello-app-xxxxxxxxxx-xxxxx
```

The hostname will change across requests, demonstrating load balancing.

**Verification:**

- Each request should return a successful HTTP 200 response
- The "Hostname" field should show different pod names across multiple requests
- The browser should display the hello-app web page

---

### Step 6: Scale the Deployment and Observe Backend Updates

**Objective:** Demonstrate how GKE automatically updates the load balancer backends when the deployment is scaled.

**Instructions:**

1. Scale the deployment to 5 replicas:
   
   ```bash
   kubectl scale deployment hello-app --replicas=5
   ```

2. Watch the new pods being created:

   ```bash
   kubectl get pods -l app=hello --watch
   ```

   Press `Ctrl+C` once all 5 pods are Running.

3. Verify the service endpoints now include 5 pods:

   ```bash
   kubectl get endpoints hello-service
   ```

4. Check the load balancer backend in Cloud Console:

   ```bash
   # Describe the ingress to see backend service details
   kubectl describe ingress hello-ingress
   ```

5. Test traffic distribution across 5 pods:

   ```bash
   for i in {1..10}; do
     echo "Request $i:"
     curl -s http://$INGRESS_IP | grep "Hostname"
   done
   ```

6. Scale back down to 2 replicas:

   ```bash
   kubectl scale deployment hello-app --replicas=2
   ```

7. Verify the reduction:

   ```bash
   kubectl get pods -l app=hello
   kubectl get endpoints hello-service
   ```

**Expected Output:**

After scaling to 5:
```
NAME                         READY   STATUS    RESTARTS   AGE
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
hello-app-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

**Verification:**

- Endpoints should reflect the current number of replicas
- All replicas should receive traffic from the load balancer
- The load balancer automatically adjusts without manual intervention

---

### Step 7: Inspect Load Balancer Configuration in Cloud Console

**Objective:** Understand how the Ingress resource maps to Google Cloud Load Balancer components.

**Instructions:**

1. Open the Cloud Console Load Balancing page:
   
   ```bash
   echo "Navigate to: https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers"
   ```

2. Find the load balancer created by the Ingress (name starts with `k8s2-`).

3. Click on the load balancer name to view details.

4. Examine the following components:
   - **Frontend**: External IP and port configuration
   - **Backend services**: Connection to the Kubernetes Service
   - **Health checks**: Automatic health check configuration
   - **Backend instances**: GKE node instances serving traffic

5. View backend health status:

   ```bash
   # Get backend service name from ingress annotations
   kubectl get ingress hello-ingress -o yaml | grep backend-service
   ```

**Expected Output:**

In the Cloud Console, you should see:
- Frontend with your external IP on port 80
- Backend service showing healthy backends
- Health check with path `/` (default)
- All backend instances in "Healthy" status

**Verification:**

- All backends should show as "Healthy" in the console
- Backend service should show connection to your GKE cluster
- Health check should have successful probe results

---

## Validation & Testing

### Success Criteria

- [ ] GKE Autopilot cluster is created and accessible via kubectl
- [ ] Deployment with 3 replicas is running successfully
- [ ] Service exposes the deployment on port 80
- [ ] Ingress resource has an external IP address assigned
- [ ] Application is accessible via HTTP on the external IP
- [ ] Traffic is distributed across multiple pods (verified by different hostnames)
- [ ] Scaling the deployment automatically updates load balancer backends
- [ ] Health checks show all backends as healthy in Cloud Console

### Testing Procedure

1. Comprehensive connectivity test:
   ```bash
   # Test basic connectivity
   curl -I http://$INGRESS_IP
   ```
   **Expected Result:** HTTP/1.1 200 OK

2. Load distribution test:
   ```bash
   # Collect hostnames from 20 requests
   for i in {1..20}; do
     curl -s http://$INGRESS_IP | grep "Hostname" | awk '{print $2}'
   done | sort | uniq -c
   ```
   **Expected Result:** Multiple different hostnames, relatively even distribution

3. Health check validation:
   ```bash
   # Check pod readiness
   kubectl get pods -l app=hello -o wide
   ```
   **Expected Result:** All pods show READY 1/1 and STATUS Running

4. Service endpoint verification:
   ```bash
   # Verify endpoints match pod count
   kubectl get endpoints hello-service -o yaml
   ```
   **Expected Result:** Number of addresses matches number of running pods

5. Ingress status check:
   ```bash
   # Get detailed ingress information
   kubectl describe ingress hello-ingress
   ```
   **Expected Result:** Shows backend service, forwarding rules, and no error events

## Troubleshooting

### Issue 1: Ingress ADDRESS Remains `<pending>`

**Symptoms:**
- `kubectl get ingress` shows ADDRESS as `<pending>` after 10+ minutes
- No external IP is assigned to the Ingress resource
- Application is not accessible from the internet

**Cause:**
The Google Cloud Load Balancer provisioning may be delayed due to backend health check failures, quota limits, or API permissions issues. GKE needs to create multiple resources (forwarding rules, backend services, health checks) which can take time.

**Solution:**

1. Check ingress events for errors:
   ```bash
   kubectl describe ingress hello-ingress
   ```

2. Verify the backend service health:
   ```bash
   # List all services and check for issues
   kubectl get events --sort-by='.lastTimestamp' | grep ingress
   ```

3. Ensure the service has healthy endpoints:
   ```bash
   kubectl get endpoints hello-service
   ```

4. Check IAM permissions for GKE service account:
   ```bash
   # Verify the GKE service account has compute.loadBalancerAdmin role
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:serviceAccount:service-*"
   ```

5. If the issue persists, delete and recreate the ingress:
   ```bash
   kubectl delete ingress hello-ingress
   kubectl apply -f ingress.yaml
   ```

6. Wait an additional 10-15 minutes and monitor:
   ```bash
   kubectl get ingress hello-ingress --watch
   ```

---

### Issue 2: HTTP Requests Return 502 Bad Gateway

**Symptoms:**
- Ingress has an external IP assigned
- Accessing the IP via browser or curl returns HTTP 502 error
- Error message: "Error: Server Error" or "502 Bad Gateway"

**Cause:**
Backend pods are not healthy or the health check is failing. The load balancer cannot route traffic to any healthy backend, resulting in 502 errors. This often happens when the Service targetPort doesn't match the container port, or pods are not ready.

**Solution:**

1. Verify all pods are running and ready:
   ```bash
   kubectl get pods -l app=hello
   ```

2. Check pod logs for errors:
   ```bash
   # Get logs from one of the pods
   kubectl logs -l app=hello --tail=50
   ```

3. Verify the Service is correctly configured:
   ```bash
   kubectl describe service hello-service
   ```
   Ensure the `TargetPort` matches the container port (8080).

4. Test pod connectivity directly:
   ```bash
   # Port-forward to a pod to test directly
   kubectl port-forward deployment/hello-app 8080:8080
   # In another terminal:
   curl http://localhost:8080
   ```

5. Check backend health in Cloud Console:
   - Navigate to Load Balancing → Select your load balancer → Backend services
   - Look for unhealthy backends and review health check details

6. Force health check configuration update:
   ```bash
   # Delete and recreate the ingress
   kubectl delete ingress hello-ingress
   sleep 30
   kubectl apply -f ingress.yaml
   ```

7. Wait 5-10 minutes for backends to become healthy and retry:
   ```bash
   curl -v http://$INGRESS_IP
   ```

---

### Issue 3: Traffic Not Distributed Across All Pods

**Symptoms:**
- Multiple requests always return the same hostname
- Only one pod appears to be receiving traffic
- Load balancing is not working as expected

**Cause:**
Session affinity may be enabled, or the load balancer is caching connections. Additionally, HTTP/1.1 keep-alive connections may route multiple requests to the same backend.

**Solution:**

1. Check service configuration for session affinity:
   ```bash
   kubectl get service hello-service -o yaml | grep sessionAffinity
   ```

2. If sessionAffinity is set to "ClientIP", change it to "None":
   ```bash
   kubectl patch service hello-service -p '{"spec":{"sessionAffinity":"None"}}'
   ```

3. Use different source IPs or close connections between requests:
   ```bash
   # Force new connections with curl
   for i in {1..10}; do
     curl -s -H "Connection: close" http://$INGRESS_IP | grep "Hostname"
     sleep 2
   done
   ```

4. Verify all pods are receiving traffic by checking access logs:
   ```bash
   # Check logs from all pods
   kubectl logs -l app=hello --tail=10 --prefix=true
   ```

5. Test from multiple external sources (different networks/machines) to verify distribution.

---

### Issue 4: Cluster Creation Fails with Quota Error

**Symptoms:**
- `gcloud container clusters create-auto` fails with quota exceeded error
- Error message mentions CPU, IP address, or in-use addresses quota

**Cause:**
Your Google Cloud project has insufficient quota for the required resources (CPU, IP addresses, or load balancers).

**Solution:**

1. Check current quota usage:
   ```bash
   gcloud compute project-info describe --project=$PROJECT_ID | grep -A 10 quotas
   ```

2. Request quota increase:
   - Navigate to: https://console.cloud.google.com/iam-admin/quotas
   - Filter by region and metric (e.g., "CPUs" in us-central1)
   - Select the quota and click "EDIT QUOTAS"
   - Request an increase

3. Alternatively, use a different region with available quota:
   ```bash
   export REGION="us-east1"
   gcloud config set compute/region $REGION
   # Retry cluster creation
   ```

4. For immediate testing, try Standard mode with minimal node count:
   ```bash
   gcloud container clusters create my-gke-cluster \
     --zone=us-central1-a \
     --num-nodes=1 \
     --machine-type=e2-medium
   ```

---

## Cleanup

**Important:** To avoid incurring charges, delete all resources created during this lab.

```bash
# Delete the Ingress (this removes the external load balancer)
kubectl delete ingress hello-ingress

# Wait for load balancer to be fully deleted (2-3 minutes)
echo "Waiting for load balancer deletion..."
sleep 120

# Delete the Service
kubectl delete service hello-service

# Delete the Deployment
kubectl delete deployment hello-app

# Verify all Kubernetes resources are deleted
kubectl get all

# Delete the GKE cluster (this may take 3-5 minutes)
gcloud container clusters delete my-gke-cluster \
  --region=$REGION \
  --quiet

# Verify cluster deletion
gcloud container clusters list --region=$REGION

# (Optional) Remove local manifest files
rm -f deployment.yaml service.yaml ingress.yaml
```

> ⚠️ **Warning:** Deleting the Ingress resource triggers deletion of the Google Cloud Load Balancer, but this process can take several minutes. Ensure you wait for complete deletion before removing the cluster to avoid orphaned load balancer resources. Always verify in the Cloud Console (Network Services → Load Balancing) that no load balancers remain.

> ⚠️ **Cost Note:** GKE clusters, especially Autopilot, incur charges for cluster management and compute resources. Load balancers also generate hourly charges. Complete cleanup promptly after finishing the lab.

## Summary

### What You Accomplished

- Created a fully managed GKE Autopilot cluster in Google Cloud
- Deployed a multi-replica containerized application using Kubernetes Deployments
- Configured a Kubernetes Service to expose the application internally
- Provisioned an external HTTP(S) Load Balancer using Kubernetes Ingress
- Verified Layer 7 load balancing with automatic health checks and traffic distribution
- Scaled the application and observed automatic backend synchronization
- Explored the integration between Kubernetes resources and Google Cloud Load Balancing infrastructure

### Key Takeaways

- **GKE Autopilot** simplifies cluster management by automatically provisioning and scaling nodes based on workload requirements
- **Kubernetes Ingress** on GKE automatically provisions Google Cloud HTTP(S) Load Balancers with health checks and backend services
- **Layer 7 load balancing** distributes HTTP traffic across multiple pods, providing high availability and scalability
- **Declarative configuration** via YAML manifests enables version-controlled, repeatable infrastructure deployments
- **Automatic health checks** ensure traffic is only routed to healthy pods, improving application reliability
- **Dynamic scaling** of deployments automatically updates load balancer backends without manual intervention
- The integration between GKE and Google Cloud networking services provides enterprise-grade load balancing with minimal configuration

### Next Steps

- Implement **HTTPS** by configuring managed SSL certificates with Google-managed certificates
- Add **path-based routing** to the Ingress to route different URL paths to different services
- Configure **Cloud Armor** security policies to protect your application from DDoS and web attacks
- Implement **horizontal pod autoscaling (HPA)** to automatically scale based on CPU or custom metrics
- Set up **Cloud Monitoring** dashboards to track load balancer metrics, latency, and error rates
- Explore **Workload Identity** to securely authenticate applications with Google Cloud services
- Practice **blue-green deployments** or canary releases using multiple services and weighted traffic splitting
- Integrate with **Cloud CDN** to cache static content and reduce latency for global users

## Additional Resources

- [GKE Autopilot Overview](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview) - Learn about GKE Autopilot mode and its benefits
- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/) - Official Kubernetes Ingress concepts and examples
- [GKE Ingress for HTTP(S) Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress) - GKE-specific Ingress implementation details
- [Configuring Ingress for External Load Balancing](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer) - Step-by-step tutorial for advanced Ingress configurations
- [GKE Best Practices: Networking](https://cloud.google.com/architecture/best-practices-for-running-cost-effective-kubernetes-applications-on-gke#networking) - Production networking recommendations
- [Troubleshooting Ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/load-balance-ingress#troubleshooting) - Official troubleshooting guide for GKE Ingress
- [HTTP(S) Load Balancing Concepts](https://cloud.google.com/load-balancing/docs/https) - Deep dive into Google Cloud Load Balancing architecture

---

# Lab 03-10-01: Práctica 9: Conectar Pub/Sub Cloud Function para procesamiento de eventos

## Metadata

| Property | Value |
|----------|-------|
| **Duration** | 30 minutes |
| **Complexity** | Beginner |
| **Bloom Level** | Understand |

## Overview

This lab introduces event-driven architecture on Google Cloud by connecting Cloud Functions with Pub/Sub messaging. You will create a Pub/Sub topic, deploy a Cloud Function that automatically triggers when messages are published to the topic, and validate end-to-end message processing. This pattern is fundamental for building decoupled, scalable microservices and real-time data pipelines in production environments.

## Learning Objectives

By completing this lab, you will be able to:

- [ ] Create a Pub/Sub topic and configure a Cloud Function Gen 2 with a Pub/Sub trigger
- [ ] Process and validate messages published to the topic using Cloud Logging
- [ ] Apply least-privilege IAM roles for Pub/Sub and Cloud Functions
- [ ] Monitor function execution metrics and logs for observability

## Prerequisites

### Required Knowledge

- Basic understanding of event-driven architectures and asynchronous messaging
- Familiarity with JSON data format
- Basic Python or Node.js syntax (reading level)
- Understanding of Google Cloud IAM roles and service accounts

### Required Access

- Google Cloud Project with billing enabled
- IAM permissions: `cloudfunctions.admin`, `pubsub.admin`, `iam.serviceAccountUser`, `logging.viewer`
- APIs enabled: Cloud Functions, Pub/Sub, Cloud Logging, Cloud Build, Eventarc

## Lab Environment

### Hardware Requirements

| Component | Specification |
|-----------|---------------|
| CPU | 2+ vCPU |
| RAM | 4+ GB |
| Network | ≥10 Mbps stable internet connection |
| Browser | Chrome, Edge, or Firefox (latest version) |

### Software Requirements

| Software | Version | Purpose |
|----------|---------|---------|
| Google Cloud SDK (gcloud) | 460.0.0+ | Deploy functions and publish Pub/Sub messages |
| Google Cloud Shell | Latest | Pre-configured environment with all tools |
| Python Runtime | 3.11 | Cloud Function runtime environment |
| Cloud Functions API | v2 | Second generation Cloud Functions with Eventarc |
| Pub/Sub API | v1 | Message queue service |
| Cloud Logging API | v2 | View function execution logs |

### Initial Setup

```bash
# Set your project ID
export PROJECT_ID=$(gcloud config get-value project)
export REGION=us-central1

# Enable required APIs
gcloud services enable cloudfunctions.googleapis.com \
  pubsub.googleapis.com \
  cloudbuild.googleapis.com \
  eventarc.googleapis.com \
  logging.googleapis.com \
  run.googleapis.com

# Verify API enablement
gcloud services list --enabled --filter="name:(cloudfunctions OR pubsub OR eventarc)"
```

## Step-by-Step Instructions

### Step 1: Create Pub/Sub Topic

**Objective:** Create a Pub/Sub topic to receive and distribute event messages.

**Instructions:**

1. Create the Pub/Sub topic named `pubsub-events`:

   ```bash
   gcloud pubsub topics create pubsub-events
   ```

2. Verify the topic was created successfully:

   ```bash
   gcloud pubsub topics list --filter="name:pubsub-events"
   ```

3. (Optional) Create a test subscription for manual message inspection:

   ```bash
   gcloud pubsub subscriptions create pubsub-events-test-sub \
     --topic=pubsub-events \
     --ack-deadline=60
   ```

**Expected Output:**

```
Created topic [projects/YOUR_PROJECT_ID/topics/pubsub-events].

---
name: projects/YOUR_PROJECT_ID/topics/pubsub-events
```

**Verification:**

- The topic `pubsub-events` appears in the list
- No error messages are displayed

---

### Step 2: Create Cloud Function Code

**Objective:** Prepare the Cloud Function source code to process Pub/Sub messages.

**Instructions:**

1. Create a working directory for the function:

   ```bash
   mkdir ~/pubsub-function
   cd ~/pubsub-function
   ```

2. Create the Python function code that logs and processes messages:

   ```bash
   cat > main.py << 'EOF'
import base64
import functions_framework
import json
import os
from datetime import datetime

@functions_framework.cloud_event
def process_pubsub_message(cloud_event):
    """
    Cloud Function triggered by Pub/Sub.
    Decodes and logs the message content.
    """
    # Extract Pub/Sub message data
    pubsub_message = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    
    # Log the received message
    print(f"[{datetime.utcnow().isoformat()}] Received Pub/Sub message:")
    print(f"Message content: {pubsub_message}")
    
    # Attempt to parse as JSON
    try:
        message_json = json.loads(pubsub_message)
        print(f"Parsed JSON: {json.dumps(message_json, indent=2)}")
        
        # Example: Extract specific fields
        if "event_type" in message_json:
            print(f"Event Type: {message_json['event_type']}")
        if "user_id" in message_json:
            print(f"User ID: {message_json['user_id']}")
            
    except json.JSONDecodeError:
        print("Message is not valid JSON, treating as plain text")
    
    print("Message processing completed successfully")
    return "OK"
EOF
   ```

3. Create the requirements file for Python dependencies:

   ```bash
   cat > requirements.txt << 'EOF'
functions-framework==3.5.0
EOF
   ```

**Expected Output:**

```
(Files created silently)
```

**Verification:**

- Confirm files exist:

  ```bash
  ls -la
  ```

- You should see `main.py` and `requirements.txt`

---

### Step 3: Deploy Cloud Function with Pub/Sub Trigger

**Objective:** Deploy the Cloud Function Gen 2 with a Pub/Sub trigger that automatically invokes the function when messages are published.

**Instructions:**

1. Deploy the function with Pub/Sub trigger:

   ```bash
   gcloud functions deploy process-pubsub-events \
     --gen2 \
     --runtime=python311 \
     --region=$REGION \
     --source=. \
     --entry-point=process_pubsub_message \
     --trigger-topic=pubsub-events \
     --max-instances=10 \
     --memory=256MB \
     --timeout=60s \
     --no-allow-unauthenticated
   ```

2. Wait for deployment to complete (this may take 2-4 minutes):

   ```bash
   gcloud functions describe process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --format="value(state)"
   ```

**Expected Output:**

```
Deploying function (may take a while - up to 2 minutes)...
...
Done.
...
state: ACTIVE
```

**Verification:**

- Function state shows `ACTIVE`
- No deployment errors in the output
- Verify the function exists:

  ```bash
  gcloud functions list --gen2 --region=$REGION
  ```

---

### Step 4: Configure IAM Permissions

**Objective:** Apply least-privilege IAM roles for secure message publishing and function invocation.

**Instructions:**

1. Get the default Compute Engine service account (used by Cloud Functions):

   ```bash
   export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
   export SERVICE_ACCOUNT="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
   echo "Service Account: $SERVICE_ACCOUNT"
   ```

2. Verify the function's service account has necessary permissions (these are auto-configured by Cloud Functions, but we verify):

   ```bash
   gcloud projects get-iam-policy $PROJECT_ID \
     --flatten="bindings[].members" \
     --filter="bindings.members:serviceAccount:$SERVICE_ACCOUNT" \
     --format="table(bindings.role)"
   ```

3. Grant Pub/Sub Publisher role to your user account for testing:

   ```bash
   gcloud pubsub topics add-iam-policy-binding pubsub-events \
     --member="user:$(gcloud config get-value account)" \
     --role="roles/pubsub.publisher"
   ```

**Expected Output:**

```
Service Account: 123456789012-compute@developer.gserviceaccount.com

Updated IAM policy for topic [pubsub-events].
bindings:
- members:
  - user:your-email@example.com
  role: roles/pubsub.publisher
```

**Verification:**

- Service account email is displayed correctly
- IAM policy binding confirms your user has `pubsub.publisher` role

---

### Step 5: Publish Test Messages

**Objective:** Publish messages to the Pub/Sub topic and trigger the Cloud Function.

**Instructions:**

1. Publish a simple text message:

   ```bash
   gcloud pubsub topics publish pubsub-events \
     --message="Hello from Pub/Sub - Test message 1"
   ```

2. Publish a JSON-formatted message with event data:

   ```bash
   gcloud pubsub topics publish pubsub-events \
     --message='{"event_type":"user_signup","user_id":"user123","timestamp":"2024-01-15T10:30:00Z","email":"user123@example.com"}'
   ```

3. Publish another JSON message simulating a different event:

   ```bash
   gcloud pubsub topics publish pubsub-events \
     --message='{"event_type":"order_placed","user_id":"user456","order_id":"ORD-789","amount":99.99}'
   ```

4. Wait 10-15 seconds for function execution to complete.

**Expected Output:**

```
messageIds:
- '12345678901234'
messageIds:
- '12345678901235'
messageIds:
- '12345678901236'
```

**Verification:**

- Each publish command returns a unique `messageId`
- No error messages appear

---

### Step 6: View Function Logs in Cloud Logging

**Objective:** Validate message processing by inspecting function execution logs.

**Instructions:**

1. Retrieve recent logs from the Cloud Function:

   ```bash
   gcloud functions logs read process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --limit=50 \
     --format="table(TIME_UTC, LOG)"
   ```

2. Filter logs to show only message content:

   ```bash
   gcloud functions logs read process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --limit=20 | grep -E "(Received|Message content|Event Type|User ID|processing completed)"
   ```

3. (Alternative) View logs in Cloud Console:

   ```bash
   echo "View logs at:"
   echo "https://console.cloud.google.com/functions/details/$REGION/process-pubsub-events?project=$PROJECT_ID&tab=logs"
   ```

**Expected Output:**

```
TIME_UTC                      LOG
2024-01-15 10:35:12.345       Received Pub/Sub message:
2024-01-15 10:35:12.346       Message content: Hello from Pub/Sub - Test message 1
2024-01-15 10:35:12.347       Message is not valid JSON, treating as plain text
2024-01-15 10:35:12.348       Message processing completed successfully
2024-01-15 10:35:18.123       Received Pub/Sub message:
2024-01-15 10:35:18.124       Message content: {"event_type":"user_signup",...}
2024-01-15 10:35:18.125       Event Type: user_signup
2024-01-15 10:35:18.126       User ID: user123
```

**Verification:**

- Logs show all three messages were received and processed
- JSON messages show parsed fields (event_type, user_id)
- Each message shows "Message processing completed successfully"

---

### Step 7: Monitor Function Metrics

**Objective:** Review function execution metrics for observability and performance analysis.

**Instructions:**

1. Get function execution count:

   ```bash
   gcloud functions describe process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --format="value(serviceConfig.availableMemory, serviceConfig.timeoutSeconds)"
   ```

2. View function details including update time:

   ```bash
   gcloud functions describe process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --format="yaml(name, state, updateTime, labels)"
   ```

3. Access Cloud Monitoring metrics in the console:

   ```bash
   echo "View metrics at:"
   echo "https://console.cloud.google.com/functions/details/$REGION/process-pubsub-events?project=$PROJECT_ID&tab=metrics"
   ```

4. Query function invocations using Cloud Monitoring (requires `gcloud alpha`):

   ```bash
   echo "In Cloud Console Metrics Explorer, query:"
   echo "Resource: Cloud Function"
   echo "Metric: function/execution_count"
   echo "Filter: function_name=process-pubsub-events"
   ```

**Expected Output:**

```
256MB
60

name: projects/YOUR_PROJECT_ID/locations/us-central1/functions/process-pubsub-events
state: ACTIVE
updateTime: '2024-01-15T10:30:00.123456Z'
```

**Verification:**

- Function state is `ACTIVE`
- Memory and timeout settings match deployment parameters
- Metrics URL is accessible in browser

---

### Step 8: Test Error Handling (Optional)

**Objective:** Observe how the function handles processing errors and retries.

**Instructions:**

1. Update the function to simulate an error condition:

   ```bash
   cat > main.py << 'EOF'
import base64
import functions_framework
import json

@functions_framework.cloud_event
def process_pubsub_message(cloud_event):
    pubsub_message = base64.b64decode(cloud_event.data["message"]["data"]).decode()
    print(f"Received message: {pubsub_message}")
    
    # Simulate error for messages containing "error"
    if "error" in pubsub_message.lower():
        print("ERROR: Simulated processing failure")
        raise Exception("Simulated error for testing")
    
    print("Message processing completed successfully")
    return "OK"
EOF
   ```

2. Redeploy the function:

   ```bash
   gcloud functions deploy process-pubsub-events \
     --gen2 \
     --runtime=python311 \
     --region=$REGION \
     --source=. \
     --entry-point=process_pubsub_message \
     --trigger-topic=pubsub-events \
     --max-instances=10 \
     --memory=256MB \
     --timeout=60s \
     --no-allow-unauthenticated \
     --quiet
   ```

3. Publish a message that triggers the error:

   ```bash
   gcloud pubsub topics publish pubsub-events \
     --message="This message contains error keyword"
   ```

4. View error logs:

   ```bash
   sleep 10
   gcloud functions logs read process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --limit=20 | grep -E "(ERROR|Simulated|Exception)"
   ```

**Expected Output:**

```
ERROR: Simulated processing failure
Exception: Simulated error for testing
```

**Verification:**

- Logs show the simulated error
- Pub/Sub automatically retries failed messages (you may see multiple attempts)

---

## Validation & Testing

### Success Criteria

- [ ] Pub/Sub topic `pubsub-events` is created and active
- [ ] Cloud Function `process-pubsub-events` is deployed in ACTIVE state
- [ ] Function successfully processes at least 3 test messages
- [ ] Cloud Logging shows message content and processing confirmation
- [ ] IAM permissions follow least-privilege principle
- [ ] Function metrics are visible in Cloud Monitoring

### Testing Procedure

1. Publish a comprehensive test message:

   ```bash
   gcloud pubsub topics publish pubsub-events \
     --message='{"event_type":"validation_test","timestamp":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","test_id":"final-validation"}'
   ```

   **Expected Result:** Message ID returned successfully

2. Verify message processing in logs:

   ```bash
   sleep 10
   gcloud functions logs read process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --limit=10 | grep "validation_test"
   ```

   **Expected Result:** Log entry shows the validation test message was received and processed

3. Confirm function is responsive:

   ```bash
   gcloud functions describe process-pubsub-events \
     --region=$REGION \
     --gen2 \
     --format="value(state, serviceConfig.uri)"
   ```

   **Expected Result:** State is `ACTIVE` and URI is displayed

## Troubleshooting

### Issue 1: Function Deployment Fails with Permission Errors

**Symptoms:**
- Error message: "Permission denied" or "requires roles/iam.serviceAccountUser"
- Deployment hangs or fails during build stage

**Cause:**
Your user account lacks the `iam.serviceAccountUser` role required to deploy functions that run as service accounts.

**Solution:**

```bash
# Grant yourself the Service Account User role
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/iam.serviceAccountUser"

# Wait 30 seconds for IAM propagation
sleep 30

# Retry deployment
gcloud functions deploy process-pubsub-events \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=. \
  --entry-point=process_pubsub_message \
  --trigger-topic=pubsub-events \
  --max-instances=10 \
  --memory=256MB \
  --timeout=60s \
  --no-allow-unauthenticated
```

---

### Issue 2: Published Messages Not Triggering Function

**Symptoms:**
- Messages publish successfully (messageId returned)
- No corresponding logs appear in Cloud Logging
- Function shows 0 invocations in metrics

**Cause:**
The Pub/Sub trigger may not be properly configured, or Eventarc service account lacks permissions.

**Solution:**

```bash
# Verify the function's trigger configuration
gcloud functions describe process-pubsub-events \
  --region=$REGION \
  --gen2 \
  --format="yaml(eventTrigger)"

# Check if Eventarc service account has Pub/Sub subscriber role
export EVENTARC_SA="service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com"

gcloud pubsub topics add-iam-policy-binding pubsub-events \
  --member="serviceAccount:${EVENTARC_SA}" \
  --role="roles/pubsub.subscriber"

# Wait 30 seconds, then publish a new test message
sleep 30
gcloud pubsub topics publish pubsub-events --message="Retry test message"
```

---

### Issue 3: Logs Not Appearing in Cloud Logging

**Symptoms:**
- Function appears to execute (metrics show invocations)
- No logs visible when running `gcloud functions logs read`
- Cloud Console Logs Explorer shows no entries

**Cause:**
Logs may take 30-60 seconds to appear, or logging permissions are missing.

**Solution:**

```bash
# Wait longer for logs to propagate
sleep 60

# Use Cloud Logging API directly with broader time window
gcloud logging read "resource.type=cloud_function AND resource.labels.function_name=process-pubsub-events" \
  --limit=20 \
  --format="table(timestamp, textPayload)" \
  --freshness=5m

# Verify Cloud Logging API is enabled
gcloud services enable logging.googleapis.com

# Check service account has logging write permissions
gcloud projects get-iam-policy $PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:$SERVICE_ACCOUNT AND bindings.role:roles/logging.logWriter"
```

---

### Issue 4: Function Timeout or Memory Errors

**Symptoms:**
- Logs show "Function execution took X ms, finished with status: 'timeout'"
- Error: "Exceeded memory limit"

**Cause:**
Function requires more time or memory than allocated.

**Solution:**

```bash
# Increase timeout and memory
gcloud functions deploy process-pubsub-events \
  --gen2 \
  --runtime=python311 \
  --region=$REGION \
  --source=. \
  --entry-point=process_pubsub_message \
  --trigger-topic=pubsub-events \
  --max-instances=10 \
  --memory=512MB \
  --timeout=120s \
  --no-allow-unauthenticated \
  --quiet

# Verify updated configuration
gcloud functions describe process-pubsub-events \
  --region=$REGION \
  --gen2 \
  --format="value(serviceConfig.availableMemory, serviceConfig.timeoutSeconds)"
```

---

## Cleanup

```bash
# Delete the Cloud Function
gcloud functions delete process-pubsub-events \
  --region=$REGION \
  --gen2 \
  --quiet

# Delete the Pub/Sub topic (this also deletes associated subscriptions)
gcloud pubsub topics delete pubsub-events --quiet

# Delete the test subscription if created separately
gcloud pubsub subscriptions delete pubsub-events-test-sub --quiet 2>/dev/null || true

# Remove local function code
cd ~
rm -rf ~/pubsub-function

# (Optional) Remove IAM policy binding if no longer needed
gcloud pubsub topics remove-iam-policy-binding pubsub-events \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/pubsub.publisher" 2>/dev/null || true

# Verify cleanup
echo "Verifying cleanup..."
gcloud functions list --region=$REGION --gen2 | grep process-pubsub-events || echo "✓ Function deleted"
gcloud pubsub topics list | grep pubsub-events || echo "✓ Topic deleted"
```

> ⚠️ **Warning:** Deleting the Pub/Sub topic will permanently remove all messages and subscriptions. Ensure you have captured any necessary data before cleanup. Cloud Functions in Gen 2 are backed by Cloud Run services; deletion may take 1-2 minutes to fully propagate.

## Summary

### What You Accomplished

- Created a Pub/Sub topic (`pubsub-events`) for asynchronous message distribution
- Deployed a Cloud Function Gen 2 with a Pub/Sub trigger that automatically processes incoming messages
- Published multiple test messages (plain text and JSON) and validated processing through Cloud Logging
- Applied IAM best practices by granting minimum necessary permissions for publishers and function execution
- Monitored function execution using Cloud Logging and reviewed available metrics in Cloud Monitoring
- Tested error handling and observed automatic retry behavior

### Key Takeaways

- **Event-driven architecture** decouples message producers from consumers, enabling scalable and resilient systems
- **Cloud Functions Gen 2** with Pub/Sub triggers provide automatic scaling and built-in retry logic for failed messages
- **Cloud Logging** is essential for debugging and validating serverless function execution
- **Least-privilege IAM** requires careful role assignment: `pubsub.publisher` for senders, automatic permissions for Eventarc service accounts
- **Base64 encoding** is used by Pub/Sub to transmit message data; Cloud Functions framework handles decoding automatically

### Next Steps

- Extend the function to write processed messages to Cloud Storage or BigQuery for persistence
- Implement message filtering using Pub/Sub message attributes
- Add dead-letter topics to handle messages that repeatedly fail processing
- Integrate with Cloud Monitoring to create alerts on function error rates or execution latency
- Explore Pub/Sub message ordering and exactly-once delivery for critical workflows
- Combine with Cloud Scheduler to create scheduled event-driven workflows

## Additional Resources

- [Cloud Functions Gen 2 Documentation](https://cloud.google.com/functions/docs/2nd-gen/overview) - Architecture and capabilities of second-generation functions
- [Pub/Sub Triggers for Cloud Functions](https://cloud.google.com/functions/docs/calling/pubsub) - Detailed guide on Pub/Sub integration patterns
- [Pub/Sub Message Format](https://cloud.google.com/pubsub/docs/reference/rest/v1/PubsubMessage) - Understanding message structure and attributes
- [Cloud Functions Best Practices](https://cloud.google.com/functions/docs/bestpractices/tips) - Performance, reliability, and security recommendations
- [Eventarc Documentation](https://cloud.google.com/eventarc/docs) - Event delivery system underlying Cloud Functions Gen 2
- [Cloud Logging Query Language](https://cloud.google.com/logging/docs/view/logging-query-language) - Advanced log filtering and analysis
- [Pub/Sub IAM Roles](https://cloud.google.com/pubsub/docs/access-control) - Detailed permissions reference for topic and subscription access
