# Azure Automated Backup System

[LinkedIn](https://linkedin.com/in/dylan-bryson-b24952181) | [GitHub](https://github.com/Dylanbeejames)

---

## Walkthrough

Watch the full project walkthrough here: https://www.loom.com/share/b06a73b5a5da4d5e9679511a64872cb7

---

## Overview

A fully automated backup system built with Terraform on Microsoft Azure. Designed to replace unreliable manual backup processes with self-running infrastructure that replicates, versions, tiers, monitors, and alerts without human intervention.

Every file uploaded is instantly replicated across geographically separate Azure data centers. Every overwrite or deletion is versioned and recoverable. Costs are controlled automatically through lifecycle tiering. A daily confirmation email proves the system is healthy without anyone having to log in and check.

---

## Architecture

```
rg-backup-dylan  (East US)
│
├── stbackupdylan  (Storage Account)
│   Standard GRS · TLS 1.2 · Versioning Enabled · Soft Delete 30d
│   │
│   ├── documents/
│   ├── database-exports/
│   └── application-files/
│
├── backup-lifecycle  (Lifecycle Policy)
│   Hot → Cool at 30d → Archive at 90d → Delete at 365d
│   Old versions purged after 30d
│
├── law-backup-dylan  (Log Analytics Workspace)
│   30d retention · StorageRead / StorageWrite / StorageDelete
│
├── la-backup-confirm-dylan  (Logic App)
│   Recurrence: Daily 8AM
│   Lists blobs → Sends confirmation email
│
└── alert-no-backup-writes  (Metric Alert)
    Checks every 1hr over a 24hr window
    Fires if PutBlob + PutBlock = 0
    Routes to ag-backup-dylan (Action Group → Email)
```

---

## Resources Deployed

| Resource | Name | Purpose |
|----------|------|---------|
| Resource Group | rg-backup-dylan | Container for all project resources |
| Storage Account | stbackupdylan | GRS replicated backup store with versioning |
| Container | documents | General document backups |
| Container | database-exports | Database dump backups |
| Container | application-files | Application asset backups |
| Lifecycle Policy | backup-lifecycle | Automatic cost tiering by file age |
| Log Analytics Workspace | law-backup-dylan | Centralized log aggregation |
| Diagnostic Setting | diag-storage-to-law | Routes storage telemetry to Log Analytics |
| Action Group | ag-backup-dylan | Alert notification routing |
| Logic App | la-backup-confirm-dylan | Daily email confirmation workflow |
| Metric Alert | alert-no-backup-writes | Triggers on zero write activity in 24hrs |

---

## Lifecycle Cost Management

| File Age | Storage Tier | Purpose |
|----------|-------------|---------|
| 0 to 30 days | Hot | Active backup window |
| 30 to 90 days | Cool | Infrequent access, reduced cost |
| 90 to 365 days | Archive | Long term retention, minimal cost |
| 365+ days | Deleted | Automatic purge |

---

## Prerequisites

- Terraform v1.0 or higher
- Azure CLI installed and authenticated via `az login`
- Active Azure subscription

---

## Deployment

Clone the repository:

```bash
git clone https://github.com/Dylanbeejames/azure-backup-system.git
cd azure-backup-system
```

Copy the example variables file and fill in your values:

```bash
cp example.tfvars terraform.tfvars
```

```hcl
yourname    = "yourname"
location    = "East US"
alert_email = "your@email.com"
```

Initialize and deploy:

```bash
terraform init
terraform plan
terraform apply
```

Expect 11 resources created in approximately 2 to 3 minutes.

Configure the Logic App in the Azure portal after deployment:

1. Navigate to la-backup-confirm-[yourname]
2. Open Logic app designer
3. Add a Recurrence trigger set to Daily at 8:00 AM
4. Add a List blobs action connected to your storage account
5. Add a Send an email action with your confirmation message
6. Save the workflow

---

## Testing Versioning

```powershell
# Upload first version
"Backup test file created $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name stbackup[yourname] `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --account-key "<your-account-key>"

# Upload second version
"Updated content - second version $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name stbackup[yourname] `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --account-key "<your-account-key>" `
  --overwrite

# List versions
az storage blob list `
  --account-name stbackup[yourname] `
  --container-name documents `
  --include v `
  --account-key "<your-account-key>" `
  --output table
```

Expected output: two rows for test/backup_test.txt with different lengths and timestamps confirming versioning is active.

---

## Verification Checklist

- [ ] Storage account visible in the Azure portal
- [ ] Versioning shows Enabled under Data Management
- [ ] Lifecycle policy backup-lifecycle rule is present
- [ ] Three containers exist: documents, database-exports, application-files
- [ ] Logic App runs on a daily recurrence trigger
- [ ] Alert rule alert-no-backup-writes exists under Monitor
- [ ] Test file upload produced two versions in blob list output

---

## Known Issues

Gmail is not compatible with Azure Blob Storage connectors inside Logic Apps due to a Microsoft policy restriction. Use Outlook.com or Office 365 Outlook as the email connector instead.

---

## If I Kept Building This

- Add an explicit secondary storage account in a paired region for manual failover control
- Refactor into a Terraform module for reuse across multiple environments
- Enforce backup tagging standards across the subscription using Azure Policy
- Integrate with Azure Backup Center for a unified management plane
- Connect Log Analytics to a Power BI dashboard for backup activity reporting

---

## Teardown

```bash
terraform destroy
```

---

Built by Dylan Bryson
[LinkedIn](https://linkedin.com/in/dylan-bryson-b24952181) | [GitHub](https://github.com/Dylanbeejames)

Last updated: May 2026
