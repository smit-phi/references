# ☁️ Azure Cloud Services: A Comprehensive Beginner's Guide

> A structured, hands-on guide to nine core Azure services — from foundational concepts to practical exercises.

---

## Table of Contents

1. [Introduction to Cloud Computing](#introduction-to-cloud-computing)
2. [Azure Virtual Machines (VMs)](#1-azure-virtual-machines-vms)
3. [Azure SQL Database](#2-azure-sql-database)
4. [Azure Blob Storage](#3-azure-blob-storage)
5. [Azure Functions](#4-azure-functions)
6. [Azure Active Directory (AD)](#5-azure-active-directory-ad)
7. [Azure Service Bus](#6-azure-service-bus)
8. [Azure Cosmos DB](#7-azure-cosmos-db)
9. [Azure SendGrid (Email Service)](#8-azure-sendgrid-email-service)
10. [Azure API Management](#9-azure-api-management)
11. [Next Steps & Learning Resources](#next-steps--learning-resources)

---

## Introduction to Cloud Computing

### What is Cloud Computing?

Cloud computing is the delivery of computing services — servers, storage, databases, networking, software, analytics, and intelligence — over the internet ("the cloud") to offer faster innovation, flexible resources, and economies of scale.

Instead of buying and maintaining physical hardware, you **rent** compute power and storage from a cloud provider and pay only for what you use.

### The Three Service Models

| Model | Full Name | You Manage | Provider Manages | Example |
|-------|-----------|------------|-----------------|---------|
| **IaaS** | Infrastructure as a Service | OS, Runtime, Apps, Data | Hardware, Networking, Virtualization | Azure VMs |
| **PaaS** | Platform as a Service | Apps & Data only | Everything below apps | Azure SQL Database |
| **SaaS** | Software as a Service | Nothing (just use it) | Everything | Microsoft 365 |

### Why Azure?

Microsoft Azure is one of the world's leading cloud platforms, offering 200+ products and services. It integrates tightly with Microsoft's ecosystem (Windows, Active Directory, Office 365) and is trusted by enterprises worldwide.

### Prerequisites

Before diving in, you will need:

- A **free Azure account**: Sign up at [azure.microsoft.com/free](https://azure.microsoft.com/free) — you get $200 in credits for 30 days
- **Azure CLI** installed: [docs.microsoft.com/cli/azure/install-azure-cli](https://docs.microsoft.com/cli/azure/install-azure-cli)
- Basic familiarity with the **Azure Portal**: [portal.azure.com](https://portal.azure.com)

---

## 1. Azure Virtual Machines (VMs)

### What Are Azure VMs?

Azure Virtual Machines (VMs) are on-demand, scalable computing resources. They give you the flexibility of virtualization without the need to buy and maintain the physical hardware. A VM is like a computer within a computer — running its own operating system and applications.

**Use cases:**
- Hosting web servers and applications
- Running development/test environments
- Lift-and-shift migration of on-premises workloads
- High-performance computing (HPC)

### Core Concepts

**VM Sizes** — Azure groups VMs into series based on workload type:

| Series | Purpose | Example |
|--------|---------|---------|
| B-series | Burstable, low-cost dev/test | Standard_B1s |
| D-series | General purpose | Standard_D2s_v5 |
| E-series | Memory optimized | Standard_E4s_v5 |
| N-series | GPU-enabled | Standard_NC6 |

**OS Disk vs Data Disk:**
- **OS Disk** — where the operating system lives (persists with VM)
- **Data Disk** — additional storage you attach for application data
- **Temporary Disk** — fast local SSD, lost on restart; use only for swap/cache

**Availability Options:**
- **Availability Sets** — spreads VMs across fault domains within a datacenter
- **Availability Zones** — spreads VMs across physically separate datacenters in a region
- **VM Scale Sets** — automatically increase/decrease the number of VMs based on demand

**Networking:**
- Every VM gets a **Virtual Network Interface Card (NIC)**
- Attached to a **Virtual Network (VNet)** and a **Subnet**
- Controlled by a **Network Security Group (NSG)** — acts as a firewall with inbound/outbound rules

---

### Hands-On Practice: Azure VMs

#### Lab 1 — Create a Linux VM via Azure Portal

1. Go to [portal.azure.com](https://portal.azure.com) → click **"Create a resource"**
2. Search **"Virtual Machine"** → click **Create**
3. Fill in the **Basics** tab:
   - **Resource Group:** Create new → `rg-vm-lab`
   - **VM Name:** `my-first-vm`
   - **Region:** East US (or your nearest)
   - **Image:** Ubuntu Server 22.04 LTS
   - **Size:** Standard_B1s (free tier eligible)
   - **Authentication:** SSH public key → generate new key pair
4. In the **Networking** tab — leave defaults (a new VNet and NSG are auto-created)
5. Click **Review + Create** → **Create**
6. Download the `.pem` private key when prompted

#### Lab 2 — Connect to Your VM via SSH

```bash
# Step 1: Fix key permissions (Linux/macOS)
chmod 400 ~/Downloads/my-first-vm_key.pem

# Step 2: Get the public IP from the Azure Portal (VM Overview page)
# Step 3: Connect
ssh -i ~/Downloads/my-first-vm_key.pem azureuser@<YOUR_PUBLIC_IP>

# Step 4: Verify you're inside the VM
uname -a
df -h
```

#### Lab 3 — Create a VM Using Azure CLI

```bash
# Log in to Azure
az login

# Create a resource group
az group create --name rg-cli-lab --location eastus

# Create the VM
az vm create \
  --resource-group rg-cli-lab \
  --name cli-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

# Open port 80 for web traffic
az vm open-port --port 80 --resource-group rg-cli-lab --name cli-vm

# Install a web server inside the VM
az vm run-command invoke \
  --resource-group rg-cli-lab \
  --name cli-vm \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx && sudo systemctl start nginx"
```

Now visit `http://<YOUR_PUBLIC_IP>` in a browser — you should see the NGINX welcome page.

#### Lab 4 — Resize and Stop a VM

```bash
# List available VM sizes in your region
az vm list-vm-resize-options \
  --resource-group rg-cli-lab \
  --name cli-vm \
  --output table

# Resize the VM (deallocate first if needed)
az vm resize \
  --resource-group rg-cli-lab \
  --name cli-vm \
  --size Standard_B2s

# Deallocate (stop billing for compute — storage still billed)
az vm deallocate --resource-group rg-cli-lab --name cli-vm

# Start it back up
az vm start --resource-group rg-cli-lab --name cli-vm
```

> **Cost Tip:** Always deallocate VMs when not in use. A *stopped* VM (not deallocated) still charges for compute.

#### Lab 5 — Cleanup

```bash
# Delete the entire resource group (removes all resources inside)
az group delete --name rg-cli-lab --yes --no-wait
```

---

## 2. Azure SQL Database

### What Is Azure SQL Database?

Azure SQL Database is a fully managed relational database service built on the Microsoft SQL Server engine. It handles patching, backups, high availability, and scaling automatically — so you focus on your data, not the infrastructure.

**Use cases:**
- Web and mobile app backends
- SaaS application data stores
- Transactional systems needing ACID guarantees

### Core Concepts

**Deployment Models:**

| Model | Description |
|-------|-------------|
| **Single Database** | One dedicated database with its own resources |
| **Elastic Pool** | Multiple databases share a pool of resources (cost-efficient for variable workloads) |
| **Managed Instance** | Near 100% compatibility with on-prem SQL Server; for full migration |

**Purchasing Models:**

| Model | Unit | Best For |
|-------|------|---------|
| **DTU** | Database Transaction Units (bundled compute+I/O+memory) | Predictable workloads |
| **vCore** | Virtual cores + separate storage | Control and hybrid benefit savings |
| **Serverless** | Auto-pause/resume; bill per second of use | Intermittent or unpredictable usage |

**Service Tiers (vCore):**
- **General Purpose** — balanced compute and storage
- **Business Critical** — high I/O, low-latency with local SSD
- **Hyperscale** — scales to 100TB+ with rapid scale-out

**Backup Strategy:**
- **Full backups** — weekly
- **Differential backups** — every 12-24 hours
- **Transaction log backups** — every 5-12 minutes
- **Retention** — 7 to 35 days (configurable)
- **Long-Term Retention (LTR)** — up to 10 years using Azure Blob Storage

**Firewall Rules:**
Azure SQL has a server-level firewall. You must explicitly allow IP addresses (or Azure services) before connections are accepted.

---

### Hands-On Practice: Azure SQL Database

#### Lab 1 — Create an Azure SQL Database

```bash
# Set variables
RESOURCE_GROUP="rg-sql-lab"
SERVER_NAME="sqlsrv-$(date +%s)"   # Must be globally unique
DB_NAME="myappdb"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssword1234!"         # Use a strong password
LOCATION="eastus"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create SQL Server (the logical server)
az sql server create \
  --name $SERVER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS

# Allow your current IP to connect
MY_IP=$(curl -s https://api.ipify.org)
az sql server firewall-rule create \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name AllowMyIP \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP

# Create the database (serverless, free tier eligible)
az sql db create \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --compute-model Serverless \
  --auto-pause-delay 60
```

#### Lab 2 — Connect and Run Queries

Install **sqlcmd** or use Azure Cloud Shell. Then:

```bash
# Get the server FQDN
SERVER_FQDN="${SERVER_NAME}.database.windows.net"

# Connect using sqlcmd
sqlcmd -S $SERVER_FQDN -U $ADMIN_USER -P $ADMIN_PASS -d $DB_NAME
```

Once inside the sqlcmd prompt:

```sql
-- Create a table
CREATE TABLE Products (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    Name NVARCHAR(100) NOT NULL,
    Price DECIMAL(10,2),
    CreatedAt DATETIME2 DEFAULT GETUTCDATE()
);
GO

-- Insert sample data
INSERT INTO Products (Name, Price) VALUES
    ('Widget A', 9.99),
    ('Widget B', 24.50),
    ('Widget C', 4.75);
GO

-- Query the data
SELECT * FROM Products;
GO

-- Exit
EXIT
```

#### Lab 3 — Configure Scaling

```bash
# Scale up to 4 vCores
az sql db update \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --capacity 4

# Check current database info
az sql db show \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --output table
```

#### Lab 4 — Configure Backup Retention

```bash
# Set short-term backup retention to 14 days
az sql db str-policy set \
  --resource-group $RESOURCE_GROUP \
  --server $SERVER_NAME \
  --name $DB_NAME \
  --retention-days 14
```

#### Lab 5 — Cleanup

```bash
az group delete --name rg-sql-lab --yes --no-wait
```

---

## 3. Azure Blob Storage

### What Is Azure Blob Storage?

Azure Blob Storage is Microsoft's object storage solution for the cloud. It is optimized for storing massive amounts of unstructured data — text files, images, videos, logs, backups, and more.

**"Blob"** stands for **Binary Large OBject**.

**Use cases:**
- Static website hosting
- Media (images, video) delivery
- Backup and archive
- Data lake for analytics
- Serving files to browser/mobile apps

### Core Concepts

**Storage Account → Container → Blob**

The hierarchy works like this:
```
Storage Account (globally unique name)
└── Container (like a folder, flat namespace)
    ├── blob1.jpg
    ├── data/report.csv      ← "/" in name simulates folders
    └── logs/app.log
```

**Blob Types:**

| Type | Description | Use Case |
|------|-------------|---------|
| **Block Blob** | Stored as blocks, up to 190.7 TB | Files, media, documents |
| **Append Blob** | Optimized for append operations | Log files |
| **Page Blob** | Optimized for random read/write | Azure VM disks (VHDs) |

**Access Tiers (Lifecycle Management):**

| Tier | Storage Cost | Access Cost | Latency | Use For |
|------|-------------|-------------|---------|---------|
| **Hot** | Higher | Lower | Immediate | Frequently accessed data |
| **Cool** | Lower | Higher | Immediate | Infrequent access (30+ day min) |
| **Cold** | Lower | Higher | Immediate | Rarely accessed (90+ day min) |
| **Archive** | Lowest | Highest | Hours (rehydrate needed) | Long-term backup/compliance |

**Redundancy Options:**

| Option | Copies | Scope |
|--------|--------|-------|
| **LRS** (Locally Redundant Storage) | 3 | Single datacenter |
| **ZRS** (Zone Redundant Storage) | 3 | 3 zones in one region |
| **GRS** (Geo-Redundant Storage) | 6 | Primary + secondary region |
| **GZRS** | 6 | ZRS primary + GRS secondary |

**Access Control:**
- **Public access** — fully public (not recommended for sensitive data)
- **SAS tokens** — Shared Access Signatures; time-limited, scoped permissions
- **Azure AD + RBAC** — enterprise identity-based access
- **Access keys** — full account-level access (rotate regularly)

---

### Hands-On Practice: Azure Blob Storage

#### Lab 1 — Create a Storage Account and Container

```bash
RESOURCE_GROUP="rg-storage-lab"
STORAGE_ACCOUNT="stglab$(date +%s | tail -c 8)"   # Must be 3-24 lowercase alphanumeric
CONTAINER_NAME="mycontainer"
LOCATION="eastus"

az group create --name $RESOURCE_GROUP --location $LOCATION

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot

# Get the connection string
CONN_STR=$(az storage account show-connection-string \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --output tsv)

# Create a container
az storage container create \
  --name $CONTAINER_NAME \
  --connection-string $CONN_STR
```

#### Lab 2 — Upload, Download, and List Blobs

```bash
# Create a test file
echo "Hello from Azure Blob Storage!" > hello.txt

# Upload the file
az storage blob upload \
  --container-name $CONTAINER_NAME \
  --file hello.txt \
  --name hello.txt \
  --connection-string $CONN_STR

# List blobs in the container
az storage blob list \
  --container-name $CONTAINER_NAME \
  --connection-string $CONN_STR \
  --output table

# Download the blob to a new file
az storage blob download \
  --container-name $CONTAINER_NAME \
  --name hello.txt \
  --file downloaded_hello.txt \
  --connection-string $CONN_STR

cat downloaded_hello.txt
```

#### Lab 3 — Generate a SAS Token

```bash
# Generate a SAS token valid for 1 hour
EXPIRY=$(date -u -d "1 hour" '+%Y-%m-%dT%H:%MZ' 2>/dev/null || \
         date -u -v +1H '+%Y-%m-%dT%H:%MZ')  # macOS fallback

SAS_TOKEN=$(az storage blob generate-sas \
  --container-name $CONTAINER_NAME \
  --name hello.txt \
  --permissions r \
  --expiry $EXPIRY \
  --connection-string $CONN_STR \
  --output tsv)

# Build the full URL
ACCOUNT_URL="https://${STORAGE_ACCOUNT}.blob.core.windows.net"
BLOB_URL="${ACCOUNT_URL}/${CONTAINER_NAME}/hello.txt?${SAS_TOKEN}"
echo "Access URL: $BLOB_URL"

# Test it
curl "$BLOB_URL"
```

#### Lab 4 — Set Up a Lifecycle Policy

This automatically moves blobs to cooler tiers or deletes them based on age.

```bash
# Create lifecycle policy JSON
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "name": "MoveToCoolAfter30Days",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
EOF

# Apply the policy
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --policy @lifecycle-policy.json
```

#### Lab 5 — Cleanup

```bash
az group delete --name rg-storage-lab --yes --no-wait
```

---

## 4. Azure Functions

### What Are Azure Functions?

Azure Functions is a **serverless compute** service that lets you run small pieces of code ("functions") without worrying about servers, infrastructure, or scaling. You write the code; Azure handles the rest.

**Key idea:** You only pay for the time your code actually runs. When idle, you pay nothing (in the Consumption plan).

**Use cases:**
- Processing uploaded files (image resize, document parsing)
- Responding to events (new queue message, database change)
- Scheduled tasks (data cleanup, report generation)
- Lightweight REST API endpoints
- IoT data processing

### Core Concepts

**Triggers** — what causes a function to run:

| Trigger | Description |
|---------|-------------|
| **HTTP** | Called via an HTTP request (like a REST API) |
| **Timer** | Runs on a CRON schedule |
| **Blob Storage** | Fires when a blob is created/modified |
| **Service Bus** | Fires when a message arrives in a queue/topic |
| **Cosmos DB** | Fires when documents change (change feed) |
| **Event Grid / Hub** | Fires on cloud events |

**Bindings** — declarative connections to other services (input or output):

```javascript
// Example: HTTP trigger → read from Blob → write to Cosmos DB
// All declared in function.json, no SDK calls needed for connections
```

**Hosting Plans:**

| Plan | Scaling | Cold Start | Best For |
|------|---------|------------|---------|
| **Consumption** | Auto (0 to ∞) | Yes | Event-driven, intermittent |
| **Premium** | Pre-warmed instances | No | Always-on, VNet integration |
| **Dedicated (App Service)** | Manual/auto | No | Predictable, co-located with apps |

**Durable Functions** — extends Azure Functions with stateful workflows. Uses the Orchestrator/Activity pattern to coordinate long-running processes without managing state yourself.

---

### Hands-On Practice: Azure Functions

#### Lab 1 — Create a Function App

```bash
RESOURCE_GROUP="rg-functions-lab"
STORAGE_ACCOUNT="stgfunc$(date +%s | tail -c 8)"
FUNCTION_APP="funcapp-$(date +%s | tail -c 8)"
LOCATION="eastus"

az group create --name $RESOURCE_GROUP --location $LOCATION

# Storage account (required by Functions)
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --sku Standard_LRS

# Create the Function App (Node.js 18, Consumption plan)
az functionapp create \
  --name $FUNCTION_APP \
  --resource-group $RESOURCE_GROUP \
  --storage-account $STORAGE_ACCOUNT \
  --consumption-plan-location $LOCATION \
  --runtime node \
  --runtime-version 18 \
  --functions-version 4 \
  --os-type Linux
```

#### Lab 2 — Create an HTTP Trigger Function

First, install Azure Functions Core Tools:

```bash
# macOS
brew tap azure/functions
brew install azure-functions-core-tools@4

# Windows (via npm)
npm install -g azure-functions-core-tools@4 --unsafe-perm true

# Ubuntu/Debian
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > /etc/apt/trusted.gpg.d/microsoft.gpg
sudo apt-get install azure-functions-core-tools-4
```

Now create and run locally:

```bash
# Create a new Functions project
mkdir my-functions && cd my-functions
func init --worker-runtime node --language javascript

# Create an HTTP trigger function
func new --name HttpGreeter --template "HTTP trigger" --authlevel anonymous
```

Edit `HttpGreeter/index.js`:

```javascript
module.exports = async function (context, req) {
    context.log('HttpGreeter function triggered.');

    const name = req.query.name || (req.body && req.body.name) || 'World';
    const message = `Hello, ${name}! This is Azure Functions.`;

    context.res = {
        status: 200,
        headers: { "Content-Type": "application/json" },
        body: { message, timestamp: new Date().toISOString() }
    };
};
```

```bash
# Run locally
func start

# Test it (in another terminal)
curl "http://localhost:7071/api/HttpGreeter?name=Azure"
```

#### Lab 3 — Deploy to Azure

```bash
# Deploy from within the project folder
func azure functionapp publish $FUNCTION_APP

# Get the function URL
az functionapp function show \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP \
  --function-name HttpGreeter \
  --query invokeUrlTemplate \
  --output tsv
```

#### Lab 4 — Timer Trigger Function

```bash
# Create a timer function (runs every minute)
func new --name MinuteTimer --template "Timer trigger"
```

Edit `MinuteTimer/index.js`:

```javascript
module.exports = async function (context, myTimer) {
    const now = new Date().toISOString();
    if (myTimer.isPastDue) {
        context.log('Timer is running late!');
    }
    context.log(`[${now}] Timer function executed.`);
};
```

Edit `MinuteTimer/function.json` to set the schedule:

```json
{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 * * * * *"
    }
  ]
}
```

#### Lab 5 — Cleanup

```bash
az group delete --name rg-functions-lab --yes --no-wait
```

---

## 5. Azure Active Directory (AD)

### What Is Azure AD?

Azure Active Directory (now called **Microsoft Entra ID**) is Microsoft's cloud-based identity and access management (IAM) service. It authenticates and authorizes users, services, and devices — both inside and outside your organization.

Think of it as the **bouncer** for all your Azure resources and Microsoft 365 apps.

**Use cases:**
- Single Sign-On (SSO) for employees across all company apps
- Multi-Factor Authentication (MFA) for security
- Controlling who can access which Azure resources
- External identity (let customers sign in with Google/Facebook)
- Application identity (service principals, managed identities)

### Core Concepts

**Key Objects:**

| Object | Description |
|--------|-------------|
| **User** | A person with credentials in the directory |
| **Group** | A collection of users (assign permissions to groups, not individuals) |
| **Application (App Registration)** | Represents an app that uses Azure AD for auth |
| **Service Principal** | The identity of an app within a specific tenant |
| **Managed Identity** | Automatically managed service identity for Azure resources (no secrets!) |

**Authentication Protocols:**
- **OAuth 2.0** — Authorization framework; used for delegated access
- **OpenID Connect (OIDC)** — Identity layer on top of OAuth 2.0; returns an ID token
- **SAML 2.0** — Older enterprise SSO protocol; supported for legacy apps

**Single Sign-On (SSO):**
Configure once in Azure AD. Users sign in once and access all connected apps (Microsoft 365, Salesforce, GitHub, etc.) without re-entering credentials.

**Multi-Factor Authentication (MFA):**
Requires users to verify identity with a second factor — authenticator app, SMS, phone call, or hardware token. Can be enforced by **Conditional Access policies** based on risk, location, device, or app.

**RBAC (Role-Based Access Control):**
Azure uses RBAC to control who can do what on which Azure resources. Roles are assigned at different scopes:
```
Management Group → Subscription → Resource Group → Resource
(broader scope inherits to narrower scope)
```

Common built-in roles:
- **Owner** — full access including managing access
- **Contributor** — create/manage resources, no access management
- **Reader** — read-only view
- **User Access Administrator** — manage access only

---

### Hands-On Practice: Azure AD

#### Lab 1 — Create a User and Group

```bash
# Create a new user
az ad user create \
  --display-name "Alice Developer" \
  --user-principal-name "alice@<YOUR_TENANT>.onmicrosoft.com" \
  --password "TempP@ssword123!" \
  --force-change-password-next-sign-in true

# List users
az ad user list --output table

# Create a group
az ad group create \
  --display-name "Developers" \
  --mail-nickname "developers"

# Add Alice to the group
ALICE_ID=$(az ad user show --id "alice@<YOUR_TENANT>.onmicrosoft.com" --query id --output tsv)
GROUP_ID=$(az ad group show --group "Developers" --query id --output tsv)

az ad group member add \
  --group $GROUP_ID \
  --member-id $ALICE_ID
```

#### Lab 2 — Register an Application

```bash
# Register an app (e.g., for a web API)
az ad app create \
  --display-name "MyWebApp" \
  --sign-in-audience AzureADMyOrg

# Get the App (client) ID
APP_ID=$(az ad app list --display-name "MyWebApp" --query "[0].appId" --output tsv)
echo "App ID: $APP_ID"

# Create a service principal for the app
az ad sp create --id $APP_ID

# Create a client secret (note: save it — shown only once)
az ad app credential reset \
  --id $APP_ID \
  --years 1
```

#### Lab 3 — Assign an RBAC Role

```bash
# Get your subscription ID
SUB_ID=$(az account show --query id --output tsv)

# Assign the Reader role to Alice on the subscription
az role assignment create \
  --assignee "alice@<YOUR_TENANT>.onmicrosoft.com" \
  --role "Reader" \
  --scope "/subscriptions/$SUB_ID"

# List role assignments
az role assignment list \
  --assignee "alice@<YOUR_TENANT>.onmicrosoft.com" \
  --output table
```

#### Lab 4 — Enable MFA via Conditional Access (Portal)

1. Go to [portal.azure.com](https://portal.azure.com) → **Microsoft Entra ID**
2. Navigate to **Security → Conditional Access → New Policy**
3. Name: `Require MFA for All Users`
4. **Users**: All users (or select a group)
5. **Cloud apps**: All cloud apps
6. **Grant**: Require multi-factor authentication
7. Enable the policy → **Save**

> Note: Exclude your admin account from MFA policies during setup to avoid lockout.

#### Lab 5 — Create a Managed Identity

```bash
# Create a User-Assigned Managed Identity
az identity create \
  --name my-managed-identity \
  --resource-group rg-functions-lab  # Reuse any RG

# Assign it the Storage Blob Data Reader role
IDENTITY_PRINCIPAL=$(az identity show \
  --name my-managed-identity \
  --resource-group rg-functions-lab \
  --query principalId --output tsv)

az role assignment create \
  --assignee $IDENTITY_PRINCIPAL \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/$SUB_ID"
```

With managed identities, your code (running in a VM, Function, or AKS) can access Azure services **without any stored secrets**.

---

## 6. Azure Service Bus

### What Is Azure Service Bus?

Azure Service Bus is a fully managed enterprise message broker. It enables decoupled communication between applications and services using **messages** — rather than direct calls. If the receiver is busy or offline, messages wait safely in the broker.

**Use cases:**
- Order processing systems (orders → fulfillment → shipping)
- Microservices communication
- Load leveling (absorb traffic spikes)
- Workflows requiring guaranteed message delivery

### Core Concepts

**Queues vs Topics:**

| Feature | Queue | Topic + Subscriptions |
|---------|-------|-----------------------|
| Pattern | Point-to-point | Publish/Subscribe |
| Receivers | One consumer per message | Many consumers (each gets a copy) |
| Use case | Task queue, work distribution | Event broadcasting |

**Message Properties:**
- **TTL (Time to Live)** — how long before an undelivered message expires
- **Dead-Letter Queue (DLQ)** — messages that fail or expire land here for inspection
- **Message Lock** — consumer locks the message during processing; unlock = requeue, complete = delete
- **Duplicate Detection** — detect and discard duplicate messages within a time window
- **Sessions** — group related messages for ordered, FIFO processing

**Tiers:**

| Tier | Max Message Size | Topics | Geo-Disaster Recovery |
|------|-----------------|--------|----------------------|
| **Basic** | 256 KB | No | No |
| **Standard** | 256 KB | Yes | No |
| **Premium** | 100 MB | Yes | Yes |

---

### Hands-On Practice: Azure Service Bus

#### Lab 1 — Create a Service Bus Namespace and Queue

```bash
RESOURCE_GROUP="rg-servicebus-lab"
NAMESPACE="sbns-$(date +%s | tail -c 8)"
QUEUE_NAME="orders"
LOCATION="eastus"

az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Service Bus namespace (Standard tier for topics support)
az servicebus namespace create \
  --name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard

# Create a queue
az servicebus queue create \
  --name $QUEUE_NAME \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --max-size 1024 \
  --default-message-time-to-live P7D \
  --dead-lettering-on-message-expiration true
```

#### Lab 2 — Send and Receive Messages (Node.js)

```bash
mkdir sb-demo && cd sb-demo
npm init -y
npm install @azure/service-bus
```

Create `send.js`:

```javascript
const { ServiceBusClient } = require("@azure/service-bus");

// Get from: Azure Portal > Service Bus > Shared Access Policies > RootManageSharedAccessKey
const CONNECTION_STRING = process.env.SERVICEBUS_CONNECTION_STRING;
const QUEUE_NAME = "orders";

async function sendMessages() {
    const sbClient = new ServiceBusClient(CONNECTION_STRING);
    const sender = sbClient.createSender(QUEUE_NAME);

    const messages = [
        { body: { orderId: "ORD-001", item: "Widget A", qty: 3 }, messageId: "ORD-001" },
        { body: { orderId: "ORD-002", item: "Widget B", qty: 1 }, messageId: "ORD-002" },
        { body: { orderId: "ORD-003", item: "Widget C", qty: 5 }, messageId: "ORD-003" },
    ];

    await sender.sendMessages(messages);
    console.log(`Sent ${messages.length} messages to queue: ${QUEUE_NAME}`);

    await sender.close();
    await sbClient.close();
}

sendMessages().catch(console.error);
```

Create `receive.js`:

```javascript
const { ServiceBusClient } = require("@azure/service-bus");

const CONNECTION_STRING = process.env.SERVICEBUS_CONNECTION_STRING;
const QUEUE_NAME = "orders";

async function receiveMessages() {
    const sbClient = new ServiceBusClient(CONNECTION_STRING);
    const receiver = sbClient.createReceiver(QUEUE_NAME, { receiveMode: "peekLock" });

    const messages = await receiver.receiveMessages(3, { maxWaitTimeInMs: 5000 });

    for (const msg of messages) {
        console.log("Received:", msg.body);
        await receiver.completeMessage(msg);   // Removes from queue
    }

    await receiver.close();
    await sbClient.close();
}

receiveMessages().catch(console.error);
```

```bash
# Get connection string
CONN_STR=$(az servicebus namespace authorization-rule keys list \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $NAMESPACE \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString --output tsv)

export SERVICEBUS_CONNECTION_STRING="$CONN_STR"

# Send messages
node send.js

# Receive messages
node receive.js
```

#### Lab 3 — Create a Topic with Subscriptions

```bash
TOPIC_NAME="notifications"

# Create topic
az servicebus topic create \
  --name $TOPIC_NAME \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP

# Create two subscriptions
az servicebus topic subscription create \
  --name "email-service" \
  --topic-name $TOPIC_NAME \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP

az servicebus topic subscription create \
  --name "sms-service" \
  --topic-name $TOPIC_NAME \
  --namespace-name $NAMESPACE \
  --resource-group $RESOURCE_GROUP
```

Now each subscription receives **its own copy** of every message sent to the topic.

#### Lab 4 — Cleanup

```bash
az group delete --name rg-servicebus-lab --yes --no-wait
```

---

## 7. Azure Cosmos DB

### What Is Azure Cosmos DB?

Azure Cosmos DB is a fully managed, globally distributed NoSQL database. It supports multiple data models and APIs, scales automatically, and offers industry-leading SLAs for throughput, latency, availability, and consistency.

**Key differentiators:**
- **Global distribution** — replicate data to any Azure region with one click
- **Multi-model** — store documents, key-value, graph, and column-family data
- **Tunable consistency** — choose your consistency-vs-latency trade-off
- **Guaranteed <10ms read/write latency** at the 99th percentile

**Use cases:**
- Real-time personalization (e-commerce, gaming)
- IoT telemetry ingestion
- Globally distributed user profiles
- Product catalogs with flexible schemas

### Core Concepts

**APIs (choose one per account):**

| API | Best For | Compatibility |
|-----|---------|---------------|
| **NoSQL** (Core SQL) | Document JSON data | Native Cosmos DB |
| **MongoDB** | MongoDB migration | MongoDB wire protocol |
| **Cassandra** | Wide-column data | Apache Cassandra |
| **Gremlin** | Graph data | Apache TinkerPop |
| **Table** | Key-value data | Azure Table Storage |

**Hierarchy:**
```
Cosmos DB Account
└── Database
    └── Container (collection/table/graph)
        └── Items (documents/rows/nodes)
```

**Consistency Levels** (from strongest to weakest):

| Level | Description | Trade-off |
|-------|-------------|-----------|
| **Strong** | Linearizable reads | Highest latency, lowest throughput |
| **Bounded Staleness** | Reads lag by at most K versions/T seconds | Predictable lag |
| **Session** | Consistent within a client session | Default; recommended for most apps |
| **Consistent Prefix** | Reads never see out-of-order writes | — |
| **Eventual** | Weakest; reads may be stale | Lowest latency, highest throughput |

**Throughput Models:**
- **Provisioned (RU/s)** — you reserve Request Units per second; predictable cost
- **Autoscale** — scales between 10%–100% of max RU/s automatically
- **Serverless** — pay per operation; best for dev/test and unpredictable traffic

**Partition Key:** The field used to distribute data across physical partitions. Choosing a good partition key is critical — aim for high cardinality and even distribution.

---

### Hands-On Practice: Azure Cosmos DB

#### Lab 1 — Create a Cosmos DB Account

```bash
RESOURCE_GROUP="rg-cosmosdb-lab"
ACCOUNT_NAME="cosmos-$(date +%s | tail -c 8)"
LOCATION="eastus"

az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Cosmos DB account (NoSQL API, serverless)
az cosmosdb create \
  --name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --locations regionName=$LOCATION \
  --capabilities EnableServerless

# Create a database
az cosmosdb sql database create \
  --account-name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --name "ecommerce"

# Create a container (partition key = /category)
az cosmosdb sql container create \
  --account-name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --database-name "ecommerce" \
  --name "products" \
  --partition-key-path "/category"
```

#### Lab 2 — CRUD Operations (Node.js SDK)

```bash
mkdir cosmos-demo && cd cosmos-demo
npm init -y
npm install @azure/cosmos
```

Create `cosmos-demo.js`:

```javascript
const { CosmosClient } = require("@azure/cosmos");

const endpoint = process.env.COSMOS_ENDPOINT;
const key = process.env.COSMOS_KEY;
const client = new CosmosClient({ endpoint, key });

const database = client.database("ecommerce");
const container = database.container("products");

async function run() {
    // CREATE — add items
    const items = [
        { id: "prod-1", category: "electronics", name: "Wireless Headphones", price: 79.99, stock: 150 },
        { id: "prod-2", category: "electronics", name: "Bluetooth Speaker", price: 49.99, stock: 200 },
        { id: "prod-3", category: "clothing", name: "Running Shoes", price: 89.99, stock: 75 },
    ];

    for (const item of items) {
        const { resource } = await container.items.create(item);
        console.log(`Created: ${resource.name}`);
    }

    // READ — query by category
    const { resources: electronics } = await container.items
        .query({
            query: "SELECT * FROM c WHERE c.category = @category AND c.price < @maxPrice",
            parameters: [
                { name: "@category", value: "electronics" },
                { name: "@maxPrice", value: 60 }
            ]
        })
        .fetchAll();

    console.log("\nElectronics under $60:");
    electronics.forEach(p => console.log(`  - ${p.name}: $${p.price}`));

    // UPDATE — change price of prod-1
    const { resource: existing } = await container.item("prod-1", "electronics").read();
    existing.price = 69.99;
    await container.item("prod-1", "electronics").replace(existing);
    console.log("\nUpdated prod-1 price to $69.99");

    // DELETE
    await container.item("prod-3", "clothing").delete();
    console.log("Deleted prod-3");
}

run().catch(console.error);
```

```bash
# Get Cosmos DB credentials
COSMOS_ENDPOINT=$(az cosmosdb show \
  --name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query documentEndpoint --output tsv)

COSMOS_KEY=$(az cosmosdb keys list \
  --name $ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query primaryMasterKey --output tsv)

export COSMOS_ENDPOINT COSMOS_KEY
node cosmos-demo.js
```

#### Lab 3 — Enable Global Distribution (Portal)

1. Navigate to your Cosmos DB account in the Azure Portal
2. Click **Replicate data globally**
3. Click a region on the map to add it as a read region
4. Click **Save** — your data is now replicated within minutes

Your app can read from the nearest region automatically with the connection policy.

#### Lab 4 — Cleanup

```bash
az group delete --name rg-cosmosdb-lab --yes --no-wait
```

---

## 8. Azure SendGrid (Email Service)

### What Is Azure SendGrid?

SendGrid (acquired by Twilio) is an email delivery platform available as a service in the Azure Marketplace. It provides a reliable, scalable infrastructure for sending **transactional emails** (order confirmations, password resets, notifications) and **marketing emails**.

**Use cases:**
- Application email notifications
- User onboarding emails
- Password reset emails
- Invoice and receipt delivery
- Email marketing campaigns

### Core Concepts

**Email Types:**
- **Transactional** — triggered by user actions (welcome email, receipt, alert)
- **Marketing** — bulk campaigns, newsletters

**Integration Methods:**
- **SMTP relay** — configure your app's SMTP settings to use SendGrid's servers
- **Web API (v3)** — REST API for programmatic, reliable email delivery (recommended)
- **SDKs** — official libraries for Node.js, Python, C#, PHP, Java, Go

**Authentication (Email Deliverability):**
- **SPF** — proves your domain is authorized to send via SendGrid
- **DKIM** — adds a digital signature to emails; prevents tampering
- **Domain Authentication** — configure SPF and DKIM for your custom domain in SendGrid settings

**Rate Limits by Plan:**

| Plan | Daily Limit | Monthly |
|------|------------|---------|
| Free | 100/day | ~3,000 |
| Essentials 50K | 2,000/day | 50,000 |
| Essentials 100K | 5,000/day | 100,000 |
| Pro | Custom | Millions |

**IP Reputation:**
- Free/shared plans use shared IPs (can be affected by other senders)
- Paid plans offer dedicated IPs for full control over sender reputation

---

### Hands-On Practice: Azure SendGrid

#### Lab 1 — Create a SendGrid Account via Azure Marketplace

```bash
RESOURCE_GROUP="rg-sendgrid-lab"
SENDGRID_NAME="sendgrid-myapp"

az group create --name $RESOURCE_GROUP --location eastus

# Create SendGrid account via Azure Marketplace
az resource create \
  --resource-group $RESOURCE_GROUP \
  --name $SENDGRID_NAME \
  --resource-type "Sendgrid.Email/accounts" \
  --api-version "2015-01-01" \
  --properties '{
    "plan": {
      "name": "free",
      "publisher": "Sendgrid",
      "product": "sendgrid_azure",
      "promotionCode": ""
    },
    "password": "SecureP@ssword123!",
    "acceptMarketingEmails": false,
    "email": "your-email@domain.com"
  }'
```

> Alternatively, create it in the **Azure Portal → Marketplace → SendGrid** with a UI wizard.

#### Lab 2 — Generate an API Key

1. Log into [app.sendgrid.com](https://app.sendgrid.com)
2. Go to **Settings → API Keys → Create API Key**
3. Name it `MyAppKey`, select **Full Access** (or limit to Mail Send only)
4. Copy and save the key — it is shown only once

#### Lab 3 — Send an Email via the REST API

```bash
# Store API key securely
export SENDGRID_API_KEY="SG.xxxxxxxxxxxxxxxx"

# Send a test email using curl
curl --request POST \
  --url https://api.sendgrid.com/v3/mail/send \
  --header "Authorization: Bearer $SENDGRID_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
    "personalizations": [
      {
        "to": [{ "email": "recipient@example.com", "name": "Test Recipient" }],
        "subject": "Hello from Azure SendGrid!"
      }
    ],
    "from": { "email": "sender@yourdomain.com", "name": "My App" },
    "content": [
      {
        "type": "text/plain",
        "value": "This is a test email sent via the SendGrid API from Azure."
      },
      {
        "type": "text/html",
        "value": "<h2>Hello!</h2><p>This is a test email sent via <strong>SendGrid</strong>.</p>"
      }
    ]
  }'
```

#### Lab 4 — Send Email with Node.js SDK

```bash
mkdir sendgrid-demo && cd sendgrid-demo
npm init -y
npm install @sendgrid/mail
```

Create `send-email.js`:

```javascript
const sgMail = require('@sendgrid/mail');
sgMail.setApiKey(process.env.SENDGRID_API_KEY);

async function sendWelcomeEmail(userEmail, userName) {
    const msg = {
        to: userEmail,
        from: 'noreply@yourdomain.com',
        subject: `Welcome to MyApp, ${userName}!`,
        text: `Hi ${userName}, thanks for signing up!`,
        html: `
            <div style="font-family: Arial, sans-serif; max-width: 600px;">
                <h1 style="color: #0078d4;">Welcome, ${userName}!</h1>
                <p>Thanks for signing up for MyApp. We're excited to have you!</p>
                <a href="https://myapp.com/dashboard"
                   style="background:#0078d4;color:white;padding:12px 24px;
                          text-decoration:none;border-radius:4px;display:inline-block;">
                    Go to Dashboard
                </a>
            </div>
        `,
        trackingSettings: {
            clickTracking: { enable: true },
            openTracking: { enable: true }
        }
    };

    try {
        await sgMail.send(msg);
        console.log(`Email sent to ${userEmail}`);
    } catch (error) {
        console.error('Error sending email:', error.response?.body?.errors);
    }
}

sendWelcomeEmail('test@example.com', 'Alice');
```

```bash
node send-email.js
```

#### Lab 5 — Use Templates (Dynamic Transactional Templates)

1. In SendGrid dashboard → **Email API → Dynamic Templates → Create Template**
2. Add a version with **Design Editor** or **Code Editor**
3. Use Handlebars syntax: `{{first_name}}`, `{{order_id}}`

```javascript
// Send using a template
const msg = {
    to: 'customer@example.com',
    from: 'orders@yourdomain.com',
    templateId: 'd-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',  // Your template ID
    dynamicTemplateData: {
        first_name: 'Alice',
        order_id: 'ORD-20240115-001',
        total: '$45.97',
        items: [
            { name: 'Widget A', qty: 2, price: '$19.99' },
            { name: 'Widget B', qty: 1, price: '$5.99' }
        ]
    }
};
await sgMail.send(msg);
```

---

## 9. Azure API Management

### What Is Azure API Management?

Azure API Management (APIM) is a turnkey solution to **publish, secure, transform, maintain, and monitor APIs**. It sits as a gateway in front of your backend APIs — whether they live in Azure, on-premises, or anywhere else.

**Use cases:**
- Exposing microservices as a unified API surface
- Partner/developer API portals
- Rate limiting and quota enforcement
- API versioning and lifecycle management
- Legacy API modernization

### Core Concepts

**Components:**

| Component | Description |
|-----------|-------------|
| **API Gateway** | Receives API calls, applies policies, routes to backends |
| **Admin Portal** | Azure Portal — configure APIs, policies, products |
| **Developer Portal** | Auto-generated, customizable portal for API consumers to discover and test APIs |

**Products:** A grouping of APIs presented to developers. Subscription required. Example: "Free Tier" (5 calls/min) vs "Pro Tier" (1000 calls/min).

**Policies:** XML rules applied at the API gateway to transform or control requests/responses. Applied at four scopes: Global → Product → API → Operation.

Common policies:

```xml
<policies>
  <inbound>
    <!-- Validate and extract JWT token -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration"/>
    </validate-jwt>

    <!-- Rate limit: 10 calls per 60 seconds per subscription -->
    <rate-limit calls="10" renewal-period="60" />

    <!-- Transform incoming request header -->
    <set-header name="X-Forwarded-For" exists-action="override">
      <value>@(context.Request.IpAddress)</value>
    </set-header>
  </inbound>

  <backend>
    <forward-request />
  </backend>

  <outbound>
    <!-- Remove internal header before sending response to client -->
    <set-header name="X-Internal-Server" exists-action="delete" />

    <!-- Cache response for 300 seconds -->
    <cache-store duration="300" />
  </outbound>

  <on-error>
    <return-response>
      <set-status code="500" reason="Internal Server Error" />
    </return-response>
  </on-error>
</policies>
```

**API Versioning Strategies:**

| Strategy | URL Example |
|----------|-------------|
| Path-based | `/api/v1/products` |
| Query string | `/api/products?api-version=1.0` |
| Header | `Api-Version: 1.0` |

**Tiers:**

| Tier | Use Case | Units |
|------|---------|-------|
| **Consumption** | Serverless, pay-per-call | Auto |
| **Developer** | Testing/dev (no SLA) | 1 |
| **Basic** | Small production | 1-2 |
| **Standard** | Medium production | 1-4 |
| **Premium** | Enterprise, multi-region | 1-31 |

---

### Hands-On Practice: Azure API Management

#### Lab 1 — Create an APIM Instance

```bash
RESOURCE_GROUP="rg-apim-lab"
APIM_NAME="apim-$(date +%s | tail -c 8)"
LOCATION="eastus"
PUBLISHER_EMAIL="admin@yourdomain.com"
PUBLISHER_NAME="My Organization"

az group create --name $RESOURCE_GROUP --location $LOCATION

# Create APIM instance (Consumption tier — fastest to provision)
az apim create \
  --name $APIM_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --publisher-email $PUBLISHER_EMAIL \
  --publisher-name "$PUBLISHER_NAME" \
  --sku-name Consumption

echo "APIM Gateway URL: https://${APIM_NAME}.azure-api.net"
```

> Note: Standard/Premium tiers take 30-45 minutes to provision. Consumption is instant.

#### Lab 2 — Import a Public API

We'll import the public JSONPlaceholder API as a demo backend.

```bash
# Create an API from an OpenAPI spec (using JSONPlaceholder)
az apim api import \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id "jsonplaceholder" \
  --path "posts" \
  --display-name "JSONPlaceholder Posts API" \
  --specification-format "OpenApi" \
  --specification-url "https://jsonplaceholder.typicode.com" \
  --protocols https \
  --service-url "https://jsonplaceholder.typicode.com"
```

Alternatively, through the portal:

1. Navigate to your APIM instance → **APIs → Add API**
2. Choose **HTTP** (manual) or import from **OpenAPI**, **WSDL**, or **WADL**
3. Enter the backend service URL
4. Configure base path and display name

#### Lab 3 — Apply Policies

**Rate Limiting Policy** (via portal):

1. Go to APIM → **APIs → JSONPlaceholder Posts API → All Operations**
2. Click **Inbound processing → Policy Editor**
3. Add the following inside `<inbound>`:

```xml
<rate-limit-by-key calls="5"
    renewal-period="30"
    counter-key="@(context.Subscription.Id)"
    increment-condition="@(context.Response.StatusCode == 200)" />
```

**Response Caching Policy:**

```xml
<cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
    <vary-by-header>Accept</vary-by-header>
</cache-lookup>

<!-- In outbound section: -->
<cache-store duration="60" />
```

**Mock Response Policy** (for testing without a real backend):

```xml
<mock-response status-code="200" content-type="application/json" />
```

Then add a **Representation** in the operation with sample JSON.

#### Lab 4 — Versioning

```bash
# Create a version set
az apim api versionset create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --version-set-id "posts-versions" \
  --display-name "Posts API Versions" \
  --versioning-scheme "Segment"

# Create v1 of the API (using path-based versioning)
az apim api create \
  --resource-group $RESOURCE_GROUP \
  --service-name $APIM_NAME \
  --api-id "posts-v1" \
  --display-name "Posts API v1" \
  --path "posts/v1" \
  --service-url "https://jsonplaceholder.typicode.com" \
  --api-version "v1" \
  --api-version-set-id "posts-versions"
```

#### Lab 5 — Test via Developer Portal

1. Navigate to your APIM in the portal → **Developer Portal → Publish**
2. Open the developer portal URL (shown in Overview)
3. Browse APIs → select an API → click **Try it**
4. Provide a subscription key and run a test request

#### Lab 6 — Cleanup

```bash
az group delete --name rg-apim-lab --yes --no-wait
```

---

## Next Steps & Learning Resources

### Putting It All Together — Sample Architecture

Here's how these services might connect in a real-world e-commerce application:

```
User → Azure API Management → Azure Functions (API handlers)
                                    ↓              ↓
                            Azure SQL DB    Azure Cosmos DB
                            (orders/users)  (product catalog)
                                    ↓
                            Azure Service Bus (order events)
                                    ↓
                            Azure Functions (fulfillment worker)
                                    ↓
                            Azure SendGrid (confirmation email)

All secured by: Azure Active Directory (user auth + RBAC)
All assets stored in: Azure Blob Storage (images, receipts)
All running on: Azure VMs or Azure App Service
```

### Free Learning Resources

| Resource | URL | Best For |
|----------|-----|---------|
| **Microsoft Learn** | learn.microsoft.com | Structured paths, free sandbox labs |
| **Azure Documentation** | docs.microsoft.com/azure | In-depth reference |
| **Azure Architecture Center** | docs.microsoft.com/azure/architecture | Real-world patterns |
| **AZ-900 Exam Prep** | learn.microsoft.com/certifications/azure-fundamentals | Foundation certification |
| **Azure Free Account** | azure.microsoft.com/free | $200 credits + always-free services |
| **Azure Quickstart Templates** | github.com/Azure/azure-quickstart-templates | Ready-to-deploy ARM templates |

### Recommended Certification Path

```
AZ-900 (Fundamentals)
    ↓
AZ-104 (Administrator) ←→ AZ-204 (Developer)
    ↓                           ↓
AZ-305 (Architect)      AZ-400 (DevOps Engineer)
```

### Cost Management Tips

- Always **delete resource groups** after labs — deleting the group deletes everything inside
- Use **Azure Cost Management + Budgets** to set spending alerts
- Use the **Azure Pricing Calculator** before deploying: [azure.microsoft.com/pricing/calculator](https://azure.microsoft.com/pricing/calculator)
- Tag all resources (`Environment: Lab`, `Owner: YourName`) for cost attribution
- Use **Azure Advisor** recommendations to identify unused or oversized resources

---

*Guide covers Azure services as of 2024. Service details, pricing, and features may change — always refer to the official Azure documentation for the most current information.*
