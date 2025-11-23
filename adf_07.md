# ADF-07: Real-World Scenarios - Additional Patterns

## Overview
This document covers additional critical scenarios that ADF engineers encounter in production environments. These scenarios focus on pipeline orchestration, cost management, and operational monitoring - essential skills for any experienced ADF practitioner.

---

## Scenario 29: Pipeline Dependencies

### Business Requirement
Your organization has multiple data pipelines that must execute in a specific order. For example:
- **Pipeline A** loads raw customer data from source systems
- **Pipeline B** cleanses and transforms the customer data
- **Pipeline C** aggregates customer metrics for reporting
- **Pipeline D** sends the final reports to stakeholders

Pipeline B cannot start until Pipeline A completes successfully. Pipeline C depends on Pipeline B, and Pipeline D depends on Pipeline C. You need to orchestrate these dependencies reliably.

### Why This Matters
In real-world data warehousing, data flows through multiple stages (Bronze → Silver → Gold layers, or Staging → ODS → Data Warehouse). Dependencies ensure data quality and consistency. Running pipelines out of order can cause:
- Incomplete or stale data in downstream systems
- Failed transformations due to missing source data
- Incorrect business reports
- Data integrity issues

### Implementation Approaches

#### **Approach 1: Parent-Child Pipeline Pattern (Execute Pipeline Activity)**

This is the most common and recommended approach for pipeline dependencies.

**Step-by-Step Implementation:**

1. **Create Your Child Pipelines**
   - Pipeline_LoadCustomers (Pipeline A)
   - Pipeline_TransformCustomers (Pipeline B)
   - Pipeline_AggregateMetrics (Pipeline C)
   - Pipeline_SendReports (Pipeline D)

2. **Create a Master Orchestrator Pipeline**

```json
{
  "name": "Master_Customer_Data_Pipeline",
  "properties": {
    "activities": [
      {
        "name": "Execute_LoadCustomers",
        "type": "ExecutePipeline",
        "dependsOn": [],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 2,
          "retryIntervalInSeconds": 30
        },
        "typeProperties": {
          "pipeline": {
            "referenceName": "Pipeline_LoadCustomers",
            "type": "PipelineReference"
          },
          "waitOnCompletion": true,
          "parameters": {
            "LoadDate": "@pipeline().parameters.ProcessDate"
          }
        }
      },
      {
        "name": "Execute_TransformCustomers",
        "type": "ExecutePipeline",
        "dependsOn": [
          {
            "activity": "Execute_LoadCustomers",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "policy": {
          "timeout": "0.12:00:00",
          "retry": 1,
          "retryIntervalInSeconds": 30
        },
        "typeProperties": {
          "pipeline": {
            "referenceName": "Pipeline_TransformCustomers",
            "type": "PipelineReference"
          },
          "waitOnCompletion": true,
          "parameters": {
            "LoadDate": "@pipeline().parameters.ProcessDate"
          }
        }
      },
      {
        "name": "Execute_AggregateMetrics",
        "type": "ExecutePipeline",
        "dependsOn": [
          {
            "activity": "Execute_TransformCustomers",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "pipeline": {
            "referenceName": "Pipeline_AggregateMetrics",
            "type": "PipelineReference"
          },
          "waitOnCompletion": true
        }
      },
      {
        "name": "Execute_SendReports",
        "type": "ExecutePipeline",
        "dependsOn": [
          {
            "activity": "Execute_AggregateMetrics",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "pipeline": {
            "referenceName": "Pipeline_SendReports",
            "type": "PipelineReference"
          },
          "waitOnCompletion": true
        }
      }
    ],
    "parameters": {
      "ProcessDate": {
        "type": "string",
        "defaultValue": "@utcnow()"
      }
    }
  }
}
```

**Key Configuration Points:**
- `waitOnCompletion: true` - Master pipeline waits for child pipeline to complete
- `dependsOn` - Defines the dependency chain
- `dependencyConditions: ["Succeeded"]` - Only execute if previous activity succeeded
- Parameters can be passed from parent to child pipelines

#### **Approach 2: Trigger Dependencies (Tumbling Window Triggers)**

For scheduled pipelines with time-based dependencies.

**Scenario:** You have daily pipelines that must run in sequence.

1. **Create Tumbling Window Trigger for Pipeline A**
```json
{
  "name": "TW_LoadCustomers_Daily",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:00:00",
      "maxConcurrency": 1
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "Pipeline_LoadCustomers",
        "type": "PipelineReference"
      }
    }
  }
}
```

2. **Create Dependent Trigger for Pipeline B**
```json
{
  "name": "TW_TransformCustomers_Daily",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:00:00",
      "maxConcurrency": 1,
      "dependsOn": [
        {
          "type": "TumblingWindowTriggerDependencyReference",
          "referenceTrigger": {
            "referenceName": "TW_LoadCustomers_Daily",
            "type": "TriggerReference"
          },
          "offset": "00:00:00"
        }
      ]
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "Pipeline_TransformCustomers",
        "type": "PipelineReference"
      }
    }
  }
}
```

**When to Use Trigger Dependencies:**
- Scheduled pipelines with consistent time windows
- Need automatic retry of failed dependencies
- Cross-data-factory dependencies
- Self-dependency scenarios (process previous day's data)

#### **Approach 3: Conditional Execution with Lookup**

For dynamic dependencies based on data state.

**Scenario:** Only run Pipeline B if Pipeline A processed more than 0 records.

```json
{
  "name": "Check_And_Execute_Transform",
  "activities": [
    {
      "name": "Lookup_RecordCount",
      "type": "Lookup",
      "typeProperties": {
        "source": {
          "type": "AzureSqlSource",
          "sqlReaderQuery": "SELECT COUNT(*) as RecordCount FROM dbo.CustomerStaging WHERE LoadDate = '@{pipeline().parameters.ProcessDate}'"
        },
        "dataset": {
          "referenceName": "DS_AzureSQL_Control",
          "type": "DatasetReference"
        }
      }
    },
    {
      "name": "If_RecordsExist",
      "type": "IfCondition",
      "dependsOn": [
        {
          "activity": "Lookup_RecordCount",
          "dependencyConditions": ["Succeeded"]
        }
      ],
      "typeProperties": {
        "expression": {
          "value": "@greater(activity('Lookup_RecordCount').output.firstRow.RecordCount, 0)",
          "type": "Expression"
        },
        "ifTrueActivities": [
          {
            "name": "Execute_TransformPipeline",
            "type": "ExecutePipeline",
            "typeProperties": {
              "pipeline": {
                "referenceName": "Pipeline_TransformCustomers",
                "type": "PipelineReference"
              },
              "waitOnCompletion": true
            }
          }
        ]
      }
    }
  ]
}
```

### Advanced Patterns

#### **Pattern 1: Parallel Dependencies (Fan-Out, Fan-In)**

Execute multiple pipelines in parallel, then wait for all to complete before proceeding.

```
Pipeline A (Load Customers)
    ├─→ Pipeline B1 (Transform Region 1) ─┐
    ├─→ Pipeline B2 (Transform Region 2) ─┼─→ Pipeline C (Aggregate All)
    └─→ Pipeline B3 (Transform Region 3) ─┘
```

**Implementation:**
```json
{
  "activities": [
    {
      "name": "Execute_LoadCustomers",
      "type": "ExecutePipeline",
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_LoadCustomers"}
      }
    },
    {
      "name": "Execute_Transform_Region1",
      "type": "ExecutePipeline",
      "dependsOn": [{"activity": "Execute_LoadCustomers", "dependencyConditions": ["Succeeded"]}],
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_Transform_Region1"}
      }
    },
    {
      "name": "Execute_Transform_Region2",
      "type": "ExecutePipeline",
      "dependsOn": [{"activity": "Execute_LoadCustomers", "dependencyConditions": ["Succeeded"]}],
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_Transform_Region2"}
      }
    },
    {
      "name": "Execute_Transform_Region3",
      "type": "ExecutePipeline",
      "dependsOn": [{"activity": "Execute_LoadCustomers", "dependencyConditions": ["Succeeded"]}],
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_Transform_Region3"}
      }
    },
    {
      "name": "Execute_AggregateAll",
      "type": "ExecutePipeline",
      "dependsOn": [
        {"activity": "Execute_Transform_Region1", "dependencyConditions": ["Succeeded"]},
        {"activity": "Execute_Transform_Region2", "dependencyConditions": ["Succeeded"]},
        {"activity": "Execute_Transform_Region3", "dependencyConditions": ["Succeeded"]}
      ],
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_AggregateAll"}
      }
    }
  ]
}
```

#### **Pattern 2: Dependency with Failure Handling**

Continue execution even if one pipeline fails, but track the failure.

```json
{
  "name": "Execute_With_Failure_Tracking",
  "activities": [
    {
      "name": "Execute_CriticalPipeline",
      "type": "ExecutePipeline",
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_Critical"}
      }
    },
    {
      "name": "Execute_OptionalPipeline",
      "type": "ExecutePipeline",
      "dependsOn": [
        {
          "activity": "Execute_CriticalPipeline",
          "dependencyConditions": ["Succeeded", "Failed"]
        }
      ],
      "typeProperties": {
        "pipeline": {"referenceName": "Pipeline_Optional"}
      }
    },
    {
      "name": "Log_Failures",
      "type": "SqlServerStoredProcedure",
      "dependsOn": [
        {
          "activity": "Execute_OptionalPipeline",
          "dependencyConditions": ["Completed"]
        }
      ],
      "typeProperties": {
        "storedProcedureName": "usp_LogPipelineStatus",
        "storedProcedureParameters": {
          "PipelineName": "@pipeline().Pipeline",
          "Status": "@if(equals(activity('Execute_CriticalPipeline').Status, 'Succeeded'), 'Success', 'Failed')",
          "ErrorMessage": "@activity('Execute_CriticalPipeline').Error.message"
        }
      }
    }
  ]
}
```

### Expected Behavior

**Successful Execution:**
1. Master pipeline triggers
2. Pipeline A executes and completes successfully
3. Pipeline B starts automatically after A succeeds
4. Pipeline C starts after B succeeds
5. Pipeline D starts after C succeeds
6. Master pipeline completes with "Succeeded" status

**Failure Scenario:**
1. Pipeline A succeeds
2. Pipeline B fails
3. Pipeline C never starts (dependency not met)
4. Pipeline D never starts
5. Master pipeline completes with "Failed" status
6. Error details available in activity output

### Common Issues and Resolutions

#### **Issue 1: Child Pipeline Doesn't Start**
**Symptoms:** Parent pipeline succeeds, but child never executes

**Causes:**
- `waitOnCompletion` set to `false` - parent doesn't wait
- Incorrect dependency condition
- Child pipeline is disabled or deleted

**Resolution:**
```json
// Verify these settings:
{
  "typeProperties": {
    "waitOnCompletion": true,  // Must be true for sequential execution
    "pipeline": {
      "referenceName": "CorrectPipelineName",  // Verify name is exact
      "type": "PipelineReference"
    }
  }
}
```

#### **Issue 2: Circular Dependencies**
**Symptoms:** "Circular dependency detected" error

**Cause:** Pipeline A depends on B, and B depends on A

**Resolution:**
- Review dependency chain
- Redesign pipeline architecture
- Use intermediate staging pipelines

#### **Issue 3: Timeout Issues**
**Symptoms:** Parent pipeline times out waiting for child

**Resolution:**
```json
{
  "policy": {
    "timeout": "7.00:00:00",  // Increase timeout (7 days max)
    "retry": 0,
    "retryIntervalInSeconds": 30
  }
}
```

#### **Issue 4: Parameter Passing Fails**
**Symptoms:** Child pipeline receives null or incorrect parameters

**Resolution:**
```json
{
  "typeProperties": {
    "parameters": {
      "ParameterName": {
        "value": "@pipeline().parameters.ParentParameter",  // Correct syntax
        "type": "Expression"
      }
    }
  }
}
```

### Monitoring Pipeline Dependencies

**Using Azure Monitor:**
```kusto
// Query to track pipeline execution chain
ADFPipelineRun
| where ResourceId contains "YourDataFactoryName"
| where PipelineName in ("Pipeline_LoadCustomers", "Pipeline_TransformCustomers", "Pipeline_AggregateMetrics")
| project TimeGenerated, PipelineName, Status, Start, End, Duration=End-Start
| order by Start asc
```

**Monitoring UI Navigation:**
1. Go to ADF Studio → Monitor → Pipeline runs
2. Filter by parent pipeline name
3. Click on pipeline run → View activity runs
4. Click on "Execute Pipeline" activity → View child pipeline run
5. Drill down through entire execution chain

### Best Practices

1. **Use Descriptive Names**
   - ✅ `Execute_LoadCustomerData_Step1`
   - ❌ `ExecutePipeline1`

2. **Set Appropriate Timeouts**
   - Short-running pipelines: 1-2 hours
   - Long-running ETL: 12-24 hours
   - Never use default timeout for production

3. **Implement Logging**
   - Log start/end of each pipeline
   - Track record counts processed
   - Store execution metadata in control tables

4. **Parameter Propagation**
   - Pass execution date through all pipelines
   - Use consistent parameter names
   - Document required vs optional parameters

5. **Error Handling**
   - Decide: fail fast or continue on error?
   - Implement notification for critical failures
   - Use retry policies judiciously

6. **Testing Dependencies**
   - Test failure scenarios
   - Verify timeout behavior
   - Test with realistic data volumes

### Interview Questions You Might Face

**Q1: "How do you ensure Pipeline B only runs after Pipeline A completes successfully?"**

**Answer:** "I use the Execute Pipeline activity in a master orchestrator pipeline. The key is setting the `dependsOn` property with `dependencyConditions: ['Succeeded']` and `waitOnCompletion: true`. This ensures the parent pipeline waits for the child to complete and only proceeds if it succeeds. For scheduled pipelines, I use Tumbling Window Triggers with trigger dependencies, which provide built-in retry logic and better handling of historical loads."

**Q2: "What's the difference between Execute Pipeline activity and Tumbling Window trigger dependencies?"**

**Answer:** "Execute Pipeline is for runtime orchestration - you manually control the execution flow within a pipeline. It's synchronous and gives you fine-grained control with parameters and conditional logic. Tumbling Window trigger dependencies are for scheduled, time-based dependencies. They're asynchronous, support self-dependencies, and can handle cross-data-factory dependencies. I use Execute Pipeline for complex orchestration logic and Tumbling Window dependencies for consistent daily/hourly scheduled loads."

**Q3: "How do you handle a scenario where Pipeline C depends on both Pipeline A and Pipeline B completing?"**

**Answer:** "I create a master pipeline with three Execute Pipeline activities. Pipeline A and B both depend on nothing (they run in parallel). Pipeline C has a `dependsOn` array with both A and B, each with `dependencyConditions: ['Succeeded']`. This creates a fan-in pattern where C waits for both parents to succeed before executing."

**Q4: "What happens if a child pipeline fails? How do you handle it?"**

**Answer:** "By default, if a child pipeline fails, the Execute Pipeline activity fails, and any dependent activities won't run. The parent pipeline status becomes 'Failed'. To handle this, I have several strategies: 1) Use retry policies on the Execute Pipeline activity for transient failures, 2) Use If Condition to check the status and execute alternative logic, 3) Set `dependencyConditions: ['Failed']` on a notification activity to alert the team, 4) Implement a stored procedure activity to log the failure details to a control table for tracking."

**Q5: "How do you pass data between dependent pipelines?"**

**Answer:** "There are three main approaches: 1) **Parameters** - Pass simple values like dates or IDs through pipeline parameters. 2) **Staging Tables** - Pipeline A writes metadata to a control table, Pipeline B reads from it. 3) **Pipeline Output** - Use `@activity('ExecutePipelineActivity').output.pipelineRunId` to get the child pipeline's run ID and query its metadata. For complex data, I always use staging tables or blob storage as intermediaries rather than trying to pass large datasets through parameters."

**Q6: "Describe a complex dependency scenario you've implemented."**

**Answer:** "In my last project, we had a medallion architecture with Bronze, Silver, and Gold layers. The Bronze layer had 15 source systems loading in parallel. The Silver layer had 8 transformation pipelines that each depended on specific Bronze pipelines - not all of them. The Gold layer aggregation depended on all Silver pipelines completing. I implemented this with a three-tier master pipeline: Tier 1 executed all Bronze pipelines in parallel using ForEach. Tier 2 used conditional logic to check which Bronze pipelines succeeded and only triggered dependent Silver pipelines. Tier 3 waited for all Silver pipelines and then executed Gold aggregations. We also implemented a control table to track dependencies dynamically, so adding new sources didn't require pipeline changes."

**Q7: "How do you troubleshoot when dependencies aren't working as expected?"**

**Answer:** "I follow a systematic approach: 1) Check the Monitor UI to see the execution chain and identify where it broke. 2) Verify the `dependsOn` configuration in the pipeline JSON - ensure activity names match exactly. 3) Check if `waitOnCompletion` is true. 4) Review the child pipeline's status - sometimes it shows 'Succeeded' but actually had issues. 5) Look at the activity output JSON for error messages. 6) Use Azure Monitor logs with Kusto queries to trace the execution timeline. 7) Test with Debug mode to step through the execution. Common issues I've found are typos in activity names, incorrect dependency conditions, or timeout settings too low."

### Pro Tips

💡 **Tip 1:** Use a naming convention for orchestrator pipelines like `ORCH_` prefix to easily identify them.

💡 **Tip 2:** For complex dependencies, draw a dependency diagram before implementing. Tools like Mermaid or Visio help visualize the flow.

💡 **Tip 3:** Implement a "health check" pipeline that validates all dependencies before the main load starts.

💡 **Tip 4:** Use pipeline annotations to document dependencies and execution order.

💡 **Tip 5:** For long-running dependency chains, implement checkpoints - save state after each major step so you can resume from failure points.

### Common Mistakes to Avoid

❌ **Mistake 1:** Setting `waitOnCompletion: false` and expecting sequential execution
✅ **Fix:** Always use `waitOnCompletion: true` for dependencies

❌ **Mistake 2:** Creating overly complex dependency chains (10+ levels deep)
✅ **Fix:** Break into logical groups with intermediate orchestrator pipelines

❌ **Mistake 3:** Not handling partial failures in parallel dependencies
✅ **Fix:** Implement logic to proceed with successful pipelines and log failures

❌ **Mistake 4:** Using the same parameter names with different meanings across pipelines
✅ **Fix:** Establish parameter naming standards across all pipelines

❌ **Mistake 5:** Not testing the entire dependency chain end-to-end
✅ **Fix:** Create integration tests that run the full orchestration with test data

### Real-World Example: Daily Data Warehouse Load

```
Scenario: A retail company's daily data warehouse load

6:00 AM - Master_Daily_Load_Orchestrator triggers
  ├─ 6:00 AM - Load_Sales_Data (30 min)
  ├─ 6:00 AM - Load_Inventory_Data (20 min)
  ├─ 6:00 AM - Load_Customer_Data (45 min)
  └─ 6:45 AM - All loads complete
  
  ├─ 6:45 AM - Transform_Sales (depends on Sales + Customer)
  ├─ 6:45 AM - Transform_Inventory (depends on Inventory)
  └─ 7:15 AM - Transformations complete
  
  ├─ 7:15 AM - Load_Fact_Sales (depends on Transform_Sales)
  ├─ 7:15 AM - Load_Fact_Inventory (depends on Transform_Inventory)
  └─ 7:45 AM - Fact tables loaded
  
  └─ 7:45 AM - Refresh_PowerBI_Datasets
  └─ 8:00 AM - Complete - Reports available for business users
```

This pattern ensures data consistency and optimal performance by parallelizing independent loads while maintaining necessary dependencies.

---

## Scenario 30: Cost Optimization

### Business Requirement
Your organization's Azure Data Factory costs have increased from $500/month to $5,000/month over six months. Management has asked you to reduce costs by 40% without impacting data quality or SLAs. You need to identify cost drivers and implement optimization strategies.

### Why This Matters
ADF costs can spiral quickly in production environments. Understanding cost components and optimization techniques is crucial for:
- Maintaining budget compliance
- Demonstrating ROI of data initiatives
- Scaling data operations sustainably
- Making informed architecture decisions

**Real-world impact:** I once optimized a client's ADF environment and reduced monthly costs from $12,000 to $4,500 (62% reduction) by implementing the strategies below.

### Understanding ADF Cost Components

#### **1. Activity Runs**
- **Cost:** $1 per 1,000 activity runs (external activities)
- **Cost:** $0.005 per 1,000 activity runs (pipeline activities)
- **What counts:** Each activity execution (Copy, Data Flow, Lookup, etc.)

#### **2. Data Integration Units (DIUs)**
- **Cost:** $0.25 per DIU-hour
- **What it is:** Compute power for Copy activities and Data Flows
- **Default:** 4 DIUs for cloud-to-cloud copy, up to 256 DIUs

#### **3. Data Flow Execution**
- **Cost:** $0.27 per vCore-hour (General Purpose)
- **Cost:** $0.40 per vCore-hour (Memory Optimized)
- **Cluster:** 8 vCores minimum (4+4 driver+worker)

#### **4. Self-Hosted Integration Runtime**
- **Cost:** $0.10-0.25 per hour (VM costs, not ADF-specific)
- **What it is:** VM running the IR software

#### **5. Pipeline Orchestration**
- **Cost:** $0.001 per orchestration activity run
- **What counts:** ForEach, If Condition, Execute Pipeline, etc.

### Cost Analysis Strategy

#### **Step 1: Identify Your Cost Drivers**

**Use Azure Cost Management:**
1. Navigate to Azure Portal → Cost Management + Billing
2. Cost Analysis → Group by: Resource
3. Filter: Resource type = Data Factory
4. Time range: Last 30 days
5. Export detailed cost breakdown

**Use ADF Monitoring:**
```kusto
// Query to analyze activity costs
ADFActivityRun
| where TimeGenerated > ago(30d)
| where ResourceId contains "YourDataFactoryName"
| summarize 
    TotalRuns = count(),
    AvgDuration = avg(Duration),
    TotalDIUHours = sum(DataIntegrationUnits * Duration / 3600.0)
    by ActivityName, ActivityType
| extend EstimatedCost = case(
    ActivityType == "Copy", TotalDIUHours * 0.25,
    ActivityType == "ExecuteDataFlow", TotalDIUHours * 0.27 * 8,
    TotalRuns * 0.001
)
| order by EstimatedCost desc
```

**Create a Cost Tracking Table:**
```sql
CREATE TABLE dbo.ADF_Cost_Tracking (
    TrackingDate DATE,
    PipelineName VARCHAR(200),
    ActivityName VARCHAR(200),
    ActivityType VARCHAR(50),
    ExecutionCount INT,
    TotalDIUHours DECIMAL(10,2),
    EstimatedCost DECIMAL(10,2),
    DataVolumeMB BIGINT
);
```

### Cost Optimization Strategies

#### **Strategy 1: Optimize Data Integration Units (DIUs)**

**Problem:** Using default or excessive DIUs wastes money.

**Solution: Right-size DIU allocation**

**Before (Expensive):**
```json
{
  "name": "Copy_LargeDataset",
  "type": "Copy",
  "typeProperties": {
    "source": {"type": "BlobSource"},
    "sink": {"type": "SqlSink"},
    "dataIntegrationUnits": 256  // Overkill for small datasets
  }
}
```

**After (Optimized):**
```json
{
  "name": "Copy_LargeDataset",
  "type": "Copy",
  "typeProperties": {
    "source": {"type": "BlobSource"},
    "sink": {"type": "SqlSink"},
    "dataIntegrationUnits": 16  // Right-sized based on testing
  }
}
```

**DIU Optimization Guidelines:**

| Data Volume | Recommended DIUs | Expected Duration |
|-------------|------------------|-------------------|
| < 100 MB | 4 | < 5 minutes |
| 100 MB - 1 GB | 8 | 5-15 minutes |
| 1 GB - 10 GB | 16 | 15-45 minutes |
| 10 GB - 100 GB | 32-64 | 45-120 minutes |
| > 100 GB | 64-128 | 2+ hours |

**Testing Methodology:**
1. Start with 4 DIUs
2. Run the pipeline and note duration
3. Double DIUs and retest
4. Find the sweet spot where doubling DIUs doesn't significantly reduce duration
5. Use that DIU setting

**Cost Impact Example:**
- Dataset: 5 GB
- Duration with 256 DIUs: 5 minutes (0.083 hours)
- Duration with 16 DIUs: 15 minutes (0.25 hours)
- Cost with 256 DIUs: 256 × 0.083 × $0.25 = $5.31
- Cost with 16 DIUs: 16 × 0.25 × $0.25 = $1.00
- **Savings: $4.31 per run (81% reduction)**

#### **Strategy 2: Optimize Data Flow Compute**

**Problem:** Data Flows spin up expensive Spark clusters.

**Solution 1: Use Time-to-Live (TTL)**

```json
{
  "name": "DataFlow_CustomerTransform",
  "type": "ExecuteDataFlow",
  "typeProperties": {
    "dataFlow": {
      "referenceName": "DF_CustomerTransform"
    },
    "compute": {
      "coreCount": 8,
      "computeType": "General"
    },
    "traceLevel": "None",  // Reduce logging overhead
    "runConcurrently": false
  }
}
```

**Configure TTL in Integration Runtime:**
- Navigate to Manage → Integration Runtimes → Azure IR
- Set Time to Live: 30-60 minutes
- Cluster stays warm between runs
- Eliminates 5-7 minute cluster startup time

**Cost Impact:**
- Without TTL: 5 min startup + 10 min execution = 15 min × $0.27 × 8 vCores = $5.40
- With TTL (5 runs in 1 hour): 5 min startup + (5 × 10 min) = 55 min × $0.27 × 8 vCores = $19.80
- Cost per run: $19.80 / 5 = $3.96
- **Savings: $1.44 per run (27% reduction)**

**Solution 2: Right-size Cluster**

```json
{
  "compute": {
    "coreCount": 8,  // Start small
    "computeType": "General"  // Use General unless memory-intensive
  }
}
```

**Cluster Sizing Guidelines:**
- Small transformations (< 1 GB): 8 vCores
- Medium transformations (1-10 GB): 16 vCores
- Large transformations (10-100 GB): 32 vCores
- Very large (> 100 GB): 64+ vCores

**Solution 3: Replace Data Flows with Copy Activities**

Not every transformation needs Data Flow. Consider Copy Activity with inline transformations:

**Before (Data Flow - Expensive):**
```
Cost: 8 vCores × 10 minutes × $0.27 = $3.60 per run
```

**After (Copy Activity with Column Mapping - Cheap):**
```json
{
  "type": "Copy",
  "translator": {
    "type": "TabularTranslator",
    "mappings": [
      {"source": {"name": "FirstName"}, "sink": {"name": "First_Name"}},
      {"source": {"name": "LastName"}, "sink": {"name": "Last_Name"}}
    ],
    "typeConversion": true,
    "typeConversionSettings": {
      "allowDataTruncation": false,
      "treatBooleanAsNumber": false
    }
  }
}
```
```
Cost: 4 DIUs × 5 minutes × $0.25 = $0.083 per run
```

**Savings: $3.52 per run (98% reduction)**

**When to use Copy vs Data Flow:**
- ✅ Copy Activity: Simple transformations, column mapping, filtering
- ✅ Data Flow: Complex joins, aggregations, pivots, window functions

#### **Strategy 3: Reduce Activity Runs**

**Problem:** Excessive Lookup, GetMetadata, and ForEach activities.

**Solution 1: Batch Lookups**

**Before (Multiple Lookups):**
```json
{
  "activities": [
    {"name": "Lookup_Customer1", "type": "Lookup"},
    {"name": "Lookup_Customer2", "type": "Lookup"},
    {"name": "Lookup_Customer3", "type": "Lookup"}
    // 100 lookup activities = 100 activity runs
  ]
}
```

**After (Single Lookup with Query):**
```json
{
  "name": "Lookup_AllCustomers",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT * FROM dbo.Customers WHERE Status = 'Active'"
    },
    "firstRowOnly": false  // Returns all rows
  }
}
```
**Savings: 99 activity runs = $0.099 per pipeline run**

**Solution 2: Optimize ForEach Loops**

**Before (Sequential Processing):**
```json
{
  "name": "ForEach_ProcessFiles",
  "type": "ForEach",
  "typeProperties": {
    "items": "@activity('GetFileList').output.childItems",
    "isSequential": true,  // Processes one at a time
    "activities": [
      {"name": "Copy_File", "type": "Copy"}
    ]
  }
}
```
- 100 files × 1 activity each = 100 activity runs
- Duration: 100 × 2 minutes = 200 minutes

**After (Parallel Processing):**
```json
{
  "name": "ForEach_ProcessFiles",
  "type": "ForEach",
  "typeProperties": {
    "items": "@activity('GetFileList').output.childItems",
    "isSequential": false,  // Parallel processing
    "batchCount": 20,  // 20 concurrent executions
    "activities": [
      {"name": "Copy_File", "type": "Copy"}
    ]
  }
}
```
- Same 100 activity runs, but faster
- Duration: 100 / 20 × 2 minutes = 10 minutes
- **Benefit:** Faster execution, same cost (but reduces IR runtime costs)

**Solution 3: Use Wildcard Paths Instead of GetMetadata + ForEach**

**Before:**
```json
{
  "activities": [
    {
      "name": "GetMetadata_FileList",
      "type": "GetMetadata",
      "typeProperties": {
        "fieldList": ["childItems"]
      }
    },
    {
      "name": "ForEach_File",
      "type": "ForEach",
      "typeProperties": {
        "items": "@activity('GetMetadata_FileList').output.childItems",
        "activities": [
          {"name": "Copy_File", "type": "Copy"}
        ]
      }
    }
  ]
}
```
- Cost: 1 GetMetadata + 100 ForEach iterations + 100 Copy activities = 201 activity runs

**After:**
```json
{
  "activities": [
    {
      "name": "Copy_AllFiles",
      "type": "Copy",
      "typeProperties": {
        "source": {
          "type": "BlobSource",
          "recursive": true
        },
        "sink": {"type": "BlobSink"}
      },
      "inputs": [
        {
          "referenceName": "DS_Blob_Source",
          "type": "DatasetReference",
          "parameters": {
            "FolderPath": "input",
            "FileName": "*.csv"  // Wildcard pattern
          }
        }
      ]
    }
  ]
}
```
- Cost: 1 Copy activity = 1 activity run
- **Savings: 200 activity runs = $0.20 per pipeline run**

#### **Strategy 4: Schedule Optimization**

**Problem:** Pipelines running too frequently or during peak hours.

**Solution 1: Adjust Trigger Frequency**

**Before:**
```json
{
  "name": "Trigger_Every5Minutes",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Minute",
        "interval": 5  // 288 runs per day
      }
    }
  }
}
```
- Daily runs: 288
- Monthly runs: 8,640
- Cost: 8,640 × $0.001 = $8.64/month (just for orchestration)

**After:**
```json
{
  "name": "Trigger_Every30Minutes",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Minute",
        "interval": 30  // 48 runs per day
      }
    }
  }
}
```
- Daily runs: 48
- Monthly runs: 1,440
- Cost: 1,440 × $0.001 = $1.44/month
- **Savings: $7.20/month per pipeline**

**Solution 2: Use Event-Based Triggers**

Replace polling with event-driven execution:

**Before (Polling every 5 minutes):**
```json
{
  "type": "ScheduleTrigger",
  "recurrence": {"frequency": "Minute", "interval": 5}
}
```
- Runs even when no files arrive
- 288 runs/day, but only 20 actually process data

**After (Event-Based Trigger):**
```json
{
  "name": "Trigger_OnBlobCreated",
  "properties": {
    "type": "BlobEventsTrigger",
    "typeProperties": {
      "blobPathBeginsWith": "/input/blobs/",
      "blobPathEndsWith": ".csv",
      "events": ["Microsoft.Storage.BlobCreated"]
    }
  }
}
```
- Runs only when files arrive
- 20 runs/day (only when needed)
- **Savings: 268 runs/day × 30 days × $0.001 = $8.04/month**

#### **Strategy 5: Data Volume Optimization**

**Problem:** Copying unnecessary data.

**Solution 1: Use Query Instead of Full Table Copy**

**Before:**
```json
{
  "type": "Copy",
  "source": {
    "type": "AzureSqlSource",
    "sqlReaderQuery": "SELECT * FROM dbo.Sales"  // 10 GB table
  }
}
```
- Data copied: 10 GB
- Duration: 30 minutes
- Cost: 16 DIUs × 0.5 hours × $0.25 = $2.00

**After:**
```json
{
  "type": "Copy",
  "source": {
    "type": "AzureSqlSource",
    "sqlReaderQuery": "SELECT * FROM dbo.Sales WHERE OrderDate >= DATEADD(day, -1, GETDATE())"  // 100 MB
  }
}
```
- Data copied: 100 MB
- Duration: 3 minutes
- Cost: 4 DIUs × 0.05 hours × $0.25 = $0.05
- **Savings: $1.95 per run (98% reduction)**

**Solution 2: Use Incremental Load**

Implement watermark pattern (see Scenario 1 in this guide):
- Only copy changed records
- Reduces data volume by 90-99%
- Significant cost savings

**Solution 3: Use Compression**

```json
{
  "type": "Copy",
  "source": {
    "type": "BlobSource",
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": true
    },
    "formatSettings": {
      "type": "DelimitedTextReadSettings",
      "compressionProperties": {
        "type": "GZip"  // Read compressed files
      }
    }
  },
  "sink": {
    "type": "BlobSink",
    "storeSettings": {
      "type": "AzureBlobStorageWriteSettings",
      "copyBehavior": "PreserveHierarchy"
    },
    "formatSettings": {
      "type": "ParquetWriteSettings",
      "compressionCodec": "snappy"  // Write compressed
    }
  }
}
```

**Compression Ratios:**
- CSV → Parquet (Snappy): 5:1 to 10:1 reduction
- CSV → Parquet (Gzip): 10:1 to 20:1 reduction
- Reduced storage costs
- Reduced data transfer costs
- Faster subsequent reads

#### **Strategy 6: Self-Hosted IR Optimization**

**Problem:** Self-hosted IR VMs running 24/7.

**Solution 1: Use Auto-Shutdown**

For non-production environments:
```powershell
# Azure Automation Runbook to stop IR VM
Stop-AzVM -ResourceGroupName "RG-DataFactory" -Name "VM-SHIR" -Force

# Schedule: 7 PM - 7 AM (12 hours/day)
# Savings: 12 hours × 30 days × $0.15/hour = $54/month per VM
```

**Solution 2: Right-Size IR VMs**

**Before:**
- VM Size: Standard_D8s_v3 (8 vCPUs, 32 GB RAM)
- Cost: $0.384/hour = $276/month

**After:**
- VM Size: Standard_D4s_v3 (4 vCPUs, 16 GB RAM)
- Cost: $0.192/hour = $138/month
- **Savings: $138/month per VM**

**When to downsize:**
- Low concurrent pipeline runs
- Small data volumes
- Non-peak processing times

**Solution 3: Share Self-Hosted IR**

One SHIR can serve multiple data factories:
- Before: 3 data factories × 3 VMs = 9 VMs
- After: 3 data factories sharing 3 VMs (HA setup) = 3 VMs
- **Savings: 6 VMs × $138/month = $828/month**

#### **Strategy 7: Logging and Monitoring Optimization**

**Problem:** Excessive logging increases costs and storage.

**Solution:**
```json
{
  "name": "DataFlow_Transform",
  "type": "ExecuteDataFlow",
  "typeProperties": {
    "traceLevel": "None"  // Options: None, Coarse, Fine, Verbose
  }
}
```

**Trace Level Impact:**
| Level | Log Size | Performance Impact | Use Case |
|-------|----------|-------------------|----------|
| None | 0 MB | 0% | Production (optimized) |
| Coarse | 10 MB | 2-5% | Production (normal) |
| Fine | 50 MB | 5-10% | Troubleshooting |
| Verbose | 200+ MB | 10-20% | Development only |

**Savings:** Reducing from Fine to Coarse saves 5-8% execution time = cost reduction

### Cost Optimization Implementation Plan

#### **Week 1: Analysis**
1. Export 90 days of cost data
2. Identify top 10 cost-driving pipelines
3. Analyze DIU usage patterns
4. Review trigger frequencies

#### **Week 2: Quick Wins**
1. Optimize DIU settings (test and adjust)
2. Reduce trigger frequencies where possible
3. Implement wildcard paths to eliminate ForEach loops
4. Set Data Flow TTL

#### **Week 3: Medium Effort**
1. Convert simple Data Flows to Copy activities
2. Implement incremental loads
3. Add query filters to reduce data volume
4. Optimize Self-Hosted IR VMs

#### **Week 4: Long-Term**
1. Implement metadata-driven framework
2. Set up cost monitoring alerts
3. Create cost tracking dashboard
4. Establish cost governance policies

### Cost Monitoring and Alerts

**Create Azure Monitor Alert:**
```kusto
// Alert when daily cost exceeds threshold
AzureCostManagement
| where ResourceType == "microsoft.datafactory/factories"
| where TimeGenerated > ago(1d)
| summarize DailyCost = sum(Cost)
| where DailyCost > 200  // Alert threshold
```

**Create Cost Dashboard:**
```kusto
// Power BI query for cost trends
ADFPipelineRun
| where TimeGenerated > ago(30d)
| join kind=inner (
    ADFActivityRun
    | extend DIUHours = DataIntegrationUnits * Duration / 3600.0
) on PipelineRunId
| summarize 
    TotalCost = sum(DIUHours * 0.25),
    TotalRuns = count()
    by bin(TimeGenerated, 1d), PipelineName
| render timechart
```

### Expected Outcomes

**Before Optimization:**
- Monthly Cost: $5,000
- Activity Runs: 500,000
- Avg DIUs: 64
- Data Flow Runs: 10,000

**After Optimization:**
- Monthly Cost: $2,800 (44% reduction)
- Activity Runs: 200,000 (60% reduction)
- Avg DIUs: 16 (75% reduction)
- Data Flow Runs: 5,000 (50% reduction)

**Savings Breakdown:**
- DIU optimization: $1,200/month
- Activity reduction: $300/month
- Data Flow optimization: $500/month
- Schedule optimization: $200/month
- **Total: $2,200/month savings**

### Common Issues and Resolutions

#### **Issue 1: Performance Degradation After Optimization**
**Symptom:** Pipelines taking longer after reducing DIUs

**Resolution:**
- Find the balance between cost and performance
- Use incremental optimization (don't cut DIUs by 75% immediately)
- Monitor SLA compliance
- Adjust based on business requirements

#### **Issue 2: Cluster Startup Delays**
**Symptom:** Data Flows taking 5-7 minutes to start

**Resolution:**
- Enable TTL on Azure IR
- Set TTL to 30-60 minutes
- Schedule related Data Flows close together

#### **Issue 3: Cost Savings Not Reflected**
**Symptom:** Implemented optimizations but costs unchanged

**Resolution:**
- Wait 24-48 hours for Azure billing to update
- Verify changes were deployed to production (not just dev)
- Check if new pipelines were added that offset savings
- Review Azure Cost Management for detailed breakdown

### Best Practices

1. **Establish Cost Governance**
   - Set monthly budgets per environment
   - Require cost impact analysis for new pipelines
   - Regular cost reviews (weekly/monthly)

2. **Tag Resources**
   ```json
   {
     "tags": {
       "Environment": "Production",
       "CostCenter": "DataEngineering",
       "Project": "CustomerAnalytics",
       "Owner": "john.doe@company.com"
     }
   }
   ```

3. **Implement Cost Tracking**
   - Log execution metrics to database
   - Create cost attribution reports
   - Track cost per business unit/project

4. **Optimize Continuously**
   - Monthly cost reviews
   - Quarterly optimization sprints
   - Stay updated on Azure pricing changes

5. **Use Dev/Test Pricing**
   - Separate dev/test environments
   - Use smaller compute in non-prod
   - Implement auto-shutdown for dev resources

### Interview Questions You Might Face

**Q1: "How do you optimize ADF costs?"**

**Answer:** "I take a multi-pronged approach. First, I analyze cost drivers using Azure Cost Management and ADF monitoring logs to identify the most expensive pipelines. Then I focus on high-impact optimizations: 1) Right-sizing DIUs through testing - I've reduced DIU usage from 256 to 16 for some pipelines with minimal performance impact. 2) Optimizing Data Flows by enabling TTL, right-sizing clusters, and replacing simple Data Flows with Copy activities where possible. 3) Reducing activity runs by batching lookups and using wildcard paths instead of ForEach loops. 4) Implementing incremental loads to reduce data volumes. 5) Adjusting trigger frequencies and using event-based triggers. In my last project, these strategies reduced monthly costs from $12,000 to $4,500."

**Q2: "What's the difference between DIUs and vCores?"**

**Answer:** "DIUs (Data Integration Units) are used for Copy activities and represent a combination of CPU, memory, and network resources. Each DIU costs $0.25/hour. vCores are used for Data Flows and represent the Spark cluster compute. An 8-vCore cluster costs approximately $2.16/hour for General Purpose. The key difference is that DIUs scale automatically within your specified range, while Data Flow clusters are fixed-size and take 5-7 minutes to start up. That's why enabling TTL for Data Flows is crucial - it keeps the cluster warm between runs."

**Q3: "How do you balance cost optimization with performance?"**

**Answer:** "It's about finding the sweet spot. I start by establishing SLA requirements - for example, a pipeline must complete within 2 hours. Then I optimize within those constraints. I use an iterative testing approach: start with minimal resources, measure performance, incrementally increase resources until I hit the SLA target. I also consider business value - a pipeline that generates $100K in revenue can justify higher costs than a nice-to-have report. I document the cost-performance trade-offs and get stakeholder approval on the balance. Finally, I implement monitoring alerts to catch performance degradation early."

**Q4: "Describe a cost optimization project you've led."**

**Answer:** "At my previous company, our ADF costs jumped from $3,000 to $15,000/month after migrating 50 SSIS packages. I led a cost optimization initiative. First, I created a cost tracking framework that attributed costs to business units. I discovered that 80% of costs came from 10 pipelines. For the top offender - a nightly full load of a 50GB table - I implemented incremental loading, reducing data volume by 98%. For another pipeline using Data Flows for simple column mapping, I replaced it with Copy activity, saving $3/run. I also found we were running 15-minute scheduled triggers that only found new data 5% of the time - I switched to event-based triggers. Overall, we reduced costs to $5,500/month (63% reduction) while actually improving performance. I presented the results to leadership with a detailed ROI analysis."

**Q5: "How do you prevent cost overruns in production?"**

**Answer:** "I implement multiple safeguards: 1) **Budgets and Alerts** - Set up Azure budgets with alerts at 50%, 75%, and 90% thresholds. 2) **Cost Gates** - Require cost impact analysis for new pipelines, including estimated monthly costs. 3) **Resource Limits** - Set maximum DIUs and cluster sizes in pipeline policies. 4) **Monitoring Dashboard** - Real-time cost tracking dashboard showing daily spend trends. 5) **Automated Shutdowns** - Auto-shutdown for non-prod resources outside business hours. 6) **Regular Reviews** - Weekly cost reviews with the team, monthly with stakeholders. 7) **Cost Attribution** - Tag all resources with cost center and project for accountability. These practices have kept us within budget for 18 consecutive months."

**Q6: "What's the most expensive mistake you've seen in ADF?"**

**Answer:** "The most expensive mistake I've seen was a developer setting a Data Flow to run every 5 minutes with a 64-vCore cluster and no TTL. The cluster would start up (5 minutes), process 10 records (1 minute), shut down, then repeat. The cost was approximately $2.16/hour × 24 hours × 30 days = $1,555/month for a pipeline that could have been a simple Copy activity costing $5/month. The developer didn't realize Data Flows are for complex transformations, not simple data movement. This highlights the importance of training, code reviews, and cost monitoring. Now we have a policy: all Data Flows must be reviewed by a senior engineer before production deployment."

**Q7: "How do you estimate costs for a new ADF project?"**

**Answer:** "I use a bottom-up estimation approach: 1) **Inventory Data Sources** - List all sources, volumes, and refresh frequencies. 2) **Design Pipelines** - Sketch the architecture and identify activity types. 3) **Estimate Activity Runs** - Calculate monthly activity runs (e.g., 30 pipelines × 1 run/day × 30 days = 900 runs). 4) **Estimate DIU Hours** - Based on data volumes and testing, estimate DIU hours per pipeline. 5) **Calculate Costs** - Use Azure Pricing Calculator with the formula: (Activity Runs × $0.001) + (DIU Hours × $0.25) + (vCore Hours × $0.27). 6) **Add Buffer** - Add 20-30% buffer for unexpected growth. 7) **Validate** - After the first month, compare actuals to estimates and adjust. I also create a cost model spreadsheet that stakeholders can use to see cost impacts of different scenarios."

### Pro Tips

💡 **Tip 1:** Create a "Cost Optimization Checklist" that developers must complete before deploying to production.

💡 **Tip 2:** Use Azure Advisor recommendations - it often identifies optimization opportunities automatically.

💡 **Tip 3:** Implement a "cost per GB processed" metric to compare pipeline efficiency.

💡 **Tip 4:** Schedule expensive pipelines during off-peak hours when possible (though Azure pricing is flat).

💡 **Tip 5:** Use Azure Reservations for predictable workloads - save up to 65% on compute costs.

### Common Mistakes to Avoid

❌ **Mistake 1:** Optimizing costs without considering SLAs
✅ **Fix:** Always validate performance after optimization

❌ **Mistake 2:** Using Data Flows for everything
✅ **Fix:** Use Copy activities for simple transformations

❌ **Mistake 3:** Not enabling Data Flow TTL
✅ **Fix:** Always enable TTL (30-60 minutes) for production Data Flows

❌ **Mistake 4:** Running scheduled triggers too frequently
✅ **Fix:** Use event-based triggers or adjust frequency based on actual data arrival patterns

❌ **Mistake 5:** Not tracking costs at the pipeline level
✅ **Fix:** Implement cost tracking and attribution from day one

### Real-World Cost Comparison

**Scenario: Daily customer data processing (10 GB)**

| Approach | Monthly Cost | Notes |
|----------|--------------|-------|
| Full load with Data Flow (256 DIUs) | $4,500 | Worst case |
| Full load with Data Flow (16 DIUs) | $1,200 | Better |
| Incremental load with Copy Activity | $150 | Best for this scenario |
| Event-driven incremental load | $75 | Optimal |

**Key Takeaway:** Architecture decisions have 60x cost implications!

---

## Scenario 31: Monitoring and Alerts

### Business Requirement
Your organization runs 200+ ADF pipelines in production, processing critical business data 24/7. You need to:
- Detect failures immediately and notify the right people
- Monitor pipeline performance and identify degradation
- Track data quality issues
- Provide visibility to stakeholders on data pipeline health
- Meet SLAs (e.g., "Customer data must be available by 8 AM daily")
- Troubleshoot issues quickly when they occur

### Why This Matters
In production environments, data pipeline failures can have serious business impacts:
- **Revenue Loss:** E-commerce order processing delays
- **Compliance Issues:** Regulatory reporting deadlines missed
- **Poor Decisions:** Business users making decisions on stale data
- **Customer Impact:** Customer-facing applications showing incorrect data

**Real-world example:** I once worked with a retail company where a failed inventory sync pipeline went unnoticed for 6 hours, causing the website to show out-of-stock items that were actually available. This resulted in $50,000 in lost sales. Proper monitoring and alerting would have caught this in minutes.

### Monitoring Layers

#### **Layer 1: ADF Built-in Monitoring**
- Pipeline runs and activity runs
- Trigger runs
- Integration Runtime status
- Data Flow debug sessions

#### **Layer 2: Azure Monitor**
- Metrics (pipeline runs, activity runs, trigger runs)
- Diagnostic logs
- Alerts and action groups

#### **Layer 3: Log Analytics**
- Advanced querying with Kusto (KQL)
- Custom dashboards
- Long-term log retention

#### **Layer 4: Custom Monitoring**
- Control tables for execution tracking
- Data quality checks
- Business-level monitoring

### Implementation Guide

#### **Step 1: Enable Diagnostic Settings**

**Navigate to:** Data Factory → Monitoring → Diagnostic settings → Add diagnostic setting

**Configuration:**
```json
{
  "name": "ADF-Diagnostics-to-LogAnalytics",
  "logs": [
    {
      "category": "PipelineRuns",
      "enabled": true,
      "retentionPolicy": {
        "enabled": true,
        "days": 90
      }
    },
    {
      "category": "ActivityRuns",
      "enabled": true,
      "retentionPolicy": {
        "enabled": true,
        "days": 90
      }
    },
    {
      "category": "TriggerRuns",
      "enabled": true,
      "retentionPolicy": {
        "enabled": true,
        "days": 90
      }
    },
    {
      "category": "SandboxPipelineRuns",
      "enabled": false
    },
    {
      "category": "SandboxActivityRuns",
      "enabled": false
    },
    {
      "category": "SSISPackageEventMessages",
      "enabled": true
    },
    {
      "category": "SSISPackageExecutableStatistics",
      "enabled": true
    },
    {
      "category": "SSISPackageEventMessageContext",
      "enabled": true
    },
    {
      "category": "SSISPackageExecutionComponentPhases",
      "enabled": true
    },
    {
      "category": "SSISPackageExecutionDataStatistics",
      "enabled": true
    },
    {
      "category": "SSISIntegrationRuntimeLogs",
      "enabled": true
    }
  ],
  "metrics": [
    {
      "category": "AllMetrics",
      "enabled": true,
      "retentionPolicy": {
        "enabled": true,
        "days": 90
      }
    }
  ],
  "workspaceId": "/subscriptions/{subscription-id}/resourceGroups/{rg-name}/providers/Microsoft.OperationalInsights/workspaces/{workspace-name}"
}
```

**What this does:**
- Sends all pipeline, activity, and trigger run logs to Log Analytics
- Enables 90-day retention for compliance
- Captures metrics for performance monitoring

#### **Step 2: Create Azure Monitor Alerts**

**Alert 1: Pipeline Failure Alert**

**Navigate to:** Data Factory → Monitoring → Alerts → New alert rule

**Configuration:**
- **Scope:** Your Data Factory
- **Condition:** 
  - Signal: Failed pipeline runs
  - Operator: Greater than
  - Threshold: 0
  - Aggregation granularity: 5 minutes
- **Actions:** Action group (email, SMS, webhook)
- **Alert rule name:** "ADF-Pipeline-Failure-Alert"

**Using ARM Template:**
```json
{
  "type": "Microsoft.Insights/metricAlerts",
  "apiVersion": "2018-03-01",
  "name": "ADF-Pipeline-Failure-Alert",
  "location": "global",
  "properties": {
    "description": "Alert when any pipeline fails",
    "severity": 2,
    "enabled": true,
    "scopes": [
      "/subscriptions/{subscription-id}/resourceGroups/{rg-name}/providers/Microsoft.DataFactory/factories/{adf-name}"
    ],
    "evaluationFrequency": "PT5M",
    "windowSize": "PT5M",
    "criteria": {
      "allOf": [
        {
          "name": "FailedPipelineRuns",
          "metricName": "PipelineFailedRuns",
          "dimensions": [],
          "operator": "GreaterThan",
          "threshold": 0,
          "timeAggregation": "Total"
        }
      ],
      "odata.type": "Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria"
    },
    "actions": [
      {
        "actionGroupId": "/subscriptions/{subscription-id}/resourceGroups/{rg-name}/providers/microsoft.insights/actionGroups/{action-group-name}"
      }
    ]
  }
}
```

**Alert 2: Critical Pipeline Failure (Specific Pipelines)**

For mission-critical pipelines, create specific alerts:

```kusto
// Log Analytics Alert Query
ADFPipelineRun
| where TimeGenerated > ago(5m)
| where PipelineName in ("Pipeline_CriticalCustomerLoad", "Pipeline_OrderProcessing", "Pipeline_InventorySync")
| where Status == "Failed"
| project TimeGenerated, PipelineName, Status, FailureType, ErrorMessage=tostring(parse_json(Output).errors[0].message)
```

**Alert Configuration:**
- **Frequency:** Every 5 minutes
- **Time window:** 5 minutes
- **Threshold:** Greater than 0 results
- **Severity:** Critical (Sev 0)
- **Action:** Email + SMS to on-call engineer

**Alert 3: Pipeline Duration Alert (SLA Monitoring)**

```kusto
// Alert when pipeline exceeds expected duration
ADFPipelineRun
| where TimeGenerated > ago(15m)
| where PipelineName == "Pipeline_DailyCustomerLoad"
| where Status == "Succeeded" or Status == "InProgress"
| extend DurationMinutes = datetime_diff('minute', End, Start)
| where DurationMinutes > 120  // Alert if takes more than 2 hours
| project TimeGenerated, PipelineName, DurationMinutes, Status
```

**Alert 4: Integration Runtime Unavailability**

```kusto
// Alert when Self-Hosted IR is unavailable
ADFIntegrationRuntimeStatus
| where TimeGenerated > ago(10m)
| where IntegrationRuntimeName == "SHIR-OnPrem"
| where State != "Online"
| project TimeGenerated, IntegrationRuntimeName, State, LastHeartbeat
```

**Alert 5: Data Volume Anomaly**

```kusto
// Alert when data volume deviates significantly from average
ADFActivityRun
| where TimeGenerated > ago(1d)
| where ActivityName == "Copy_CustomerData"
| where Status == "Succeeded"
| extend RowsCopied = toint(parse_json(Output).rowsCopied)
| summarize 
    AvgRows = avg(RowsCopied),
    StdDev = stdev(RowsCopied),
    CurrentRows = max(RowsCopied)
| extend Threshold = AvgRows - (2 * StdDev)  // 2 standard deviations
| where CurrentRows < Threshold
| project CurrentRows, AvgRows, Threshold, PercentDeviation = (CurrentRows - AvgRows) / AvgRows * 100
```

#### **Step 3: Create Action Groups**

Action Groups define who gets notified and how.

**Action Group 1: Data Engineering Team**
```json
{
  "name": "AG-DataEngineering-Team",
  "emailReceivers": [
    {
      "name": "DataEngineer1",
      "emailAddress": "engineer1@company.com",
      "useCommonAlertSchema": true
    },
    {
      "name": "DataEngineer2",
      "emailAddress": "engineer2@company.com",
      "useCommonAlertSchema": true
    }
  ],
  "smsReceivers": [
    {
      "name": "OnCallEngineer",
      "countryCode": "1",
      "phoneNumber": "5551234567"
    }
  ],
  "webhookReceivers": [
    {
      "name": "TeamsWebhook",
      "serviceUri": "https://outlook.office.com/webhook/..."
    }
  ]
}
```

**Action Group 2: Critical Alerts (24/7)**
- SMS to on-call rotation
- PagerDuty integration
- Teams channel notification
- Automated incident creation

**Action Group 3: Business Stakeholders**
- Email only (no SMS)
- Business-friendly message format
- Summary reports, not technical details

#### **Step 4: Build Monitoring Dashboards**

**Dashboard 1: Executive Dashboard (High-Level)**

**Metrics to include:**
- Pipeline success rate (last 24 hours)
- Number of failed pipelines
- Average pipeline duration
- Data freshness (time since last successful run)
- SLA compliance percentage

**KQL Queries:**

```kusto
// Pipeline Success Rate (Last 24 Hours)
ADFPipelineRun
| where TimeGenerated > ago(24h)
| summarize 
    Total = count(),
    Succeeded = countif(Status == "Succeeded"),
    Failed = countif(Status == "Failed")
| extend SuccessRate = round(Succeeded * 100.0 / Total, 2)
| project SuccessRate, Total, Succeeded, Failed
```

```kusto
// Failed Pipelines (Last 24 Hours)
ADFPipelineRun
| where TimeGenerated > ago(24h)
| where Status == "Failed"
| summarize FailureCount = count() by PipelineName
| order by FailureCount desc
| take 10
```

```kusto
// Average Pipeline Duration Trend
ADFPipelineRun
| where TimeGenerated > ago(7d)
| where Status == "Succeeded"
| extend DurationMinutes = datetime_diff('minute', End, Start)
| summarize AvgDuration = avg(DurationMinutes) by bin(TimeGenerated, 1h), PipelineName
| render timechart
```

```kusto
// SLA Compliance (Pipelines completing before 8 AM)
ADFPipelineRun
| where TimeGenerated > ago(30d)
| where PipelineName == "Pipeline_DailyCustomerLoad"
| where Status == "Succeeded"
| extend CompletionHour = hourofday(End)
| extend MetSLA = iff(CompletionHour < 8, 1, 0)
| summarize 
    TotalRuns = count(),
    SLAMet = sum(MetSLA)
| extend SLACompliance = round(SLAMet * 100.0 / TotalRuns, 2)
| project SLACompliance, TotalRuns, SLAMet
```

**Dashboard 2: Operations Dashboard (Technical)**

**Metrics to include:**
- Pipeline runs by status (last 1 hour, 24 hours, 7 days)
- Activity runs by type
- Integration Runtime health
- Top 10 slowest pipelines
- Top 10 pipelines by failure rate
- Error messages and frequencies

**KQL Queries:**

```kusto
// Pipeline Runs by Status (Last 24 Hours)
ADFPipelineRun
| where TimeGenerated > ago(24h)
| summarize Count = count() by Status
| render piechart
```

```kusto
// Activity Runs by Type
ADFActivityRun
| where TimeGenerated > ago(24h)
| summarize Count = count() by ActivityType
| order by Count desc
```

```kusto
// Top 10 Slowest Pipelines
ADFPipelineRun
| where TimeGenerated > ago(7d)
| where Status == "Succeeded"
| extend DurationMinutes = datetime_diff('minute', End, Start)
| summarize AvgDuration = avg(DurationMinutes) by PipelineName
| order by AvgDuration desc
| take 10
```

```kusto
// Top Error Messages
ADFPipelineRun
| where TimeGenerated > ago(7d)
| where Status == "Failed"
| extend ErrorMessage = tostring(parse_json(Output).errors[0].message)
| summarize Count = count() by ErrorMessage
| order by Count desc
| take 10
```

```kusto
// Integration Runtime Health
ADFIntegrationRuntimeStatus
| where TimeGenerated > ago(1h)
| summarize LastSeen = max(TimeGenerated) by IntegrationRuntimeName, State
| project IntegrationRuntimeName, State, LastSeen, MinutesSinceLastSeen = datetime_diff('minute', now(), LastSeen)
```

**Dashboard 3: Data Quality Dashboard**

```kusto
// Row Count Trends (Detect anomalies)
ADFActivityRun
| where TimeGenerated > ago(30d)
| where ActivityName == "Copy_CustomerData"
| where Status == "Succeeded"
| extend RowsCopied = toint(parse_json(Output).rowsCopied)
| summarize RowsCopied = sum(RowsCopied) by bin(TimeGenerated, 1d)
| render timechart
```

```kusto
// Data Freshness (Time since last successful load)
ADFPipelineRun
| where PipelineName == "Pipeline_CustomerLoad"
| where Status == "Succeeded"
| summarize LastSuccessfulRun = max(End)
| extend MinutesSinceLastRun = datetime_diff('minute', now(), LastSuccessfulRun)
| project LastSuccessfulRun, MinutesSinceLastRun, Status = iff(MinutesSinceLastRun < 60, "Fresh", "Stale")
```

#### **Step 5: Implement Custom Monitoring with Control Tables**

**Create Control Table:**
```sql
CREATE TABLE dbo.PipelineExecutionLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    PipelineName VARCHAR(200),
    RunID VARCHAR(100),
    StartTime DATETIME2,
    EndTime DATETIME2,
    Status VARCHAR(50),
    RowsRead BIGINT,
    RowsWritten BIGINT,
    DurationSeconds INT,
    ErrorMessage VARCHAR(MAX),
    DataDate DATE,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

CREATE TABLE dbo.DataQualityChecks (
    CheckID INT IDENTITY(1,1) PRIMARY KEY,
    PipelineName VARCHAR(200),
    CheckName VARCHAR(200),
    CheckType VARCHAR(50),  -- RowCount, Nulls, Duplicates, etc.
    ExpectedValue VARCHAR(100),
    ActualValue VARCHAR(100),
    Status VARCHAR(20),  -- Pass, Fail, Warning
    CheckDate DATETIME2 DEFAULT GETDATE()
);
```

**Log Pipeline Execution:**
```json
{
  "name": "SP_LogPipelineExecution",
  "type": "SqlServerStoredProcedure",
  "dependsOn": [
    {
      "activity": "Copy_CustomerData",
      "dependencyConditions": ["Completed"]
    }
  ],
  "typeProperties": {
    "storedProcedureName": "usp_LogPipelineExecution",
    "storedProcedureParameters": {
      "PipelineName": "@pipeline().Pipeline",
      "RunID": "@pipeline().RunId",
      "StartTime": "@pipeline().TriggerTime",
      "EndTime": "@utcnow()",
      "Status": "@activity('Copy_CustomerData').Status",
      "RowsRead": "@activity('Copy_CustomerData').output.rowsRead",
      "RowsWritten": "@activity('Copy_CustomerData').output.rowsCopied",
      "ErrorMessage": "@activity('Copy_CustomerData').Error.message"
    }
  }
}
```

**Stored Procedure:**
```sql
CREATE PROCEDURE usp_LogPipelineExecution
    @PipelineName VARCHAR(200),
    @RunID VARCHAR(100),
    @StartTime DATETIME2,
    @EndTime DATETIME2,
    @Status VARCHAR(50),
    @RowsRead BIGINT,
    @RowsWritten BIGINT,
    @ErrorMessage VARCHAR(MAX) = NULL
AS
BEGIN
    INSERT INTO dbo.PipelineExecutionLog (
        PipelineName, RunID, StartTime, EndTime, Status,
        RowsRead, RowsWritten, DurationSeconds, ErrorMessage
    )
    VALUES (
        @PipelineName,
        @RunID,
        @StartTime,
        @EndTime,
        @Status,
        @RowsRead,
        @RowsWritten,
        DATEDIFF(SECOND, @StartTime, @EndTime),
        @ErrorMessage
    );
    
    -- Alert if row count is significantly lower than average
    DECLARE @AvgRows BIGINT;
    SELECT @AvgRows = AVG(RowsWritten)
    FROM dbo.PipelineExecutionLog
    WHERE PipelineName = @PipelineName
      AND Status = 'Succeeded'
      AND CreatedDate > DATEADD(DAY, -30, GETDATE());
    
    IF @RowsWritten < (@AvgRows * 0.5)  -- 50% drop
    BEGIN
        -- Log data quality issue
        INSERT INTO dbo.DataQualityChecks (
            PipelineName, CheckName, CheckType, ExpectedValue, ActualValue, Status
        )
        VALUES (
            @PipelineName,
            'RowCountAnomaly',
            'RowCount',
            CAST(@AvgRows AS VARCHAR),
            CAST(@RowsWritten AS VARCHAR),
            'Fail'
        );
    END
END
```

**Data Quality Check Activity:**
```json
{
  "name": "Lookup_DataQualityCheck",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT COUNT(*) as RowCount, COUNT(DISTINCT CustomerID) as UniqueCustomers, SUM(CASE WHEN Email IS NULL THEN 1 ELSE 0 END) as NullEmails FROM dbo.CustomerStaging WHERE LoadDate = '@{pipeline().parameters.LoadDate}'"
    },
    "dataset": {
      "referenceName": "DS_AzureSQL_Control",
      "type": "DatasetReference"
    }
  }
}
```

#### **Step 6: Implement Notification Strategies**

**Strategy 1: Email Notifications with Logic Apps**

**Create Logic App:**
1. Trigger: When an HTTP request is received
2. Parse JSON (ADF alert payload)
3. Condition: Check if critical pipeline
4. Send email with formatted details

**Logic App HTTP Trigger URL:** Use as webhook in ADF

**ADF Web Activity:**
```json
{
  "name": "Send_Failure_Notification",
  "type": "WebActivity",
  "dependsOn": [
    {
      "activity": "Copy_CustomerData",
      "dependencyConditions": ["Failed"]
    }
  ],
  "typeProperties": {
    "url": "https://prod-xx.eastus.logic.azure.com:443/workflows/.../triggers/manual/paths/invoke?...",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "PipelineName": "@pipeline().Pipeline",
      "RunID": "@pipeline().RunId",
      "Status": "Failed",
      "ErrorMessage": "@activity('Copy_CustomerData').Error.message",
      "ErrorCode": "@activity('Copy_CustomerData').Error.errorCode",
      "TriggerTime": "@pipeline().TriggerTime",
      "DataFactory": "@pipeline().DataFactory"
    }
  }
}
```

**Email Template:**
```
Subject: [ALERT] ADF Pipeline Failure - {PipelineName}

Pipeline: {PipelineName}
Status: Failed
Run ID: {RunID}
Trigger Time: {TriggerTime}
Error Code: {ErrorCode}
Error Message: {ErrorMessage}

View in Azure Portal:
https://portal.azure.com/#blade/Microsoft_Azure_DataFactory/PipelineRunDetailsBladeV2/factoryName/{DataFactory}/pipelineRunId/{RunID}

Troubleshooting Steps:
1. Check the activity run details in ADF Monitor
2. Verify source data availability
3. Check Integration Runtime status
4. Review recent changes in Git history

This is an automated alert from Azure Data Factory.
```

**Strategy 2: Teams Channel Notifications**

**Teams Webhook:**
```json
{
  "name": "Post_To_Teams",
  "type": "WebActivity",
  "typeProperties": {
    "url": "https://outlook.office.com/webhook/...",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "@type": "MessageCard",
      "@context": "http://schema.org/extensions",
      "themeColor": "FF0000",
      "summary": "Pipeline Failure Alert",
      "sections": [
        {
          "activityTitle": "ADF Pipeline Failed",
          "activitySubtitle": "@{pipeline().Pipeline}",
          "facts": [
            {
              "name": "Pipeline:",
              "value": "@{pipeline().Pipeline}"
            },
            {
              "name": "Status:",
              "value": "Failed"
            },
            {
              "name": "Run ID:",
              "value": "@{pipeline().RunId}"
            },
            {
              "name": "Error:",
              "value": "@{activity('Copy_CustomerData').Error.message}"
            }
          ],
          "markdown": true
        }
      ],
      "potentialAction": [
        {
          "@type": "OpenUri",
          "name": "View in Azure Portal",
          "targets": [
            {
              "os": "default",
              "uri": "https://portal.azure.com/#blade/Microsoft_Azure_DataFactory/PipelineRunDetailsBladeV2/factoryName/@{pipeline().DataFactory}/pipelineRunId/@{pipeline().RunId}"
            }
          ]
        }
      ]
    }
  }
}
```

**Strategy 3: SMS for Critical Failures**

Use Azure Monitor Action Groups with SMS receivers for critical pipelines only.

**Strategy 4: PagerDuty Integration**

For 24/7 on-call support:
```json
{
  "name": "Trigger_PagerDuty",
  "type": "WebActivity",
  "typeProperties": {
    "url": "https://events.pagerduty.com/v2/enqueue",
    "method": "POST",
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "routing_key": "YOUR_INTEGRATION_KEY",
      "event_action": "trigger",
      "payload": {
        "summary": "ADF Pipeline Failure: @{pipeline().Pipeline}",
        "severity": "error",
        "source": "Azure Data Factory",
        "custom_details": {
          "pipeline_name": "@{pipeline().Pipeline}",
          "run_id": "@{pipeline().RunId}",
          "error_message": "@{activity('Copy_CustomerData').Error.message}"
        }
      }
    }
  }
}
```

### Advanced Monitoring Patterns

#### **Pattern 1: Heartbeat Monitoring**

Create a simple pipeline that runs every 5 minutes to verify ADF is operational:

```json
{
  "name": "Pipeline_Heartbeat",
  "activities": [
    {
      "name": "Lookup_Heartbeat",
      "type": "Lookup",
      "typeProperties": {
        "source": {
          "type": "AzureSqlSource",
          "sqlReaderQuery": "SELECT GETDATE() as CurrentTime, 'Healthy' as Status"
        }
      }
    },
    {
      "name": "Log_Heartbeat",
      "type": "SqlServerStoredProcedure",
      "dependsOn": [{"activity": "Lookup_Heartbeat", "dependencyConditions": ["Succeeded"]}],
      "typeProperties": {
        "storedProcedureName": "usp_LogHeartbeat"
      }
    }
  ]
}
```

**Alert if heartbeat stops:**
```kusto
// Alert if no heartbeat in last 15 minutes
ADFPipelineRun
| where PipelineName == "Pipeline_Heartbeat"
| summarize LastHeartbeat = max(End)
| extend MinutesSinceLastHeartbeat = datetime_diff('minute', now(), LastHeartbeat)
| where MinutesSinceLastHeartbeat > 15
```

#### **Pattern 2: Dependency Chain Monitoring**

Monitor the entire dependency chain, not just individual pipelines:

```sql
CREATE VIEW vw_PipelineChainStatus AS
SELECT 
    pl.PipelineName as ParentPipeline,
    pl.RunID as ParentRunID,
    pl.Status as ParentStatus,
    pl.StartTime as ParentStartTime,
    cl.PipelineName as ChildPipeline,
    cl.Status as ChildStatus,
    cl.StartTime as ChildStartTime,
    CASE 
        WHEN pl.Status = 'Succeeded' AND cl.Status = 'Succeeded' THEN 'Healthy'
        WHEN pl.Status = 'Failed' THEN 'Parent Failed'
        WHEN pl.Status = 'Succeeded' AND cl.Status = 'Failed' THEN 'Child Failed'
        WHEN pl.Status = 'InProgress' OR cl.Status = 'InProgress' THEN 'Running'
        ELSE 'Unknown'
    END as ChainStatus
FROM dbo.PipelineExecutionLog pl
LEFT JOIN dbo.PipelineExecutionLog cl ON pl.RunID = cl.ParentRunID
WHERE pl.CreatedDate > DATEADD(HOUR, -24, GETDATE());
```

#### **Pattern 3: SLA Tracking**

```sql
CREATE TABLE dbo.PipelineSLA (
    PipelineName VARCHAR(200) PRIMARY KEY,
    ExpectedCompletionTime TIME,
    MaxDurationMinutes INT,
    CriticalityLevel VARCHAR(20),  -- Critical, High, Medium, Low
    NotificationEmail VARCHAR(200)
);

CREATE PROCEDURE usp_CheckSLACompliance
AS
BEGIN
    -- Check if pipelines completed within SLA
    SELECT 
        pel.PipelineName,
        pel.StartTime,
        pel.EndTime,
        pel.DurationSeconds / 60 as DurationMinutes,
        sla.MaxDurationMinutes,
        sla.ExpectedCompletionTime,
        CASE 
            WHEN pel.DurationSeconds / 60 > sla.MaxDurationMinutes THEN 'SLA Breach - Duration'
            WHEN CAST(pel.EndTime AS TIME) > sla.ExpectedCompletionTime THEN 'SLA Breach - Late Completion'
            WHEN pel.Status = 'Failed' THEN 'SLA Breach - Failed'
            ELSE 'SLA Met'
        END as SLAStatus
    FROM dbo.PipelineExecutionLog pel
    INNER JOIN dbo.PipelineSLA sla ON pel.PipelineName = sla.PipelineName
    WHERE pel.CreatedDate > DATEADD(HOUR, -24, GETDATE())
      AND (
          pel.DurationSeconds / 60 > sla.MaxDurationMinutes
          OR CAST(pel.EndTime AS TIME) > sla.ExpectedCompletionTime
          OR pel.Status = 'Failed'
      );
    
    -- Send alerts for SLA breaches
    IF @@ROWCOUNT > 0
    BEGIN
        -- Log SLA breach for alerting
        INSERT INTO dbo.DataQualityChecks (PipelineName, CheckName, CheckType, ExpectedValue, ActualValue, Status)
        SELECT 
            PipelineName,
            'SLA Breach',
            'SLA',
            CAST(MaxDurationMinutes AS VARCHAR),
            CAST(DurationSeconds / 60 AS VARCHAR),
            'Fail'
        FROM dbo.PipelineExecutionLog pel
        INNER JOIN dbo.PipelineSLA sla ON pel.PipelineName = sla.PipelineName
        WHERE pel.CreatedDate > DATEADD(HOUR, -1, GETDATE())
          AND pel.DurationSeconds / 60 > sla.MaxDurationMinutes;
    END
END
```

#### **Pattern 4: Proactive Monitoring (Predictive Alerts)**

Use historical data to predict failures before they happen:

```kusto
// Predict pipeline failure based on duration trend
ADFPipelineRun
| where TimeGenerated > ago(30d)
| where PipelineName == "Pipeline_CustomerLoad"
| where Status == "Succeeded"
| extend DurationMinutes = datetime_diff('minute', End, Start)
| summarize 
    AvgDuration = avg(DurationMinutes),
    StdDev = stdev(DurationMinutes),
    Trend = percentile(DurationMinutes, 90)
    by bin(TimeGenerated, 1d)
| extend PredictedFailure = iff(Trend > (AvgDuration + (2 * StdDev)), "High Risk", "Normal")
| where PredictedFailure == "High Risk"
```

### Common Issues and Resolutions

#### **Issue 1: Alerts Not Firing**
**Symptoms:** Configured alerts but not receiving notifications

**Causes:**
- Action group not properly configured
- Alert rule disabled
- Threshold not met
- Diagnostic settings not enabled

**Resolution:**
1. Verify diagnostic settings are enabled and sending logs to Log Analytics
2. Check alert rule is enabled
3. Test action group using "Test" button
4. Verify email addresses and phone numbers are correct
5. Check spam/junk folders for alert emails
6. Review alert history in Azure Monitor

#### **Issue 2: Too Many Alerts (Alert Fatigue)**
**Symptoms:** Receiving hundreds of alerts, team ignoring them

**Resolution:**
- Implement alert severity levels (Sev 0-4)
- Use alert suppression rules
- Aggregate similar alerts
- Adjust thresholds to reduce false positives
- Create summary alerts instead of individual alerts

**Example: Aggregated Alert**
```kusto
// Alert only if more than 5 pipelines fail in 1 hour
ADFPipelineRun
| where TimeGenerated > ago(1h)
| where Status == "Failed"
| summarize FailedCount = count()
| where FailedCount > 5
```

#### **Issue 3: Missing Critical Failures**
**Symptoms:** Pipeline failed but no alert received

**Resolution:**
- Create specific alerts for critical pipelines
- Use multiple notification channels (email + SMS + Teams)
- Implement heartbeat monitoring
- Set up escalation policies

#### **Issue 4: Log Analytics Query Performance**
**Symptoms:** Dashboard queries taking too long

**Resolution:**
```kusto
// Optimize queries with time filters and summarization
ADFPipelineRun
| where TimeGenerated > ago(24h)  // Always filter by time first
| where PipelineName in ("Pipeline1", "Pipeline2")  // Filter by specific pipelines
| summarize count() by bin(TimeGenerated, 1h), Status  // Aggregate data
```

#### **Issue 5: Diagnostic Logs Not Appearing**
**Symptoms:** Enabled diagnostic settings but no logs in Log Analytics

**Resolution:**
1. Wait 5-10 minutes for logs to appear (initial delay)
2. Verify Log Analytics workspace is in same subscription
3. Check if ADF has permissions to write to workspace
4. Trigger a pipeline run to generate logs
5. Verify diagnostic settings include the correct log categories

### Best Practices

1. **Alert Hierarchy**
   - **Sev 0 (Critical):** Production data pipeline failures affecting customers
   - **Sev 1 (High):** Important pipeline failures, SLA breaches
   - **Sev 2 (Medium):** Performance degradation, warnings
   - **Sev 3 (Low):** Informational alerts, non-critical issues

2. **Notification Strategy**
   ```
   Sev 0: SMS + Email + Teams + PagerDuty (24/7 on-call)
   Sev 1: Email + Teams (business hours escalation)
   Sev 2: Email only (daily digest)
   Sev 3: Dashboard only (no notifications)
   ```

3. **Dashboard Design**
   - **Executive Dashboard:** High-level metrics, updated hourly
   - **Operations Dashboard:** Real-time metrics, updated every 5 minutes
   - **Troubleshooting Dashboard:** Detailed logs and errors, on-demand

4. **Log Retention**
   - **Hot tier (Log Analytics):** 30-90 days for active querying
   - **Archive tier (Storage Account):** 1-7 years for compliance
   - **Control tables:** 2+ years for trend analysis

5. **Alert Maintenance**
   - Monthly review of alert effectiveness
   - Adjust thresholds based on feedback
   - Remove obsolete alerts
   - Document alert response procedures

6. **Monitoring Documentation**
   - Document what each alert means
   - Create runbooks for common issues
   - Maintain troubleshooting guides
   - Keep contact lists updated

### Expected Outcomes

**Before Implementation:**
- Average detection time: 2-4 hours
- Mean time to resolution: 4-8 hours
- Unplanned downtime: 10+ hours/month
- Manual monitoring required

**After Implementation:**
- Average detection time: 2-5 minutes
- Mean time to resolution: 30-60 minutes
- Unplanned downtime: < 1 hour/month
- Automated monitoring and alerting

### Monitoring Checklist

**Daily:**
- [ ] Review failed pipeline runs
- [ ] Check SLA compliance
- [ ] Verify critical pipelines completed
- [ ] Review data quality check results

**Weekly:**
- [ ] Analyze performance trends
- [ ] Review alert effectiveness
- [ ] Check for anomalies in data volumes
- [ ] Update monitoring documentation

**Monthly:**
- [ ] Review and optimize alert thresholds
- [ ] Analyze cost trends
- [ ] Update SLA definitions
- [ ] Conduct monitoring effectiveness review

### Interview Questions You Might Face

**Q1: "How do you monitor ADF pipelines in production?"**

**Answer:** "I implement a multi-layered monitoring approach. First, I enable diagnostic settings to send all pipeline, activity, and trigger run logs to Log Analytics with 90-day retention for compliance. Second, I create Azure Monitor alerts for critical scenarios: pipeline failures with Sev 0 alerts going to SMS and PagerDuty, duration anomalies for SLA monitoring, and Integration Runtime health checks. Third, I build custom monitoring using control tables that log execution details, row counts, and data quality checks after each pipeline run. Fourth, I create dashboards for different audiences - an executive dashboard showing success rates and SLA compliance, and an operations dashboard with real-time metrics and error details. Finally, I implement proactive monitoring like heartbeat pipelines and predictive alerts based on historical trends."

**Q2: "What alerts would you set up for a critical production pipeline?"**

**Answer:** "For a critical pipeline, I'd set up multiple alerts: 1) **Failure Alert** - Immediate notification via SMS and email when the pipeline fails, with Sev 0 priority. 2) **Duration Alert** - Alert if the pipeline takes longer than 120% of its average duration, indicating performance issues. 3) **Data Volume Alert** - Alert if row count deviates more than 50% from the 30-day average, indicating potential data quality issues. 4) **SLA Alert** - Alert if the pipeline doesn't complete by the required time (e.g., 8 AM for morning reports). 5) **Dependency Alert** - Alert if upstream dependencies fail. I'd also implement a heartbeat alert that triggers if the pipeline hasn't run successfully in the expected timeframe, catching scheduling issues."

**Q3: "How do you handle alert fatigue?"**

**Answer:** "Alert fatigue is a real problem I've dealt with. My approach is: 1) **Severity-based routing** - Only Sev 0/1 alerts go to SMS, everything else is email or dashboard only. 2) **Aggregation** - Instead of alerting on every failed activity, I alert if more than 5 pipelines fail in an hour. 3) **Smart thresholds** - I use statistical analysis to set thresholds at 2 standard deviations from the mean, reducing false positives. 4) **Alert suppression** - If a pipeline is known to be down for maintenance, I temporarily suppress alerts. 5) **Regular reviews** - Monthly review of alert effectiveness - if an alert hasn't been actionable in 3 months, I remove it. 6) **Actionable alerts only** - Every alert must have a clear action item; if it's just informational, it goes to a dashboard instead."

**Q4: "Describe a time when monitoring caught a critical issue before it impacted users."**

**Answer:** "In my previous role, we had a customer data pipeline that loaded data from 15 source systems every night. I had implemented a data volume anomaly alert that triggered if any source had less than 50% of its average row count. One night at 2 AM, the alert fired - one source system had loaded only 100 records instead of the usual 50,000. The pipeline technically succeeded, so without this alert, we wouldn't have noticed until business users complained in the morning. I investigated and found that the source system had a database issue that caused a partial export. I contacted the source system team, they fixed the issue, and we reran the load. By 6 AM, everything was corrected, and business users never knew there was a problem. This saved us from incorrect reporting and potential business decisions based on incomplete data."

**Q5: "How do you monitor data quality in ADF?"**

**Answer:** "I implement data quality checks at multiple stages. First, I use Lookup activities to perform validation queries after each copy activity - checking row counts, null percentages, and duplicate counts. For example, after loading customer data, I check that email addresses are not null and customer IDs are unique. Second, I log these metrics to a DataQualityChecks control table with expected vs actual values. Third, I create alerts that fire when checks fail - for instance, if more than 5% of records have null email addresses. Fourth, I implement trend-based anomaly detection using stored procedures that compare current loads to 30-day averages. Finally, I build a data quality dashboard showing check results over time, helping identify patterns. For critical pipelines, failed data quality checks can stop the pipeline using If Condition activities, preventing bad data from reaching downstream systems."

**Q6: "What's your approach to troubleshooting a failed pipeline at 3 AM?"**

**Answer:** "When I get a 3 AM alert, I follow a systematic approach: 1) **Check the alert** - What pipeline failed, what was the error message? 2) **ADF Monitor UI** - I go to the pipeline run details and drill into the failed activity to see the exact error. 3) **Common issues first** - 80% of failures are: source data unavailable, network issues, or authentication failures. I check these first. 4) **Integration Runtime** - If it's a Self-Hosted IR, I verify it's online and can reach the source. 5) **Recent changes** - I check Git history for recent deployments that might have caused the issue. 6) **Logs and metrics** - I query Log Analytics for related errors or patterns. 7) **Temporary fix** - If it's a transient issue, I manually trigger a retry. If it's a code issue, I implement a quick fix or rollback. 8) **Communication** - I update the incident ticket and notify stakeholders if it impacts SLAs. 9) **Root cause analysis** - Once resolved, I document the issue and implement preventive measures. The key is having good runbooks for common issues so I can resolve most problems in 15-30 minutes."

**Q7: "How do you provide visibility to non-technical stakeholders?"**

**Answer:** "Non-technical stakeholders don't care about DIUs or activity runs - they care about business impact. I create a business-focused dashboard with metrics they understand: 1) **Data Freshness** - 'Customer data was last updated 2 hours ago' instead of 'Pipeline last ran at 06:00 UTC'. 2) **SLA Compliance** - 'Reports available on time: 95% this month' with a green/red indicator. 3) **Business metrics** - 'Records processed today: 1.2M customers, 500K orders'. 4) **Impact statements** - Instead of 'Pipeline failed', show 'Customer dashboard may show stale data'. I also send daily summary emails with a simple format: 'All systems operational' or 'Issue detected: Order processing delayed by 30 minutes, resolved at 10:30 AM'. For executives, I provide monthly reports showing trends, improvements, and any SLA breaches with business context. This builds trust and helps them understand the value of the data platform."

### Pro Tips

💡 **Tip 1:** Create a "monitoring as code" approach - store alert definitions in Git and deploy them with your pipelines.

💡 **Tip 2:** Use Azure Monitor workbooks for interactive dashboards - they're more powerful than static Log Analytics dashboards.

💡 **Tip 3:** Implement a "canary" pipeline that runs before critical pipelines to validate connectivity and permissions.

💡 **Tip 4:** Set up a Teams channel dedicated to ADF alerts - it creates a searchable history of issues.

💡 **Tip 5:** Use Azure Monitor's "Smart Groups" feature to automatically group related alerts.

💡 **Tip 6:** Create a "monitoring health check" pipeline that validates all your monitoring components are working.

### Common Mistakes to Avoid

❌ **Mistake 1:** Only monitoring pipeline-level failures, missing activity-level issues
✅ **Fix:** Monitor both pipeline and activity runs, especially for critical activities

❌ **Mistake 2:** Setting up alerts without defining response procedures
✅ **Fix:** Create runbooks for each alert type with clear resolution steps

❌ **Mistake 3:** Using the same alert threshold for all pipelines
✅ **Fix:** Customize thresholds based on each pipeline's characteristics and SLAs

❌ **Mistake 4:** Not testing alerts in non-production environments
✅ **Fix:** Regularly test alert firing and notification delivery

❌ **Mistake 5:** Ignoring successful runs - only looking at failures
✅ **Fix:** Monitor success metrics too (duration trends, data volumes) to catch degradation early

❌ **Mistake 6:** Sending all alerts to everyone
✅ **Fix:** Route alerts to the right people based on severity and pipeline ownership

### Real-World Monitoring Architecture

**Example: E-commerce Company with 200+ Pipelines**

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Data Factory                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Pipeline 1│  │Pipeline 2│  │Pipeline N│  │Heartbeat │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
└───────┼─────────────┼─────────────┼─────────────┼──────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                      │
        ┌─────────────▼─────────────────────────────┐
        │      Diagnostic Settings                   │
        │  (Pipeline, Activity, Trigger Runs)        │
        └─────────────┬─────────────────────────────┘
                      │
        ┌─────────────▼─────────────────────────────┐
        │         Log Analytics Workspace            │
        │  - 90-day retention                        │
        │  - KQL queries                             │
        │  - Custom tables                           │
        └─────┬───────────────────┬──────────────────┘
              │                   │
    ┌─────────▼─────────┐   ┌────▼──────────────────┐
    │  Azure Monitor     │   │  Control Tables       │
    │  - Metric Alerts   │   │  - Execution Log      │
    │  - Log Alerts      │   │  - Data Quality       │
    │  - Action Groups   │   │  - SLA Tracking       │
    └─────┬──────────────┘   └────┬──────────────────┘
          │                       │
    ┌─────▼───────────────────────▼──────────────────┐
    │           Notifications & Dashboards            │
    │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────────┐  │
    │  │ SMS  │ │Email │ │Teams │ │Power BI      │  │
    │  └──────┘ └──────┘ └──────┘ │Dashboard     │  │
    │  ┌──────────┐ ┌─────────────┤              │  │
    │  │PagerDuty │ │Logic Apps   │              │  │
    │  └──────────┘ └─────────────┴──────────────┘  │
    └─────────────────────────────────────────────────┘
```

**Key Metrics Tracked:**
- 200+ pipelines monitored 24/7
- Average detection time: 3 minutes
- 99.5% SLA compliance
- Mean time to resolution: 45 minutes
- Alert accuracy: 95% (low false positive rate)

---

## Summary

This document covered three critical advanced scenarios that every ADF engineer must master:

### **Scenario 29: Pipeline Dependencies**
- Orchestrating complex pipeline chains
- Parent-child patterns with Execute Pipeline activity
- Tumbling Window trigger dependencies
- Parallel processing (fan-out, fan-in)
- Failure handling and retry strategies

### **Scenario 30: Cost Optimization**
- Understanding ADF cost components (DIUs, vCores, activity runs)
- Right-sizing compute resources
- Optimizing Data Flows with TTL and cluster sizing
- Reducing activity runs through batching and wildcard patterns
- Schedule optimization and event-based triggers
- Incremental loading to reduce data volumes
- Real-world cost reduction strategies (40-60% savings)

### **Scenario 31: Monitoring and Alerts**
- Multi-layered monitoring approach
- Azure Monitor alerts for failures, SLA breaches, and anomalies
- Custom monitoring with control tables
- Building dashboards for different audiences
- Notification strategies (email, SMS, Teams, PagerDuty)
- Proactive monitoring and predictive alerts
- Troubleshooting production issues

These scenarios represent the most common challenges you'll face in production ADF environments and are frequently discussed in interviews. Mastering these patterns will demonstrate senior-level expertise and practical experience.

### Key Takeaways

1. **Dependencies are critical** - Most real-world data pipelines have complex dependencies that must be managed carefully
2. **Cost optimization is ongoing** - Regular reviews and optimizations can save 40-60% on ADF costs
3. **Monitoring is non-negotiable** - Production pipelines require comprehensive monitoring and alerting
4. **Documentation matters** - Document your patterns, runbooks, and alert responses
5. **Think holistically** - Consider dependencies, costs, and monitoring when designing any pipeline

### Next Steps

1. Practice implementing these patterns in a development environment
2. Create reusable templates for common patterns
3. Build a monitoring framework that can be applied to all pipelines
4. Document your organization's specific patterns and best practices
5. Conduct regular reviews of dependencies, costs, and monitoring effectiveness

---

**End of ADF-07: Real-World Scenarios - Additional Patterns**