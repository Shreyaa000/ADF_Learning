# ADF-04: Integration Runtime Mastery

## What is Integration Runtime?

**Think of Integration Runtime like:** The delivery service for your data. Just like you need different delivery methods (truck, plane, ship) for different scenarios, you need different Integration Runtimes for different data movement scenarios.

**Technical Definition:** Integration Runtime (IR) is the compute infrastructure used by Azure Data Factory to provide data integration capabilities across different network environments.

### The Three Types

```
Integration Runtime
├── Azure IR (Cloud-to-Cloud)
├── Self-Hosted IR (Hybrid/On-Premises)
└── Azure-SSIS IR (SSIS Package Execution)
```

---

## AZURE INTEGRATION RUNTIME

### What is Azure IR?

**Simple Explanation:** Microsoft-managed compute infrastructure in Azure. You don't manage servers, patches, or updates. It just works.

**When to Use:**
- ✅ Copying data between cloud data stores
- ✅ Running Data Flows
- ✅ Dispatching activities to cloud compute services (Databricks, HDInsight)
- ✅ Public endpoint access is acceptable
- ✅ No on-premises data sources

**When NOT to Use:**
- ❌ Accessing on-premises data sources
- ❌ Data sources behind firewalls/private networks
- ❌ Compliance requires data to stay in specific location

### Azure IR Features

**1. Auto-Resolve**
- ADF automatically selects the optimal region
- Chooses region closest to your data sink
- Reduces latency and costs

**2. Region-Specific**
- Manually specify region
- Use when: Data residency requirements, compliance, performance optimization

**3. Managed Virtual Network**
- Private endpoints to data sources
- Network isolation
- Enhanced security

### Configuration

#### Auto-Resolve Integration Runtime (Default)

```json
{
  "name": "AutoResolveIntegrationRuntime",
  "properties": {
    "type": "Managed",
    "typeProperties": {
      "computeProperties": {
        "location": "AutoResolve"
      }
    }
  }
}
```

**How Auto-Resolve Works:**
```
Copy Activity: Azure SQL (East US) → Blob Storage (West US)
ADF Decision: Use West US IR (closer to sink)
```

#### Region-Specific Integration Runtime

```json
{
  "name": "EastUSIntegrationRuntime",
  "properties": {
    "type": "Managed",
    "typeProperties": {
      "computeProperties": {
        "location": "East US",
        "dataFlowProperties": {
          "computeType": "General",
          "coreCount": 8,
          "timeToLive": 10,
          "cleanup": false
        }
      }
    }
  }
}
```

**Region Selection Criteria:**
1. **Data Residency:** EU data must stay in EU regions
2. **Compliance:** HIPAA, GDPR requirements
3. **Performance:** Minimize cross-region data transfer
4. **Cost:** Cross-region transfer costs money

#### Managed Virtual Network Integration Runtime

```json
{
  "name": "ManagedVNetIR",
  "properties": {
    "type": "Managed",
    "typeProperties": {
      "computeProperties": {
        "location": "East US"
      }
    },
    "managedVirtualNetwork": {
      "referenceName": "default",
      "type": "ManagedVirtualNetworkReference"
    }
  }
}
```

**Setup Steps:**
1. Create Managed Virtual Network in ADF
2. Create Managed Private Endpoints to data sources
3. Approve private endpoint connections on data sources
4. Use Managed VNet IR in linked services

**Benefits:**
- No public internet exposure
- Network isolation
- Compliance with security policies
- Reduced attack surface

### Data Flow Compute Configuration

When using Data Flows, configure compute properties:

```json
{
  "dataFlowProperties": {
    "computeType": "General",        // General, MemoryOptimized, ComputeOptimized
    "coreCount": 8,                  // 8, 16, 32, 64, 128, 256
    "timeToLive": 10,                // Minutes to keep cluster warm
    "cleanup": false                 // Cleanup resources after execution
  }
}
```

**Compute Types:**

| Type | Use Case | Memory per Core |
|------|----------|-----------------|
| General | Balanced workloads | 4 GB |
| Memory Optimized | Large joins, aggregations | 8 GB |
| Compute Optimized | CPU-intensive transformations | 2 GB |

**Core Count Selection:**
- **8 cores:** Small datasets (< 1 GB), development/testing
- **16 cores:** Medium datasets (1-10 GB), most production workloads
- **32 cores:** Large datasets (10-100 GB)
- **64+ cores:** Very large datasets (100+ GB), complex transformations

**Time to Live (TTL):**
- Keeps Spark cluster warm for specified minutes
- Subsequent Data Flows reuse warm cluster (no startup time)
- Balance between cost and performance
- **Recommended:** 10-15 minutes for frequent runs

**Cost Consideration:**
```
Data Flow Cost = Core Count × Hours × Price per Core-Hour

Example:
- 16 cores, 30 minutes execution, $0.27/core-hour
- Cost = 16 × 0.5 × $0.27 = $2.16 per run

With TTL=10 and 3 runs in 10 minutes:
- Without TTL: 3 × 5 min startup + 3 × 30 min = 1.75 hours
- With TTL: 5 min startup + 30 min + 30 min + 30 min = 1.58 hours
- Savings: ~10% time, ~10% cost
```

### Real-World Example 1: Multi-Region Architecture

**Scenario:** Global company with data in US, EU, and APAC.

**Requirements:**
- EU data must stay in EU (GDPR)
- Minimize cross-region transfer costs
- Optimize performance

**Solution:**

**Integration Runtimes:**
```
1. EastUSIntegrationRuntime (location: East US)
2. WestEuropeIntegrationRuntime (location: West Europe)
3. SoutheastAsiaIntegrationRuntime (location: Southeast Asia)
```

**Linked Services:**
```
US Data Sources → Use EastUSIntegrationRuntime
EU Data Sources → Use WestEuropeIntegrationRuntime
APAC Data Sources → Use SoutheastAsiaIntegrationRuntime
```

**Pipeline Design:**
```
Pipeline: GlobalDataIngestion

ForEach: Loop through regions
  ├─ US: Execute Pipeline (US-specific pipeline)
  │   Uses: EastUSIntegrationRuntime
  ├─ EU: Execute Pipeline (EU-specific pipeline)
  │   Uses: WestEuropeIntegrationRuntime
  └─ APAC: Execute Pipeline (APAC-specific pipeline)
      Uses: SoutheastAsiaIntegrationRuntime
```

**Benefits:**
- Data residency compliance
- Reduced cross-region costs
- Better performance (local processing)

### Real-World Example 2: Managed VNet for Security

**Scenario:** Financial institution with strict security requirements.

**Requirements:**
- No public internet access to data sources
- Network isolation
- Private connectivity only

**Solution:**

**Architecture:**
```
ADF (Managed VNet IR)
    ↓ (Private Endpoint)
Azure SQL Database (Private Endpoint enabled)
    ↓ (Private Endpoint)
Azure Storage (Private Endpoint enabled)
```

**Setup:**
1. Enable Managed Virtual Network in ADF
2. Create Managed Private Endpoints:
   - To Azure SQL Database
   - To Azure Storage
   - To Azure Key Vault
3. Approve private endpoint connections
4. Update linked services to use Managed VNet IR

**Result:**
- All data movement happens over Microsoft backbone network
- No public internet exposure
- Compliant with security policies

---

## SELF-HOSTED INTEGRATION RUNTIME

### What is Self-Hosted IR?

**Simple Explanation:** Software you install on your own Windows machine (on-premises or VM). It acts as a secure bridge between Azure and your private network.

**Think of it like:** A VPN client that allows ADF to securely access your on-premises data.

### When to Use Self-Hosted IR

**Use Cases:**
- ✅ Accessing on-premises data sources (SQL Server, Oracle, file shares)
- ✅ Data sources behind corporate firewall
- ✅ Private network/VNet data sources
- ✅ Compliance requires data processing in specific location
- ✅ Data sources requiring specific drivers (Oracle, SAP)

**Real-World Scenarios:**
1. **Hybrid Cloud:** On-premises SQL Server → Azure SQL Database
2. **Legacy Systems:** Mainframe files → Azure Data Lake
3. **Corporate Network:** File share behind firewall → Blob Storage
4. **Compliance:** Healthcare data that can't leave premises

### Architecture

```
On-Premises Network
├── Self-Hosted IR (Windows Server)
│   ├── Outbound HTTPS 443 to Azure
│   ├── Access to local data sources
│   └── No inbound ports required
├── SQL Server
├── File Share
└── Oracle Database
    ↓ (Encrypted HTTPS)
Azure Data Factory
```

**Key Points:**
- **Outbound only:** Only outbound HTTPS (443) required
- **No inbound:** No inbound ports needed (more secure)
- **Encrypted:** All communication encrypted with TLS
- **Firewall-friendly:** Works through corporate firewalls

### Installation and Setup

#### Step 1: Create Self-Hosted IR in ADF

**Via Azure Portal:**
1. Go to ADF → Manage → Integration Runtimes
2. Click "+ New"
3. Select "Self-Hosted"
4. Enter name: "OnPremisesIR"
5. Click "Create"
6. Copy one of the authentication keys

**Via JSON:**
```json
{
  "name": "OnPremisesIR",
  "properties": {
    "type": "SelfHosted",
    "description": "Self-Hosted IR for on-premises SQL Server"
  }
}
```

#### Step 2: Install IR Software

**System Requirements:**
- Windows Server 2012 R2 or later (Windows 10/11 also supported)
- .NET Framework 4.7.2 or later
- Minimum 4 cores, 8 GB RAM (recommended: 8 cores, 16 GB RAM)
- Outbound internet access (HTTPS 443)

**Installation Steps:**
1. Download IR installer from Azure Portal
2. Run installer on Windows machine
3. Enter authentication key from Step 1
4. Complete setup wizard

**PowerShell Installation:**
```powershell
# Download installer
Invoke-WebRequest -Uri "https://download.microsoft.com/download/..." -OutFile "IntegrationRuntime.msi"

# Install silently
msiexec /i IntegrationRuntime.msi /quiet /norestart

# Register with key
cd "C:\Program Files\Microsoft Integration Runtime\5.0\Shared"
.\dmgcmd.exe -Key "YOUR_AUTHENTICATION_KEY"
```

#### Step 3: Configure Firewall

**Outbound Rules Required:**

| Protocol | Port | Destination | Purpose |
|----------|------|-------------|---------|
| HTTPS | 443 | *.servicebus.windows.net | Service Bus communication |
| HTTPS | 443 | *.frontend.clouddatahub.net | ADF service |
| HTTPS | 443 | *.core.windows.net | Azure Storage (staging) |
| HTTPS | 443 | download.microsoft.com | Updates |

**No Inbound Rules Needed!**

**Windows Firewall Configuration:**
```powershell
# Allow outbound HTTPS
New-NetFirewallRule -DisplayName "ADF Self-Hosted IR - HTTPS Out" `
  -Direction Outbound -Protocol TCP -LocalPort Any -RemotePort 443 `
  -Action Allow
```

#### Step 4: Test Connectivity

**Test from ADF Portal:**
1. Go to Integration Runtimes
2. Select your Self-Hosted IR
3. Click "Test connection"
4. Status should show "Connected"

**Test from IR Machine:**
```powershell
# Check IR service status
Get-Service -Name "DIAHostService"

# Test connectivity
cd "C:\Program Files\Microsoft Integration Runtime\5.0\Shared"
.\dmgcmd.exe -TestConnection
```

### High Availability Setup

**Scenario:** Production workloads require redundancy.

**Solution: Multi-Node Self-Hosted IR**

**Architecture:**
```
Azure Data Factory
    ↓
Self-Hosted IR (Logical)
    ├── Node 1 (Primary) - Server 1
    ├── Node 2 (Secondary) - Server 2
    ├── Node 3 (Secondary) - Server 3
    └── Node 4 (Secondary) - Server 4
```

**Setup Steps:**

**1. Install Primary Node:**
```
- Create Self-Hosted IR in ADF
- Install on Server 1 with authentication key
- This becomes the primary node
```

**2. Add Secondary Nodes:**
```
For each additional server:
1. Install IR software
2. Use same authentication key OR
3. Go to primary node, get registration key:
   dmgcmd.exe -Key <auth_key> -RegisterNewNode
4. Use registration key on secondary node
```

**Configuration:**
```json
{
  "name": "OnPremisesIR_HA",
  "properties": {
    "type": "SelfHosted",
    "description": "High Availability Self-Hosted IR",
    "typeProperties": {
      "linkedInfo": {
        "resourceId": "/subscriptions/.../integrationRuntimes/OnPremisesIR_HA",
        "authorizationType": "Key"
      }
    }
  }
}
```

**Benefits:**
- **Load Balancing:** ADF distributes work across nodes
- **Failover:** If one node fails, others continue
- **Scalability:** Add nodes for more capacity
- **Maintenance:** Update nodes one at a time (zero downtime)

**Best Practices:**
- **Minimum 2 nodes** for production
- **4 nodes** for high-volume workloads
- **Same configuration** on all nodes
- **Monitor all nodes** regularly

### Performance Tuning

#### 1. Concurrent Jobs

**Default:** 4 concurrent jobs per node

**Increase for high-volume:**
```json
{
  "typeProperties": {
    "maxConcurrentJobs": 8  // Increase based on server capacity
  }
}
```

**Calculation:**
```
Recommended Concurrent Jobs = (CPU Cores / 2)

Example:
- 8-core server: maxConcurrentJobs = 4
- 16-core server: maxConcurrentJobs = 8
```

#### 2. Data Integration Units (DIU)

For cloud-to-cloud copy via Self-Hosted IR:
```json
{
  "dataIntegrationUnits": 4  // 2, 4, 8, 16, 32
}
```

#### 3. Parallel Copies

```json
{
  "parallelCopies": 8  // Number of parallel threads
}
```

#### 4. Staging

For large datasets, enable staging:
```json
{
  "enableStaging": true,
  "stagingSettings": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLS",
      "type": "LinkedServiceReference"
    },
    "path": "staging"
  }
}
```

### Real-World Example 1: On-Premises SQL Server to Azure

**Scenario:** Migrate 100 tables from on-premises SQL Server to Azure SQL Database.

**Architecture:**
```
On-Premises Data Center
├── SQL Server 2019
└── Self-Hosted IR (Windows Server 2022, 8 cores, 16 GB RAM)
    ↓ (HTTPS 443)
Azure
├── Azure Data Factory
└── Azure SQL Database
```

**Setup:**

**1. Self-Hosted IR Configuration:**
```json
{
  "name": "OnPremSqlServerIR",
  "properties": {
    "type": "SelfHosted",
    "description": "IR for on-premises SQL Server",
    "typeProperties": {
      "maxConcurrentJobs": 4
    }
  }
}
```

**2. Linked Services:**

**Source (On-Premises SQL Server):**
```json
{
  "name": "OnPremSqlServerLS",
  "properties": {
    "type": "SqlServer",
    "typeProperties": {
      "connectionString": "Server=SQLSERVER01;Database=ProductionDB;Integrated Security=True;",
      "userName": "domain\\serviceaccount",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "OnPremSqlPassword"
      }
    },
    "connectVia": {
      "referenceName": "OnPremSqlServerIR",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

**Destination (Azure SQL Database):**
```json
{
  "name": "AzureSqlDatabaseLS",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=TargetDB;",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

**3. Pipeline:**
```
Pipeline: MigrateOnPremTables

1. Lookup: Get table list from control table
   Query: SELECT SchemaName, TableName FROM TableConfig WHERE IsActive = 1

2. ForEach: Loop through tables (batchCount = 4)
   Activities:
     ├─ Copy Activity: Copy table
     │   Source: OnPremSqlServerLS
     │   Sink: AzureSqlDatabaseLS
     │   Settings:
     │     - parallelCopies: 4
     │     - enableStaging: true
     │     - DIU: 4
     │
     └─ Stored Procedure: Log completion
```

**Performance Results:**
```
Without optimization:
- 100 tables × 5 minutes = 500 minutes (8.3 hours)

With optimization (4 parallel):
- 100 tables / 4 parallel × 5 minutes = 125 minutes (2.1 hours)

With staging enabled:
- 100 tables / 4 parallel × 3 minutes = 75 minutes (1.25 hours)
```

### Real-World Example 2: File Share to Data Lake

**Scenario:** Daily file ingestion from corporate file share.

**Requirements:**
- 500+ CSV files daily
- File share behind corporate firewall
- Files range from 10 MB to 2 GB
- Process only new files

**Architecture:**
```
Corporate Network
├── File Share: \\fileserver\data\incoming
└── Self-Hosted IR (Windows Server)
    ↓
Azure Data Lake Storage Gen2
```

**Pipeline:**
```
Pipeline: IngestFileShareFiles

1. Get Metadata: Get list of files
   Dataset: File Share (via Self-Hosted IR)
   Field List: ["childItems"]

2. Filter: Get only today's files
   Condition: @greater(
     item().lastModified,
     addDays(utcnow(), -1)
   )

3. ForEach: Loop through files (batchCount = 10)
   Activities:
     ├─ Copy Activity: Copy file
     │   Source: File Share (Self-Hosted IR)
     │   Sink: ADLS Gen2 (Azure IR)
     │   Settings:
     │     - Binary copy (preserve format)
     │     - Compression: gzip
     │
     ├─ Copy Activity: Archive file
     │   Source: \\fileserver\data\incoming\@{item().name}
     │   Sink: \\fileserver\data\archive\@{formatDateTime(utcnow(),'yyyy-MM-dd')}\@{item().name}
     │
     └─ Delete Activity: Delete from incoming (optional)
```

### Troubleshooting Self-Hosted IR

#### Issue 1: IR Offline

**Symptoms:**
- IR shows "Offline" in ADF portal
- Pipelines fail with "Integration Runtime is offline"

**Troubleshooting Steps:**
```powershell
# 1. Check IR service status
Get-Service -Name "DIAHostService"

# If stopped, start it
Start-Service -Name "DIAHostService"

# 2. Check event logs
Get-EventLog -LogName Application -Source "DIAHostService" -Newest 50

# 3. Check connectivity
cd "C:\Program Files\Microsoft Integration Runtime\5.0\Shared"
.\dmgcmd.exe -TestConnection

# 4. Check firewall
Test-NetConnection -ComputerName "*.servicebus.windows.net" -Port 443

# 5. Review IR logs
Get-Content "C:\ProgramData\Microsoft\Integration Runtime\*\Logs\*" -Tail 100
```

**Common Causes:**
- Service stopped
- Network connectivity issues
- Firewall blocking outbound HTTPS
- Certificate issues
- IR machine rebooted (service not set to auto-start)

**Solutions:**
```powershell
# Set service to auto-start
Set-Service -Name "DIAHostService" -StartupType Automatic

# Restart service
Restart-Service -Name "DIAHostService"

# Re-register if needed
cd "C:\Program Files\Microsoft Integration Runtime\5.0\Shared"
.\dmgcmd.exe -Key "YOUR_NEW_KEY"
```

#### Issue 2: Slow Performance

**Symptoms:**
- Copy activities taking longer than expected
- High CPU/memory on IR machine

**Troubleshooting:**
```powershell
# Check resource usage
Get-Counter '\Processor(_Total)\% Processor Time'
Get-Counter '\Memory\Available MBytes'

# Check concurrent jobs
# Review ADF monitoring for concurrent activities
```

**Solutions:**
1. **Add more nodes** (scale out)
2. **Upgrade IR machine** (scale up)
3. **Reduce concurrent jobs** if resource-constrained
4. **Enable staging** for large copies
5. **Optimize queries** (filter at source)

#### Issue 3: Connection to Data Source Fails

**Symptoms:**
- "Cannot connect to database"
- "Network path not found"

**Troubleshooting:**
```powershell
# Test from IR machine
# For SQL Server:
Test-NetConnection -ComputerName "SQLSERVER01" -Port 1433

# For file share:
Test-Path "\\fileserver\share"

# Check DNS resolution
Resolve-DnsName "SQLSERVER01"

# Test SQL connection
sqlcmd -S SQLSERVER01 -d ProductionDB -U username -P password
```

**Common Causes:**
- Firewall on data source blocking IR machine
- DNS resolution issues
- Incorrect credentials
- SQL Server not accepting remote connections
- File share permissions

**Solutions:**
1. **Add IR machine to firewall whitelist** on data source
2. **Verify credentials** in Key Vault
3. **Enable TCP/IP** in SQL Server Configuration Manager
4. **Grant permissions** on file share to IR service account
5. **Use IP address** instead of hostname if DNS issues

#### Issue 4: Certificate Errors

**Symptoms:**
- "The remote certificate is invalid"
- SSL/TLS errors

**Solutions:**
```powershell
# Import certificate to IR machine
Import-Certificate -FilePath "C:\cert.cer" -CertStoreLocation Cert:\LocalMachine\Root

# Or disable certificate validation (not recommended for production)
# In linked service, set: "TrustServerCertificate=True"
```

---

## AZURE-SSIS INTEGRATION RUNTIME

### What is Azure-SSIS IR?

**Simple Explanation:** A managed cluster of Azure VMs that runs your SSIS packages in the cloud.

**Think of it like:** Renting a fleet of servers pre-configured with SQL Server Integration Services, without managing the infrastructure.

### When to Use Azure-SSIS IR

**Use Cases:**
- ✅ Migrating existing SSIS packages to cloud (lift-and-shift)
- ✅ Running SSIS packages without on-premises infrastructure
- ✅ Need SSIS-specific features not available in ADF
- ✅ Large investment in SSIS packages (200+ packages)

**When NOT to Use:**
- ❌ Building new cloud-native solutions (use ADF activities/Data Flows)
- ❌ Simple data movement (use Copy Activity)
- ❌ Cost-sensitive (Azure-SSIS IR is more expensive)

### Architecture

```
Azure Data Factory
    ↓
Azure-SSIS Integration Runtime
├── VM Node 1 (SSIS Runtime)
├── VM Node 2 (SSIS Runtime)
└── VM Node 3 (SSIS Runtime)
    ↓
SSIS Packages (stored in SSISDB or file system)
```

### Configuration

```json
{
  "name": "AzureSsisIR",
  "properties": {
    "type": "Managed",
    "typeProperties": {
      "computeProperties": {
        "location": "East US",
        "nodeSize": "Standard_D2_v3",
        "numberOfNodes": 2,
        "maxParallelExecutionsPerNode": 2
      },
      "ssisProperties": {
        "catalogInfo": {
          "catalogServerEndpoint": "myserver.database.windows.net",
          "catalogAdminUserName": "ssisadmin",
          "catalogAdminPassword": {
            "type": "SecureString",
            "value": "**********"
          },
          "catalogPricingTier": "S1"
        },
        "edition": "Standard",
        "licenseType": "LicenseIncluded"
      }
    }
  }
}
```

**Key Configuration Options:**

**Node Size:**
- Standard_D2_v3: 2 cores, 8 GB RAM (dev/test)
- Standard_D4_v3: 4 cores, 16 GB RAM (small production)
- Standard_D8_v3: 8 cores, 32 GB RAM (medium production)
- Standard_D16_v3: 16 cores, 64 GB RAM (large production)

**Number of Nodes:**
- 1 node: Development/testing
- 2-4 nodes: Production (high availability)
- 5+ nodes: High-volume workloads

**Max Parallel Executions Per Node:**
- Depends on node size and package complexity
- Recommended: 1-2 per node for complex packages

**License Type:**
- **LicenseIncluded:** Pay for SQL Server license in hourly rate
- **BasePrice:** Use existing SQL Server license (cost savings)

### Package Deployment

**Option 1: SSISDB (Recommended)**
```
Packages stored in SSISDB (Azure SQL Database)
- Centralized management
- Built-in logging
- Environment variables
- Versioning
```

**Option 2: File System**
```
Packages stored in Azure Files or Blob Storage
- Simple deployment
- No SSISDB needed
- Manual logging
```

**Option 3: Package Store**
```
Packages stored in SQL Server or file system
- Hybrid approach
- Legacy compatibility
```

### Real-World Example: SSIS Migration

**Scenario:** Company has 300 SSIS packages running on-premises. Want to migrate to cloud.

**Current State:**
```
On-Premises
├── SQL Server 2019 (SSISDB)
├── 300 SSIS packages
├── SQL Server Agent jobs (scheduling)
└── 5 years of development investment
```

**Migration Strategy:**

**Phase 1: Assessment (2 weeks)**
```
1. Inventory all packages
2. Identify dependencies
3. Assess complexity
4. Identify candidates for rewrite vs lift-and-shift
```

**Phase 2: Setup Azure-SSIS IR (1 week)**
```
1. Create Azure SQL Database for SSISDB
2. Provision Azure-SSIS IR
3. Configure VNet (if accessing on-premises)
4. Setup Azure Files for package storage
```

**Phase 3: Migrate Packages (4-8 weeks)**
```
1. Deploy packages to Azure SSISDB
2. Update connection strings
3. Test packages in Azure
4. Update error handling
```

**Phase 4: Scheduling (2 weeks)**
```
Replace SQL Server Agent with ADF triggers:

ADF Pipeline:
1. Execute SSIS Package Activity
   Package: /SSISDB/Production/DailyLoad/LoadOrders.dtsx
   Parameters:
     - ProcessDate: @pipeline().parameters.ProcessDate
     - Environment: Production

2. Schedule Trigger: Daily at 2 AM
```

**Migration Results:**
```
Before (On-Premises):
- Infrastructure cost: $5,000/month (servers, licenses)
- Maintenance: 20 hours/month
- Scalability: Limited

After (Azure-SSIS IR):
- Infrastructure cost: $3,500/month (pay-per-use)
- Maintenance: 5 hours/month (managed service)
- Scalability: Scale up/down as needed
- High availability: Built-in
```

### Cost Optimization

**Problem:** Azure-SSIS IR is expensive when running 24/7.

**Solution: Start/Stop on Schedule**

**Pipeline: StartSSISIR**
```json
{
  "name": "StartSSISIR",
  "activities": [
    {
      "name": "StartIR",
      "type": "WebActivity",
      "typeProperties": {
        "url": "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.DataFactory/factories/{factoryName}/integrationRuntimes/{irName}/start?api-version=2018-06-01",
        "method": "POST",
        "authentication": {
          "type": "MSI",
          "resource": "https://management.azure.com/"
        }
      }
    }
  ]
}
```

**Pipeline: StopSSISIR**
```json
{
  "name": "StopSSISIR",
  "activities": [
    {
      "name": "StopIR",
      "type": "WebActivity",
      "typeProperties": {
        "url": "https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroup}/providers/Microsoft.DataFactory/factories/{factoryName}/integrationRuntimes/{irName}/stop?api-version=2018-06-01",
        "method": "POST",
        "authentication": {
          "type": "MSI",
          "resource": "https://management.azure.com/"
        }
      }
    }
  ]
}
```

**Schedule:**
```
6:00 AM: Trigger StartSSISIR
6:15 AM: Run SSIS packages
8:00 PM: Trigger StopSSISIR
```

**Cost Savings:**
```
24/7 Operation:
- 2 nodes × Standard_D4_v3 × 730 hours/month × $0.40/hour = $584/month

14 hours/day (6 AM - 8 PM):
- 2 nodes × Standard_D4_v3 × 420 hours/month × $0.40/hour = $336/month

Savings: $248/month (42%)
```

---

## INTEGRATION RUNTIME COMPARISON

| Feature | Azure IR | Self-Hosted IR | Azure-SSIS IR |
|---------|----------|----------------|---------------|
| **Management** | Fully managed | You manage | Fully managed |
| **Location** | Azure regions | Your network | Azure regions |
| **Use Case** | Cloud-to-cloud | Hybrid/On-prem | SSIS packages |
| **Setup Time** | Instant | 30-60 minutes | 15-30 minutes |
| **Cost** | Low | Medium (infrastructure) | High |
| **Scalability** | Auto-scale | Manual (add nodes) | Manual (add nodes) |
| **Maintenance** | None | Patches, updates | None |
| **HA** | Built-in | Multi-node setup | Multi-node setup |

---

## BEST PRACTICES

### Azure IR:
1. **Use Auto-Resolve** for most scenarios
2. **Use region-specific** for compliance/performance
3. **Enable Managed VNet** for security
4. **Configure TTL** for Data Flows (10-15 minutes)
5. **Right-size compute** based on workload

### Self-Hosted IR:
1. **Use dedicated machine** (not shared with other apps)
2. **Minimum 2 nodes** for production (HA)
3. **Monitor resource usage** (CPU, memory, network)
4. **Keep IR updated** (auto-update or manual)
5. **Use Windows authentication** when possible
6. **Store credentials in Key Vault**
7. **Document firewall rules**
8. **Regular health checks**

### Azure-SSIS IR:
1. **Right-size nodes** based on package complexity
2. **Use SSISDB** for centralized management
3. **Implement start/stop** for cost savings
4. **Monitor package execution** logs
5. **Consider rewriting** simple packages as ADF pipelines

---

## TROUBLESHOOTING GUIDE

### General IR Issues

**Issue: IR Not Showing in Dropdown**
- **Cause:** IR in different region than linked service
- **Solution:** Use Auto-Resolve IR or create region-specific IR

**Issue: High Costs**
- **Cause:** IR running 24/7, oversized compute
- **Solution:** Implement start/stop, right-size compute, use TTL

**Issue: Performance Degradation**
- **Cause:** Resource contention, network latency
- **Solution:** Scale up/out, optimize queries, enable staging

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: Explain the three types of Integration Runtime and when to use each.**

**A:** "There are three types:

**Azure IR** - Fully managed by Microsoft, used for cloud-to-cloud data movement. I use this for copying between Azure services like Blob Storage to SQL Database. It's the default and handles most scenarios. Benefits: no management overhead, auto-scaling, pay-per-use.

**Self-Hosted IR** - Software you install on your own machine, used for hybrid scenarios. I use this when accessing on-premises data sources or data behind firewalls. For example, in my last project, we had SQL Server in our data center. We installed Self-Hosted IR on a Windows Server, and it securely connected to Azure over HTTPS 443. We set up 2 nodes for high availability.

**Azure-SSIS IR** - Managed cluster for running SSIS packages. I use this for lift-and-shift SSIS migrations. In one project, we had 200+ SSIS packages. Instead of rewriting them all, we used Azure-SSIS IR to run them as-is in Azure, saving 6 months of development time.

The key is choosing based on where your data is and what you're trying to do."

**Q2: How do you set up high availability for Self-Hosted IR?**

**A:** "High availability requires multiple nodes:

1. **Install primary node:** Create Self-Hosted IR in ADF, install software on first server with authentication key
2. **Add secondary nodes:** Install IR software on additional servers using the same authentication key or a registration key from the primary node
3. **Verify:** Check ADF portal shows all nodes as 'Running'

**Benefits:**
- Load balancing: ADF distributes work across nodes
- Failover: If one node fails, others continue
- Maintenance: Update nodes one at a time with zero downtime

**Best practices:**
- Minimum 2 nodes for production
- Same configuration on all nodes
- Monitor all nodes regularly
- Use 4 nodes for high-volume workloads

In my experience, we had 4 nodes processing 500+ files daily. When we needed to patch one node, we took it offline, updated it, and brought it back - no downtime for the business."

**Q3: How do you optimize costs for Azure-SSIS IR?**

**A:** "Azure-SSIS IR is expensive if running 24/7. My optimization strategies:

**1. Start/Stop on Schedule:**
```
- Start IR at 6 AM (when packages run)
- Stop IR at 8 PM (after packages complete)
- Savings: ~40-50% cost reduction
```

**2. Right-Size Nodes:**
```
- Don't over-provision
- Start with Standard_D4_v3, scale up if needed
- Monitor CPU/memory usage
```

**3. Use Existing SQL License:**
```
- License Type: BasePrice (vs LicenseIncluded)
- Savings: ~30% on compute costs
```

**4. Consolidate Package Runs:**
```
- Run multiple packages in parallel
- Reduce total IR runtime
```

**5. Consider Rewriting:**
```
- Simple packages → ADF Copy Activity
- Complex packages → Keep in SSIS
```

In my last project, we reduced Azure-SSIS IR costs from $8,000/month to $4,500/month by implementing start/stop and rewriting 50 simple packages as ADF pipelines."

**Q4: What firewall rules are needed for Self-Hosted IR?**

**A:** "Self-Hosted IR only requires **outbound** HTTPS (443) - no inbound ports needed, which is great for security.

**Required outbound destinations:**
- *.servicebus.windows.net (Service Bus communication)
- *.frontend.clouddatahub.net (ADF service)
- *.core.windows.net (Azure Storage for staging)
- download.microsoft.com (Updates)

**No inbound rules required** because IR initiates all connections to Azure. This is firewall-friendly and more secure.

**Common issues:**
- Corporate proxy blocking HTTPS
- SSL inspection breaking encrypted communication
- Firewall rules too restrictive

**Solutions:**
- Whitelist ADF domains
- Configure proxy settings in IR
- Disable SSL inspection for ADF traffic

In my experience, most corporate firewalls allow outbound HTTPS, so Self-Hosted IR works without special firewall changes. The key is testing connectivity during setup."

**Q5: How do you troubleshoot Self-Hosted IR offline issues?**

**A:** "I follow a systematic approach:

**1. Check Service Status:**
```powershell
Get-Service -Name 'DIAHostService'
# If stopped: Start-Service -Name 'DIAHostService'
```

**2. Check Connectivity:**
```powershell
cd 'C:\Program Files\Microsoft Integration Runtime\5.0\Shared'
.\dmgcmd.exe -TestConnection
```

**3. Review Logs:**
```
Location: C:\ProgramData\Microsoft\Integration Runtime\*\Logs\
Look for: Connection errors, authentication failures
```

**4. Check Network:**
```powershell
Test-NetConnection -ComputerName '*.servicebus.windows.net' -Port 443
```

**5. Re-register if needed:**
```powershell
.\dmgcmd.exe -Key 'NEW_AUTHENTICATION_KEY'
```

**Common causes:**
- Service stopped (set to auto-start)
- Network connectivity issues
- Firewall blocking outbound HTTPS
- Certificate expiration
- Machine rebooted

In my experience, 80% of offline issues are service-related. I always set the service to auto-start and monitor it with Azure Monitor alerts."

---

## Summary

**Key Takeaways:**

1. **Azure IR:** Cloud-to-cloud, fully managed, use for most scenarios
2. **Self-Hosted IR:** Hybrid/on-premises, requires setup, use for private networks
3. **Azure-SSIS IR:** SSIS packages, expensive, use for lift-and-shift
4. **High Availability:** Multi-node setup for production
5. **Cost Optimization:** Start/stop, right-sizing, monitoring
6. **Security:** Managed Identity, Key Vault, Private Endpoints

**Next Steps:**
- Practice setting up each IR type
- Implement HA for Self-Hosted IR
- Learn triggers and scheduling
- Master monitoring and troubleshooting

Remember: Choose the right IR for your scenario. Don't use Self-Hosted IR if Azure IR works. Don't use Azure-SSIS IR if ADF activities can do the job!

