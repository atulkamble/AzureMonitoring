# Azure Monitoring & Troubleshooting – Practical Commands and Scripts

A hands-on lab guide covering Azure Monitor, Log Analytics, Azure CLI, Virtual Machines, Networking, Web Apps, and AKS troubleshooting.

## 1. Prerequisites and configuration

Install Azure CLI and authenticate.

```
# Check Azure CLI
az version

# Login
az login

# List subscriptions
az account list -o table

# Set subscription
az account set --subscription "<SUBSCRIPTION_ID>"

# Check current subscription
az account show -o table

# Install monitoring extension if needed
az extension add --name application-insights --upgrade
```

Configure variables:

```
RG="myRG"
LOCATION="eastus"
VM="myVM"
LAW="my-log-workspace"
AKS="myAKS"
WEBAPP="myWebApp"
```

## 2. Azure resource monitoring

```
# List resource groups
az group list -o table

# List all resources
az resource list -o table

# List VMs
az vm list -d -o table

# Check VM status
az vm get-instance-view \
  -g $RG -n $VM \
  --query "instanceView.statuses" -o table

# List metric definitions for VM
VM_ID=$(az vm show -g $RG -n $VM --query id -o tsv)

az monitor metrics list-definitions \
  --resource "$VM_ID" -o table

# Monitor CPU usage
az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --interval PT5M \
  --aggregation Average \
  -o json
```

## 3. Azure VM troubleshooting

### Check VM health

```
# VM power state
az vm get-instance-view \
  -g $RG -n $VM \
  --query "instanceView.statuses[-1]" -o json

# Boot diagnostics
az vm boot-diagnostics get-boot-log \
  -g $RG -n $VM

# VM network interfaces
az vm nic list \
  -g $RG --vm-name $VM -o table

# Restart VM
az vm restart -g $RG -n $VM
```

### Inside Linux VM

```
# System status
uptime
hostnamectl

# CPU and memory
top
free -h
vmstat 1 5

# Disk usage
df -h
lsblk
du -sh /var/log

# Running processes
ps aux --sort=-%cpu | head

# System logs
sudo journalctl -p err -n 50
sudo journalctl -xe

# Network listening ports
sudo ss -tulnp

# Apache service
sudo systemctl status httpd

# Nginx service
sudo systemctl status nginx
```

For Ubuntu Apache, use `apache2` instead of `httpd`.

## 4. Log Analytics Workspace

```
# Create workspace
az monitor log-analytics workspace create \
  -g $RG \
  -n $LAW \
  -l $LOCATION

# List workspaces
az monitor log-analytics workspace list \
  -g $RG -o table

# Show workspace
az monitor log-analytics workspace show \
  -g $RG -n $LAW

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  -g $RG -n $LAW \
  --query customerId -o tsv)

echo "$WORKSPACE_ID"
```

### Run KQL queries

```
az monitor log-analytics query \
  --workspace "$WORKSPACE_ID" \
  --analytics-query \
  "Heartbeat | take 10" \
  -o table
```

Heartbeat records require a configured data collection rule and a connected monitoring agent. Creating a workspace alone does not populate VM logs.

## 5. KQL troubleshooting queries

Run these in Azure Portal → Log Analytics Workspace → Logs.

Check VM heartbeat

```
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastSeen=max(TimeGenerated) by Computer
| order by LastSeen desc
```

Check high CPU usage

```
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| where InstanceName == "_Total"
| summarize AvgCPU=avg(CounterValue) by Computer
| order by AvgCPU desc
```

Check low memory

```
Perf
| where TimeGenerated > ago(1h)
| where CounterName == "Available MBytes"
| summarize AvailableMB=avg(CounterValue) by Computer
| order by AvailableMB asc
```

Check Linux errors

```
Syslog
| where TimeGenerated > ago(24h)
| where SeverityLevel in ("err", "crit", "alert", "emerg")
| project TimeGenerated, Computer, Facility, SeverityLevel, SyslogMessage
| order by TimeGenerated desc
```

Check Windows errors

```
Event
| where TimeGenerated > ago(24h)
| where EventLevelName == "Error"
| project TimeGenerated, Computer, Source, RenderedDescription
| order by TimeGenerated desc
```

These queries depend on the corresponding performance counters and logs being collected.

## 6. Azure Monitor alerts

Create a CPU utilization alert for a VM.

```
az monitor metrics alert create \
  --name "HighCPUAlert" \
  --resource-group "$RG" \
  --scopes "$VM_ID" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "VM CPU above 80 percent"
```

Check alerts:

```
# List metric alert rules
az monitor metrics alert list \
  -g $RG -o table

# Show alert configuration
az monitor metrics alert show \
  -g $RG -n HighCPUAlert

# List fired alerts
az monitor alert list \
  --resource-group $RG -o table
```

An action group must be associated separately to deliver email, SMS, or webhook notifications.

## 7. Azure networking troubleshooting

```
# List VNets
az network vnet list -o table

# List NSGs
az network nsg list -o table

# List public IPs
az network public-ip list -o table

# List load balancers
az network lb list -o table

# List network interfaces
az network nic list -o table
```

### Test connectivity using Network Watcher

```
# Check Network Watcher
az network watcher list -o table

# Get VM resource ID
VM_ID=$(az vm show \
  -g $RG -n $VM --query id -o tsv)

# Test outbound connectivity to HTTPS
az network watcher test-connectivity \
  --source-resource "$VM_ID" \
  --dest-address www.microsoft.com \
  --dest-port 443
```

### Test effective NSG rules

```
az network nic list-effective-nsg \
  --resource-group "$RG" \
  --name "<NIC_NAME>" \
  -o json
```

### Inside VM

```
# DNS resolution
nslookup microsoft.com

# HTTPS connectivity
curl -I https://www.microsoft.com

# Test TCP port
nc -zv <PRIVATE_IP> 3306

# Routing table
ip route

# IP addresses
ip addr

# Trace network path
traceroute 8.8.8.8
```

## 8. Azure App Service troubleshooting

```
# List Web Apps
az webapp list -o table

# Check Web App status
az webapp show \
  -g $RG -n $WEBAPP \
  --query "{name:name,state:state,host:defaultHostName}"

# Enable application logging (Linux)
az webapp log config \
  -g $RG -n $WEBAPP \
  --application-logging filesystem \
  --level information

# Stream logs
az webapp log tail \
  -g $RG -n $WEBAPP

# Restart application
az webapp restart \
  -g $RG -n $WEBAPP

# Test HTTP response
curl -I "https://${WEBAPP}.azurewebsites.net"
```

For App Service, also inspect Application Insights for exceptions, requests, and dependencies.

## 9. AKS monitoring and troubleshooting

```
# List AKS clusters
az aks list -o table

# Get cluster credentials
az aks get-credentials \
  -g $RG -n $AKS \
  --overwrite-existing

# Cluster information
kubectl cluster-info

# Check nodes
kubectl get nodes -o wide

# Check all pods
kubectl get pods -A

# Check services
kubectl get svc -A

# Check deployments
kubectl get deployments -A

# Check resource usage
kubectl top nodes
kubectl top pods -A
```

### Troubleshoot failing pods

```
# Describe pod
kubectl describe pod <POD_NAME> -n <NAMESPACE>

# Check logs
kubectl logs <POD_NAME> -n <NAMESPACE>

# Previous crashed container logs
kubectl logs <POD_NAME> \
  -n <NAMESPACE> --previous

# Cluster events
kubectl get events -A \
  --sort-by=.lastTimestamp

# Check rollout
kubectl rollout status \
  deployment/<DEPLOYMENT_NAME> \
  -n <NAMESPACE>

# Restart deployment
kubectl rollout restart \
  deployment/<DEPLOYMENT_NAME> \
  -n <NAMESPACE>
```

`kubectl top` requires Metrics Server or an equivalent metrics API provider.

## 10. Practical troubleshooting scripts

### Script 1 – Linux VM health check

Save as `vm-health-check.sh`.

```
#!/bin/bash

echo "===== SYSTEM HEALTH ====="

echo "Hostname:"
hostname

echo "Uptime:"
uptime

echo "Memory:"
free -h

echo "Disk:"
df -h

echo "CPU:"
top -bn1 | head -n 10

echo "Network:"
ss -tuln

echo "Recent Errors:"
journalctl -p err -n 10 --no-pager
```

Run:

```
chmod +x vm-health-check.sh
./vm-health-check.sh
```

### Script 2 – Azure VM CPU monitor

Save as `azure-cpu-monitor.sh`.

```
#!/bin/bash
set -euo pipefail

RG="myRG"
VM="myVM"

VM_ID=$(az vm show \
  -g "$RG" -n "$VM" \
  --query id -o tsv)

echo "Monitoring CPU for $VM"

az monitor metrics list \
  --resource "$VM_ID" \
  --metric "Percentage CPU" \
  --interval PT1M \
  --aggregation Average \
  --query "value[0].timeseries[0].data[].{Time:timeStamp,CPU:average}" \
  -o table
```

### Script 3 – AKS troubleshooting report

Save as `aks-troubleshoot.sh`.

```
#!/bin/bash
set -euo pipefail

echo "===== AKS TROUBLESHOOTING ====="

kubectl get nodes -o wide

echo "===== PODS ====="
kubectl get pods -A -o wide

echo "===== SERVICES ====="
kubectl get svc -A

echo "===== RESOURCE USAGE ====="
kubectl top nodes || true

echo "===== RECENT EVENTS ====="
kubectl get events -A \
  --sort-by=.lastTimestamp | tail -30
```

Run:

```
chmod +x aks-troubleshoot.sh
./aks-troubleshoot.sh
```

## 11. Hands-on monitoring test scenarios

| Test            | Action                                               | Expected observation                 |
| --------------- | ---------------------------------------------------- | ------------------------------------ |
| High CPU        | Run `stress-ng --cpu 2 --timeout 120s` on VM         | CPU metric increases                 |
| High memory     | Run `stress-ng --vm 1 --vm-bytes 256M --timeout 60s` | Memory consumption increases         |
| Disk usage      | Run `fallocate -l 100M /tmp/testfile`                | Disk usage increases                 |
| Apache failure  | `sudo systemctl stop httpd`                          | HTTP service becomes unavailable     |
| NSG restriction | Temporarily block a test port                        | Connectivity test fails              |
| AKS pod failure | Deploy an invalid container image                    | `ImagePullBackOff`                   |
| Web App failure | Deploy an application with a startup error           | Application logs show errors         |
| CPU alert       | Generate CPU load exceeding configured threshold     | Alert rule may fire after evaluation |

Use a dedicated test VM and verify available resources before generating load. `stress-ng` may need installation. Revert NSG changes, restart stopped services, and delete test files after testing.

## 12. Troubleshooting command quick reference

| Issue             | Commands                                                        |
| ----------------- | --------------------------------------------------------------- |
| VM not running    | `az vm get-instance-view`                                       |
| High CPU          | `top`, `az monitor metrics list`                                |
| Memory pressure   | `free -h`, `vmstat`                                             |
| Disk full         | `df -h`, `du -sh`                                               |
| SSH failure       | `az network nic list-effective-nsg`, `ss -tuln`                 |
| DNS issue         | `nslookup`, `dig`                                               |
| Web server down   | `systemctl status httpd`                                        |
| Application error | `az webapp log tail`                                            |
| AKS pod crash     | `kubectl describe pod`, `kubectl logs`                          |
| AKS node issue    | `kubectl describe node`                                         |
| Missing logs      | Check Azure Monitor Agent, DCR, and workspace                   |
| Alert not firing  | Check alert rule, metric data, threshold, and evaluation window |

Recommended practical lab flow:

Azure VM → Azure Monitor Metrics → Log Analytics Workspace → Azure Monitor Agent + Data Collection Rule → KQL Queries → Metric Alerts → Action Groups → Network Watcher → Application Insights → AKS Monitoring → Troubleshooting Tests.

This sequence provides an end-to-end Azure monitoring and troubleshooting practice lab.
