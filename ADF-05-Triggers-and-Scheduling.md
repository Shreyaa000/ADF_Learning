# ADF-05: Triggers and Scheduling

## What are Triggers?

**Think of triggers like:** Alarm clocks for your data pipelines. Just like you set different alarms for different purposes (wake up, medication reminder, meeting alert), you set different triggers to run your pipelines at the right time or when specific events occur.

**Technical Definition:** A trigger represents a unit of processing that determines when a pipeline execution needs to be kicked off. Triggers are the "when" in your data orchestration.

### Why Triggers Matter

**Without Triggers:**
- Manual pipeline execution only
- No automation
- Someone needs to click "Trigger Now" every time
- Inconsistent data refresh schedules

**With Triggers:**
- ✅ Automated execution on schedule
- ✅ Event-driven processing
- ✅ Complex dependencies between pipelines
- ✅ Consistent and reliable data refresh
- ✅ No human intervention needed

---

## Types of Triggers

```
Triggers
├── Schedule Trigger (Time-based)
├── Tumbling Window Trigger (Time-based with state)
├── Event-based Trigger (Storage events)
└── Manual Trigger (On-demand)
```

---

## SCHEDULE TRIGGER

### What is a Schedule Trigger?

**Simple Explanation:** Like setting a recurring calendar event. "Run this pipeline every day at 2 AM" or "Run every Monday at 9 AM."

**When to Use:**
- ✅ Regular, predictable data loads
- ✅ Daily/weekly/monthly reports
- ✅ Batch processing at specific times
- ✅ Business hour vs off-hour processing

**When NOT to Use:**
- ❌ Need to track pipeline state between runs
- ❌ Need to reprocess specific time windows
- ❌ Need dependencies between time windows
- ❌ Event-driven scenarios (use Event-based trigger)

### Schedule Trigger Configuration

#### Basic Properties

```json
{
  "name": "DailyScheduleTrigger",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Day",
        "interval": 1,
        "startTime": "2024-01-01T02:00:00Z",
        "endTime": "2024-12-31T23:59:59Z",
        "timeZone": "Eastern Standard Time"
      }
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "DailySalesLoadPipeline",
          "type": "PipelineReference"
        },
        "parameters": {
          "loadDate": "@trigger().scheduledTime"
        }
      }
    ]
  }
}
```

#### Frequency Options

**1. Minute**
```json
{
  "frequency": "Minute",
  "interval": 15  // Every 15 minutes
}
```
**Use Case:** Near real-time data sync, frequent API polling

**2. Hour**
```json
{
  "frequency": "Hour",
  "interval": 2  // Every 2 hours
}
```
**Use Case:** Hourly sales updates, log aggregation

**3. Day**
```json
{
  "frequency": "Day",
  "interval": 1,  // Every day
  "schedule": {
    "hours": [2, 14],  // At 2 AM and 2 PM
    "minutes": [0]
  }
}
```
**Use Case:** Daily batch loads, EOD processing

**4. Week**
```json
{
  "frequency": "Week",
  "interval": 1,
  "schedule": {
    "weekDays": ["Monday", "Wednesday", "Friday"],
    "hours": [9],
    "minutes": [0]
  }
}
```
**Use Case:** Weekly reports, periodic data cleanup

**5. Month**
```json
{
  "frequency": "Month",
  "interval": 1,
  "schedule": {
    "monthDays": [1, 15],  // 1st and 15th of month
    "hours": [0],
    "minutes": [0]
  }
}
```
**Use Case:** Monthly financial reports, billing cycles

### Advanced Scheduling Patterns

#### Pattern 1: Business Hours Only
```json
{
  "frequency": "Hour",
  "interval": 1,
  "schedule": {
    "hours": [9, 10, 11, 12, 13, 14, 15, 16, 17],  // 9 AM to 5 PM
    "weekDays": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
  },
  "timeZone": "Pacific Standard Time"
}
```

**Real-World Scenario:**
*"We need to sync customer data from Salesforce every hour, but only during business hours to avoid overloading the system during peak usage."*

#### Pattern 2: Multiple Times Per Day
```json
{
  "frequency": "Day",
  "interval": 1,
  "schedule": {
    "hours": [6, 12, 18, 23],  // 6 AM, 12 PM, 6 PM, 11 PM
    "minutes": [30]
  }
}
```

**Real-World Scenario:**
*"Our retail stores need inventory updates 4 times a day to keep stock levels accurate."*

#### Pattern 3: Last Day of Month
```json
{
  "frequency": "Month",
  "interval": 1,
  "schedule": {
    "monthDays": [-1],  // Last day of month
    "hours": [23],
    "minutes": [59]
  }
}
```

**Real-World Scenario:**
*"Generate month-end financial reports on the last day of every month."*

#### Pattern 4: Quarter-End Processing
```json
{
  "frequency": "Month",
  "interval": 3,  // Every 3 months
  "schedule": {
    "monthlyOccurrences": [
      {
        "day": "Sunday",
        "occurrence": -1  // Last Sunday of quarter-end month
      }
    ],
    "hours": [0],
    "minutes": [0]
  }
}
```

### Time Zone Considerations

**Critical for Global Operations:**

```json
{
  "timeZone": "Eastern Standard Time"  // Automatically handles DST
}
```

**Common Time Zones:**
- `"UTC"` - Universal Coordinated Time (recommended for consistency)
- `"Eastern Standard Time"` - US East Coast
- `"Pacific Standard Time"` - US West Coast
- `"GMT Standard Time"` - London
- `"India Standard Time"` - India
- `"China Standard Time"` - China

**Pro Tip:** Always use UTC for global operations and convert in your pipeline if needed. This avoids DST confusion.

### Passing Parameters to Pipelines

```json
{
  "pipelines": [
    {
      "pipelineReference": {
        "referenceName": "IncrementalLoadPipeline",
        "type": "PipelineReference"
      },
      "parameters": {
        "windowStart": "@trigger().scheduledTime",
        "windowEnd": "@addHours(trigger().scheduledTime, 1)",
        "environment": "PROD",
        "loadType": "Incremental"
      }
    }
  ]
}
```

**Available Trigger Functions:**
- `@trigger().scheduledTime` - When the trigger was scheduled to run
- `@trigger().startTime` - When the trigger actually started
- `@trigger().name` - Name of the trigger

---

## TUMBLING WINDOW TRIGGER

### What is a Tumbling Window Trigger?

**Simple Explanation:** Imagine processing data in fixed time buckets (windows) that don't overlap. Like processing hourly data where each hour is processed separately, and you can track which hours succeeded or failed.

**Key Difference from Schedule Trigger:**
- **Schedule:** "Run at 2 AM every day" (stateless)
- **Tumbling Window:** "Process data for each 1-hour window" (stateful)

**When to Use:**
- ✅ Need to reprocess specific time windows
- ✅ Need dependencies between time windows
- ✅ Need to track state per window
- ✅ Incremental data processing with watermarks
- ✅ Backfilling historical data

**When NOT to Use:**
- ❌ Simple scheduled runs without state tracking
- ❌ Event-driven scenarios
- ❌ Don't need to reprocess specific windows

### Tumbling Window Configuration

#### Basic Configuration

```json
{
  "name": "HourlyTumblingWindowTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "endTime": "2024-12-31T23:59:59Z",
      "delay": "00:15:00",  // Wait 15 minutes after window closes
      "maxConcurrency": 5,  // Process 5 windows in parallel
      "retryPolicy": {
        "count": 3,
        "intervalInSeconds": 300  // 5 minutes between retries
      }
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "IncrementalLoadPipeline",
        "type": "PipelineReference"
      },
      "parameters": {
        "windowStart": "@trigger().outputs.windowStartTime",
        "windowEnd": "@trigger().outputs.windowEndTime"
      }
    }
  }
}
```

### Understanding Windows

**Visual Representation:**
```
Time:     00:00   01:00   02:00   03:00   04:00   05:00
          |-------|-------|-------|-------|-------|
Window 1: [=======]
Window 2:         [=======]
Window 3:                 [=======]
Window 4:                         [=======]
Window 5:                                 [=======]
```

**Each window:**
- Has a start time (inclusive)
- Has an end time (exclusive)
- Processes independently
- Can be retried independently
- Can have dependencies

### Key Properties Explained

#### 1. Delay
**Purpose:** Wait for late-arriving data before processing the window.

```json
"delay": "00:30:00"  // Wait 30 minutes after window closes
```

**Example:**
- Window: 2:00 PM - 3:00 PM
- Window closes: 3:00 PM
- Delay: 30 minutes
- Pipeline starts: 3:30 PM

**Real-World Scenario:**
*"Our IoT devices sometimes send data with a 15-minute delay. We set a 20-minute delay on our tumbling window to ensure we capture all data before processing."*

#### 2. Max Concurrency
**Purpose:** Control how many windows can run in parallel.

```json
"maxConcurrency": 10
```

**Example:**
- You have 24 failed hourly windows from yesterday
- maxConcurrency = 10
- ADF will process 10 windows simultaneously
- As each completes, the next one starts

**Pro Tip:** Set based on your data source capacity. Don't overwhelm your source system.

#### 3. Retry Policy
```json
"retryPolicy": {
  "count": 3,
  "intervalInSeconds": 600  // 10 minutes
}
```

**Retry Behavior:**
- Attempt 1: Immediate
- Attempt 2: After 10 minutes
- Attempt 3: After another 10 minutes
- After 3 failures: Window marked as failed

### Tumbling Window Parameters

**Available in Pipeline:**

```json
{
  "parameters": {
    "windowStart": "@trigger().outputs.windowStartTime",
    "windowEnd": "@trigger().outputs.windowEndTime",
    "windowInterval": "@trigger().outputs.windowInterval"
  }
}
```

**Example Values:**
- `windowStart`: "2024-01-15T14:00:00.0000000Z"
- `windowEnd`: "2024-01-15T15:00:00.0000000Z"
- `windowInterval`: "PT1H" (1 hour)

### Real-World Scenarios

#### Scenario 1: Hourly Incremental Load

**Business Requirement:**
*"Load sales transactions from SQL Server to Data Lake every hour. Each hour should process only that hour's data."*

**Implementation:**

```json
{
  "name": "HourlySalesLoadTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:05:00",  // 5-minute delay for late data
      "maxConcurrency": 3
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "SalesIncrementalLoad",
        "type": "PipelineReference"
      },
      "parameters": {
        "startDate": "@trigger().outputs.windowStartTime",
        "endDate": "@trigger().outputs.windowEndTime"
      }
    }
  }
}
```

**In Pipeline (Copy Activity Source Query):**
```sql
SELECT *
FROM Sales
WHERE OrderDateTime >= '@{pipeline().parameters.startDate}'
  AND OrderDateTime < '@{pipeline().parameters.endDate}'
```

#### Scenario 2: Daily Data with Dependencies

**Business Requirement:**
*"Process daily sales data, but only after the previous day's processing succeeds."*

**Implementation:**

```json
{
  "name": "DailySalesTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-01-01T02:00:00Z",
      "delay": "00:00:00",
      "maxConcurrency": 1  // Process one day at a time
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "DailySalesProcessing",
        "type": "PipelineReference"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "offset": "-1.00:00:00",  // Previous day
        "size": "1.00:00:00"      // 1 day window
      }
    ]
  }
}
```

**What This Does:**
- Today's window waits for yesterday's window to succeed
- If yesterday failed, today won't run
- Ensures sequential processing

#### Scenario 3: Backfilling Historical Data

**Business Requirement:**
*"We just set up our pipeline. We need to load the last 30 days of historical data."*

**Implementation:**

```json
{
  "name": "BackfillTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",  // 30 days ago
      "endTime": "2024-01-31T00:00:00Z",    // Today
      "maxConcurrency": 5  // Process 5 days in parallel
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "HistoricalDataLoad",
        "type": "PipelineReference"
      }
    }
  }
}
```

**What Happens:**
- Creates 30 windows (one per day)
- Processes 5 windows concurrently
- Each window loads one day's data
- Can monitor each day's success/failure independently

### Tumbling Window Dependencies

#### Self-Dependency
**Use Case:** Today depends on yesterday

```json
"dependsOn": [
  {
    "type": "SelfDependencyTumblingWindowTriggerReference",
    "offset": "-1.00:00:00",  // Previous window
    "size": "1.00:00:00"      // Same size as current window
  }
]
```

#### Other Trigger Dependency
**Use Case:** Pipeline B depends on Pipeline A

**Trigger A (Source Data Load):**
```json
{
  "name": "SourceDataLoadTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "LoadSourceData",
        "type": "PipelineReference"
      }
    }
  }
}
```

**Trigger B (Data Transformation - depends on A):**
```json
{
  "name": "TransformDataTrigger",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:10:00"  // Extra delay after dependency
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "TransformData",
        "type": "PipelineReference"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "SourceDataLoadTrigger",
        "offset": "00:00:00",  // Same window
        "size": "01:00:00"     // 1 hour window
      }
    ]
  }
}
```

**Execution Flow:**
```
Hour 1:
  10:00 AM - SourceDataLoadTrigger starts (window 10:00-11:00)
  10:15 AM - SourceDataLoadTrigger completes
  10:15 AM - TransformDataTrigger starts (waits for dependency)
  10:30 AM - TransformDataTrigger completes

Hour 2:
  11:00 AM - SourceDataLoadTrigger starts (window 11:00-12:00)
  11:10 AM - SourceDataLoadTrigger completes
  11:10 AM - TransformDataTrigger starts
  11:25 AM - TransformDataTrigger completes
```

#### Complex Dependency: Multiple Windows

**Use Case:** Aggregate last 3 hours of data

```json
"dependsOn": [
  {
    "type": "TumblingWindowTriggerDependencyReference",
    "referenceName": "HourlyDataTrigger",
    "offset": "-2.00:00:00",  // 2 hours ago
    "size": "03:00:00"        // 3-hour window
  }
]
```

**What This Means:**
- Current window: 12:00 PM - 1:00 PM
- Depends on: 10:00 AM - 1:00 PM (3 hours)
- Waits for windows at 10 AM, 11 AM, and 12 PM to all succeed

### Rerunning Failed Windows

**Scenario:** Window for 2:00 PM - 3:00 PM failed due to network issue.

**How to Rerun:**

1. **Via Portal:**
   - Go to Monitor → Trigger Runs
   - Find the failed window
   - Click "Rerun"
   - Only that specific window reruns

2. **Via PowerShell:**
```powershell
Invoke-AzDataFactoryV2TriggerRun `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -TriggerName "HourlyTumblingWindowTrigger" `
    -TriggerRunStartTime "2024-01-15T14:00:00Z" `
    -TriggerRunEndTime "2024-01-15T15:00:00Z"
```

**Pro Tip:** Failed windows don't block future windows (unless there's a dependency). You can fix the issue and rerun failed windows later.

---

## EVENT-BASED TRIGGER

### What is an Event-Based Trigger?

**Simple Explanation:** Like a motion sensor light. The light turns on when it detects movement. Similarly, your pipeline runs when it detects a specific event (like a file being created).

**When to Use:**
- ✅ Process files as soon as they arrive
- ✅ Event-driven architecture
- ✅ Real-time or near-real-time processing
- ✅ React to blob storage events

**When NOT to Use:**
- ❌ Scheduled batch processing
- ❌ Need to process multiple files together
- ❌ Source doesn't support events

### Event-Based Trigger Configuration

#### Storage Events Trigger

```json
{
  "name": "BlobCreatedTrigger",
  "properties": {
    "type": "BlobEventsTrigger",
    "typeProperties": {
      "blobPathBeginsWith": "/raw-data/sales/",
      "blobPathEndsWith": ".csv",
      "ignoreEmptyBlobs": true,
      "events": [
        "Microsoft.Storage.BlobCreated"
      ],
      "scope": "/subscriptions/{subscription-id}/resourceGroups/{rg-name}/providers/Microsoft.Storage/storageAccounts/{storage-account}"
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "ProcessNewFile",
          "type": "PipelineReference"
        },
        "parameters": {
          "fileName": "@triggerBody().fileName",
          "folderPath": "@triggerBody().folderPath",
          "fileUrl": "@triggerBody().data.url"
        }
      }
    ]
  }
}
```

### Event Types

**1. Microsoft.Storage.BlobCreated**
- Triggered when a blob is created
- Includes: New uploads, copy operations, rename operations

**2. Microsoft.Storage.BlobDeleted**
- Triggered when a blob is deleted
- Use case: Archive cleanup, audit logging

### Path Filtering

#### Begins With (Folder Filter)
```json
"blobPathBeginsWith": "/raw-data/sales/"
```
- Only triggers for files in the `sales` folder
- Includes subfolders: `/raw-data/sales/2024/01/file.csv` ✅

#### Ends With (File Extension Filter)
```json
"blobPathEndsWith": ".csv"
```
- Only triggers for CSV files
- Ignores: `.txt`, `.json`, `.parquet`

#### Combined Filtering
```json
"blobPathBeginsWith": "/raw-data/sales/",
"blobPathEndsWith": ".csv"
```
- Only CSV files in the sales folder

### Available Trigger Data

**In Pipeline Parameters:**

```json
{
  "parameters": {
    "fileName": "@triggerBody().fileName",
    "folderPath": "@triggerBody().folderPath",
    "fileUrl": "@triggerBody().data.url",
    "fileSize": "@triggerBody().data.contentLength",
    "eventTime": "@triggerBody().eventTime",
    "eventType": "@triggerBody().eventType"
  }
}
```

**Example Values:**
```json
{
  "fileName": "sales_20240115.csv",
  "folderPath": "raw-data/sales",
  "fileUrl": "https://mystorageaccount.blob.core.windows.net/raw-data/sales/sales_20240115.csv",
  "fileSize": "1048576",
  "eventTime": "2024-01-15T14:30:00.0000000Z",
  "eventType": "Microsoft.Storage.BlobCreated"
}
```

### Real-World Scenarios

#### Scenario 1: Process Files as They Arrive

**Business Requirement:**
*"Vendors upload sales files to our blob storage throughout the day. We need to process each file immediately after upload."*

**Implementation:**

**Trigger:**
```json
{
  "name": "VendorFileArrivalTrigger",
  "properties": {
    "type": "BlobEventsTrigger",
    "typeProperties": {
      "blobPathBeginsWith": "/vendor-uploads/",
      "blobPathEndsWith": ".csv",
      "ignoreEmptyBlobs": true,
      "events": ["Microsoft.Storage.BlobCreated"],
      "scope": "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/vendorstorage"
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "ProcessVendorFile",
          "type": "PipelineReference"
        },
        "parameters": {
          "sourceFile": "@triggerBody().folderPath/@triggerBody().fileName",
          "fileSize": "@triggerBody().data.contentLength"
        }
      }
    ]
  }
}
```

**Pipeline Activities:**
1. **Validation Activity** - Check file size, format
2. **Copy Activity** - Load to staging
3. **Data Flow** - Transform and validate
4. **Copy Activity** - Load to final destination
5. **Delete Activity** - Remove processed file or move to archive

#### Scenario 2: Multiple File Types, Different Processing

**Business Requirement:**
*"We receive CSV files and JSON files in the same folder. Each type needs different processing."*

**Solution: Two Triggers**

**Trigger 1: CSV Files**
```json
{
  "name": "CSVFileTrigger",
  "typeProperties": {
    "blobPathBeginsWith": "/incoming/",
    "blobPathEndsWith": ".csv",
    "events": ["Microsoft.Storage.BlobCreated"]
  },
  "pipelines": [
    {
      "pipelineReference": {
        "referenceName": "ProcessCSVFile"
      }
    }
  ]
}
```

**Trigger 2: JSON Files**
```json
{
  "name": "JSONFileTrigger",
  "typeProperties": {
    "blobPathBeginsWith": "/incoming/",
    "blobPathEndsWith": ".json",
    "events": ["Microsoft.Storage.BlobCreated"]
  },
  "pipelines": [
    {
      "pipelineReference": {
        "referenceName": "ProcessJSONFile"
      }
    }
  ]
}
```

#### Scenario 3: Archive Cleanup

**Business Requirement:**
*"When files are deleted from our active storage, log the deletion for audit purposes."*

**Implementation:**

```json
{
  "name": "FileDeletionAuditTrigger",
  "properties": {
    "type": "BlobEventsTrigger",
    "typeProperties": {
      "blobPathBeginsWith": "/active-data/",
      "events": ["Microsoft.Storage.BlobDeleted"],
      "scope": "/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/datastorage"
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "LogFileDeletion",
          "type": "PipelineReference"
        },
        "parameters": {
          "deletedFile": "@triggerBody().data.url",
          "deletionTime": "@triggerBody().eventTime"
        }
      }
    ]
  }
}
```

### Event-Based Trigger Limitations

**Important Constraints:**

1. **Storage Account Requirements:**
   - Must be Azure Blob Storage or ADLS Gen2
   - Must have hierarchical namespace enabled (for ADLS Gen2)
   - Event Grid must be enabled

2. **Latency:**
   - Not truly real-time (typically 1-2 minutes delay)
   - Event Grid delivery can be delayed under load

3. **Duplicate Events:**
   - Event Grid guarantees "at least once" delivery
   - Your pipeline should be idempotent (safe to run multiple times)

4. **No Batch Processing:**
   - One event = one pipeline run
   - If 100 files arrive, 100 pipeline runs trigger
   - Can be expensive for high-volume scenarios

**Pro Tip:** For high-volume file arrivals, consider using a Schedule Trigger with GetMetadata to batch process multiple files instead of event-based triggers.

---

## MANUAL TRIGGER

### What is a Manual Trigger?

**Simple Explanation:** The "Run Now" button. You manually start the pipeline when needed.

**When to Use:**
- ✅ Ad-hoc data loads
- ✅ Testing and debugging
- ✅ One-time data migrations
- ✅ Manual intervention scenarios

**How to Use:**

**Via Portal:**
1. Go to your pipeline
2. Click "Add trigger" → "Trigger now"
3. Provide parameter values
4. Click "OK"

**Via PowerShell:**
```powershell
Invoke-AzDataFactoryV2Pipeline `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -PipelineName "MyPipeline" `
    -Parameter @{
        "startDate" = "2024-01-01"
        "endDate" = "2024-01-31"
    }
```

**Via REST API:**
```bash
POST https://management.azure.com/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.DataFactory/factories/{factory-name}/pipelines/{pipeline-name}/createRun?api-version=2018-06-01

Body:
{
  "startDate": "2024-01-01",
  "endDate": "2024-01-31"
}
```

---

## TRIGGER MANAGEMENT

### Starting and Stopping Triggers

**Important:** Creating a trigger doesn't automatically start it!

**Start a Trigger:**
```powershell
Start-AzDataFactoryV2Trigger `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -Name "DailyScheduleTrigger"
```

**Stop a Trigger:**
```powershell
Stop-AzDataFactoryV2Trigger `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -Name "DailyScheduleTrigger"
```

**Via Portal:**
1. Go to Manage → Triggers
2. Find your trigger
3. Toggle the "Started" switch

### Monitoring Triggers

**View Trigger Runs:**
1. Go to Monitor → Trigger runs
2. Filter by:
   - Trigger name
   - Status (Succeeded, Failed, In Progress)
   - Time range

**What You See:**
- Trigger name
- Trigger type
- Trigger time
- Status
- Associated pipeline run

### Trigger Best Practices

#### 1. Naming Conventions

**Good Names:**
- `Daily_Sales_Load_2AM_EST`
- `Hourly_Incremental_Customer_Sync`
- `BlobCreated_Vendor_Files`

**Bad Names:**
- `Trigger1`
- `MyTrigger`
- `Test`

**Pattern:**
`{Frequency}_{PipelinePurpose}_{TimeOrEvent}_{OptionalDetails}`

#### 2. Time Zone Consistency

**Pro Tip:** Use UTC for all triggers unless you have a specific business requirement for local time zones.

**Why UTC:**
- No DST confusion
- Consistent across regions
- Easier to troubleshoot

**When to Use Local Time:**
- Business users expect reports at "9 AM local time"
- Compliance requires local time processing

#### 3. Delay Configuration

**For Tumbling Windows:**
```json
"delay": "00:15:00"  // 15-minute delay
```

**Guidelines:**
- **No delay:** Source data is immediately available
- **5-10 minutes:** Typical cloud-to-cloud scenarios
- **15-30 minutes:** On-premises sources, IoT devices
- **1+ hours:** Batch systems, external vendors

#### 4. Max Concurrency

**For Tumbling Windows:**
```json
"maxConcurrency": 5
```

**Guidelines:**
- **1:** Sequential processing, dependencies between windows
- **3-5:** Balanced approach for most scenarios
- **10+:** Backfilling, independent windows, high-capacity sources
- **50:** Maximum allowed by ADF

**Consider:**
- Source system capacity
- Destination system capacity
- Network bandwidth
- Cost (more concurrent runs = higher cost)

#### 5. Retry Policy

```json
"retryPolicy": {
  "count": 3,
  "intervalInSeconds": 300
}
```

**Guidelines:**
- **Transient errors:** 3-5 retries, 5-10 minute intervals
- **External API calls:** 2-3 retries, 1-2 minute intervals
- **Database operations:** 3 retries, 5 minute intervals
- **No retries:** Idempotent operations, manual intervention needed

---

## ADVANCED TRIGGER SCENARIOS

### Scenario 1: Complex Scheduling - Business Days Only

**Requirement:** Run every hour on business days (Mon-Fri), excluding holidays.

**Solution:**

**Trigger (Hourly on Weekdays):**
```json
{
  "name": "BusinessHoursTrigger",
  "properties": {
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Hour",
        "interval": 1,
        "schedule": {
          "weekDays": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
          "hours": [9, 10, 11, 12, 13, 14, 15, 16, 17]
        },
        "timeZone": "Eastern Standard Time"
      }
    },
    "pipelines": [
      {
        "pipelineReference": {
          "referenceName": "BusinessDayProcessing",
          "type": "PipelineReference"
        }
      }
    ]
  }
}
```

**Pipeline (Check for Holidays):**

**Activity 1: Lookup - Check Holiday Table**
```json
{
  "name": "CheckHoliday",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT COUNT(*) as IsHoliday FROM Holidays WHERE HolidayDate = CAST(GETDATE() AS DATE)"
    }
  }
}
```

**Activity 2: If Condition**
```json
{
  "name": "IfNotHoliday",
  "type": "IfCondition",
  "dependsOn": [
    {
      "activity": "CheckHoliday",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "expression": {
      "value": "@equals(activity('CheckHoliday').output.firstRow.IsHoliday, 0)",
      "type": "Expression"
    },
    "ifTrueActivities": [
      {
        "name": "ProcessData",
        "type": "Copy"
        // ... actual processing
      }
    ],
    "ifFalseActivities": [
      {
        "name": "LogHolidaySkip",
        "type": "WebActivity"
        // ... log that processing was skipped
      }
    ]
  }
}
```

### Scenario 2: Cascading Pipelines with Dependencies

**Requirement:** 
1. Load raw data (hourly)
2. Transform data (depends on raw load)
3. Aggregate data (depends on transform)
4. Generate reports (depends on aggregate)

**Solution:**

**Trigger 1: Raw Data Load**
```json
{
  "name": "T1_RawDataLoad",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "P1_LoadRawData"
      }
    }
  }
}
```

**Trigger 2: Transform Data (depends on T1)**
```json
{
  "name": "T2_TransformData",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:05:00"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "P2_TransformData"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T1_RawDataLoad",
        "offset": "00:00:00",
        "size": "01:00:00"
      }
    ]
  }
}
```

**Trigger 3: Aggregate Data (depends on T2)**
```json
{
  "name": "T3_AggregateData",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:05:00"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "P3_AggregateData"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T2_TransformData",
        "offset": "00:00:00",
        "size": "01:00:00"
      }
    ]
  }
}
```

**Trigger 4: Generate Reports (depends on T3)**
```json
{
  "name": "T4_GenerateReports",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:05:00"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "P4_GenerateReports"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T3_AggregateData",
        "offset": "00:00:00",
        "size": "01:00:00"
      }
    ]
  }
}
```

**Execution Flow:**
```
Hour 1 (10:00 AM - 11:00 AM):
10:00 - T1_RawDataLoad starts
10:15 - T1_RawDataLoad completes
10:20 - T2_TransformData starts (5-min delay)
10:35 - T2_TransformData completes
10:40 - T3_AggregateData starts (5-min delay)
10:50 - T3_AggregateData completes
10:55 - T4_GenerateReports starts (5-min delay)
11:05 - T4_GenerateReports completes
```

### Scenario 3: Multi-Region Processing

**Requirement:** Process data from 3 regions, then aggregate all regions.

**Solution:**

**Trigger: Regional Processing (3 separate triggers)**

**Region 1:**
```json
{
  "name": "T_Region1_Processing",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "ProcessRegionalData"
      },
      "parameters": {
        "region": "US-EAST",
        "windowStart": "@trigger().outputs.windowStartTime",
        "windowEnd": "@trigger().outputs.windowEndTime"
      }
    }
  }
}
```

**Region 2 & 3:** Similar configuration with different region parameters

**Aggregation Trigger (depends on all 3 regions):**
```json
{
  "name": "T_GlobalAggregation",
  "properties": {
    "type": "TumblingWindowTrigger",
    "typeProperties": {
      "frequency": "Hour",
      "interval": 1,
      "startTime": "2024-01-01T00:00:00Z",
      "delay": "00:10:00"
    },
    "pipeline": {
      "pipelineReference": {
        "referenceName": "AggregateAllRegions"
      }
    },
    "dependsOn": [
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T_Region1_Processing",
        "offset": "00:00:00",
        "size": "01:00:00"
      },
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T_Region2_Processing",
        "offset": "00:00:00",
        "size": "01:00:00"
      },
      {
        "type": "TumblingWindowTriggerDependencyReference",
        "referenceName": "T_Region3_Processing",
        "offset": "00:00:00",
        "size": "01:00:00"
      }
    ]
  }
}
```

**What This Does:**
- All 3 regional triggers run in parallel
- Aggregation trigger waits for ALL 3 to complete
- If any region fails, aggregation doesn't run
- Each region can be retried independently

---

## TROUBLESHOOTING TRIGGERS

### Common Issues and Solutions

#### Issue 1: Trigger Not Firing

**Symptoms:**
- Trigger is started
- No pipeline runs appear
- No errors shown

**Checklist:**
1. **Is the trigger started?**
   - Go to Manage → Triggers
   - Check "Started" status

2. **Is the schedule correct?**
   - Check start time (hasn't passed?)
   - Check end time (already passed?)
   - Check time zone

3. **For Tumbling Windows: Check dependencies**
   - Are dependent triggers running?
   - Are dependent windows succeeding?

4. **For Event-Based: Check Event Grid**
   - Is Event Grid subscription active?
   - Are events being generated?
   - Check path filters (beginsWith/endsWith)

**Solution Example:**
```powershell
# Check trigger status
Get-AzDataFactoryV2Trigger `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -Name "MyTrigger"

# Start trigger if stopped
Start-AzDataFactoryV2Trigger `
    -ResourceGroupName "myResourceGroup" `
    -DataFactoryName "myDataFactory" `
    -Name "MyTrigger"
```

#### Issue 2: Tumbling Window Stuck

**Symptoms:**
- One window keeps failing
- Subsequent windows are blocked

**Cause:**
- Dependency on failed window
- maxConcurrency = 1

**Solution:**

**Option 1: Fix and Rerun**
1. Fix the underlying issue
2. Rerun the failed window
3. Subsequent windows will proceed

**Option 2: Skip Failed Window**
```powershell
# Mark window as succeeded (use with caution!)
# This is not a direct ADF command, but you can:
# 1. Stop the trigger
# 2. Fix the issue
# 3. Adjust start time to skip the problematic window
# 4. Restart trigger
```

**Option 3: Increase Max Concurrency**
- If windows are independent
- Change maxConcurrency to allow parallel processing
- Failed windows won't block others

#### Issue 3: Event-Based Trigger Firing Multiple Times

**Symptoms:**
- One file upload triggers multiple pipeline runs
- Duplicate processing

**Cause:**
- Event Grid "at least once" delivery guarantee
- File operations that generate multiple events (rename = delete + create)

**Solution:**

**Make Pipeline Idempotent:**

```json
// In pipeline, check if file already processed
{
  "name": "CheckIfProcessed",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT COUNT(*) as Processed FROM ProcessedFiles WHERE FileName = '@{pipeline().parameters.fileName}' AND ProcessedDate = CAST(GETDATE() AS DATE)"
    }
  }
}

{
  "name": "IfNotProcessed",
  "type": "IfCondition",
  "dependsOn": [
    {
      "activity": "CheckIfProcessed",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "expression": {
      "value": "@equals(activity('CheckIfProcessed').output.firstRow.Processed, 0)",
      "type": "Expression"
    },
    "ifTrueActivities": [
      // Process the file
    ],
    "ifFalseActivities": [
      // Log that file was already processed, skip
    ]
  }
}
```

#### Issue 4: Schedule Trigger Running at Wrong Time

**Symptoms:**
- Trigger runs at unexpected times
- Off by several hours

**Cause:**
- Time zone misconfiguration
- DST transition

**Solution:**

**Check Time Zone:**
```json
{
  "recurrence": {
    "frequency": "Day",
    "interval": 1,
    "startTime": "2024-01-01T02:00:00Z",  // Z = UTC
    "timeZone": "Eastern Standard Time"    // Specify time zone
  }
}
```

**Pro Tip:** Use UTC and convert in pipeline if needed:
```json
{
  "parameters": {
    "utcTime": "@trigger().scheduledTime",
    "estTime": "@convertTimeZone(trigger().scheduledTime, 'UTC', 'Eastern Standard Time')"
  }
}
```

#### Issue 5: Too Many Concurrent Runs

**Symptoms:**
- High costs
- Source/destination systems overloaded
- Throttling errors

**Cause:**
- maxConcurrency too high
- Multiple triggers on same pipeline

**Solution:**

**Reduce Max Concurrency:**
```json
{
  "maxConcurrency": 3  // Reduce from 10 to 3
}
```

**Add Pipeline Concurrency Limit:**
```json
// In pipeline JSON
{
  "properties": {
    "concurrency": 5  // Max 5 concurrent runs of this pipeline
  }
}
```

---

## TRIGGER EXPRESSIONS AND FUNCTIONS

### Common Trigger Functions

#### For Schedule and Tumbling Window Triggers

**@trigger().scheduledTime**
- When the trigger was scheduled to run
- Format: "2024-01-15T14:00:00.0000000Z"

**@trigger().startTime**
- When the trigger actually started
- May differ from scheduledTime if delayed

**@trigger().name**
- Name of the trigger

**@trigger().outputs.windowStartTime** (Tumbling Window only)
- Start of the time window
- Format: "2024-01-15T14:00:00.0000000Z"

**@trigger().outputs.windowEndTime** (Tumbling Window only)
- End of the time window
- Format: "2024-01-15T15:00:00.0000000Z"

#### For Event-Based Triggers

**@triggerBody().fileName**
- Name of the file that triggered the pipeline
- Example: "sales_20240115.csv"

**@triggerBody().folderPath**
- Path to the folder containing the file
- Example: "raw-data/sales"

**@triggerBody().data.url**
- Full URL to the blob
- Example: "https://mystorageaccount.blob.core.windows.net/container/file.csv"

**@triggerBody().data.contentLength**
- Size of the file in bytes
- Example: "1048576"

**@triggerBody().eventTime**
- When the event occurred
- Format: "2024-01-15T14:30:00.0000000Z"

### Useful Expressions

#### Calculate Previous Day
```json
"@formatDateTime(addDays(trigger().scheduledTime, -1), 'yyyy-MM-dd')"
```

#### Calculate Start of Month
```json
"@formatDateTime(startOfMonth(trigger().scheduledTime), 'yyyy-MM-dd')"
```

#### Calculate End of Month
```json
"@formatDateTime(addDays(startOfMonth(addMonths(trigger().scheduledTime, 1)), -1), 'yyyy-MM-dd')"
```

#### Extract Date Parts
```json
"year": "@formatDateTime(trigger().scheduledTime, 'yyyy')",
"month": "@formatDateTime(trigger().scheduledTime, 'MM')",
"day": "@formatDateTime(trigger().scheduledTime, 'dd')"
```

#### Build Dynamic File Path
```json
"@concat('raw-data/', formatDateTime(trigger().scheduledTime, 'yyyy'), '/', formatDateTime(trigger().scheduledTime, 'MM'), '/', formatDateTime(trigger().scheduledTime, 'dd'), '/sales.csv')"
```
Result: "raw-data/2024/01/15/sales.csv"

---

## INTERVIEW QUESTIONS

### Basic Questions

**Q1: What are the different types of triggers in ADF?**

**Answer:**
"There are four types of triggers in ADF:

1. **Schedule Trigger** - Time-based, runs on a schedule like daily or hourly. It's stateless and simple to set up.

2. **Tumbling Window Trigger** - Also time-based but stateful. It processes data in fixed time windows and allows you to track state, reprocess specific windows, and create dependencies between windows.

3. **Event-Based Trigger** - Event-driven, triggers when a specific event occurs like a blob being created or deleted in Azure Storage.

4. **Manual Trigger** - On-demand execution when you click 'Trigger Now' or call the API.

I've used Schedule Triggers for daily batch loads, Tumbling Windows for incremental hourly processing with dependencies, and Event-Based triggers for real-time file processing scenarios."

**Q2: What's the difference between Schedule Trigger and Tumbling Window Trigger?**

**Answer:**
"The key differences are:

**Schedule Trigger:**
- Stateless - doesn't track which runs succeeded or failed
- Simple time-based execution
- Can't reprocess specific time periods easily
- Can't create dependencies between runs
- Good for simple batch jobs

**Tumbling Window Trigger:**
- Stateful - tracks each time window independently
- Can reprocess specific windows without affecting others
- Supports dependencies between windows and between triggers
- Has properties like maxConcurrency, delay, and retry policy
- Better for incremental data processing

For example, in my last project, we used Schedule Trigger for a simple daily report generation, but switched to Tumbling Window for our hourly sales data load because we needed to track which hours succeeded and reprocess failed hours without rerunning everything."

**Q3: How do you pass parameters from a trigger to a pipeline?**

**Answer:**
"You define parameters in the trigger configuration and use trigger expressions to pass values. For example:

```json
{
  "pipelines": [{
    "pipelineReference": {
      "referenceName": "MyPipeline"
    },
    "parameters": {
      "loadDate": "@trigger().scheduledTime",
      "windowStart": "@trigger().outputs.windowStartTime",
      "fileName": "@triggerBody().fileName"
    }
  }]
}
```

The expressions available depend on trigger type:
- Schedule/Tumbling Window: `@trigger().scheduledTime`, `@trigger().outputs.windowStartTime/EndTime`
- Event-Based: `@triggerBody().fileName`, `@triggerBody().folderPath`, `@triggerBody().data.url`

In my projects, I always pass time-based parameters from triggers so pipelines can process the correct data range."

### Intermediate Questions

**Q4: Explain tumbling window dependencies with an example.**

**Answer:**
"Tumbling window dependencies ensure that one trigger waits for another trigger's window to complete before running. There are two types:

**1. Self-Dependency:**
Today's window waits for yesterday's window to succeed. Example:
```json
{
  "dependsOn": [{
    "type": "SelfDependencyTumblingWindowTriggerReference",
    "offset": "-1.00:00:00",  // Previous day
    "size": "1.00:00:00"
  }]
}
```

**2. Cross-Trigger Dependency:**
One pipeline depends on another. For example, we had a data transformation pipeline that depended on the raw data load pipeline:
```json
{
  "dependsOn": [{
    "type": "TumblingWindowTriggerDependencyReference",
    "referenceName": "RawDataLoadTrigger",
    "offset": "00:00:00",  // Same window
    "size": "01:00:00"
  }]
}
```

In my last project, we had a 4-stage pipeline: Load → Transform → Aggregate → Report. Each stage was a separate trigger with dependencies, so if the Load failed at 2 PM, the subsequent stages for that hour wouldn't run, but the 3 PM load could still proceed independently."

**Q5: How do you handle late-arriving data in tumbling windows?**

**Answer:**
"I use the 'delay' property in tumbling window triggers. This tells ADF to wait a specified time after the window closes before starting the pipeline.

For example:
```json
{
  "frequency": "Hour",
  "interval": 1,
  "delay": "00:15:00"  // Wait 15 minutes
}
```

This means for the 2:00-3:00 PM window, the pipeline won't start until 3:15 PM, giving late data 15 minutes to arrive.

In a recent IoT project, we discovered that about 5% of device data arrived 5-10 minutes late due to network issues. We set a 15-minute delay, which reduced our data completeness issues from 5% to less than 0.1%. The trade-off is slightly delayed processing, but the data quality improvement was worth it."

**Q6: How do event-based triggers work? What are their limitations?**

**Answer:**
"Event-based triggers use Azure Event Grid to detect storage events like blob creation or deletion. When a matching event occurs, the trigger fires the pipeline.

**How it works:**
1. You configure the trigger with path filters (beginsWith/endsWith)
2. Event Grid monitors the storage account
3. When a matching blob event occurs, Event Grid notifies ADF
4. ADF starts the pipeline with event details (file name, path, URL)

**Key Limitations:**

1. **Not truly real-time** - Typically 1-2 minute delay, can be longer under load

2. **At-least-once delivery** - Event Grid may deliver the same event multiple times, so your pipeline must be idempotent

3. **Storage account requirements** - Only works with Azure Blob Storage and ADLS Gen2

4. **High volume costs** - If you receive 1000 files/hour, you get 1000 pipeline runs, which can be expensive

5. **No batch processing** - Can't group multiple files into one run

In my experience, event-based triggers work great for moderate-volume scenarios (< 100 files/hour). For higher volumes, I've used Schedule Triggers with GetMetadata to batch process multiple files in one run."

### Advanced Questions

**Q7: How would you design a solution to process files from multiple vendors with different schedules?**

**Answer:**
"I would use a metadata-driven approach with a combination of triggers:

**Architecture:**

1. **Vendor Configuration Table:**
```sql
CREATE TABLE VendorConfig (
    VendorID INT,
    VendorName VARCHAR(100),
    FilePattern VARCHAR(200),
    FolderPath VARCHAR(500),
    Schedule VARCHAR(50),  -- 'Hourly', 'Daily', 'Event-Based'
    ProcessingPipeline VARCHAR(100)
)
```

2. **Event-Based Trigger for Real-Time Vendors:**
```json
{
  "name": "RealTimeVendorTrigger",
  "typeProperties": {
    "blobPathBeginsWith": "/vendor-uploads/",
    "events": ["Microsoft.Storage.BlobCreated"]
  },
  "pipelines": [{
    "pipelineReference": {
      "referenceName": "DynamicVendorProcessor"
    },
    "parameters": {
      "filePath": "@triggerBody().folderPath",
      "fileName": "@triggerBody().fileName"
    }
  }]
}
```

3. **Schedule Triggers for Batch Vendors:**
- Hourly trigger for vendors that send files every hour
- Daily trigger for vendors that send once per day

4. **Master Processing Pipeline:**
- Lookup vendor config based on file path
- Execute appropriate processing logic
- Handle errors and notifications

**Real-World Example:**
In my last project, we had 15 vendors:
- 5 sent files continuously (event-based trigger)
- 7 sent files hourly (hourly schedule trigger)
- 3 sent files daily (daily schedule trigger)

The master pipeline looked up vendor-specific rules (file format, validation rules, destination) and processed accordingly. This gave us a scalable solution without creating 15 separate pipelines."

**Q8: How do you handle trigger failures and implement retry logic?**

**Answer:**
"I implement retry logic at multiple levels:

**1. Trigger-Level Retry (Tumbling Window):**
```json
{
  "retryPolicy": {
    "count": 3,
    "intervalInSeconds": 300  // 5 minutes
  }
}
```

**2. Activity-Level Retry (in pipeline):**
```json
{
  "name": "CopyActivity",
  "policy": {
    "retry": 3,
    "retryIntervalInSeconds": 30,
    "timeout": "01:00:00"
  }
}
```

**3. Custom Retry Logic (for complex scenarios):**
I use an Until loop with exponential backoff:

```json
{
  "name": "RetryUntilSuccess",
  "type": "Until",
  "typeProperties": {
    "expression": {
      "value": "@equals(variables('success'), true)",
      "type": "Expression"
    },
    "timeout": "01:00:00",
    "activities": [
      {
        "name": "TryOperation",
        "type": "Copy"
      },
      {
        "name": "SetSuccessTrue",
        "type": "SetVariable",
        "dependsOn": [{
          "activity": "TryOperation",
          "dependencyConditions": ["Succeeded"]
        }]
      },
      {
        "name": "WaitBeforeRetry",
        "type": "Wait",
        "dependsOn": [{
          "activity": "TryOperation",
          "dependencyConditions": ["Failed"]
        }],
        "typeProperties": {
          "waitTimeInSeconds": 60
        }
      }
    ]
  }
}
```

**4. Monitoring and Alerting:**
- Azure Monitor alerts for repeated failures
- Logic App integration for email notifications
- Log Analytics queries to identify patterns

**Real Example:**
We had a vendor API that occasionally returned 429 (too many requests). I implemented:
- 3 retries with 2-minute intervals at activity level
- If all 3 failed, pipeline marked as failed
- Tumbling window trigger retried the entire window after 10 minutes
- Alert sent to ops team if same window failed 3 times

This reduced manual intervention from 10 incidents/week to 1-2/month."

**Q9: How do you optimize costs for high-frequency triggers?**

**Answer:**
"Cost optimization for triggers involves several strategies:

**1. Batch Processing Instead of Event-Based:**

Instead of:
- Event-based trigger: 1000 files = 1000 pipeline runs = high cost

Use:
- Schedule trigger every 5 minutes
- GetMetadata to list new files
- ForEach to process all files in one run
- Result: 288 pipeline runs/day instead of potentially thousands

**2. Optimize Max Concurrency:**
```json
{
  "maxConcurrency": 3  // Process 3 windows at a time
}
```
Lower concurrency = fewer parallel runs = lower cost (but slower backfilling)

**3. Consolidate Pipelines:**
Instead of:
- 10 separate triggers → 10 separate pipelines

Use:
- 1 trigger → 1 master pipeline → Execute Pipeline activities for each sub-pipeline
- Reduces trigger overhead

**4. Use Appropriate Trigger Types:**
- Don't use Tumbling Window if you don't need state tracking
- Schedule Trigger is simpler and has less overhead

**5. Optimize Activity Execution:**
- Use higher DIU for Copy activities (faster = less execution time)
- Combine multiple Copy activities into Data Flow if doing transformations
- Use PolyBase for SQL loads (faster = cheaper)

**Real Example:**
We had a vendor sending 50-100 files/hour. Initial design:
- Event-based trigger: ~1,200 pipeline runs/day
- Cost: ~$150/month just for pipeline runs

Optimized design:
- Schedule trigger every 10 minutes
- Batch process all new files
- Result: 144 pipeline runs/day
- Cost: ~$20/month
- Saved $130/month (87% reduction)

The trade-off was processing delay (max 10 minutes instead of 1-2 minutes), which was acceptable for this use case."

**Q10: Describe a complex trigger scenario you've implemented.**

**Answer:**
"I implemented a multi-region, multi-stage data processing system for a global retail company:

**Requirement:**
- 3 regions (US, EU, APAC) sending hourly sales data
- Each region processed independently
- After all regions complete, global aggregation
- If any region fails, retry that region only
- Generate reports only after successful global aggregation
- Handle backfilling for missed hours

**Solution:**

**Stage 1: Regional Processing (3 Tumbling Window Triggers)**
```json
{
  "name": "T_US_HourlySales",
  "typeProperties": {
    "frequency": "Hour",
    "interval": 1,
    "delay": "00:10:00",  // 10-min delay for late data
    "maxConcurrency": 5   // Process 5 hours in parallel if backfilling
  },
  "pipeline": {
    "pipelineReference": {
      "referenceName": "ProcessRegionalSales"
    },
    "parameters": {
      "region": "US",
      "windowStart": "@trigger().outputs.windowStartTime",
      "windowEnd": "@trigger().outputs.windowEndTime"
    }
  }
}
```
(Similar triggers for EU and APAC)

**Stage 2: Global Aggregation (Tumbling Window with Dependencies)**
```json
{
  "name": "T_GlobalAggregation",
  "typeProperties": {
    "frequency": "Hour",
    "interval": 1,
    "delay": "00:15:00",  // Extra delay after regions
    "maxConcurrency": 3
  },
  "pipeline": {
    "pipelineReference": {
      "referenceName": "AggregateGlobalSales"
    }
  },
  "dependsOn": [
    {
      "type": "TumblingWindowTriggerDependencyReference",
      "referenceName": "T_US_HourlySales",
      "offset": "00:00:00",
      "size": "01:00:00"
    },
    {
      "type": "TumblingWindowTriggerDependencyReference",
      "referenceName": "T_EU_HourlySales",
      "offset": "00:00:00",
      "size": "01:00:00"
    },
    {
      "type": "TumblingWindowTriggerDependencyReference",
      "referenceName": "T_APAC_HourlySales",
      "offset": "00:00:00",
      "size": "01:00:00"
    }
  ]
}
```

**Stage 3: Report Generation (Depends on Aggregation)**
```json
{
  "name": "T_HourlyReports",
  "typeProperties": {
    "frequency": "Hour",
    "interval": 1,
    "delay": "00:05:00"
  },
  "pipeline": {
    "pipelineReference": {
      "referenceName": "GenerateHourlyReports"
    }
  },
  "dependsOn": [
    {
      "type": "TumblingWindowTriggerDependencyReference",
      "referenceName": "T_GlobalAggregation",
      "offset": "00:00:00",
      "size": "01:00:00"
    }
  ]
}
```

**Key Features:**

1. **Independent Regional Processing:**
   - US, EU, APAC process in parallel
   - If US fails at 2 PM, EU and APAC continue
   - Can rerun just US 2 PM window

2. **Dependency Chain:**
   - Global aggregation waits for ALL regions
   - Reports wait for aggregation
   - Ensures data consistency

3. **Backfilling:**
   - maxConcurrency = 5 for regional triggers
   - If system was down for 10 hours, processes 5 hours in parallel
   - Respects dependencies (aggregation waits for all regions)

4. **Monitoring:**
   - Each trigger tracked independently
   - Can see exactly which region/hour failed
   - Alerts configured for repeated failures

**Results:**
- Processed 72 hours of data per day (24 hours × 3 regions)
- 99.5% success rate
- Average processing time: 15 minutes per hour
- Failed windows easily identified and rerun
- Saved ~40 hours/month of manual troubleshooting

This architecture scaled to 5 regions and handled Black Friday traffic (10x normal volume) without issues."

---

## SUMMARY

### Quick Reference

| Trigger Type | Use Case | Stateful | Dependencies | Reprocessing |
|-------------|----------|----------|--------------|--------------|
| Schedule | Simple batch jobs | No | No | Rerun entire pipeline |
| Tumbling Window | Incremental loads | Yes | Yes | Rerun specific windows |
| Event-Based | File arrival processing | No | No | Idempotent design needed |
| Manual | Ad-hoc, testing | No | No | Manual execution |

### When to Use What

**Use Schedule Trigger when:**
- ✅ Simple time-based execution
- ✅ Don't need to track state
- ✅ Don't need dependencies
- ✅ Don't need to reprocess specific time periods

**Use Tumbling Window Trigger when:**
- ✅ Incremental data processing
- ✅ Need to track which time windows succeeded/failed
- ✅ Need dependencies between pipelines
- ✅ Need to reprocess specific time windows
- ✅ Backfilling historical data

**Use Event-Based Trigger when:**
- ✅ Process files as they arrive
- ✅ Event-driven architecture
- ✅ Near real-time processing
- ✅ Moderate file volume (< 100/hour)

**Use Manual Trigger when:**
- ✅ Ad-hoc data loads
- ✅ Testing and debugging
- ✅ One-time migrations

### Pro Tips

1. **Always use UTC** for global operations unless business specifically requires local time
2. **Make pipelines idempotent** especially with event-based triggers
3. **Use tumbling windows** for any incremental data processing
4. **Set appropriate delays** based on your data source latency
5. **Monitor trigger runs** regularly to catch issues early
6. **Use dependencies** to ensure data consistency across pipelines
7. **Optimize maxConcurrency** based on source system capacity
8. **Implement retry logic** at both trigger and activity levels
9. **Use metadata-driven patterns** for scalable solutions
10. **Consider costs** when choosing trigger frequency and type

---

**Next Steps:**
- Practice creating different trigger types
- Implement tumbling window dependencies
- Test event-based triggers with blob storage
- Build a metadata-driven trigger solution
- Monitor and optimize trigger performance

**Related Documentation:**
- ADF-01: Core Concepts
- ADF-02: Activities Reference
- ADF-06: Building Your First Pipeline
- ADF-07: Real-World Scenarios
- ADF-10: Monitoring and Troubleshooting

