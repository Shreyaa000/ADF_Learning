# ADF-01: Introduction and Core Concepts

## What is Azure Data Factory?

### The Simple Explanation
Imagine you're a logistics manager who needs to move packages from various warehouses to different stores. You need to:
- Pick up packages from multiple locations
- Transform or repackage some items
- Deliver them to the right destinations
- Do this on a schedule or when triggered by events
- Track everything that happens

**Azure Data Factory (ADF) is exactly that, but for data.** It's Microsoft's cloud-based data integration service that allows you to create, schedule, and orchestrate data workflows at scale.

### The Technical Definition
Azure Data Factory is a fully managed, serverless data integration service that enables you to:
- **Ingest** data from 90+ data sources
- **Transform** data using various compute services
- **Orchestrate** complex data workflows
- **Monitor** all data movement and transformations
- **Schedule** automated data pipelines

### Why Azure Data Factory Exists

**Business Problem:**
Companies have data scattered across:
- On-premises databases (SQL Server, Oracle, SAP)
- Cloud storage (Azure Blob, AWS S3, Google Cloud)
- SaaS applications (Salesforce, Dynamics 365)
- APIs and web services
- File shares and FTP servers

**The Challenge:**
- Manual data movement is time-consuming and error-prone
- Different data formats need standardization
- Data needs to be available for analytics and reporting
- Compliance requires audit trails
- Business needs real-time or near-real-time data

**ADF Solution:**
- Code-free visual interface for building pipelines
- Pre-built connectors for 90+ data sources
- Scalable cloud infrastructure
- Built-in monitoring and alerting
- Pay-per-use pricing model

## Complete Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE DATA FACTORY                            │
│                                                                   │
│  ┌────────────────┐      ┌────────────────┐                    │
│  │   AUTHOR       │      │    MONITOR     │                    │
│  │   (Design)     │      │   (Observe)    │                    │
│  └────────────────┘      └────────────────┘                    │
│           │                       │                              │
│           ▼                       ▼                              │
│  ┌─────────────────────────────────────────┐                   │
│  │         ORCHESTRATION ENGINE             │                   │
│  │  (Pipelines, Activities, Triggers)       │                   │
│  └─────────────────────────────────────────┘                   │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────┐                   │
│  │      INTEGRATION RUNTIME                 │                   │
│  │  (Execution Environment)                 │                   │
│  └─────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
         │                  │                  │
         ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   SOURCE     │   │  TRANSFORM   │   │ DESTINATION  │
│   SYSTEMS    │   │   COMPUTE    │   │   SYSTEMS    │
└──────────────┘   └──────────────┘   └──────────────┘
```

### Component Interaction Flow

```
Trigger → Pipeline → Activity → Integration Runtime → Data Source/Sink
   ↓          ↓          ↓              ↓                    ↓
Schedule   Workflow   Task         Execution          Linked Service
           Control    Unit         Environment         + Dataset
```

## Core Components Explained in Detail

### 1. Integration Runtime (IR)

**Think of it like:** The delivery truck that actually moves your packages. You need different trucks for different routes.

#### Types of Integration Runtime:

**A. Azure Integration Runtime**
- **What it is:** Microsoft-managed compute infrastructure in Azure
- **When to use:** 
  - Copying data between cloud data stores
  - Running Data Flows
  - Dispatching activities to cloud compute services
- **Key Features:**
  - Fully managed (no maintenance required)
  - Auto-scaling
  - Available in multiple Azure regions
  - Supports public endpoints

**Configuration Example:**
```json
{
  "name": "AutoResolveIntegrationRuntime",
  "properties": {
    "type": "Managed",
    "typeProperties": {
      "computeProperties": {
        "location": "AutoResolve",
        "dataFlowProperties": {
          "computeType": "General",
          "coreCount": 8,
          "timeToLive": 10
        }
      }
    }
  }
}
```

**B. Self-Hosted Integration Runtime**
- **What it is:** Software you install on your own machine/VM
- **When to use:**
  - Accessing on-premises data sources
  - Data sources behind firewalls/private networks
  - Data sources requiring specific drivers
  - Compliance requires data to stay in specific locations

**Real-World Scenario:**
"Your company has a SQL Server database in your corporate data center. Azure can't directly access it because of firewall rules. You install Self-Hosted IR on a server in your data center, and it acts as a secure bridge."

**Installation Steps:**
1. Download IR software from Azure Portal
2. Install on Windows Server (on-premises or VM)
3. Register with registration key from ADF
4. Configure firewall rules (outbound HTTPS 443)
5. Test connectivity

**C. Azure-SSIS Integration Runtime**
- **What it is:** Managed cluster of Azure VMs for running SSIS packages
- **When to use:**
  - Migrating existing SSIS packages to cloud
  - Running SSIS packages without maintaining infrastructure
  - Lift-and-shift SSIS workloads

**Interview Tip:** "In my previous project, we had 200+ SSIS packages. Instead of rewriting them in ADF, we used Azure-SSIS IR to run them as-is, saving 6 months of development time."

### 2. Linked Services

**Think of it like:** Connection strings or contact information for your data sources.

**What it is:** A connection definition that contains the information needed for ADF to connect to external resources.

**Components of a Linked Service:**
- **Connection string/endpoint**
- **Authentication method** (Key, MSI, Service Principal, SQL Auth)
- **Additional properties** (database name, container name, etc.)

**Example: Azure Blob Storage Linked Service**
```json
{
  "name": "AzureBlobStorageLinkedService",
  "properties": {
    "type": "AzureBlobStorage",
    "typeProperties": {
      "connectionString": "DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=<key>",
      "encryptedCredential": null
    },
    "connectVia": {
      "referenceName": "AutoResolveIntegrationRuntime",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

**Example: SQL Database Linked Service with Managed Identity**
```json
{
  "name": "AzureSqlDatabaseLinkedService",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

**Common Linked Service Types:**
- Azure Blob Storage / ADLS Gen2
- Azure SQL Database / SQL Managed Instance
- Azure Synapse Analytics
- REST API / HTTP
- Snowflake
- Amazon S3
- Databricks
- Azure Key Vault

**Pro Tip:** Always use Azure Key Vault to store connection strings and passwords. Never hardcode credentials!

### 3. Datasets

**Think of it like:** A specific folder or table you're working with. The Linked Service is the warehouse; the Dataset is the specific shelf.

**What it is:** A named view of data that points to or references the data you want to use in activities.

**Dataset Components:**
- **Linked Service reference** (where the data is)
- **Structure/schema** (optional, what the data looks like)
- **Type properties** (file path, table name, query, etc.)
- **Parameters** (for dynamic datasets)

**Example: CSV File Dataset**
```json
{
  "name": "CsvDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLinkedService",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "folderPath": {
        "type": "string"
      },
      "fileName": {
        "type": "string"
      }
    },
    "type": "DelimitedText",
    "typeProperties": {
      "location": {
        "type": "AzureBlobStorageLocation",
        "fileName": {
          "value": "@dataset().fileName",
          "type": "Expression"
        },
        "folderPath": {
          "value": "@dataset().folderPath",
          "type": "Expression"
        },
        "container": "raw-data"
      },
      "columnDelimiter": ",",
      "escapeChar": "\\",
      "firstRowAsHeader": true,
      "quoteChar": "\""
    }
  }
}
```

**Example: SQL Table Dataset**
```json
{
  "name": "SqlTableDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureSqlDatabaseLinkedService",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "schemaName": {
        "type": "string"
      },
      "tableName": {
        "type": "string"
      }
    },
    "type": "AzureSqlTable",
    "typeProperties": {
      "schema": {
        "value": "@dataset().schemaName",
        "type": "Expression"
      },
      "table": {
        "value": "@dataset().tableName",
        "type": "Expression"
      }
    }
  }
}
```

**Parameterized Dataset Benefits:**
- Single dataset definition for multiple files/tables
- Reduces maintenance overhead
- Enables metadata-driven pipelines
- Improves reusability

### 4. Activities

**Think of it like:** Individual tasks in your workflow - copy this file, run that query, send an email.

**What it is:** A processing step in a pipeline. Activities represent actions to be performed.

**Activity Categories:**

**A. Data Movement Activities**
- **Copy Activity:** Move data from source to destination
- Most commonly used activity in ADF

**B. Data Transformation Activities**
- **Data Flow:** Visual data transformation
- **Databricks Notebook:** Run Spark transformations
- **HDInsight Activities:** Hive, Pig, MapReduce, Spark
- **Stored Procedure:** Execute SQL stored procedures
- **Azure Function:** Custom code execution

**C. Control Flow Activities**
- **If Condition:** Conditional branching
- **ForEach:** Loop through items
- **Until:** Loop until condition is met
- **Wait:** Pause execution
- **Execute Pipeline:** Call another pipeline
- **Set Variable:** Set pipeline variable value
- **Append Variable:** Add to array variable

**D. External Activities**
- **Web Activity:** Call REST APIs
- **Webhook:** Call endpoint and wait for callback
- **Lookup:** Retrieve data for use in pipeline
- **Get Metadata:** Get file/folder properties

**Activity Structure:**
```json
{
  "name": "CopyActivity1",
  "type": "Copy",
  "dependsOn": [],
  "policy": {
    "timeout": "7.00:00:00",
    "retry": 3,
    "retryIntervalInSeconds": 30,
    "secureOutput": false,
    "secureInput": false
  },
  "userProperties": [],
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource"
    },
    "sink": {
      "type": "AzureSqlSink"
    },
    "enableStaging": false
  }
}
```

**Activity Dependencies:**
Activities can depend on the success, failure, or completion of previous activities:
```json
"dependsOn": [
  {
    "activity": "LookupActivity",
    "dependencyConditions": ["Succeeded"]
  }
]
```

### 5. Pipelines

**Think of it like:** A complete workflow or process. A recipe that lists all the steps in order.

**What it is:** A logical grouping of activities that together perform a task.

**Pipeline Components:**
- **Activities:** The tasks to execute
- **Parameters:** Input values passed at runtime
- **Variables:** Values that can change during execution
- **Annotations:** Metadata for organization
- **Concurrency:** Control parallel execution

**Simple Pipeline Example:**
```json
{
  "name": "CopyPipeline",
  "properties": {
    "activities": [
      {
        "name": "CopyFromBlobToSQL",
        "type": "Copy",
        "typeProperties": {
          "source": {
            "type": "DelimitedTextSource"
          },
          "sink": {
            "type": "AzureSqlSink",
            "writeBehavior": "insert"
          }
        },
        "inputs": [
          {
            "referenceName": "CsvDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "SqlTableDataset",
            "type": "DatasetReference"
          }
        ]
      }
    ],
    "parameters": {
      "sourceFileName": {
        "type": "string"
      }
    }
  }
}
```

**Complex Pipeline with Control Flow:**
```
Pipeline: DailyDataLoad
│
├─ Lookup: Get list of tables to load
│   └─ Success →
│
├─ ForEach: Loop through tables
│   ├─ Copy Activity: Load table
│   ├─ Stored Procedure: Update watermark
│   └─ If Condition: Check row count
│       ├─ True → Send success email
│       └─ False → Send failure email
│
└─ Execute Pipeline: Run aggregation pipeline
```

### 6. Triggers

**Think of it like:** The alarm clock or event that starts your workflow.

**What it is:** Determines when a pipeline execution should be kicked off.

**Trigger Types:**

**A. Schedule Trigger**
- Runs on wall-clock schedule
- Uses cron-like expressions
- Can specify start/end dates

**Example:**
```json
{
  "name": "DailyTrigger",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Day",
        "interval": 1,
        "startTime": "2024-01-01T00:00:00Z",
        "timeZone": "UTC",
        "schedule": {
          "hours": [2],
          "minutes": [0]
        }
      }
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "DailyLoadPipeline",
          "type": "PipelineReference"
        }
      }
    ]
  }
}
```

**B. Tumbling Window Trigger**
- Runs at periodic intervals
- Maintains state
- Supports dependencies between windows
- Can backfill historical data

**Use Case:** "Process data in 1-hour windows. If 2 PM window fails, you can rerun just that window without affecting others."

**C. Event-Based Trigger (Storage Events)**
- Triggers when blob is created/deleted
- Near real-time processing
- Requires Event Grid subscription

**Example Scenario:**
"Every time a CSV file lands in the 'incoming' folder, automatically trigger a pipeline to process it."

**D. Manual Trigger**
- On-demand execution
- Triggered via UI, API, or PowerShell

### 7. Data Flows

**Think of it like:** A visual transformation designer, similar to SSIS Data Flow but in the cloud.

**What it is:** Code-free data transformation at scale using a visual interface.

**Data Flow Components:**

**Source → Transformations → Sink**

**Available Transformations:**
- **Select:** Choose/rename columns
- **Filter:** Filter rows based on conditions
- **Derived Column:** Create calculated columns
- **Aggregate:** Group by and aggregate
- **Join:** Combine datasets
- **Lookup:** Enrich data with reference tables
- **Union:** Combine rows from multiple sources
- **Conditional Split:** Route rows to different paths
- **Sort:** Order data
- **Pivot/Unpivot:** Reshape data
- **Window:** Ranking, lead/lag functions
- **Surrogate Key:** Generate surrogate keys

**When to Use Data Flows:**
- Complex transformations (joins, aggregations)
- Need visual transformation designer
- Schema drift handling
- Large-scale transformations (uses Spark)

**When NOT to Use Data Flows:**
- Simple copy operations (use Copy Activity)
- Small datasets (overhead not worth it)
- Need real-time streaming (use Stream Analytics)

**Pro Tip:** Data Flows run on Spark clusters. They have startup time (cluster provisioning) but scale well for large datasets.

### 8. Control Flow vs Data Flow

**Control Flow:**
- **What:** Orchestration logic - the order and conditions of execution
- **Examples:** If-then-else, loops, error handling, calling other pipelines
- **Think of it as:** The project manager deciding what to do next

**Data Flow:**
- **What:** Actual data movement and transformation
- **Examples:** Copy data, transform columns, join tables
- **Think of it as:** The workers doing the actual data work

**Visual Representation:**
```
Control Flow (Pipeline Level):
┌─────────────────────────────────────────────┐
│ If (file exists)                             │
│   ├─ True: Copy Activity → Data Flow        │
│   └─ False: Send Alert                      │
└─────────────────────────────────────────────┘

Data Flow (Inside Data Flow Activity):
┌─────────────────────────────────────────────┐
│ Source → Filter → Join → Aggregate → Sink   │
└─────────────────────────────────────────────┘
```

## ADF vs Other ETL Tools

### ADF vs SSIS (SQL Server Integration Services)

| Aspect | ADF | SSIS |
|--------|-----|------|
| **Deployment** | Cloud (PaaS) | On-premises or VM |
| **Infrastructure** | Fully managed | You manage servers |
| **Scaling** | Auto-scales | Manual scaling |
| **Cost Model** | Pay-per-use | License + infrastructure |
| **Data Sources** | 90+ connectors | Limited without custom code |
| **Learning Curve** | Moderate | Steep |
| **Transformation** | Data Flows (Spark-based) | Rich transformation tasks |
| **Best For** | Cloud-first, hybrid scenarios | On-premises, complex transformations |

**When to Choose ADF over SSIS:**
- Building new cloud solutions
- Need to integrate cloud and on-premises
- Want to avoid infrastructure management
- Need elastic scaling
- Modern DevOps practices (Git, CI/CD)

**When to Choose SSIS over ADF:**
- Heavy investment in existing SSIS packages
- Complex transformations requiring custom .NET code
- All data sources are on-premises
- Need very specific control over execution

**Migration Path:** Use Azure-SSIS IR to run SSIS packages in ADF while gradually rewriting them.

### ADF vs Azure Databricks

| Aspect | ADF | Databricks |
|--------|-----|------------|
| **Primary Purpose** | Orchestration & ETL | Data engineering & ML |
| **Interface** | Visual designer | Code (Python, Scala, SQL) |
| **Transformations** | Data Flows, Copy Activity | Spark DataFrames, SQL |
| **Complexity** | Low to medium | Medium to high |
| **Performance** | Good for most scenarios | Excellent for big data |
| **Cost** | Lower for simple pipelines | Higher (compute costs) |
| **Best For** | Orchestration, simple ETL | Complex transformations, ML |

**Best Practice:** Use them together!
- ADF orchestrates the workflow
- Databricks does heavy transformations
- ADF handles scheduling and monitoring

**Example Architecture:**
```
ADF Pipeline:
1. Copy raw data to ADLS (ADF Copy Activity)
2. Transform data (ADF calls Databricks Notebook)
3. Load to SQL DW (ADF Copy Activity)
4. Send notification (ADF Web Activity)
```

### ADF vs Azure Synapse Analytics

**Important:** Synapse includes ADF capabilities plus more!

| Feature | ADF | Synapse |
|---------|-----|---------|
| **Data Integration** | ✅ Full | ✅ Full (same as ADF) |
| **Data Warehousing** | ❌ No | ✅ Dedicated SQL pools |
| **Spark Analytics** | ❌ No (calls Databricks) | ✅ Built-in Spark pools |
| **Serverless SQL** | ❌ No | ✅ Query data without loading |
| **Studio Experience** | ADF Studio | Synapse Studio (unified) |
| **Use Case** | Data integration only | End-to-end analytics |

**When to Choose ADF:**
- Only need data integration/ETL
- Already have separate analytics tools
- Cost-conscious (pay only for integration)

**When to Choose Synapse:**
- Need integrated analytics platform
- Want unified experience
- Building data warehouse + ETL together

## Real-World Use Cases

### Use Case 1: E-commerce Order Processing
**Scenario:** Online retailer receives orders from website, mobile app, and phone orders.

**Requirements:**
- Consolidate orders from 3 sources
- Enrich with customer data
- Load to data warehouse
- Run every 15 minutes

**ADF Solution:**
```
Tumbling Window Trigger (15 min) →
Pipeline:
  ├─ Copy: Website orders (REST API → ADLS)
  ├─ Copy: Mobile orders (Cosmos DB → ADLS)
  ├─ Copy: Phone orders (SQL Server → ADLS)
  ├─ Data Flow:
  │   ├─ Union all orders
  │   ├─ Lookup customer details
  │   ├─ Derive calculated columns
  │   └─ Load to Synapse
  └─ Stored Procedure: Update summary tables
```

### Use Case 2: Financial Compliance Reporting
**Scenario:** Bank needs to generate daily regulatory reports.

**Requirements:**
- Extract from 20+ source systems
- Apply business rules and validations
- Archive raw data for 7 years
- Generate audit trail
- Must complete by 6 AM

**ADF Solution:**
```
Schedule Trigger (11 PM) →
Pipeline:
  ├─ Lookup: Get list of source systems
  ├─ ForEach: Loop through sources
  │   ├─ Copy: Extract data
  │   ├─ Copy: Archive to ADLS (Parquet)
  │   └─ Data Flow: Validate and transform
  ├─ Execute Pipeline: Consolidation pipeline
  ├─ Stored Procedure: Generate reports
  └─ If Condition: Check completion time
      ├─ Success: Send confirmation email
      └─ Failure: Alert on-call team
```

### Use Case 3: IoT Sensor Data Ingestion
**Scenario:** Manufacturing company with 1000+ sensors generating data.

**Requirements:**
- Ingest data from IoT Hub
- Process in near real-time
- Store raw and aggregated data
- Alert on anomalies

**ADF Solution:**
```
Event-Based Trigger (IoT Hub) →
Pipeline:
  ├─ Copy: Raw data to ADLS (JSON)
  ├─ Data Flow:
  │   ├─ Parse JSON
  │   ├─ Filter invalid readings
  │   ├─ Aggregate by time windows
  │   └─ Load to SQL Database
  └─ If Condition: Check for anomalies
      └─ True: Call Azure Function (send alert)
```

### Use Case 4: Data Lake Ingestion
**Scenario:** Enterprise building data lake for analytics.

**Requirements:**
- Ingest from 50+ data sources
- Maintain bronze/silver/gold layers
- Metadata-driven approach
- Handle schema changes

**ADF Solution:**
```
Metadata-Driven Framework:
  ├─ Control Table (SQL): Lists all sources
  ├─ Master Pipeline:
  │   ├─ Lookup: Read control table
  │   └─ ForEach: Process each source
  │       ├─ Copy: Bronze layer (raw data)
  │       ├─ Data Flow: Silver layer (cleansed)
  │       └─ Data Flow: Gold layer (aggregated)
  └─ Schedule: Multiple triggers for different frequencies
```

## When to Choose ADF

### ✅ Choose ADF When:
1. **Cloud-first strategy:** Moving to or already in Azure
2. **Hybrid scenarios:** Mix of cloud and on-premises data
3. **Orchestration needs:** Complex workflows with dependencies
4. **Multiple data sources:** Need 90+ pre-built connectors
5. **Serverless preference:** Don't want to manage infrastructure
6. **Cost optimization:** Pay only for what you use
7. **DevOps integration:** Need Git, CI/CD pipelines
8. **Managed service:** Want Microsoft to handle updates/patches

### ❌ Don't Choose ADF When:
1. **Real-time streaming:** Use Azure Stream Analytics or Event Hubs
2. **Complex transformations:** Consider Databricks for heavy Spark work
3. **On-premises only:** SSIS might be more appropriate
4. **Simple file copies:** Azure Storage sync tools might suffice
5. **Extremely high frequency:** Sub-second processing needs
6. **Custom .NET code required:** SSIS has better support
7. **Very small datasets:** Overhead might not be worth it

## Interview Red Flags to Avoid

❌ **Don't Say:**
- "ADF is just like SSIS in the cloud" (it's different!)
- "I always use Data Flows for everything" (overkill for simple copies)
- "Self-hosted IR is outdated" (still very relevant for hybrid scenarios)
- "ADF can do real-time streaming" (it's for batch/micro-batch)

✅ **Do Say:**
- "ADF is an orchestration and integration service with both data movement and transformation capabilities"
- "I choose the right tool for each scenario - Copy Activity for simple moves, Data Flows for complex transformations"
- "Self-hosted IR is essential for secure hybrid connectivity"
- "For real-time needs, I'd recommend Stream Analytics, but ADF works great for micro-batch with tumbling windows"

## Pro Tips from Experience

**Tip 1: Start Simple**
Don't over-engineer. Begin with basic Copy Activities and add complexity only when needed.

**Tip 2: Parameterize Everything**
Make pipelines reusable from day one. Future you will thank you.

**Tip 3: Use Managed Identity**
Avoid storing credentials. Use Managed Identity wherever possible.

**Tip 4: Monitor from Day One**
Set up alerts early. Don't wait for production issues.

**Tip 5: Document Your Patterns**
Create internal documentation for your team's common patterns.

**Tip 6: Test in Lower Environments**
Always test in DEV/UAT before PROD. Use parameterized linked services.

**Tip 7: Understand Pricing**
Know what costs money (DIU hours, pipeline runs, data flows) to optimize costs.

## Common Mistakes to Avoid

**Mistake 1: Not Using Parameters**
Creating separate pipelines for similar tasks instead of one parameterized pipeline.

**Mistake 2: Ignoring Error Handling**
Not implementing try-catch patterns or retry logic.

**Mistake 3: Over-Using Data Flows**
Using Data Flows for simple copies when Copy Activity would suffice.

**Mistake 4: Poor Naming Conventions**
Using generic names like "Pipeline1", "Copy1" instead of descriptive names.

**Mistake 5: Not Monitoring Performance**
Not tracking DIU usage, pipeline duration, or identifying bottlenecks.

**Mistake 6: Hardcoding Values**
Embedding environment-specific values instead of using parameters.

**Mistake 7: Not Using Source Control**
Not integrating with Git, making change tracking difficult.

## Day-to-Day Work as an ADF Engineer

### Morning Routine:
1. **Check Monitoring Dashboard**
   - Review overnight pipeline runs
   - Check for failures or warnings
   - Verify SLAs were met

2. **Review Alerts**
   - Investigate any email alerts
   - Check Azure Monitor for anomalies
   - Review performance metrics

3. **Standup Meeting**
   - Discuss pipeline failures and resolutions
   - Share progress on new pipelines
   - Identify blockers

### Typical Tasks:
- **Development:** Building new pipelines, modifying existing ones
- **Troubleshooting:** Investigating failures, performance issues
- **Optimization:** Improving pipeline performance, reducing costs
- **Documentation:** Updating runbooks, creating design docs
- **Code Review:** Reviewing team members' pipeline changes
- **Meetings:** Requirements gathering, architecture discussions

### Common Support Tickets:
1. "Pipeline failed with timeout error"
2. "Data not showing up in destination"
3. "Pipeline running slower than usual"
4. "Need to add new data source"
5. "Schema changed in source system"

### Production Incidents:
**Incident:** "Critical pipeline failed - reports not generated"

**Response:**
1. Check monitoring logs
2. Identify failure point
3. Review error message
4. Quick fix if possible (rerun, increase timeout)
5. Implement permanent fix
6. Update documentation
7. Post-mortem meeting

## Summary

Azure Data Factory is a powerful, cloud-based data integration service that enables you to:
- **Orchestrate** complex data workflows
- **Integrate** data from diverse sources
- **Transform** data at scale
- **Monitor** all data operations
- **Automate** data pipelines

**Key Takeaways:**
1. ADF is about orchestration and integration, not just data movement
2. Choose the right component for each task (IR, Activity type, etc.)
3. Parameterization and reusability are crucial
4. Monitoring and error handling are not optional
5. Understand when to use ADF vs other tools

**Next Steps:**
- Understand all activity types in detail (next file)
- Learn about linked services and datasets
- Master Integration Runtime configurations
- Practice building pipelines

---

**Interview Confidence Builder:**
After understanding these concepts, you should be able to:
- Explain ADF architecture to a non-technical stakeholder
- Justify why you chose ADF over alternatives
- Describe the role of each component
- Discuss real-world implementation scenarios

Remember: Interviewers want to know you understand not just WHAT each component does, but WHY and WHEN to use it!

