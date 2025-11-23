# ADF-02: Complete Activities Reference

## Introduction to Activities

**Think of activities like:** Individual LEGO blocks. Each block has a specific shape and purpose, and you combine them to build something amazing.

Activities are the fundamental building blocks of Azure Data Factory pipelines. Each activity represents a single task or operation in your data workflow.

## Activity Categories Overview

```
Activities
├── Data Movement
│   └── Copy Activity
├── Data Transformation
│   ├── Data Flow
│   ├── Databricks Notebook/Jar/Python
│   ├── HDInsight (Hive, Pig, MapReduce, Spark, Streaming)
│   ├── Stored Procedure
│   ├── Azure Function
│   └── Custom Activity
├── Control Flow
│   ├── If Condition
│   ├── ForEach
│   ├── Until
│   ├── Wait
│   ├── Execute Pipeline
│   ├── Set Variable
│   ├── Append Variable
│   └── Filter
└── External/Utility
    ├── Web Activity
    ├── Webhook
    ├── Lookup
    ├── Get Metadata
    ├── Delete
    └── Validation
```

---

## DATA MOVEMENT ACTIVITIES

### Copy Activity

**What it does:** Moves data from a source to a destination. This is the most used activity in ADF.

**Think of it like:** A smart copy-paste operation that can handle millions of rows and transform data during the copy.

#### When to Use:
- Moving data between any supported data stores
- Simple data transformations during copy (column mapping, type conversion)
- Bulk data ingestion
- Data archival
- Data replication

#### Configuration Parameters:

**Source Configuration:**
```json
{
  "source": {
    "type": "AzureSqlSource",
    "sqlReaderQuery": "SELECT * FROM Orders WHERE OrderDate > '@{pipeline().parameters.LastRunDate}'",
    "queryTimeout": "02:00:00",
    "isolationLevel": "ReadCommitted",
    "partitionOption": "DynamicRange",
    "partitionSettings": {
      "partitionColumnName": "OrderID",
      "partitionUpperBound": "1000000",
      "partitionLowerBound": "1"
    }
  }
}
```

**Key Source Parameters:**
- **sqlReaderQuery / sqlReaderStoredProcedureName:** Custom query or stored procedure
- **queryTimeout:** Maximum time for query execution
- **isolationLevel:** Transaction isolation level
- **partitionOption:** Enable parallel reading (None, PhysicalPartitionsOfTable, DynamicRange)
- **additionalColumns:** Add static columns during copy

**Sink Configuration:**
```json
{
  "sink": {
    "type": "AzureSqlSink",
    "writeBehavior": "upsert",
    "upsertSettings": {
      "useTempDB": true,
      "keys": ["OrderID"]
    },
    "sqlWriterStoredProcedureName": "spMergeOrders",
    "sqlWriterTableType": "OrderTableType",
    "preCopyScript": "TRUNCATE TABLE StagingOrders",
    "tableOption": "autoCreate",
    "disableMetricsCollection": false
  }
}
```

**Key Sink Parameters:**
- **writeBehavior:** insert, upsert, or storedProcedure
- **preCopyScript:** SQL to run before copy
- **tableOption:** autoCreate creates table if not exists
- **maxConcurrentConnections:** Parallel connections to sink
- **writeBatchSize:** Rows per batch
- **writeBatchTimeout:** Timeout per batch

**Performance Settings:**
```json
{
  "enableStaging": true,
  "stagingSettings": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLinkedService",
      "type": "LinkedServiceReference"
    },
    "path": "staging-container/staging-folder",
    "enableCompression": true
  },
  "parallelCopies": 32,
  "dataIntegrationUnits": 16,
  "enableSkipIncompatibleRow": true,
  "redirectIncompatibleRowSettings": {
    "linkedServiceName": {
      "referenceName": "ErrorLogStorage",
      "type": "LinkedServiceReference"
    },
    "path": "error-logs"
  }
}
```

**Performance Parameters Explained:**

**Data Integration Units (DIU):**
- Measure of compute power for cloud-to-cloud copy
- Range: 2 to 256 DIUs
- Default: Auto (ADF decides)
- Higher DIU = faster copy but higher cost
- Formula: Cost = DIU × Hours × Price per DIU-hour

**Parallel Copies:**
- Number of parallel threads
- Range: 1 to 256
- Default: Auto-determined based on source/sink pattern
- Too high can overwhelm source/sink

**Staging:**
- Use intermediate blob storage for better performance
- Recommended for: Large datasets, cross-region copies, SQL DW loads
- Enables PolyBase for SQL DW

#### Real-World Example 1: Incremental Load

**Scenario:** Load only new/modified orders from SQL Server to Azure SQL Database.

**Business Requirement:**
"We have 10 million orders. We can't copy all of them every day. Only copy orders modified since last run."

**Implementation:**
```json
{
  "name": "IncrementalCopyOrders",
  "type": "Copy",
  "inputs": [
    {
      "referenceName": "SqlServerOrdersDataset",
      "type": "DatasetReference"
    }
  ],
  "outputs": [
    {
      "referenceName": "AzureSqlOrdersDataset",
      "type": "DatasetReference"
    }
  ],
  "typeProperties": {
    "source": {
      "type": "SqlServerSource",
      "sqlReaderQuery": {
        "value": "SELECT * FROM Orders WHERE ModifiedDate > '@{activity('LookupLastWatermark').output.firstRow.WatermarkValue}'",
        "type": "Expression"
      }
    },
    "sink": {
      "type": "AzureSqlSink",
      "writeBehavior": "upsert",
      "upsertSettings": {
        "keys": ["OrderID"]
      }
    },
    "enableStaging": false,
    "dataIntegrationUnits": 4
  }
}
```

**Complete Pipeline Flow:**
```
1. Lookup Activity: Get last watermark from control table
2. Copy Activity: Copy new/modified records
3. Stored Procedure: Update watermark to current timestamp
```

#### Real-World Example 2: File Pattern Copy

**Scenario:** Copy all CSV files from blob storage that match a pattern.

**Business Requirement:**
"Every day, vendors drop files like 'Sales_20240101.csv', 'Sales_20240102.csv' in blob storage. Copy all files from today."

**Implementation:**
```json
{
  "name": "CopyTodaysFiles",
  "type": "Copy",
  "inputs": [
    {
      "referenceName": "CsvFilesDataset",
      "type": "DatasetReference",
      "parameters": {
        "folderPath": "incoming",
        "fileName": {
          "value": "@concat('Sales_', formatDateTime(utcnow(), 'yyyyMMdd'), '*.csv')",
          "type": "Expression"
        }
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "ProcessedFilesDataset",
      "type": "DatasetReference"
    }
  ],
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings",
        "recursive": true,
        "wildcardFileName": "*.csv",
        "enablePartitionDiscovery": false
      }
    },
    "sink": {
      "type": "DelimitedTextSink"
    }
  }
}
```

#### Common Pitfalls:

**Pitfall 1: Not Using Staging for Large Copies**
- **Problem:** Copying 1TB directly to SQL DW is slow
- **Solution:** Enable staging with PolyBase
- **Impact:** 10x performance improvement

**Pitfall 2: Too Many DIUs**
- **Problem:** Setting DIU to 256 for small dataset
- **Solution:** Use Auto or start with 4-8 DIUs
- **Impact:** Unnecessary cost

**Pitfall 3: Ignoring Skip Incompatible Rows**
- **Problem:** One bad row fails entire copy
- **Solution:** Enable skipIncompatibleRow and log errors
- **Impact:** Pipeline resilience

**Pitfall 4: Not Using Partition Options**
- **Problem:** Single-threaded read from large table
- **Solution:** Use DynamicRange partitioning
- **Impact:** Parallel reads, faster extraction

#### Best Practices:

1. **Use Query Instead of Table for Source**
   - Filter at source to reduce data movement
   - Apply transformations in source query when possible

2. **Enable Staging for:**
   - Copies > 1GB
   - Cross-region copies
   - Loading to SQL DW/Synapse

3. **Monitor DIU Usage**
   - Check actual DIU used in monitoring
   - Adjust based on performance needs

4. **Implement Error Handling**
   - Use redirectIncompatibleRowSettings
   - Log errors for investigation

5. **Use Upsert for Idempotency**
   - Makes pipeline rerunnable
   - Prevents duplicates

#### Interview Questions:

**Q1: What's the difference between DIU and Parallel Copies?**
**A:** "DIU is the compute power allocated to the copy operation - it's like the size of your truck. Parallel Copies is how many threads/workers are copying simultaneously - it's like how many workers are loading the truck. You need both optimized. In my last project, we had a 500GB daily load. We used 16 DIUs with 32 parallel copies and staging enabled, which reduced our copy time from 4 hours to 45 minutes."

**Q2: How do you handle schema drift in Copy Activity?**
**A:** "Copy Activity has limited schema drift handling. For simple cases, I use 'enableStaging' and 'translator' with column mapping. For complex drift, I use Data Flows which have better schema drift capabilities. In a recent project, our source added columns weekly. We used Data Flow with 'Allow schema drift' enabled and 'Auto-map drifted columns' which automatically handled new columns without pipeline changes."

**Q3: Explain the upsert operation.**
**A:** "Upsert is 'update if exists, insert if not.' In Copy Activity, you configure it in the sink with writeBehavior='upsert' and specify key columns. ADF uses a staging table and MERGE statement behind the scenes. For example, when loading customer data, I use CustomerID as the key. If a customer exists, their record updates; if new, it inserts. This makes the pipeline idempotent - you can rerun it safely without duplicates."

**Q4: When would you NOT use Copy Activity?**
**A:** "I wouldn't use Copy Activity when:
1. Complex transformations needed (joins, aggregations) - use Data Flow
2. Row-by-row processing required - use ForEach with Stored Procedure
3. Real-time streaming - use Stream Analytics
4. Custom business logic - use Azure Function or Databricks
Copy Activity is best for bulk data movement with simple transformations."

---

## DATA TRANSFORMATION ACTIVITIES

### Data Flow Activity

**What it does:** Executes a mapping data flow for visual, code-free data transformation at scale.

**Think of it like:** SSIS Data Flow but cloud-based and running on Spark.

#### When to Use:
- Complex transformations (joins, aggregations, pivots)
- Schema drift handling
- Large-scale transformations (GB to TB)
- Visual transformation design preferred
- Need to avoid writing code

#### Configuration Parameters:

```json
{
  "name": "ExecuteDataFlow",
  "type": "ExecuteDataFlow",
  "typeProperties": {
    "dataFlow": {
      "referenceName": "TransformOrdersDataFlow",
      "type": "DataFlowReference",
      "parameters": {
        "ProcessDate": {
          "value": "@pipeline().parameters.ProcessDate",
          "type": "Expression"
        }
      }
    },
    "compute": {
      "coreCount": 8,
      "computeType": "General"
    },
    "traceLevel": "Fine",
    "runConcurrently": false,
    "continueOnError": false,
    "staging": {
      "linkedService": {
        "referenceName": "StagingStorage",
        "type": "LinkedServiceReference"
      },
      "folderPath": "dataflow-staging"
    }
  }
}
```

**Key Parameters:**
- **coreCount:** Spark cluster size (8, 16, 32, 64, 128, 256)
- **computeType:** General, MemoryOptimized, ComputeOptimized
- **traceLevel:** None, Coarse, Fine (for debugging)
- **staging:** Required for certain transformations

#### Real-World Example: Customer 360 View

**Scenario:** Combine customer data from multiple sources into a single view.

**Business Requirement:**
"We have customer data in CRM (Salesforce), transactions in SQL, and web behavior in Cosmos DB. Create a unified customer profile."

**Data Flow Design:**
```
Source: CRM Customers
Source: SQL Transactions
Source: Cosmos Web Events
    ↓
Join: Customers + Transactions (on CustomerID)
Join: Result + Web Events (on CustomerID)
    ↓
Aggregate: Calculate total purchases, visit count
Derived Column: Create customer segment (High/Medium/Low value)
    ↓
Conditional Split:
  - High Value → Gold Tier Sink
  - Medium Value → Silver Tier Sink
  - Low Value → Bronze Tier Sink
```

#### Best Practices:

1. **Use Data Flow Debug**
   - Test with sample data before full run
   - Debug sessions stay warm for 60 minutes
   - Cost: ~$0.50/hour for 8-core cluster

2. **Optimize Partitioning**
   - Use "Optimize" tab in each transformation
   - Round robin for even distribution
   - Hash partition for joins/aggregations
   - Key partition for lookups

3. **Enable Cluster Reuse**
   - Set Time to Live (TTL) for cluster
   - Subsequent flows reuse warm cluster
   - Reduces startup time from 5-7 minutes to seconds

4. **Monitor Data Flow Performance**
   - Check execution plan
   - Identify bottlenecks (skewed partitions)
   - Optimize partition counts

#### Common Pitfalls:

**Pitfall 1: Not Using Broadcast Joins**
- **Problem:** Joining large table with small lookup table
- **Solution:** Enable broadcast for small table
- **Impact:** Massive performance improvement

**Pitfall 2: Too Many Partitions**
- **Problem:** 1000 partitions for 100MB data
- **Solution:** Reduce partitions (1 partition per 100MB)
- **Impact:** Reduces overhead

**Pitfall 3: Not Caching Reference Data**
- **Problem:** Reading lookup table multiple times
- **Solution:** Enable sink cache for lookups
- **Impact:** Faster execution

#### Interview Questions:

**Q: Data Flow vs Copy Activity - when to use which?**
**A:** "Copy Activity is for data movement with simple transformations - column mapping, basic filtering. Data Flow is for complex transformations. For example, if I need to copy 100 tables as-is, I use Copy Activity. But if I need to join 5 tables, apply business rules, and create aggregations, I use Data Flow. In my last project, we used Copy Activity for raw layer ingestion (fast, cheap) and Data Flow for curated layer transformations (complex logic)."

---

### Stored Procedure Activity

**What it does:** Executes a stored procedure in a database.

**Think of it like:** Running a SQL script or stored procedure from your pipeline.

#### When to Use:
- Execute database-specific logic
- Update control tables (watermarks)
- Data validation
- Post-processing tasks
- Leverage existing stored procedures

#### Configuration:

```json
{
  "name": "UpdateWatermark",
  "type": "SqlServerStoredProcedure",
  "typeProperties": {
    "storedProcedureName": "usp_UpdateWatermark",
    "storedProcedureParameters": {
      "TableName": {
        "value": "Orders",
        "type": "String"
      },
      "WatermarkValue": {
        "value": {
          "value": "@activity('CopyOrders').output.rowsCopied",
          "type": "Expression"
        },
        "type": "DateTime"
      }
    }
  },
  "linkedServiceName": {
    "referenceName": "AzureSqlDatabase",
    "type": "LinkedServiceReference"
  }
}
```

#### Real-World Example: Watermark Pattern

**Scenario:** Track last successful load time for incremental loads.

**Stored Procedure:**
```sql
CREATE PROCEDURE usp_UpdateWatermark
    @TableName NVARCHAR(100),
    @WatermarkValue DATETIME
AS
BEGIN
    UPDATE ControlTable
    SET WatermarkValue = @WatermarkValue,
        LastUpdated = GETDATE()
    WHERE TableName = @TableName
END
```

**Pipeline Flow:**
```
1. Lookup: Get current watermark
2. Copy: Load data WHERE ModifiedDate > watermark
3. Stored Procedure: Update watermark to current time
```

#### Best Practices:

1. **Return Result Sets for Validation**
   - Stored procedure can return data
   - Use in downstream If Condition

2. **Use for Database-Specific Operations**
   - Merge operations
   - Index rebuilds
   - Statistics updates

3. **Handle Errors in Stored Procedure**
   - Use TRY-CATCH in SQL
   - Return error codes

#### Interview Question:

**Q: Why use Stored Procedure Activity instead of Copy Activity with stored procedure sink?**
**A:** "Copy Activity with stored procedure sink is for loading data using a stored procedure - you're copying data and the SP handles the insert/merge. Stored Procedure Activity is for executing any database logic without data movement - updating control tables, running maintenance, or validation. For example, after copying orders, I use Stored Procedure Activity to update the watermark table and run data quality checks."

---

### Databricks Notebook Activity

**What it does:** Executes a Databricks notebook for Spark-based transformations.

**Think of it like:** Running a Python/Scala script on a powerful Spark cluster.

#### When to Use:
- Complex transformations requiring code
- Machine learning pipelines
- Large-scale data processing (TB+)
- Custom business logic
- Python/Scala libraries needed

#### Configuration:

```json
{
  "name": "RunTransformation",
  "type": "DatabricksNotebook",
  "typeProperties": {
    "notebookPath": "/Shared/Transformations/ProcessOrders",
    "baseParameters": {
      "process_date": {
        "value": "@pipeline().parameters.ProcessDate",
        "type": "Expression"
      },
      "input_path": "/mnt/raw/orders",
      "output_path": "/mnt/curated/orders"
    }
  },
  "linkedServiceName": {
    "referenceName": "AzureDatabricksLinkedService",
    "type": "LinkedServiceReference"
  }
}
```

#### Real-World Example: ML Feature Engineering

**Scenario:** Prepare features for machine learning model.

**Business Requirement:**
"Transform raw transaction data into features for churn prediction model."

**Databricks Notebook:**
```python
# Get parameters from ADF
dbutils.widgets.text("process_date", "")
dbutils.widgets.text("input_path", "")
dbutils.widgets.text("output_path", "")

process_date = dbutils.widgets.get("process_date")
input_path = dbutils.widgets.get("input_path")
output_path = dbutils.widgets.get("output_path")

# Read data
df = spark.read.parquet(input_path)

# Feature engineering
from pyspark.sql.functions import *

features_df = df.groupBy("CustomerID").agg(
    count("TransactionID").alias("transaction_count"),
    sum("Amount").alias("total_spent"),
    avg("Amount").alias("avg_transaction"),
    datediff(current_date(), max("TransactionDate")).alias("days_since_last_purchase")
)

# Write features
features_df.write.mode("overwrite").parquet(output_path)

# Return results to ADF
dbutils.notebook.exit(f"Processed {features_df.count()} customers")
```

**ADF Pipeline:**
```
1. Copy: Raw transactions to ADLS
2. Databricks Notebook: Feature engineering
3. Copy: Features to SQL for ML model
```

#### Best Practices:

1. **Use Job Clusters, Not Interactive**
   - Job clusters are cheaper
   - Auto-terminate after job
   - Interactive clusters are for development

2. **Pass Parameters from ADF**
   - Makes notebooks reusable
   - Environment-agnostic

3. **Return Values to ADF**
   - Use dbutils.notebook.exit()
   - Return status, counts, or error messages

4. **Handle Errors**
   - Wrap in try-except
   - Return error status to ADF

#### Interview Question:

**Q: When would you use Databricks Notebook vs Data Flow?**
**A:** "Data Flow is great for visual transformations and when you want code-free development. Databricks is better when you need:
1. Custom Python/Scala libraries
2. Machine learning (MLlib, scikit-learn)
3. Very complex business logic
4. Existing Spark code to reuse
5. Performance optimization with custom Spark code

In my experience, I use Data Flow for standard ETL (joins, aggregations) and Databricks for ML pipelines and complex custom transformations. For example, we used Data Flow to prepare customer data, then Databricks for feature engineering and model training."

---

## CONTROL FLOW ACTIVITIES

### If Condition Activity

**What it does:** Conditional branching - execute different activities based on a condition.

**Think of it like:** An if-else statement in programming.

#### When to Use:
- Conditional logic based on previous activity results
- Error handling patterns
- Different processing paths based on data
- Validation checks

#### Configuration:

```json
{
  "name": "CheckFileExists",
  "type": "IfCondition",
  "typeProperties": {
    "expression": {
      "value": "@greater(activity('GetMetadata').output.itemsCount, 0)",
      "type": "Expression"
    },
    "ifTrueActivities": [
      {
        "name": "ProcessFile",
        "type": "Copy"
      }
    ],
    "ifFalseActivities": [
      {
        "name": "SendAlertEmail",
        "type": "Web"
      }
    ]
  },
  "dependsOn": [
    {
      "activity": "GetMetadata",
      "dependencyConditions": ["Succeeded"]
    }
  ]
}
```

#### Real-World Example 1: File Validation

**Scenario:** Process file only if it meets criteria.

**Business Requirement:**
"Only process CSV files that have more than 1000 rows. If less, send alert."

**Pipeline:**
```
1. Get Metadata: Get file row count
2. If Condition: Check if rowCount > 1000
   ├─ True: Copy Activity → Process file
   └─ False: Web Activity → Send alert to Logic App
```

**Expression:**
```javascript
@greater(activity('GetMetadata').output.rowCount, 1000)
```

#### Real-World Example 2: Error Handling

**Scenario:** Send different notifications based on success/failure.

**Pipeline:**
```
1. Copy Activity: Load data
2. If Condition: Check if copy succeeded
   ├─ True: 
   │   ├─ Stored Procedure: Update success log
   │   └─ Web Activity: Send success email
   └─ False:
       ├─ Stored Procedure: Log error
       └─ Web Activity: Send failure alert
```

**Expression:**
```javascript
@equals(activity('CopyData').status, 'Succeeded')
```

#### Common Expressions:

**Check if activity succeeded:**
```javascript
@equals(activity('CopyData').status, 'Succeeded')
```

**Check row count:**
```javascript
@greater(activity('CopyData').output.rowsCopied, 0)
```

**Check file exists:**
```javascript
@activity('GetMetadata').output.exists
```

**Compare values:**
```javascript
@and(
  greater(activity('Lookup').output.firstRow.Count, 1000),
  less(activity('Lookup').output.firstRow.Count, 1000000)
)
```

**Check time window:**
```javascript
@less(
  activity('CopyData').output.copyDuration,
  3600
)
```

#### Best Practices:

1. **Keep Conditions Simple**
   - Complex logic is hard to debug
   - Break into multiple If Conditions if needed

2. **Always Handle Both Branches**
   - Even if false branch is empty, document why

3. **Use for Validation**
   - Validate before expensive operations

4. **Combine with Lookup**
   - Lookup gets data, If Condition decides what to do

#### Interview Question:

**Q: How do you implement try-catch pattern in ADF?**
**A:** "ADF doesn't have native try-catch, but we implement it using activity dependencies and If Condition. Set up the activity with dependency conditions: 'Succeeded', 'Failed', 'Skipped', 'Completed'. For example:

```
Copy Activity
├─ On Success → Update success log
└─ On Failure → 
    ├─ Set Variable: Capture error
    └─ Send alert
```

For more complex scenarios, I use Execute Pipeline activity with 'waitOnCompletion: false' and check its status. In my last project, we had a critical nightly load. We used this pattern to capture failures, log to database, send alerts, and trigger a backup pipeline - all automatically."

---

### ForEach Activity

**What it does:** Loops through a collection and executes activities for each item.

**Think of it like:** A for loop in programming.

#### When to Use:
- Process multiple files/tables
- Iterate through lookup results
- Parallel processing
- Metadata-driven pipelines

#### Configuration:

```json
{
  "name": "ForEachTable",
  "type": "ForEach",
  "typeProperties": {
    "items": {
      "value": "@activity('GetTableList').output.value",
      "type": "Expression"
    },
    "isSequential": false,
    "batchCount": 10,
    "activities": [
      {
        "name": "CopyTable",
        "type": "Copy",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
              "value": "@concat('SELECT * FROM ', item().SchemaName, '.', item().TableName)",
              "type": "Expression"
            }
          }
        }
      }
    ]
  }
}
```

**Key Parameters:**
- **items:** Array to iterate (from Lookup, pipeline parameter, or static)
- **isSequential:** true = one at a time, false = parallel
- **batchCount:** Max parallel iterations (1-50, default 20)

#### Real-World Example 1: Metadata-Driven Load

**Scenario:** Load multiple tables based on control table.

**Business Requirement:**
"We have 50 tables to load daily. Configuration is in a control table. Load them all in parallel."

**Control Table:**
```sql
CREATE TABLE ControlTable (
    TableID INT,
    SchemaName NVARCHAR(50),
    TableName NVARCHAR(100),
    SourceQuery NVARCHAR(MAX),
    IsActive BIT
)
```

**Pipeline:**
```
1. Lookup: SELECT * FROM ControlTable WHERE IsActive = 1
2. ForEach: Loop through lookup results (isSequential = false, batchCount = 10)
   └─ Copy Activity: 
      Source Query: @item().SourceQuery
      Destination Table: @item().TableName
```

**ForEach Item Access:**
```javascript
// Access current item properties
@item().SchemaName
@item().TableName
@item().SourceQuery
```

#### Real-World Example 2: Process Multiple Files

**Scenario:** Process all CSV files in a folder.

**Pipeline:**
```
1. Get Metadata: Get list of files in folder
   - Field List: ["childItems"]
2. Filter: Filter only .csv files
3. ForEach: Loop through filtered files
   └─ Copy Activity: Process each file
      Source: @item().name
      Destination: Processed/@{item().name}
```

**Filter Expression:**
```javascript
@filter(
  activity('GetMetadata').output.childItems,
  endswith(item().name, '.csv')
)
```

#### Real-World Example 3: Parallel API Calls

**Scenario:** Call REST API for each customer.

**Pipeline:**
```
1. Lookup: Get list of customers
2. ForEach: Loop through customers (parallel)
   └─ Web Activity: Call API
      URL: @concat('https://api.example.com/customers/', item().CustomerID)
      Method: GET
```

#### Sequential vs Parallel:

**Sequential (isSequential = true):**
- Processes one item at a time
- Use when: Order matters, avoiding overload, dependent operations
- Example: Loading dimension tables in specific order

**Parallel (isSequential = false):**
- Processes multiple items simultaneously
- batchCount controls max parallel (default 20, max 50)
- Use when: Independent operations, need speed
- Example: Loading 100 fact tables

**Performance Comparison:**
```
50 tables, 5 minutes each:
- Sequential: 50 × 5 = 250 minutes (4+ hours)
- Parallel (batchCount=10): ~25 minutes
```

#### Best Practices:

1. **Use Parallel When Possible**
   - Massive time savings
   - Monitor to avoid overwhelming targets

2. **Combine with Lookup**
   - Lookup gets list, ForEach processes

3. **Use Filter for Conditional Iteration**
   - Filter array before ForEach
   - More efficient than If Condition inside loop

4. **Monitor Batch Count**
   - Start with 10-20
   - Increase if target can handle
   - Decrease if seeing timeouts

5. **Handle Errors in Loop**
   - Use activity dependencies
   - Log failures but continue processing

#### Common Pitfalls:

**Pitfall 1: Sequential When Parallel Would Work**
- **Problem:** 100 tables taking 8 hours
- **Solution:** Use parallel with batchCount=20
- **Impact:** Reduce to 30 minutes

**Pitfall 2: Too High Batch Count**
- **Problem:** batchCount=50 overwhelming database
- **Solution:** Reduce to 10-15
- **Impact:** Prevents timeouts

**Pitfall 3: Not Handling Failures**
- **Problem:** One failure stops entire loop
- **Solution:** Use activity dependencies with "Completed" condition
- **Impact:** Process all items even if some fail

#### Interview Questions:

**Q1: How do you handle failures in ForEach loop?**
**A:** "By default, if one iteration fails, the entire ForEach fails. To handle this, I use activity dependencies with 'Completed' condition instead of 'Succeeded'. This allows the loop to continue even if some iterations fail. Then I log failures to a table and send a summary alert at the end. For example:

```
ForEach:
  ├─ Copy Activity
  ├─ On Success → Log success
  └─ On Failure → Log failure (dependency: Completed)
```

After the loop, I use a Lookup to check the failure log and send alerts if needed."

**Q2: What's the maximum batch count and how do you determine the right value?**
**A:** "Maximum is 50, default is 20. The right value depends on your target system's capacity. I start with 10-20 and monitor:
1. Target system load (CPU, connections)
2. Pipeline duration
3. Timeout errors

In my last project, we were loading 80 tables to SQL Database. Started with batchCount=20, but saw connection pool exhaustion. Reduced to 10, which was the sweet spot - fast enough (40 minutes total) without overwhelming the database. I also added a Wait activity between batches to give the system breathing room."

**Q3: Can you nest ForEach loops?**
**A:** "Yes, but it's not recommended due to complexity and debugging difficulty. Instead, I flatten the data structure. For example, if I need to process multiple databases, each with multiple tables, I create a single control table with all database-table combinations and use one ForEach. If nesting is unavoidable, I use Execute Pipeline activity to call a child pipeline, which makes it more manageable and reusable."

---

### Until Activity

**What it does:** Repeats activities until a condition is true.

**Think of it like:** A while loop in programming.

#### When to Use:
- Polling for status
- Waiting for file arrival
- Retry logic with conditions
- Processing until queue is empty

#### Configuration:

```json
{
  "name": "WaitForFileProcessing",
  "type": "Until",
  "typeProperties": {
    "expression": {
      "value": "@equals(activity('CheckStatus').output.firstRow.Status, 'Completed')",
      "type": "Expression"
    },
    "timeout": "01:00:00",
    "activities": [
      {
        "name": "CheckStatus",
        "type": "Lookup"
      },
      {
        "name": "Wait30Seconds",
        "type": "Wait",
        "typeProperties": {
          "waitTimeInSeconds": 30
        }
      }
    ]
  }
}
```

**Key Parameters:**
- **expression:** Condition to check (must evaluate to boolean)
- **timeout:** Maximum time to loop (format: HH:MM:SS)
- **activities:** Activities to execute in each iteration

#### Real-World Example: Polling External API

**Scenario:** Submit job to external system and wait for completion.

**Business Requirement:**
"We submit data processing job to external API. It returns a job ID. We need to poll every 30 seconds until status is 'Completed' or timeout after 1 hour."

**Pipeline:**
```
1. Web Activity: Submit job
   - Returns: { "jobId": "12345", "status": "Running" }

2. Until Activity: Poll until complete
   - Timeout: 01:00:00
   - Activities:
     ├─ Wait: 30 seconds
     └─ Web Activity: Check status
         URL: https://api.example.com/jobs/@{activity('SubmitJob').output.jobId}
   - Condition: @equals(activity('CheckStatus').output.status, 'Completed')

3. If Condition: Check if completed or timed out
   ├─ Completed: Process results
   └─ Timeout: Send alert
```

#### Real-World Example: Wait for File

**Scenario:** Wait for file to arrive before processing.

**Pipeline:**
```
1. Until Activity: Wait for file
   - Timeout: 02:00:00
   - Activities:
     ├─ Get Metadata: Check if file exists
     └─ Wait: 60 seconds
   - Condition: @activity('GetMetadata').output.exists

2. Copy Activity: Process file
```

#### Best Practices:

1. **Always Set Timeout**
   - Prevents infinite loops
   - Default is 7 days (too long!)

2. **Add Wait Activity**
   - Don't hammer the system
   - 30-60 seconds between checks

3. **Use for Polling**
   - External system status
   - File arrival
   - Queue processing

4. **Handle Timeout**
   - Check if loop exited due to timeout
   - Send appropriate alerts

#### Common Pitfall:

**Pitfall: No Wait Between Iterations**
- **Problem:** Checking status 100 times per second
- **Solution:** Add Wait activity (30-60 seconds)
- **Impact:** Prevents API throttling

#### Interview Question:

**Q: Until vs ForEach - when to use which?**
**A:** "ForEach is for iterating through a known collection - you know how many items upfront. Until is for repeating until a condition is met - you don't know how many iterations. 

ForEach example: Process 50 files from a list.
Until example: Poll API until job completes.

In my experience, I used Until when integrating with external systems. We submitted batch jobs to a third-party system and polled every minute until completion. We set a 2-hour timeout and added error handling for timeout scenarios."

---

### Execute Pipeline Activity

**What it does:** Calls another pipeline from within a pipeline.

**Think of it like:** Calling a function or subroutine.

#### When to Use:
- Modular pipeline design
- Reusable pipeline components
- Parent-child pipeline patterns
- Complex orchestration

#### Configuration:

```json
{
  "name": "ExecuteChildPipeline",
  "type": "ExecutePipeline",
  "typeProperties": {
    "pipeline": {
      "referenceName": "ChildPipeline",
      "type": "PipelineReference"
    },
    "waitOnCompletion": true,
    "parameters": {
      "SourceTable": "Orders",
      "ProcessDate": {
        "value": "@pipeline().parameters.ProcessDate",
        "type": "Expression"
      }
    }
  }
}
```

**Key Parameters:**
- **pipeline:** Reference to pipeline to execute
- **waitOnCompletion:** true = wait for child to finish, false = fire and forget
- **parameters:** Parameters to pass to child pipeline

#### Real-World Example: Master-Child Pattern

**Scenario:** Load multiple data domains, each with its own pipeline.

**Master Pipeline:**
```
ExecutePipeline: Load Customers
ExecutePipeline: Load Products
ExecutePipeline: Load Orders
ExecutePipeline: Load Inventory
```

**Benefits:**
- Each domain pipeline is independent and reusable
- Can run child pipelines in parallel
- Easier to maintain and debug
- Team members can work on different pipelines

#### Real-World Example: Error Handling

**Scenario:** If main pipeline fails, trigger backup pipeline.

**Pipeline:**
```
Execute Pipeline: MainLoadPipeline
  ├─ On Success → Log success
  └─ On Failure → 
      ├─ Execute Pipeline: BackupLoadPipeline
      └─ Send Alert
```

#### Wait On Completion:

**waitOnCompletion = true (default):**
- Parent waits for child to finish
- Can access child pipeline output
- Use when: Need results from child, sequential processing

**waitOnCompletion = false:**
- Parent continues immediately
- Fire and forget
- Use when: Parallel processing, independent pipelines

**Example - Parallel Processing:**
```json
{
  "name": "ExecuteRegionPipelines",
  "type": "ForEach",
  "typeProperties": {
    "items": ["US", "EU", "APAC"],
    "isSequential": false,
    "activities": [
      {
        "name": "ExecuteRegionPipeline",
        "type": "ExecutePipeline",
        "typeProperties": {
          "pipeline": {
            "referenceName": "RegionLoadPipeline",
            "type": "PipelineReference"
          },
          "waitOnCompletion": true,
          "parameters": {
            "Region": "@item()"
          }
        }
      }
    ]
  }
}
```

#### Best Practices:

1. **Use for Modularity**
   - Break complex pipelines into smaller, reusable ones
   - Each pipeline has single responsibility

2. **Pass Parameters**
   - Make child pipelines configurable
   - Avoid hardcoding

3. **Handle Child Pipeline Failures**
   - Use activity dependencies
   - Log failures appropriately

4. **Avoid Deep Nesting**
   - Max 3-4 levels deep
   - Too deep = hard to debug

#### Interview Question:

**Q: How do you design a metadata-driven framework using Execute Pipeline?**
**A:** "I create a master pipeline that reads configuration from a control table and executes child pipelines dynamically. Here's the pattern:

**Control Table:**
```sql
CREATE TABLE PipelineConfig (
    PipelineID INT,
    PipelineName NVARCHAR(100),
    Parameters NVARCHAR(MAX), -- JSON
    ExecutionOrder INT,
    IsActive BIT
)
```

**Master Pipeline:**
```
1. Lookup: Get active pipelines from control table
2. ForEach: Loop through pipelines
   └─ Execute Pipeline: 
      Pipeline Name: @item().PipelineName
      Parameters: @json(item().Parameters)
```

This approach gives us:
- Configuration-driven execution
- No code changes for new pipelines
- Easy enable/disable of pipelines
- Centralized control

In my last project, we managed 50+ pipelines this way. Adding a new pipeline was just inserting a row in the control table - no code deployment needed."

---

### Set Variable & Append Variable Activities

**What they do:** 
- **Set Variable:** Assigns a value to a pipeline variable
- **Append Variable:** Adds an item to an array variable

**Think of it like:** Variable assignment in programming.

#### When to Use:
- Store intermediate results
- Build dynamic values
- Accumulate data across activities
- Pass data between activities

#### Configuration:

**Set Variable:**
```json
{
  "name": "SetProcessDate",
  "type": "SetVariable",
  "typeProperties": {
    "variableName": "ProcessDate",
    "value": {
      "value": "@formatDateTime(utcnow(), 'yyyy-MM-dd')",
      "type": "Expression"
    }
  }
}
```

**Append Variable:**
```json
{
  "name": "AppendFailedTable",
  "type": "AppendVariable",
  "typeProperties": {
    "variableName": "FailedTables",
    "value": {
      "value": "@item().TableName",
      "type": "Expression"
    }
  }
}
```

#### Variable Declaration (Pipeline Level):

```json
{
  "name": "MyPipeline",
  "properties": {
    "variables": {
      "ProcessDate": {
        "type": "String"
      },
      "RowCount": {
        "type": "Integer"
      },
      "FailedTables": {
        "type": "Array"
      },
      "IsValid": {
        "type": "Boolean"
      }
    }
  }
}
```

**Variable Types:**
- String
- Integer
- Boolean
- Array

#### Real-World Example: Accumulate Errors

**Scenario:** Track which tables failed during ForEach loop.

**Pipeline:**
```
Variables: FailedTables (Array)

ForEach: Loop through tables
  └─ Copy Activity: Load table
      ├─ On Success → Continue
      └─ On Failure → 
          └─ Append Variable: Add table name to FailedTables

After ForEach:
  If Condition: @greater(length(variables('FailedTables')), 0)
    └─ True: Send email with failed tables list
```

#### Real-World Example: Dynamic File Path

**Scenario:** Build file path dynamically.

**Pipeline:**
```
1. Set Variable: Year
   Value: @formatDateTime(utcnow(), 'yyyy')

2. Set Variable: Month
   Value: @formatDateTime(utcnow(), 'MM')

3. Set Variable: FilePath
   Value: @concat('data/', variables('Year'), '/', variables('Month'), '/sales.csv')

4. Copy Activity:
   Source: @variables('FilePath')
```

#### Best Practices:

1. **Use Parameters Over Variables When Possible**
   - Parameters are immutable (safer)
   - Variables are mutable (can change)

2. **Initialize Variables at Start**
   - Declare all variables in pipeline properties
   - Set initial values early

3. **Use Meaningful Names**
   - ProcessDate, not var1
   - FailedTables, not arr

4. **Limit Variable Usage**
   - Too many variables = hard to debug
   - Consider using Lookup output directly

#### Common Pitfall:

**Pitfall: Set Variable in Parallel Activities**
- **Problem:** Two parallel activities setting same variable
- **Result:** Race condition, unpredictable value
- **Solution:** Use Set Variable only in sequential flow

#### Interview Question:

**Q: Variables vs Parameters - what's the difference?**
**A:** "Parameters are inputs to the pipeline - they're set when the pipeline is triggered and can't be changed during execution. They're like function parameters. Variables are internal to the pipeline and can be modified during execution using Set Variable or Append Variable activities.

Use parameters for:
- Pipeline inputs (dates, file names, etc.)
- Values that don't change during run
- Configuration values

Use variables for:
- Intermediate calculations
- Accumulating results (like error lists)
- Dynamic values built during execution

In my projects, I use parameters for things like ProcessDate and SourceSystem, and variables for tracking progress like RowsProcessed or FailedItems."

---

## EXTERNAL/UTILITY ACTIVITIES

### Lookup Activity

**What it does:** Retrieves data from a data source (typically a single row or set of rows).

**Think of it like:** Running a SELECT query and storing the result for use in the pipeline.

#### When to Use:
- Get configuration data
- Retrieve watermark values
- Get list of files/tables to process
- Validation checks
- Get metadata for downstream activities

#### Configuration:

```json
{
  "name": "GetWatermark",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT WatermarkValue FROM ControlTable WHERE TableName = 'Orders'"
    },
    "dataset": {
      "referenceName": "AzureSqlDataset",
      "type": "DatasetReference"
    },
    "firstRowOnly": true
  }
}
```

**Key Parameters:**
- **firstRowOnly:** true = return first row only, false = return all rows (max 5000)
- **source:** Query or table to read from

#### Accessing Lookup Output:

**Single Row (firstRowOnly = true):**
```javascript
@activity('GetWatermark').output.firstRow.WatermarkValue
@activity('GetWatermark').output.firstRow.TableName
```

**Multiple Rows (firstRowOnly = false):**
```javascript
@activity('GetTableList').output.value
// Returns array of objects
// Use in ForEach: @activity('GetTableList').output.value
```

#### Real-World Example 1: Watermark Pattern

**Scenario:** Get last load timestamp for incremental load.

**Control Table:**
```sql
CREATE TABLE WatermarkTable (
    TableName NVARCHAR(100),
    WatermarkValue DATETIME,
    LastUpdated DATETIME
)
```

**Pipeline:**
```
1. Lookup: Get watermark
   Query: SELECT WatermarkValue FROM WatermarkTable WHERE TableName = 'Orders'
   firstRowOnly: true

2. Copy Activity: Incremental load
   Source Query: 
   SELECT * FROM Orders 
   WHERE ModifiedDate > '@{activity('GetWatermark').output.firstRow.WatermarkValue}'

3. Stored Procedure: Update watermark
   Parameters: 
   - TableName: 'Orders'
   - WatermarkValue: @utcnow()
```

#### Real-World Example 2: Dynamic Table List

**Scenario:** Get list of tables to load from control table.

**Control Table:**
```sql
CREATE TABLE TableConfig (
    TableID INT,
    SchemaName NVARCHAR(50),
    TableName NVARCHAR(100),
    SourceQuery NVARCHAR(MAX),
    IsActive BIT
)
```

**Pipeline:**
```
1. Lookup: Get table list
   Query: SELECT SchemaName, TableName, SourceQuery 
          FROM TableConfig 
          WHERE IsActive = 1
   firstRowOnly: false

2. ForEach: Loop through tables
   Items: @activity('GetTableList').output.value
   Activities:
     └─ Copy Activity:
        Source Query: @item().SourceQuery
        Destination: @concat(item().SchemaName, '.', item().TableName)
```

#### Real-World Example 3: Validation

**Scenario:** Check if source data exists before processing.

**Pipeline:**
```
1. Lookup: Check record count
   Query: SELECT COUNT(*) as RecordCount FROM SourceTable
   firstRowOnly: true

2. If Condition: Validate count
   Expression: @greater(activity('CheckCount').output.firstRow.RecordCount, 0)
   ├─ True: Process data
   └─ False: Send alert "No data to process"
```

#### Best Practices:

1. **Use firstRowOnly When Possible**
   - Faster performance
   - Less memory usage
   - Use false only when you need multiple rows

2. **Limit Rows Returned**
   - Max 5000 rows
   - Use TOP or LIMIT in query

3. **Combine with ForEach**
   - Lookup gets list, ForEach processes

4. **Use for Configuration**
   - Centralize configuration in database
   - Avoid hardcoding

#### Common Pitfall:

**Pitfall: Returning Too Many Rows**
- **Problem:** Lookup returns 10,000 rows (max is 5000)
- **Solution:** Add TOP 5000 or use pagination
- **Impact:** Activity fails

#### Interview Questions:

**Q1: What's the maximum number of rows Lookup can return?**
**A:** "5000 rows when firstRowOnly is false. If you need more, you have two options:
1. Use Copy Activity to load data to a staging table, then process from there
2. Implement pagination with multiple Lookup activities

In my experience, if you're hitting this limit, you should reconsider your design. Lookup is for configuration and metadata, not bulk data retrieval. For example, I once had a scenario where we needed to process 10,000 files. Instead of Lookup returning all file names, we used Get Metadata with childItems and filtered in the pipeline."

**Q2: How do you handle null values from Lookup?**
**A:** "Always check if the Lookup returned data before accessing it. Use coalesce or if-else logic:

```javascript
// Check if value exists
@if(
  equals(activity('GetWatermark').output.firstRow, null),
  '1900-01-01',
  activity('GetWatermark').output.firstRow.WatermarkValue
)
```

Or use coalesce in the SQL query itself:
```sql
SELECT COALESCE(WatermarkValue, '1900-01-01') as WatermarkValue
FROM WatermarkTable
WHERE TableName = 'Orders'
```

I prefer handling it in SQL when possible - cleaner and easier to debug."

---

### Get Metadata Activity

**What it does:** Retrieves metadata about files, folders, or datasets.

**Think of it like:** Running dir or ls command to get file/folder information.

#### When to Use:
- Check if file/folder exists
- Get file size, last modified date
- Get list of files in folder
- Validate before processing
- Dynamic file discovery

#### Configuration:

```json
{
  "name": "GetFileMetadata",
  "type": "GetMetadata",
  "typeProperties": {
    "dataset": {
      "referenceName": "BlobStorageDataset",
      "type": "DatasetReference",
      "parameters": {
        "folderPath": "incoming",
        "fileName": "sales.csv"
      }
    },
    "fieldList": [
      "exists",
      "itemName",
      "itemType",
      "size",
      "lastModified",
      "childItems"
    ],
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": true
    }
  }
}
```

**Available Fields:**
- **exists:** Boolean - file/folder exists
- **itemName:** String - file/folder name
- **itemType:** String - "File" or "Folder"
- **size:** Long - size in bytes
- **lastModified:** DateTime - last modified timestamp
- **childItems:** Array - list of files/folders (for folders)
- **structure:** Array - schema/columns (for structured datasets)
- **columnCount:** Integer - number of columns
- **rowCount:** Long - number of rows (limited support)

#### Accessing Get Metadata Output:

```javascript
// Check if exists
@activity('GetFileMetadata').output.exists

// Get file name
@activity('GetFileMetadata').output.itemName

// Get size
@activity('GetFileMetadata').output.size

// Get last modified
@activity('GetFileMetadata').output.lastModified

// Get child items (array)
@activity('GetFileMetadata').output.childItems
```

#### Real-World Example 1: File Existence Check

**Scenario:** Process file only if it exists.

**Pipeline:**
```
1. Get Metadata: Check file
   Field List: ["exists"]

2. If Condition: File exists?
   Expression: @activity('GetFileMetadata').output.exists
   ├─ True: Copy Activity → Process file
   └─ False: Web Activity → Send alert
```

#### Real-World Example 2: Process All Files in Folder

**Scenario:** Dynamically discover and process all CSV files.

**Pipeline:**
```
1. Get Metadata: Get folder contents
   Field List: ["childItems"]

2. Filter: Get only CSV files
   Items: @activity('GetFolderMetadata').output.childItems
   Condition: @endswith(item().name, '.csv')

3. ForEach: Loop through CSV files
   Items: @activity('FilterCSV').output.value
   Activities:
     └─ Copy Activity: Process file
        Source: @item().name
```

**Filter Activity Configuration:**
```json
{
  "name": "FilterCSV",
  "type": "Filter",
  "typeProperties": {
    "items": {
      "value": "@activity('GetFolderMetadata').output.childItems",
      "type": "Expression"
    },
    "condition": {
      "value": "@endswith(item().name, '.csv')",
      "type": "Expression"
    }
  }
}
```

#### Real-World Example 3: File Size Validation

**Scenario:** Process only if file is within size limits.

**Pipeline:**
```
1. Get Metadata: Get file size
   Field List: ["exists", "size"]

2. If Condition: Validate size
   Expression: 
   @and(
     activity('GetFileMetadata').output.exists,
     less(activity('GetFileMetadata').output.size, 1073741824)
   )
   // 1073741824 bytes = 1GB
   
   ├─ True: Process file
   └─ False: Send alert "File too large"
```

#### Real-World Example 4: Process Only New Files

**Scenario:** Process files modified in last 24 hours.

**Pipeline:**
```
1. Get Metadata: Get all files
   Field List: ["childItems"]

2. ForEach: Loop through files
   Activities:
     ├─ Get Metadata: Get file details
        Field List: ["lastModified"]
     
     ├─ If Condition: Check if recent
        Expression: 
        @less(
          div(sub(ticks(utcnow()), ticks(activity('GetFileDetails').output.lastModified)), 10000000),
          86400
        )
        // Check if modified in last 86400 seconds (24 hours)
        
        └─ True: Copy Activity → Process file
```

#### Best Practices:

1. **Request Only Needed Fields**
   - Each field has performance cost
   - Don't request all fields if you only need "exists"

2. **Use for Validation**
   - Check before expensive operations
   - Fail fast if conditions not met

3. **Combine with Filter**
   - Get Metadata returns all items
   - Use Filter to select specific ones

4. **Cache Results**
   - Don't call Get Metadata multiple times for same file
   - Store output in variable if needed multiple times

#### Common Pitfalls:

**Pitfall 1: Using childItems on Large Folders**
- **Problem:** Folder with 100,000 files
- **Solution:** Use folder structure/partitioning, or process in batches
- **Impact:** Timeout, performance issues

**Pitfall 2: Not Checking Exists Before Other Fields**
- **Problem:** Accessing size of non-existent file
- **Solution:** Always check exists first
- **Impact:** Activity fails

#### Interview Questions:

**Q1: Get Metadata vs Lookup - when to use which?**
**A:** "Get Metadata is for file/folder metadata - existence, size, list of files. Lookup is for data content - reading rows from tables or files.

Get Metadata examples:
- Check if file exists
- Get list of files in folder
- Get file size

Lookup examples:
- Get watermark value from table
- Get list of tables to process
- Get configuration data

In my projects, I use Get Metadata for file-based operations and Lookup for database-driven operations. For example, in a file ingestion pipeline, Get Metadata checks if the file arrived, then Lookup gets the processing configuration from a control table."

**Q2: How do you handle large numbers of files?**
**A:** "Get Metadata with childItems can be slow for folders with thousands of files. I use these strategies:

1. **Folder Partitioning:** Organize files by date/category
   ```
   /data/2024/01/01/file1.csv
   /data/2024/01/02/file1.csv
   ```
   Get Metadata on specific date folder, not root.

2. **File Naming Conventions:** Use patterns that allow filtering
   ```
   Sales_20240101_*.csv
   ```
   Get Metadata with wildcard support.

3. **Event-Based Triggers:** Instead of scanning folders, use blob events to trigger on file arrival.

In my last project, we had 50,000+ files per day. We partitioned by hour (/YYYY/MM/DD/HH/) and processed each hour's folder separately. This reduced Get Metadata time from 5 minutes to 10 seconds."

---

### Web Activity

**What it does:** Calls REST API endpoints.

**Think of it like:** Making HTTP requests from your pipeline.

#### When to Use:
- Call external APIs
- Send notifications
- Trigger external systems
- Get data from web services
- Integrate with Logic Apps, Azure Functions

#### Configuration:

```json
{
  "name": "CallAPI",
  "type": "WebActivity",
  "typeProperties": {
    "url": "https://api.example.com/data",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json",
      "Authorization": "Bearer @{activity('GetToken').output.access_token}"
    },
    "body": {
      "processDate": "@{pipeline().parameters.ProcessDate}",
      "recordCount": "@{activity('CopyData').output.rowsCopied}"
    },
    "authentication": {
      "type": "MSI",
      "resource": "https://management.azure.com/"
    }
  }
}
```

**HTTP Methods:**
- GET: Retrieve data
- POST: Create/submit data
- PUT: Update data
- DELETE: Delete data
- PATCH: Partial update

**Authentication Types:**
- **None:** No authentication
- **Basic:** Username/password
- **ClientCertificate:** Certificate-based
- **MSI:** Managed Service Identity

#### Real-World Example 1: Send Email via Logic App

**Scenario:** Send notification email on pipeline completion.

**Logic App:** Create Logic App with HTTP trigger and Send Email action.

**Pipeline:**
```
1. Copy Activity: Load data

2. Web Activity: Send success email
   URL: https://prod-123.logic.azure.com:443/workflows/abc.../triggers/manual/paths/invoke
   Method: POST
   Headers:
     Content-Type: application/json
   Body:
     {
       "subject": "Pipeline Completed Successfully",
       "body": "Loaded @{activity('CopyData').output.rowsCopied} rows",
       "recipient": "team@company.com"
     }
```

#### Real-World Example 2: Call External API with Pagination

**Scenario:** Retrieve data from paginated API.

**Pipeline:**
```
Variables: 
  - PageNumber: 1
  - HasMorePages: true

Until Activity: @equals(variables('HasMorePages'), false)
  Activities:
    1. Web Activity: Get page
       URL: @concat('https://api.example.com/data?page=', variables('PageNumber'))
       Method: GET
    
    2. Copy Activity: Save page data to ADLS
       Source: @activity('GetPage').output.data
    
    3. Set Variable: Increment page
       Variable: PageNumber
       Value: @add(variables('PageNumber'), 1)
    
    4. Set Variable: Check if more pages
       Variable: HasMorePages
       Value: @activity('GetPage').output.hasMore
    
    5. Wait: 1 second (rate limiting)
```

#### Real-World Example 3: Trigger Azure Function

**Scenario:** Call Azure Function for custom processing.

**Pipeline:**
```
1. Copy Activity: Load raw data

2. Web Activity: Trigger function
   URL: https://myfunctionapp.azurewebsites.net/api/ProcessData?code=xxx
   Method: POST
   Body:
     {
       "fileName": "@{pipeline().parameters.FileName}",
       "rowCount": "@{activity('CopyData').output.rowsCopied}"
     }

3. If Condition: Check function result
   Expression: @equals(activity('TriggerFunction').output.status, 'Success')
   ├─ True: Continue
   └─ False: Send alert
```

#### Real-World Example 4: Webhook Pattern

**Scenario:** Submit long-running job and wait for callback.

**Pipeline:**
```
1. Web Activity: Submit job
   URL: https://api.example.com/jobs
   Method: POST
   Body: { "jobType": "DataProcessing" }
   Returns: { "jobId": "12345", "callbackUrl": "..." }

2. Webhook Activity: Wait for completion
   URL: @activity('SubmitJob').output.callbackUrl
   Method: POST
   Timeout: 01:00:00

3. Web Activity: Get results
   URL: @concat('https://api.example.com/jobs/', activity('SubmitJob').output.jobId, '/results')
   Method: GET
```

#### Accessing Web Activity Output:

```javascript
// Status code
@activity('CallAPI').output.statusCode

// Response body
@activity('CallAPI').output.response

// Specific field from response
@activity('CallAPI').output.response.data.recordCount

// Headers
@activity('CallAPI').output.headers
```

#### Best Practices:

1. **Store Secrets in Key Vault**
   - Don't hardcode API keys
   - Use Key Vault linked service

2. **Handle Rate Limiting**
   - Add Wait activities between calls
   - Implement retry logic

3. **Validate Responses**
   - Check status code
   - Use If Condition for error handling

4. **Use Managed Identity When Possible**
   - Avoid storing credentials
   - More secure

5. **Log API Calls**
   - Store request/response for debugging
   - Use Stored Procedure to log

#### Common Pitfalls:

**Pitfall 1: Not Handling HTTP Errors**
- **Problem:** API returns 500, pipeline fails
- **Solution:** Use activity dependency with "Completed" condition
- **Impact:** Better error handling

**Pitfall 2: Exposing Secrets in URL**
- **Problem:** API key in URL visible in logs
- **Solution:** Use headers or Key Vault
- **Impact:** Security risk

**Pitfall 3: Not Implementing Retry**
- **Problem:** Transient network error fails pipeline
- **Solution:** Configure retry policy
- **Impact:** Resilience

#### Retry Policy Configuration:

```json
{
  "name": "CallAPI",
  "type": "WebActivity",
  "policy": {
    "timeout": "00:05:00",
    "retry": 3,
    "retryIntervalInSeconds": 30,
    "secureOutput": true,
    "secureInput": true
  }
}
```

#### Interview Questions:

**Q1: How do you secure API keys in Web Activity?**
**A:** "Never hardcode API keys. I use Azure Key Vault:

1. Store API key in Key Vault
2. Create Key Vault linked service in ADF
3. Reference secret in Web Activity:

```json
{
  "headers": {
    "Authorization": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVault",
        "type": "LinkedServiceReference"
      },
      "secretName": "ApiKey"
    }
  }
}
```

Also, I set secureInput and secureOutput to true to prevent keys from appearing in logs. In my last project, we integrated with 15+ external APIs, all using Key Vault. This made key rotation easy - update in Key Vault, no pipeline changes needed."

**Q2: How do you handle API pagination in ADF?**
**A:** "I use Until loop with Web Activity:

```
Variables: PageNumber=1, HasMore=true

Until: @equals(variables('HasMore'), false)
  1. Web Activity: Get page
     URL: https://api.com/data?page=@{variables('PageNumber')}
  
  2. Copy Activity: Save data
  
  3. Set Variable: PageNumber = PageNumber + 1
  
  4. Set Variable: HasMore = @activity('GetPage').output.hasNextPage
  
  5. Wait: 1 second
```

For APIs with cursor-based pagination:
```
Variables: NextCursor=""

Until: @equals(variables('NextCursor'), null)
  1. Web Activity: Get page
     URL: https://api.com/data?cursor=@{variables('NextCursor')}
  
  2. Set Variable: NextCursor = @activity('GetPage').output.nextCursor
```

In my experience with Salesforce API integration, we processed 500K records using pagination. We also added error handling and logging to track which page failed if issues occurred."

---

### Delete Activity

**What it does:** Deletes files or folders from data stores.

**Think of it like:** rm or del command.

#### When to Use:
- Clean up processed files
- Archive management
- Temporary file cleanup
- Data retention policies

#### Configuration:

```json
{
  "name": "DeleteProcessedFiles",
  "type": "Delete",
  "typeProperties": {
    "dataset": {
      "referenceName": "BlobStorageDataset",
      "type": "DatasetReference",
      "parameters": {
        "folderPath": "processed",
        "fileName": "*.csv"
      }
    },
    "enableLogging": true,
    "logStorageSettings": {
      "linkedServiceName": {
        "referenceName": "LogStorage",
        "type": "LinkedServiceReference"
      },
      "path": "delete-logs"
    },
    "recursive": true,
    "maxConcurrentConnections": 10
  }
}
```

**Key Parameters:**
- **recursive:** Delete subfolders
- **enableLogging:** Log deleted files
- **maxConcurrentConnections:** Parallel deletes

#### Real-World Example: Archive Pattern

**Scenario:** Move processed files to archive, then delete from source.

**Pipeline:**
```
1. Get Metadata: Get list of processed files
   Field List: ["childItems"]

2. ForEach: Loop through files
   Activities:
     ├─ Copy Activity: Copy to archive
        Source: processed/@{item().name}
        Destination: archive/@{formatDateTime(utcnow(),'yyyy/MM/dd')}/@{item().name}
     
     └─ Delete Activity: Delete from source
        File: processed/@{item().name}
```

#### Real-World Example: Retention Policy

**Scenario:** Delete files older than 30 days.

**Pipeline:**
```
1. Get Metadata: Get all files
   Field List: ["childItems"]

2. ForEach: Loop through files
   Activities:
     ├─ Get Metadata: Get file last modified
        Field List: ["lastModified"]
     
     ├─ If Condition: Check if old
        Expression: 
        @greater(
          div(sub(ticks(utcnow()), ticks(activity('GetFileDetails').output.lastModified)), 10000000),
          2592000
        )
        // 2592000 seconds = 30 days
        
        └─ True: Delete Activity → Delete file
```

#### Best Practices:

1. **Always Archive Before Delete**
   - Copy to archive location first
   - Then delete source

2. **Enable Logging**
   - Track what was deleted
   - Audit trail

3. **Test with Small Dataset**
   - Delete is permanent!
   - Test thoroughly before production

4. **Use Wildcards Carefully**
   - *.csv deletes ALL CSV files
   - Be specific

#### Interview Question:

**Q: How do you implement a data retention policy in ADF?**
**A:** "I create a scheduled pipeline that:
1. Gets list of files with Get Metadata
2. Checks lastModified date
3. Archives old files to cold storage
4. Deletes from hot storage

For database retention, I use Stored Procedure Activity:
```sql
DELETE FROM TransactionLog
WHERE LogDate < DATEADD(DAY, -90, GETDATE())
```

In my last project, we had a 7-year retention policy for financial data. We used ADF to:
- Keep last 30 days in SQL Database (hot)
- Move 30-365 days to ADLS (warm)
- Move 1-7 years to Archive Storage (cold)
- Delete after 7 years

This reduced storage costs by 70% while maintaining compliance."

---

## Summary and Activity Selection Guide

### Quick Reference: Which Activity to Use?

**Data Movement:**
- Simple copy → **Copy Activity**
- Complex transformations → **Data Flow**

**Database Operations:**
- Run stored procedure → **Stored Procedure Activity**
- Get data for decisions → **Lookup Activity**

**File Operations:**
- Check file exists → **Get Metadata Activity**
- Delete files → **Delete Activity**
- Validate file → **Validation Activity**

**Control Flow:**
- Conditional logic → **If Condition Activity**
- Loop through items → **ForEach Activity**
- Repeat until condition → **Until Activity**
- Call another pipeline → **Execute Pipeline Activity**
- Store values → **Set/Append Variable Activities**

**External Integration:**
- Call REST API → **Web Activity**
- Wait for callback → **Webhook Activity**
- Run custom code → **Azure Function Activity**
- Spark processing → **Databricks Activity**

**Timing:**
- Pause execution → **Wait Activity**

### Performance Considerations

**Fast Activities (< 1 second):**
- Set Variable
- Append Variable
- If Condition
- Wait (configurable)

**Medium Activities (seconds to minutes):**
- Lookup
- Get Metadata
- Web Activity
- Stored Procedure

**Slow Activities (minutes to hours):**
- Copy Activity (depends on data size)
- Data Flow (cluster startup + processing)
- Databricks (cluster startup + processing)

### Cost Considerations

**Free/Cheap:**
- Control flow activities (If, ForEach, Set Variable)
- Lookup, Get Metadata (minimal cost)

**Moderate:**
- Copy Activity (DIU-hours)
- Web Activity (per 1000 calls)

**Expensive:**
- Data Flow (Spark cluster time)
- Databricks (compute costs)

### Interview Confidence Builder

After mastering these activities, you should be able to:
1. Choose the right activity for any scenario
2. Explain trade-offs between different approaches
3. Design complex pipelines with multiple activities
4. Troubleshoot activity failures
5. Optimize pipeline performance
6. Discuss real-world implementations confidently

**Remember:** Interviewers want to see you understand:
- **WHAT** each activity does
- **WHEN** to use it
- **WHY** you chose it over alternatives
- **HOW** to configure it properly
- **WHAT** can go wrong and how to fix it

---

**Next Steps:**
- Practice building pipelines with these activities
- Understand linked services and datasets in depth
- Learn about Integration Runtime configurations
- Master triggers and scheduling

**Pro Tip:** In interviews, always explain your reasoning. Don't just say "I'd use Copy Activity." Say "I'd use Copy Activity because it's optimized for bulk data movement, supports parallel processing with DIU scaling, and has built-in error handling. For this scenario with 10GB of data, I'd enable staging and use 8 DIUs for optimal performance."

