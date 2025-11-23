# ADF-07: 30+ Real-World Scenarios - Missing Scenarios

## Scenario 26: Timeouts - Setting and Handling Activity Timeouts

### Business Requirement
A financial services company needs to process daily transaction files from external vendors. Sometimes vendor APIs become slow or unresponsive, causing pipelines to hang indefinitely. The business requires that any activity taking longer than expected should timeout gracefully, trigger alerts, and allow the pipeline to continue with alternative logic.

### Problem Statement
- Long-running activities can block pipeline execution
- Need to prevent indefinite waits on external dependencies
- Must handle timeout scenarios differently from failures
- Need to implement fallback mechanisms when timeouts occur

### Implementation Steps

#### Step 1: Configure Activity Timeout
Every activity in ADF has a timeout property that can be configured:

```json
{
    "name": "CallVendorAPI",
    "type": "WebActivity",
    "typeProperties": {
        "url": "https://vendor-api.com/transactions",
        "method": "GET",
        "authentication": {
            "type": "MSI"
        }
    },
    "policy": {
        "timeout": "0.00:05:00",
        "retry": 0,
        "retryIntervalInSeconds": 30
    }
}
```

**Timeout Format**: `D.HH:MM:SS`
- `0.00:05:00` = 5 minutes
- `0.01:00:00` = 1 hour
- `7.00:00:00` = 7 days (maximum)

#### Step 2: Create Timeout Handling Pattern

**Pattern 1: Timeout with Retry Logic**

```
Pipeline Structure:
├── Web Activity (API Call) - Timeout: 5 minutes
│   └── On Failure → If Condition
│       ├── Check if error is timeout
│       ├── True → Wait Activity (2 minutes) → Retry Web Activity
│       └── False → Send Error Notification
```

**If Condition Expression to Detect Timeout:**
```javascript
@equals(
    activity('CallVendorAPI').error.errorCode,
    'ActivityTimedOut'
)
```

#### Step 3: Implement Complete Timeout Pipeline

```json
{
    "name": "TimeoutHandlingPipeline",
    "activities": [
        {
            "name": "CallExternalAPI",
            "type": "WebActivity",
            "policy": {
                "timeout": "0.00:05:00",
                "retry": 0
            },
            "typeProperties": {
                "url": "@pipeline().parameters.ApiEndpoint",
                "method": "POST",
                "body": {
                    "date": "@formatDateTime(utcnow(), 'yyyy-MM-dd')"
                }
            }
        },
        {
            "name": "CheckIfTimeout",
            "type": "IfCondition",
            "dependsOn": [
                {
                    "activity": "CallExternalAPI",
                    "dependencyConditions": ["Failed"]
                }
            ],
            "typeProperties": {
                "expression": {
                    "value": "@equals(activity('CallExternalAPI').error.errorCode, 'ActivityTimedOut')",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "LogTimeout",
                        "type": "Lookup",
                        "typeProperties": {
                            "source": {
                                "type": "AzureSqlSource",
                                "sqlReaderQuery": {
                                    "value": "INSERT INTO ErrorLog (PipelineName, ActivityName, ErrorType, ErrorMessage, LogTime) VALUES ('@{pipeline().Pipeline}', 'CallExternalAPI', 'Timeout', 'Activity exceeded 5 minute timeout', GETDATE()); SELECT 1 as Result",
                                    "type": "Expression"
                                }
                            },
                            "dataset": {
                                "referenceName": "LogDatabase",
                                "type": "DatasetReference"
                            }
                        }
                    },
                    {
                        "name": "UseFallbackData",
                        "type": "Copy",
                        "dependsOn": [{"activity": "LogTimeout", "dependencyConditions": ["Succeeded"]}],
                        "typeProperties": {
                            "source": {
                                "type": "AzureSqlSource",
                                "sqlReaderQuery": "SELECT * FROM PreviousDayData"
                            },
                            "sink": {
                                "type": "AzureSqlSink"
                            }
                        }
                    }
                ],
                "ifFalseActivities": [
                    {
                        "name": "HandleOtherError",
                        "type": "WebActivity",
                        "typeProperties": {
                            "url": "@pipeline().parameters.ErrorWebhook",
                            "method": "POST",
                            "body": {
                                "error": "@activity('CallExternalAPI').error"
                            }
                        }
                    }
                ]
            }
        }
    ]
}
```

#### Step 4: Advanced Timeout Patterns

**Pattern 2: Progressive Timeout with Multiple Retries**

```
Attempt 1: Timeout 5 minutes
  ↓ (if timeout)
Wait 2 minutes
  ↓
Attempt 2: Timeout 10 minutes
  ↓ (if timeout)
Wait 5 minutes
  ↓
Attempt 3: Timeout 15 minutes
  ↓ (if timeout)
Use Fallback Logic
```

**Pattern 3: Parallel Execution with Timeout**

```json
{
    "name": "ParallelWithTimeout",
    "type": "ForEach",
    "typeProperties": {
        "items": "@pipeline().parameters.FileList",
        "isSequential": false,
        "batchCount": 5,
        "activities": [
            {
                "name": "ProcessFile",
                "type": "Copy",
                "policy": {
                    "timeout": "0.00:30:00"
                }
            }
        ]
    }
}
```

### Configuration Best Practices

#### 1. Timeout Values by Activity Type

| Activity Type | Recommended Timeout | Rationale |
|--------------|---------------------|-----------|
| Web Activity (API) | 5-10 minutes | APIs should respond quickly |
| Copy Activity (Small files) | 30 minutes | Depends on data volume |
| Copy Activity (Large files) | 2-4 hours | Large data transfers |
| Stored Procedure | 15-30 minutes | Database operations |
| Data Flow | 1-3 hours | Complex transformations |
| Databricks Notebook | 2-8 hours | Long-running analytics |
| Execute Pipeline | Based on child | Sum of child activities + buffer |

#### 2. Timeout Calculation Formula

```
Recommended Timeout = (Average Execution Time × 2) + Buffer

Example:
- Average API response: 2 minutes
- Recommended timeout: (2 × 2) + 1 = 5 minutes
```

### Common Timeout Scenarios

#### Scenario 1: Database Query Timeout

```json
{
    "name": "LongRunningQuery",
    "type": "Lookup",
    "policy": {
        "timeout": "0.00:30:00"
    },
    "typeProperties": {
        "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "EXEC sp_GenerateReport @Date = '@{formatDateTime(utcnow(), 'yyyy-MM-dd')}'",
            "queryTimeout": "00:25:00"
        }
    }
}
```

**Note**: `queryTimeout` (activity-specific) vs `timeout` (policy-level)

#### Scenario 2: File Transfer Timeout

```json
{
    "name": "CopyLargeFile",
    "type": "Copy",
    "policy": {
        "timeout": "0.02:00:00"
    },
    "typeProperties": {
        "source": {
            "type": "BinarySource"
        },
        "sink": {
            "type": "BinarySink"
        },
        "dataIntegrationUnits": 32
    }
}
```

### Monitoring and Alerting

#### Create Timeout Alert Query (Log Analytics)

```kql
ADFActivityRun
| where Status == "Failed"
| where ErrorCode == "ActivityTimedOut"
| project 
    TimeGenerated,
    PipelineName,
    ActivityName,
    ActivityType,
    Duration = End - Start,
    ErrorMessage
| order by TimeGenerated desc
```

#### Timeout Metrics Dashboard

```kql
ADFActivityRun
| where Status == "Failed" and ErrorCode == "ActivityTimedOut"
| summarize TimeoutCount = count() by ActivityName, bin(TimeGenerated, 1h)
| render timechart
```

### Expected Behavior

1. **Normal Execution**: Activity completes within timeout → Success
2. **Timeout Occurs**: Activity exceeds timeout → Status = "Failed", ErrorCode = "ActivityTimedOut"
3. **Retry After Timeout**: Can retry with same or increased timeout
4. **Fallback Logic**: Execute alternative activities when timeout occurs

### Common Issues and Resolutions

#### Issue 1: Timeout Too Short
**Symptom**: Activities consistently timing out
**Solution**: 
- Analyze historical execution times
- Increase timeout to 2× average execution time
- Optimize the underlying operation (query, API, etc.)

#### Issue 2: Timeout Too Long
**Symptom**: Pipeline hangs for hours before failing
**Solution**:
- Set realistic timeouts based on SLA
- Use progressive timeout strategy
- Implement health checks before long operations

#### Issue 3: Cannot Distinguish Timeout from Other Failures
**Solution**:
```javascript
// Check error code explicitly
@equals(activity('MyActivity').error.errorCode, 'ActivityTimedOut')

// Or check error message
@contains(activity('MyActivity').error.message, 'timeout')
```

#### Issue 4: Timeout Not Triggering
**Symptom**: Activity runs longer than timeout setting
**Cause**: Timeout format incorrect
**Solution**:
```json
// WRONG
"timeout": "5:00"

// CORRECT
"timeout": "0.00:05:00"
```

### Interview Questions You Might Face

**Q1: How do you handle activities that might take unpredictable amounts of time?**

**Answer**: "I implement a multi-layered timeout strategy. First, I analyze historical execution times to set a baseline timeout at 2× the average. Then I use an If Condition to detect timeout errors specifically using `activity().error.errorCode == 'ActivityTimedOut'`. For critical operations, I implement a progressive retry pattern where each retry has an increased timeout. I also always have a fallback mechanism—for example, if an API times out, we might use cached data or previous day's data. Finally, I set up Azure Monitor alerts to track timeout patterns so we can proactively optimize."

**Q2: What's the difference between activity timeout and query timeout?**

**Answer**: "Activity timeout is a policy-level setting that applies to the entire activity execution in ADF, including any overhead from the integration runtime. Query timeout is specific to database activities like Lookup or Stored Procedure and controls how long the actual SQL query can run. The activity timeout should always be slightly longer than the query timeout. For example, if my query timeout is 25 minutes, I'd set activity timeout to 30 minutes to account for connection establishment and result processing."

**Q3: How do you decide appropriate timeout values?**

**Answer**: "I follow a data-driven approach. First, I run the pipeline in development and measure actual execution times. Then I calculate timeout as 2× average execution time plus a buffer. For example, if a Copy activity averages 15 minutes, I'd set timeout to 35 minutes. I also consider business SLAs—if data must be ready by 8 AM, I work backward to set timeouts that ensure the pipeline completes on time. I document these decisions and review them quarterly based on monitoring data."

**Q4: Can you give an example of a complex timeout scenario you've handled?**

**Answer**: "In my previous project, we had a pipeline calling multiple vendor APIs in parallel. Some vendors had unreliable response times. I implemented a pattern where each API call had a 5-minute timeout. If any API timed out, we logged it to a control table and continued with other APIs. At the end, we checked if critical APIs succeeded—if yes, we proceeded with partial data and flagged it for review. If critical APIs failed, we triggered an immediate alert and used previous day's data as fallback. This pattern reduced our pipeline failures by 70% while maintaining data freshness."

**Q5: How do you test timeout scenarios?**

**Answer**: "I create test pipelines with artificially low timeouts to verify the handling logic works. For example, I'll set a 10-second timeout on a Web activity and use a test endpoint that delays 15 seconds. I verify that the timeout triggers correctly, the error code is captured, and the fallback logic executes. I also use Wait activities to simulate delays in testing. In production, I monitor the first few runs closely and adjust timeouts based on actual performance."

### Pro Tips

1. **Always set explicit timeouts**: Default is 7 days—way too long for most scenarios
2. **Use variables to store timeout values**: Makes it easier to adjust across environments
3. **Log timeout events**: Create a dedicated table to track timeout patterns
4. **Consider time zones**: UTC vs local time can affect timeout calculations
5. **Test with realistic data volumes**: Small test files may complete quickly, but production volumes might timeout

### Common Mistakes to Avoid

❌ Using default timeout (7 days) for all activities
❌ Setting timeout shorter than the operation can possibly complete
❌ Not handling timeout differently from other failures
❌ Forgetting to account for retry time in overall pipeline timeout
❌ Not monitoring timeout trends over time

### Interview Red Flags to Avoid

❌ "I just use the default timeout"
❌ "I've never had timeout issues"
❌ "I set all timeouts to 1 hour to be safe"
❌ Not knowing the timeout format syntax
❌ Not understanding the difference between activity and query timeout

---

## Scenario 27: Conditional Execution - Execute Activities Based on Previous Results

### Business Requirement
An e-commerce company needs to process daily sales data with complex business logic: validate file format, check if data is complete, process only if validation passes, send different notifications based on success/failure, and archive files to different locations based on processing results.

### Problem Statement
- Need to execute activities based on runtime conditions
- Different paths based on data validation results
- Dynamic decision-making within pipelines
- Handle multiple conditional branches
- Implement complex business logic without coding

### Implementation Steps

#### Step 1: Understanding Dependency Conditions

ADF provides four dependency conditions:

```json
{
    "dependsOn": [
        {
            "activity": "PreviousActivity",
            "dependencyConditions": [
                "Succeeded",    // Execute if previous succeeded
                "Failed",       // Execute if previous failed
                "Skipped",      // Execute if previous was skipped
                "Completed"     // Execute regardless of outcome
            ]
        }
    ]
}
```

#### Step 2: Basic Conditional Execution Pattern

**Scenario**: Process file only if validation succeeds

```json
{
    "name": "ConditionalProcessingPipeline",
    "activities": [
        {
            "name": "ValidateFile",
            "type": "Lookup",
            "typeProperties": {
                "source": {
                    "type": "AzureSqlSource",
                    "sqlReaderQuery": {
                        "value": "SELECT CASE WHEN COUNT(*) > 0 AND COUNT(*) = SUM(CASE WHEN Amount > 0 THEN 1 ELSE 0 END) THEN 1 ELSE 0 END as IsValid FROM StagingTable",
                        "type": "Expression"
                    }
                },
                "dataset": {
                    "referenceName": "ValidationDB",
                    "type": "DatasetReference"
                }
            }
        },
        {
            "name": "CheckValidationResult",
            "type": "IfCondition",
            "dependsOn": [
                {
                    "activity": "ValidateFile",
                    "dependencyConditions": ["Succeeded"]
                }
            ],
            "typeProperties": {
                "expression": {
                    "value": "@equals(activity('ValidateFile').output.firstRow.IsValid, 1)",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "ProcessValidData",
                        "type": "Copy",
                        "typeProperties": {
                            "source": {"type": "AzureSqlSource"},
                            "sink": {"type": "AzureSqlSink"}
                        }
                    },
                    {
                        "name": "SendSuccessEmail",
                        "type": "WebActivity",
                        "dependsOn": [
                            {
                                "activity": "ProcessValidData",
                                "dependencyConditions": ["Succeeded"]
                            }
                        ],
                        "typeProperties": {
                            "url": "@pipeline().parameters.LogicAppSuccessURL",
                            "method": "POST"
                        }
                    }
                ],
                "ifFalseActivities": [
                    {
                        "name": "LogValidationFailure",
                        "type": "Lookup",
                        "typeProperties": {
                            "source": {
                                "type": "AzureSqlSource",
                                "sqlReaderQuery": "INSERT INTO ErrorLog VALUES ('Validation Failed', GETDATE()); SELECT 1 as Result"
                            }
                        }
                    },
                    {
                        "name": "SendFailureEmail",
                        "type": "WebActivity",
                        "dependsOn": [
                            {
                                "activity": "LogValidationFailure",
                                "dependencyConditions": ["Succeeded"]
                            }
                        ],
                        "typeProperties": {
                            "url": "@pipeline().parameters.LogicAppFailureURL",
                            "method": "POST"
                        }
                    }
                ]
            }
        }
    ]
}
```

#### Step 3: Multiple Conditional Branches

**Scenario**: Route data based on file size

```json
{
    "name": "GetFileMetadata",
    "type": "GetMetadata",
    "typeProperties": {
        "dataset": {
            "referenceName": "InputFile",
            "type": "DatasetReference"
        },
        "fieldList": ["size", "itemName"]
    }
},
{
    "name": "CheckFileSize",
    "type": "IfCondition",
    "dependsOn": [
        {
            "activity": "GetFileMetadata",
            "dependencyConditions": ["Succeeded"]
        }
    ],
    "typeProperties": {
        "expression": {
            "value": "@greater(activity('GetFileMetadata').output.size, 104857600)",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "ProcessLargeFile",
                "type": "Copy",
                "typeProperties": {
                    "enableStaging": true,
                    "stagingSettings": {
                        "linkedServiceName": {"referenceName": "BlobStaging"}
                    }
                }
            }
        ],
        "ifFalseActivities": [
            {
                "name": "ProcessSmallFile",
                "type": "Copy",
                "typeProperties": {
                    "enableStaging": false
                }
            }
        ]
    }
}
```

#### Step 4: Complex Multi-Condition Logic

**Scenario**: Execute different logic based on day of week and data volume

```json
{
    "name": "SetDayOfWeek",
    "type": "SetVariable",
    "typeProperties": {
        "variableName": "DayOfWeek",
        "value": {
            "value": "@formatDateTime(utcnow(), 'dddd')",
            "type": "Expression"
        }
    }
},
{
    "name": "GetRecordCount",
    "type": "Lookup",
    "dependsOn": [{"activity": "SetDayOfWeek", "dependencyConditions": ["Succeeded"]}],
    "typeProperties": {
        "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT COUNT(*) as RecordCount FROM StagingTable"
        }
    }
},
{
    "name": "IsWeekendAndHighVolume",
    "type": "IfCondition",
    "dependsOn": [{"activity": "GetRecordCount", "dependencyConditions": ["Succeeded"]}],
    "typeProperties": {
        "expression": {
            "value": "@and(or(equals(variables('DayOfWeek'), 'Saturday'), equals(variables('DayOfWeek'), 'Sunday')), greater(activity('GetRecordCount').output.firstRow.RecordCount, 100000))",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "UseHighPerformanceProcessing",
                "type": "ExecuteDataFlow",
                "typeProperties": {
                    "compute": {
                        "coreCount": 32,
                        "computeType": "MemoryOptimized"
                    }
                }
            }
        ],
        "ifFalseActivities": [
            {
                "name": "UseStandardProcessing",
                "type": "Copy"
            }
        ]
    }
}
```

#### Step 5: Chained Conditional Execution

**Pattern**: Sequential validation checks

```
Check 1: File Exists
  ↓ (if true)
Check 2: File Format Valid
  ↓ (if true)
Check 3: Schema Matches
  ↓ (if true)
Check 4: Data Quality Passes
  ↓ (if true)
Process Data
```

```json
{
    "activities": [
        {
            "name": "CheckFileExists",
            "type": "GetMetadata",
            "typeProperties": {
                "fieldList": ["exists"]
            }
        },
        {
            "name": "IfFileExists",
            "type": "IfCondition",
            "dependsOn": [{"activity": "CheckFileExists", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "expression": {
                    "value": "@activity('CheckFileExists').output.exists",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "ValidateFormat",
                        "type": "Lookup"
                    },
                    {
                        "name": "IfFormatValid",
                        "type": "IfCondition",
                        "dependsOn": [{"activity": "ValidateFormat", "dependencyConditions": ["Succeeded"]}],
                        "typeProperties": {
                            "expression": {
                                "value": "@equals(activity('ValidateFormat').output.firstRow.IsValid, 1)",
                                "type": "Expression"
                            },
                            "ifTrueActivities": [
                                {
                                    "name": "ValidateSchema",
                                    "type": "Lookup"
                                },
                                {
                                    "name": "IfSchemaValid",
                                    "type": "IfCondition",
                                    "dependsOn": [{"activity": "ValidateSchema", "dependencyConditions": ["Succeeded"]}],
                                    "typeProperties": {
                                        "expression": {
                                            "value": "@equals(activity('ValidateSchema').output.firstRow.IsValid, 1)",
                                            "type": "Expression"
                                        },
                                        "ifTrueActivities": [
                                            {
                                                "name": "ProcessData",
                                                "type": "Copy"
                                            }
                                        ],
                                        "ifFalseActivities": [
                                            {
                                                "name": "LogSchemaError",
                                                "type": "Lookup"
                                            }
                                        ]
                                    }
                                }
                            ],
                            "ifFalseActivities": [
                                {
                                    "name": "LogFormatError",
                                    "type": "Lookup"
                                }
                            ]
                        }
                    }
                ],
                "ifFalseActivities": [
                    {
                        "name": "LogFileNotFound",
                        "type": "Lookup"
                    }
                ]
            }
        }
    ]
}
```

### Advanced Conditional Patterns

#### Pattern 1: Switch-Case Logic (Multiple Conditions)

Since ADF doesn't have a native Switch activity, implement using nested If Conditions:

```json
{
    "name": "CheckStatus",
    "type": "Lookup",
    "typeProperties": {
        "source": {
            "sqlReaderQuery": "SELECT Status FROM ControlTable WHERE ProcessID = 1"
        }
    }
},
{
    "name": "SwitchOnStatus",
    "type": "IfCondition",
    "typeProperties": {
        "expression": {
            "value": "@equals(activity('CheckStatus').output.firstRow.Status, 'NEW')",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {"name": "ProcessNewRecords", "type": "Copy"}
        ],
        "ifFalseActivities": [
            {
                "name": "CheckIfPending",
                "type": "IfCondition",
                "typeProperties": {
                    "expression": {
                        "value": "@equals(activity('CheckStatus').output.firstRow.Status, 'PENDING')",
                        "type": "Expression"
                    },
                    "ifTrueActivities": [
                        {"name": "ProcessPendingRecords", "type": "Copy"}
                    ],
                    "ifFalseActivities": [
                        {
                            "name": "CheckIfError",
                            "type": "IfCondition",
                            "typeProperties": {
                                "expression": {
                                    "value": "@equals(activity('CheckStatus').output.firstRow.Status, 'ERROR')",
                                    "type": "Expression"
                                },
                                "ifTrueActivities": [
                                    {"name": "ReprocessErrors", "type": "Copy"}
                                ],
                                "ifFalseActivities": [
                                    {"name": "HandleUnknownStatus", "type": "Lookup"}
                                ]
                            }
                        }
                    ]
                }
            }
        ]
    }
}
```

#### Pattern 2: Conditional Execution Based on Activity Output

```json
{
    "name": "CopyData",
    "type": "Copy",
    "typeProperties": {
        "source": {"type": "AzureSqlSource"},
        "sink": {"type": "AzureSqlSink"}
    }
},
{
    "name": "CheckRowsCopied",
    "type": "IfCondition",
    "dependsOn": [
        {
            "activity": "CopyData",
            "dependencyConditions": ["Succeeded"]
        }
    ],
    "typeProperties": {
        "expression": {
            "value": "@greater(activity('CopyData').output.rowsCopied, 0)",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "UpdateProcessedFlag",
                "type": "Lookup",
                "typeProperties": {
                    "source": {
                        "sqlReaderQuery": {
                            "value": "UPDATE ControlTable SET ProcessedRecords = @{activity('CopyData').output.rowsCopied}, LastProcessed = GETDATE(); SELECT 1 as Result",
                            "type": "Expression"
                        }
                    }
                }
            },
            {
                "name": "TriggerDownstreamPipeline",
                "type": "ExecutePipeline",
                "dependsOn": [{"activity": "UpdateProcessedFlag", "dependencyConditions": ["Succeeded"]}],
                "typeProperties": {
                    "pipeline": {"referenceName": "DownstreamPipeline"},
                    "parameters": {
                        "RecordCount": "@activity('CopyData').output.rowsCopied"
                    }
                }
            }
        ],
        "ifFalseActivities": [
            {
                "name": "LogNoDataFound",
                "type": "Lookup",
                "typeProperties": {
                    "source": {
                        "sqlReaderQuery": "INSERT INTO AuditLog VALUES ('No data to process', GETDATE()); SELECT 1 as Result"
                    }
                }
            }
        ]
    }
}
```

#### Pattern 3: Failure Handling with Conditional Recovery

```json
{
    "name": "MainProcessing",
    "type": "Copy"
},
{
    "name": "OnFailureCheck",
    "type": "IfCondition",
    "dependsOn": [
        {
            "activity": "MainProcessing",
            "dependencyConditions": ["Failed"]
        }
    ],
    "typeProperties": {
        "expression": {
            "value": "@contains(activity('MainProcessing').error.message, 'timeout')",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "RetryWithSmallerBatch",
                "type": "Copy",
                "typeProperties": {
                    "source": {
                        "type": "AzureSqlSource",
                        "sqlReaderQuery": "SELECT TOP 1000 * FROM SourceTable"
                    }
                }
            }
        ],
        "ifFalseActivities": [
            {
                "name": "CheckIfPermissionError",
                "type": "IfCondition",
                "typeProperties": {
                    "expression": {
                        "value": "@contains(activity('MainProcessing').error.message, 'permission')",
                        "type": "Expression"
                    },
                    "ifTrueActivities": [
                        {
                            "name": "NotifyAdmin",
                            "type": "WebActivity"
                        }
                    ],
                    "ifFalseActivities": [
                        {
                            "name": "LogUnknownError",
                            "type": "Lookup"
                        }
                    ]
                }
            }
        ]
    }
}
```

### Useful Conditional Expressions

#### Comparison Expressions

```javascript
// Equality
@equals(variables('Status'), 'Active')

// Greater than
@greater(activity('GetCount').output.firstRow.Count, 1000)

// Less than
@less(activity('GetSize').output.size, 1048576)

// Greater or equal
@greaterOrEquals(variables('Threshold'), 100)

// Less or equal
@lessOrEquals(variables('RetryCount'), 3)
```

#### Logical Expressions

```javascript
// AND
@and(equals(variables('Status'), 'Active'), greater(variables('Count'), 0))

// OR
@or(equals(variables('Status'), 'Active'), equals(variables('Status'), 'Pending'))

// NOT
@not(equals(variables('Status'), 'Inactive'))

// Complex combination
@and(
    or(equals(variables('Type'), 'Full'), equals(variables('Type'), 'Incremental')),
    greater(variables('RecordCount'), 0)
)
```

#### String Expressions

```javascript
// Contains
@contains(activity('GetMetadata').output.itemName, '.csv')

// Starts with
@startsWith(variables('FileName'), 'SALES_')

// Ends with
@endsWith(variables('FileName'), '.parquet')

// Empty check
@empty(variables('ErrorMessage'))
```

#### Date/Time Conditional

```javascript
// Check if today is Monday
@equals(formatDateTime(utcnow(), 'dddd'), 'Monday')

// Check if current hour is between 9 AM and 5 PM
@and(
    greaterOrEquals(int(formatDateTime(utcnow(), 'HH')), 9),
    lessOrEquals(int(formatDateTime(utcnow(), 'HH')), 17)
)

// Check if it's first day of month
@equals(formatDateTime(utcnow(), 'dd'), '01')
```

### Real-World Conditional Scenarios

#### Scenario 1: Environment-Based Execution

```json
{
    "name": "CheckEnvironment",
    "type": "IfCondition",
    "typeProperties": {
        "expression": {
            "value": "@equals(pipeline().globalParameters.Environment, 'PROD')",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "ProductionProcessing",
                "type": "Copy",
                "typeProperties": {
                    "enableStaging": true,
                    "validateDataConsistency": true
                }
            },
            {
                "name": "SendProdNotification",
                "type": "WebActivity"
            }
        ],
        "ifFalseActivities": [
            {
                "name": "DevProcessing",
                "type": "Copy",
                "typeProperties": {
                    "enableStaging": false,
                    "validateDataConsistency": false
                }
            }
        ]
    }
}
```

#### Scenario 2: Business Hours Processing

```json
{
    "name": "CheckBusinessHours",
    "type": "SetVariable",
    "typeProperties": {
        "variableName": "IsBusinessHours",
        "value": {
            "value": "@and(greaterOrEquals(int(formatDateTime(convertFromUtc(utcnow(), 'Eastern Standard Time'), 'HH')), 9), lessOrEquals(int(formatDateTime(convertFromUtc(utcnow(), 'Eastern Standard Time'), 'HH')), 17))",
            "type": "Expression"
        }
    }
},
{
    "name": "ConditionalProcessing",
    "type": "IfCondition",
    "dependsOn": [{"activity": "CheckBusinessHours", "dependencyConditions": ["Succeeded"]}],
    "typeProperties": {
        "expression": {
            "value": "@variables('IsBusinessHours')",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {
                "name": "LimitedProcessing",
                "type": "Copy",
                "typeProperties": {
                    "dataIntegrationUnits": 4
                }
            }
        ],
        "ifFalseActivities": [
            {
                "name": "FullProcessing",
                "type": "Copy",
                "typeProperties": {
                    "dataIntegrationUnits": 32
                }
            }
        ]
    }
}
```

#### Scenario 3: Data Quality Gate

```json
{
    "name": "ValidateDataQuality",
    "type": "Lookup",
    "typeProperties": {
        "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT CASE WHEN (SELECT COUNT(*) FROM StagingTable WHERE Amount IS NULL) = 0 AND (SELECT COUNT(DISTINCT CustomerID) FROM StagingTable) > 10 THEN 1 ELSE 0 END as PassedQuality"
        }
    }
},
{
    "name": "QualityGate",
    "type": "IfCondition",
    "dependsOn": [{"activity": "ValidateDataQuality", "dependencyConditions": ["Succeeded"]}],
    "typeProperties": {
        "expression": {
            "value": "@equals(activity('ValidateDataQuality').output.firstRow.PassedQuality, 1)",
            "type": "Expression"
        },
        "ifTrueActivities": [
            {"name": "LoadToProduction", "type": "Copy"},
            {"name": "UpdateQualityMetrics", "type": "Lookup"}
        ],
        "ifFalseActivities": [
            {"name": "QuarantineData", "type": "Copy"},
            {"name": "AlertDataQualityTeam", "type": "WebActivity"}
        ]
    }
}
```

### Expected Behavior

1. **If Condition True**: Activities in `ifTrueActivities` execute
2. **If Condition False**: Activities in `ifFalseActivities` execute
3. **Dependency Conditions**: Control when activities execute based on previous activity status
4. **Nested Conditions**: Can nest If Conditions up to 40 levels deep (though not recommended)
5. **Expression Evaluation**: Expressions evaluated at runtime with actual values

### Common Issues and Resolutions

#### Issue 1: Expression Syntax Errors
**Symptom**: Pipeline validation fails
**Common Errors**:
```javascript
// WRONG
@activity('MyActivity').output.firstRow.Count > 100

// CORRECT
@greater(activity('MyActivity').output.firstRow.Count, 100)
```

#### Issue 2: Null Reference Errors
**Symptom**: Pipeline fails during execution
**Solution**:
```javascript
// Add null checks
@if(
    and(
        not(empty(activity('Lookup').output)),
        not(empty(activity('Lookup').output.firstRow))
    ),
    activity('Lookup').output.firstRow.Value,
    'DefaultValue'
)
```

#### Issue 3: Complex Nested Conditions Hard to Debug
**Solution**: Break into multiple pipelines or use variables
```json
{
    "name": "SetCondition1",
    "type": "SetVariable",
    "typeProperties": {
        "variableName": "Condition1Met",
        "value": "@greater(variables('Count'), 100)"
    }
},
{
    "name": "SetCondition2",
    "type": "SetVariable",
    "typeProperties": {
        "variableName": "Condition2Met",
        "value": "@equals(variables('Status'), 'Active')"
    }
},
{
    "name": "CheckBothConditions",
    "type": "IfCondition",
    "typeProperties": {
        "expression": {
            "value": "@and(variables('Condition1Met'), variables('Condition2Met'))",
            "type": "Expression"
        }
    }
}
```

#### Issue 4: Activity Output Not Available
**Symptom**: Cannot access previous activity output
**Cause**: Missing dependency or activity failed
**Solution**:
```json
{
    "dependsOn": [
        {
            "activity": "PreviousActivity",
            "dependencyConditions": ["Succeeded"]  // Ensure this is set
        }
    ]
}
```

### Interview Questions You Might Face

**Q1: How do you implement complex conditional logic in ADF?**

**Answer**: "I use a combination of If Condition activities and dependency conditions. For simple branching, I use If Condition with expressions like `@equals()` or `@greater()`. For multiple conditions, I either nest If Conditions or use logical operators like `@and()` and `@or()`. For switch-case scenarios, I create nested If Conditions that check each case sequentially. I also leverage variables to store intermediate condition results, making the logic more readable and debuggable. For very complex logic, I sometimes use a Lookup activity to execute SQL with CASE statements, returning a decision flag that drives the pipeline flow."

**Q2: What's the difference between If Condition and dependency conditions?**

**Answer**: "If Condition is an activity that evaluates an expression and executes different activities based on true/false. It's used for business logic decisions like 'if record count > 1000, use high-performance processing.' Dependency conditions, on the other hand, control activity execution based on the status of previous activities—Succeeded, Failed, Skipped, or Completed. They're used for error handling and flow control. For example, I use 'Failed' dependency to execute error logging only when an activity fails. Often, I combine both: use dependency conditions to handle technical failures and If Condition for business logic decisions."

**Q3: How do you handle scenarios requiring switch-case logic?**

**Answer**: "Since ADF doesn't have a native Switch activity, I implement it using nested If Conditions. I start by getting the value to switch on, usually via a Lookup or variable. Then I create a series of nested If Conditions checking each case. For better performance and readability, I sometimes use a Lookup activity with a SQL CASE statement that returns a decision code, then use a single If Condition to route based on that code. For example, in a project where we had 5 different processing types, I used a Lookup that returned 'TYPE_A', 'TYPE_B', etc., then nested If Conditions to route to the appropriate processing pipeline."

**Q4: Can you describe a complex conditional scenario you've implemented?**

**Answer**: "In my previous role, we had a data validation pipeline with multiple quality gates. First, we checked if the file existed and wasn't empty using GetMetadata. If true, we validated the schema by comparing column names and data types using a Lookup against a metadata table. If that passed, we ran data quality checks—null checks, range validations, referential integrity. Each check was a separate Lookup returning pass/fail. Only if all checks passed did we load to production. If any check failed, we routed the data to a quarantine zone, logged the specific failure reason, and sent targeted notifications. We also had time-based logic—during business hours, we'd alert immediately; after hours, we'd batch alerts. This reduced production data issues by 85%."

**Q5: How do you debug conditional logic in ADF?**

**Answer**: "I use several techniques. First, I add Set Variable activities before If Conditions to store the expression result, making it visible in monitoring. Second, I use Lookup activities to log decision points to a table with timestamps and values. Third, I test each branch independently by temporarily hardcoding the condition to true or false. Fourth, I use the debug feature to step through the pipeline and inspect activity outputs. For complex expressions, I test them in isolation using a simple pipeline with just a Set Variable activity. I also document the logic in the activity description field so anyone debugging can understand the intent."

**Q6: What are common mistakes with conditional execution?**

**Answer**: "The biggest mistake is not handling null values in expressions, which causes runtime failures. Always check if output exists before accessing it. Second, people forget that activity outputs are only available if the activity succeeded and has a Succeeded dependency. Third, overly complex nested conditions become unmaintainable—I've seen 8 levels deep, which is a nightmare to debug. Fourth, not considering all possible paths—forgetting to handle the 'else' case. Fifth, using string comparisons without considering case sensitivity. I always use lowercase/uppercase conversions for string comparisons. Finally, not testing all branches—people test the happy path but forget to test failure scenarios."

### Pro Tips

1. **Use variables for complex expressions**: Store intermediate results for better readability
2. **Document your conditions**: Use activity descriptions to explain the business logic
3. **Test all branches**: Use debug mode to verify both true and false paths
4. **Keep nesting shallow**: More than 3 levels deep becomes hard to maintain
5. **Consider Execute Pipeline**: For complex logic, break into separate pipelines
6. **Use Lookup for complex logic**: SQL CASE statements can simplify multiple conditions
7. **Handle nulls explicitly**: Always check for null/empty before accessing values

### Common Mistakes to Avoid

❌ Not handling null values in expressions
❌ Forgetting to set proper dependency conditions
❌ Overly complex nested conditions (more than 3 levels)
❌ Not testing all conditional branches
❌ Using incorrect expression syntax (using > instead of @greater())
❌ Not documenting the business logic behind conditions
❌ Hardcoding values instead of using parameters

### Interview Red Flags to Avoid

❌ "I just use If Condition for everything"
❌ Not knowing the difference between If Condition and dependency conditions
❌ Unable to explain how to implement switch-case logic
❌ Not understanding expression syntax
❌ Never having debugged conditional logic
❌ Not considering null handling

---

## Scenario 28: Variables Usage - Setting and Using Variables Within Pipelines

### Business Requirement
A retail company needs to build dynamic pipelines that adapt based on runtime conditions: track processing state, accumulate results from multiple activities, pass data between activities, implement counters and flags, and build flexible, reusable pipeline logic.

### Problem Statement
- Need to store temporary values during pipeline execution
- Pass data between activities without using datasets
- Implement counters, flags, and accumulators
- Build dynamic expressions based on runtime calculations
- Create reusable pipeline patterns with state management

### Implementation Steps

#### Step 1: Understanding Variable Types

ADF supports three variable types:

```json
{
    "variables": {
        "StringVariable": {
            "type": "String",
            "defaultValue": "Initial Value"
        },
        "IntegerVariable": {
            "type": "Integer",
            "defaultValue": 0
        },
        "BooleanVariable": {
            "type": "Boolean",
            "defaultValue": false
        },
        "ArrayVariable": {
            "type": "Array",
            "defaultValue": []
        }
    }
}
```

**Note**: Variables are pipeline-scoped, not global. Each pipeline run has its own variable instance.

#### Step 2: Declaring Variables

At the pipeline level:

```json
{
    "name": "VariableUsagePipeline",
    "properties": {
        "variables": {
            "ProcessedCount": {
                "type": "Integer",
                "defaultValue": 0
            },
            "CurrentStatus": {
                "type": "String",
                "defaultValue": "NotStarted"
            },
            "HasErrors": {
                "type": "Boolean",
                "defaultValue": false
            },
            "FileList": {
                "type": "Array",
                "defaultValue": []
            },
            "ErrorMessage": {
                "type": "String"
            }
        },
        "activities": [...]
    }
}
```

#### Step 3: Setting Variable Values

Use **Set Variable** activity:

```json
{
    "name": "SetProcessedCount",
    "type": "SetVariable",
    "typeProperties": {
        "variableName": "ProcessedCount",
        "value": {
            "value": "@activity('CopyData').output.rowsCopied",
            "type": "Expression"
        }
    },
    "dependsOn": [
        {
            "activity": "CopyData",
            "dependencyConditions": ["Succeeded"]
        }
    ]
}
```

#### Step 4: Appending to Array Variables

Use **Append Variable** activity:

```json
{
    "name": "AppendFileName",
    "type": "AppendVariable",
    "typeProperties": {
        "variableName": "FileList",
        "value": {
            "value": "@item().name",
            "type": "Expression"
        }
    }
}
```

#### Step 5: Using Variables in Expressions

```json
{
    "name": "CheckProcessedCount",
    "type": "IfCondition",
    "typeProperties": {
        "expression": {
            "value": "@greater(variables('ProcessedCount'), 1000)",
            "type": "Expression"
        }
    }
},
{
    "name": "DynamicCopy",
    "type": "Copy",
    "typeProperties": {
        "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
                "value": "SELECT * FROM @{variables('CurrentTable')} WHERE Status = '@{variables('CurrentStatus')}'",
                "type": "Expression"
            }
        }
    }
}
```

### Common Variable Patterns

#### Pattern 1: Counter/Accumulator

**Scenario**: Track total records processed across multiple copy activities

```json
{
    "variables": {
        "TotalRecordsProcessed": {
            "type": "Integer",
            "defaultValue": 0
        }
    },
    "activities": [
        {
            "name": "CopyTable1",
            "type": "Copy"
        },
        {
            "name": "UpdateCount1",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CopyTable1", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "TotalRecordsProcessed",
                "value": {
                    "value": "@add(variables('TotalRecordsProcessed'), activity('CopyTable1').output.rowsCopied)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CopyTable2",
            "type": "Copy",
            "dependsOn": [{"activity": "UpdateCount1", "dependencyConditions": ["Succeeded"]}]
        },
        {
            "name": "UpdateCount2",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CopyTable2", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "TotalRecordsProcessed",
                "value": {
                    "value": "@add(variables('TotalRecordsProcessed'), activity('CopyTable2').output.rowsCopied)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "LogTotalCount",
            "type": "Lookup",
            "dependsOn": [{"activity": "UpdateCount2", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "source": {
                    "type": "AzureSqlSource",
                    "sqlReaderQuery": {
                        "value": "INSERT INTO ProcessingLog (TotalRecords, ProcessDate) VALUES (@{variables('TotalRecordsProcessed')}, GETDATE()); SELECT 1 as Result",
                        "type": "Expression"
                    }
                }
            }
        }
    ]
}
```

#### Pattern 2: Status Tracking

**Scenario**: Track pipeline execution status through different stages

```json
{
    "variables": {
        "PipelineStatus": {
            "type": "String",
            "defaultValue": "Initializing"
        }
    },
    "activities": [
        {
            "name": "SetStatusValidating",
            "type": "SetVariable",
            "typeProperties": {
                "variableName": "PipelineStatus",
                "value": "Validating"
            }
        },
        {
            "name": "ValidateData",
            "type": "Lookup",
            "dependsOn": [{"activity": "SetStatusValidating", "dependencyConditions": ["Succeeded"]}]
        },
        {
            "name": "SetStatusProcessing",
            "type": "SetVariable",
            "dependsOn": [{"activity": "ValidateData", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "PipelineStatus",
                "value": "Processing"
            }
        },
        {
            "name": "ProcessData",
            "type": "Copy",
            "dependsOn": [{"activity": "SetStatusProcessing", "dependencyConditions": ["Succeeded"]}]
        },
        {
            "name": "SetStatusCompleted",
            "type": "SetVariable",
            "dependsOn": [{"activity": "ProcessData", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "PipelineStatus",
                "value": "Completed"
            }
        },
        {
            "name": "SetStatusFailed",
            "type": "SetVariable",
            "dependsOn": [{"activity": "ProcessData", "dependencyConditions": ["Failed"]}],
            "typeProperties": {
                "variableName": "PipelineStatus",
                "value": {
                    "value": "@concat('Failed: ', activity('ProcessData').error.message)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "LogFinalStatus",
            "type": "Lookup",
            "dependsOn": [
                {"activity": "SetStatusCompleted", "dependencyConditions": ["Succeeded"]},
                {"activity": "SetStatusFailed", "dependencyConditions": ["Succeeded"]}
            ],
            "typeProperties": {
                "source": {
                    "sqlReaderQuery": {
                        "value": "INSERT INTO PipelineAudit (PipelineName, Status, LogTime) VALUES ('@{pipeline().Pipeline}', '@{variables('PipelineStatus')}', GETDATE()); SELECT 1 as Result",
                        "type": "Expression"
                    }
                }
            }
        }
    ]
}
```

#### Pattern 3: Error Collection

**Scenario**: Collect all errors from multiple activities

```json
{
    "variables": {
        "ErrorMessages": {
            "type": "Array",
            "defaultValue": []
        },
        "HasErrors": {
            "type": "Boolean",
            "defaultValue": false
        }
    },
    "activities": [
        {
            "name": "ProcessFile1",
            "type": "Copy"
        },
        {
            "name": "CaptureError1",
            "type": "AppendVariable",
            "dependsOn": [{"activity": "ProcessFile1", "dependencyConditions": ["Failed"]}],
            "typeProperties": {
                "variableName": "ErrorMessages",
                "value": {
                    "value": "@concat('File1 Error: ', activity('ProcessFile1').error.message)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "SetHasErrors1",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CaptureError1", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "HasErrors",
                "value": true
            }
        },
        {
            "name": "ProcessFile2",
            "type": "Copy",
            "dependsOn": [
                {"activity": "ProcessFile1", "dependencyConditions": ["Succeeded", "Failed"]}
            ]
        },
        {
            "name": "CaptureError2",
            "type": "AppendVariable",
            "dependsOn": [{"activity": "ProcessFile2", "dependencyConditions": ["Failed"]}],
            "typeProperties": {
                "variableName": "ErrorMessages",
                "value": {
                    "value": "@concat('File2 Error: ', activity('ProcessFile2').error.message)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CheckForErrors",
            "type": "IfCondition",
            "dependsOn": [
                {"activity": "ProcessFile2", "dependencyConditions": ["Succeeded", "Failed"]}
            ],
            "typeProperties": {
                "expression": {
                    "value": "@variables('HasErrors')",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "SendErrorReport",
                        "type": "WebActivity",
                        "typeProperties": {
                            "url": "@pipeline().parameters.ErrorWebhook",
                            "method": "POST",
                            "body": {
                                "errors": "@variables('ErrorMessages')",
                                "pipeline": "@pipeline().Pipeline",
                                "runId": "@pipeline().RunId"
                            }
                        }
                    }
                ]
            }
        }
    ]
}
```

#### Pattern 4: Dynamic Table/File Name Building

**Scenario**: Build dynamic names based on date and parameters

```json
{
    "variables": {
        "SourceTableName": {
            "type": "String"
        },
        "TargetFileName": {
            "type": "String"
        },
        "ProcessDate": {
            "type": "String"
        }
    },
    "activities": [
        {
            "name": "SetProcessDate",
            "type": "SetVariable",
            "typeProperties": {
                "variableName": "ProcessDate",
                "value": {
                    "value": "@formatDateTime(utcnow(), 'yyyyMMdd')",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "BuildSourceTableName",
            "type": "SetVariable",
            "dependsOn": [{"activity": "SetProcessDate", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "SourceTableName",
                "value": {
                    "value": "@concat(pipeline().parameters.TablePrefix, '_', variables('ProcessDate'))",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "BuildTargetFileName",
            "type": "SetVariable",
            "dependsOn": [{"activity": "SetProcessDate", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "TargetFileName",
                "value": {
                    "value": "@concat('Extract_', pipeline().parameters.EntityName, '_', variables('ProcessDate'), '.parquet')",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CopyData",
            "type": "Copy",
            "dependsOn": [
                {"activity": "BuildSourceTableName", "dependencyConditions": ["Succeeded"]},
                {"activity": "BuildTargetFileName", "dependencyConditions": ["Succeeded"]}
            ],
            "typeProperties": {
                "source": {
                    "type": "AzureSqlSource",
                    "sqlReaderQuery": {
                        "value": "SELECT * FROM @{variables('SourceTableName')}",
                        "type": "Expression"
                    }
                },
                "sink": {
                    "type": "ParquetSink",
                    "storeSettings": {
                        "type": "AzureBlobFSWriteSettings",
                        "fileName": {
                            "value": "@variables('TargetFileName')",
                            "type": "Expression"
                        }
                    }
                }
            }
        }
    ]
}
```

#### Pattern 5: Conditional Flag

**Scenario**: Set flags based on conditions and use them later

```json
{
    "variables": {
        "IsHighPriority": {
            "type": "Boolean",
            "defaultValue": false
        },
        "RequiresValidation": {
            "type": "Boolean",
            "defaultValue": false
        }
    },
    "activities": [
        {
            "name": "CheckRecordCount",
            "type": "Lookup",
            "typeProperties": {
                "source": {
                    "sqlReaderQuery": "SELECT COUNT(*) as RecordCount FROM StagingTable"
                }
            }
        },
        {
            "name": "SetHighPriorityFlag",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CheckRecordCount", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "IsHighPriority",
                "value": {
                    "value": "@greater(activity('CheckRecordCount').output.firstRow.RecordCount, 100000)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "SetValidationFlag",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CheckRecordCount", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "RequiresValidation",
                "value": {
                    "value": "@or(greater(activity('CheckRecordCount').output.firstRow.RecordCount, 50000), equals(pipeline().parameters.Environment, 'PROD'))",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "ConditionalProcessing",
            "type": "IfCondition",
            "dependsOn": [
                {"activity": "SetHighPriorityFlag", "dependencyConditions": ["Succeeded"]},
                {"activity": "SetValidationFlag", "dependencyConditions": ["Succeeded"]}
            ],
            "typeProperties": {
                "expression": {
                    "value": "@variables('IsHighPriority')",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "HighPriorityProcessing",
                        "type": "ExecuteDataFlow",
                        "typeProperties": {
                            "compute": {
                                "coreCount": 32,
                                "computeType": "MemoryOptimized"
                            }
                        }
                    }
                ],
                "ifFalseActivities": [
                    {
                        "name": "StandardProcessing",
                        "type": "Copy"
                    }
                ]
            }
        },
        {
            "name": "ConditionalValidation",
            "type": "IfCondition",
            "dependsOn": [{"activity": "ConditionalProcessing", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "expression": {
                    "value": "@variables('RequiresValidation')",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "RunValidation",
                        "type": "Lookup"
                    }
                ]
            }
        }
    ]
}
```

#### Pattern 6: Array Manipulation

**Scenario**: Build and iterate through dynamic lists

```json
{
    "variables": {
        "TableList": {
            "type": "Array",
            "defaultValue": []
        },
        "ProcessedTables": {
            "type": "Array",
            "defaultValue": []
        }
    },
    "activities": [
        {
            "name": "GetTableList",
            "type": "Lookup",
            "typeProperties": {
                "source": {
                    "sqlReaderQuery": "SELECT TableName FROM ControlTable WHERE IsActive = 1"
                },
                "firstRowOnly": false
            }
        },
        {
            "name": "SetTableList",
            "type": "SetVariable",
            "dependsOn": [{"activity": "GetTableList", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "TableList",
                "value": {
                    "value": "@activity('GetTableList').output.value",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "ForEachTable",
            "type": "ForEach",
            "dependsOn": [{"activity": "SetTableList", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "items": {
                    "value": "@variables('TableList')",
                    "type": "Expression"
                },
                "activities": [
                    {
                        "name": "ProcessTable",
                        "type": "Copy"
                    },
                    {
                        "name": "AddToProcessedList",
                        "type": "AppendVariable",
                        "dependsOn": [{"activity": "ProcessTable", "dependencyConditions": ["Succeeded"]}],
                        "typeProperties": {
                            "variableName": "ProcessedTables",
                            "value": {
                                "value": "@item().TableName",
                                "type": "Expression"
                            }
                        }
                    }
                ]
            }
        },
        {
            "name": "LogProcessedTables",
            "type": "Lookup",
            "dependsOn": [{"activity": "ForEachTable", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "source": {
                    "sqlReaderQuery": {
                        "value": "INSERT INTO ProcessingLog (ProcessedTables, ProcessDate) VALUES ('@{join(variables('ProcessedTables'), ',')}', GETDATE()); SELECT 1 as Result",
                        "type": "Expression"
                    }
                }
            }
        }
    ]
}
```

### Variable Operations and Functions

#### String Operations

```javascript
// Concatenation
@concat(variables('Prefix'), '_', variables('Suffix'))

// Substring
@substring(variables('FileName'), 0, 10)

// Replace
@replace(variables('Path'), '\\', '/')

// ToUpper/ToLower
@toUpper(variables('Status'))
@toLower(variables('FileName'))

// Trim
@trim(variables('InputValue'))

// Split (returns array)
@split(variables('CommaSeparatedList'), ',')

// Join (array to string)
@join(variables('ArrayVariable'), ',')
```

#### Numeric Operations

```javascript
// Addition
@add(variables('Count'), 1)

// Subtraction
@sub(variables('Total'), variables('Processed'))

// Multiplication
@mul(variables('Quantity'), variables('Price'))

// Division
@div(variables('Total'), variables('Count'))

// Modulo
@mod(variables('Number'), 10)

// Convert to int
@int(variables('StringNumber'))

// Convert to string
@string(variables('IntegerValue'))
```

#### Array Operations

```javascript
// Get array length
@length(variables('ArrayVariable'))

// Get first element
@first(variables('ArrayVariable'))

// Get last element
@last(variables('ArrayVariable'))

// Check if array contains value
@contains(variables('ArrayVariable'), 'SearchValue')

// Get element at index
@variables('ArrayVariable')[0]

// Create array from range
@range(0, 10)  // [0, 1, 2, ..., 9]

// Union (combine arrays)
@union(variables('Array1'), variables('Array2'))

// Intersection
@intersection(variables('Array1'), variables('Array2'))
```

#### Boolean Operations

```javascript
// AND
@and(variables('Flag1'), variables('Flag2'))

// OR
@or(variables('Flag1'), variables('Flag2'))

// NOT
@not(variables('Flag'))

// Comparison
@equals(variables('Status'), 'Completed')
@greater(variables('Count'), 100)
@less(variables('Count'), 1000)
```

### Advanced Variable Scenarios

#### Scenario 1: Retry Counter

```json
{
    "variables": {
        "RetryCount": {
            "type": "Integer",
            "defaultValue": 0
        },
        "MaxRetries": {
            "type": "Integer",
            "defaultValue": 3
        }
    },
    "activities": [
        {
            "name": "AttemptProcess",
            "type": "Copy"
        },
        {
            "name": "IncrementRetryCount",
            "type": "SetVariable",
            "dependsOn": [{"activity": "AttemptProcess", "dependencyConditions": ["Failed"]}],
            "typeProperties": {
                "variableName": "RetryCount",
                "value": {
                    "value": "@add(variables('RetryCount'), 1)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CheckRetryLimit",
            "type": "IfCondition",
            "dependsOn": [{"activity": "IncrementRetryCount", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "expression": {
                    "value": "@less(variables('RetryCount'), variables('MaxRetries'))",
                    "type": "Expression"
                },
                "ifTrueActivities": [
                    {
                        "name": "WaitBeforeRetry",
                        "type": "Wait",
                        "typeProperties": {
                            "waitTimeInSeconds": {
                                "value": "@mul(variables('RetryCount'), 60)",
                                "type": "Expression"
                            }
                        }
                    },
                    {
                        "name": "RetryProcess",
                        "type": "ExecutePipeline",
                        "dependsOn": [{"activity": "WaitBeforeRetry", "dependencyConditions": ["Succeeded"]}],
                        "typeProperties": {
                            "pipeline": {"referenceName": "@{pipeline().Pipeline}"}
                        }
                    }
                ],
                "ifFalseActivities": [
                    {
                        "name": "MaxRetriesExceeded",
                        "type": "WebActivity",
                        "typeProperties": {
                            "url": "@pipeline().parameters.ErrorWebhook",
                            "method": "POST",
                            "body": {
                                "message": "Max retries exceeded",
                                "retryCount": "@variables('RetryCount')"
                            }
                        }
                    }
                ]
            }
        }
    ]
}
```

#### Scenario 2: Performance Metrics Collection

```json
{
    "variables": {
        "StartTime": {"type": "String"},
        "EndTime": {"type": "String"},
        "DurationSeconds": {"type": "Integer"},
        "RecordsProcessed": {"type": "Integer"},
        "RecordsPerSecond": {"type": "Integer"}
    },
    "activities": [
        {
            "name": "CaptureStartTime",
            "type": "SetVariable",
            "typeProperties": {
                "variableName": "StartTime",
                "value": {
                    "value": "@utcnow()",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "ProcessData",
            "type": "Copy",
            "dependsOn": [{"activity": "CaptureStartTime", "dependencyConditions": ["Succeeded"]}]
        },
        {
            "name": "CaptureEndTime",
            "type": "SetVariable",
            "dependsOn": [{"activity": "ProcessData", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "EndTime",
                "value": {
                    "value": "@utcnow()",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CalculateDuration",
            "type": "SetVariable",
            "dependsOn": [{"activity": "CaptureEndTime", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "DurationSeconds",
                "value": {
                    "value": "@div(sub(ticks(variables('EndTime')), ticks(variables('StartTime'))), 10000000)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "SetRecordsProcessed",
            "type": "SetVariable",
            "dependsOn": [{"activity": "ProcessData", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "variableName": "RecordsProcessed",
                "value": {
                    "value": "@activity('ProcessData').output.rowsCopied",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "CalculateThroughput",
            "type": "SetVariable",
            "dependsOn": [
                {"activity": "CalculateDuration", "dependencyConditions": ["Succeeded"]},
                {"activity": "SetRecordsProcessed", "dependencyConditions": ["Succeeded"]}
            ],
            "typeProperties": {
                "variableName": "RecordsPerSecond",
                "value": {
                    "value": "@if(greater(variables('DurationSeconds'), 0), div(variables('RecordsProcessed'), variables('DurationSeconds')), 0)",
                    "type": "Expression"
                }
            }
        },
        {
            "name": "LogMetrics",
            "type": "Lookup",
            "dependsOn": [{"activity": "CalculateThroughput", "dependencyConditions": ["Succeeded"]}],
            "typeProperties": {
                "source": {
                    "sqlReaderQuery": {
                        "value": "INSERT INTO PerformanceMetrics (PipelineName, StartTime, EndTime, DurationSeconds, RecordsProcessed, RecordsPerSecond) VALUES ('@{pipeline().Pipeline}', '@{variables('StartTime')}', '@{variables('EndTime')}', @{variables('DurationSeconds')}, @{variables('RecordsProcessed')}, @{variables('RecordsPerSecond')}); SELECT 1 as Result",
                        "type": "Expression"
                    }
                }
            }
        }
    ]
}
```

### Expected Behavior

1. **Variable Scope**: Variables are scoped to the pipeline run—each run has its own instance
2. **Set Variable**: Replaces the entire value
3. **Append Variable**: Only works with Array type, adds to the end
4. **Default Values**: Used when pipeline starts, can be overridden
5. **Type Safety**: Must match declared type (String, Integer, Boolean, Array)
6. **Sequential Execution**: Set Variable activities execute sequentially, not in parallel

### Common Issues and Resolutions

#### Issue 1: Cannot Set Variable in Parallel
**Symptom**: Error when trying to set same variable from multiple activities
**Cause**: Set Variable activities must execute sequentially
**Solution**:
```json
// WRONG - parallel execution
{
    "name": "ForEach",
    "typeProperties": {
        "isSequential": false,
        "activities": [
            {
                "name": "SetVariable",  // This will fail!
                "type": "SetVariable"
            }
        ]
    }
}

// CORRECT - use Append Variable or make sequential
{
    "name": "ForEach",
    "typeProperties": {
        "isSequential": false,
        "activities": [
            {
                "name": "AppendVariable",  // This works
                "type": "AppendVariable"
            }
        ]
    }
}
```

#### Issue 2: Variable Type Mismatch
**Symptom**: Pipeline validation or runtime error
**Solution**:
```javascript
// WRONG
"variableName": "IntegerVar",
"value": "123"  // String value for Integer variable

// CORRECT
"variableName": "IntegerVar",
"value": "@int('123')"  // Convert to integer
```

#### Issue 3: Accessing Variable Before It's Set
**Symptom**: Empty or default value returned
**Solution**: Ensure proper dependency chain
```json
{
    "name": "UseVariable",
    "dependsOn": [
        {
            "activity": "SetVariable",
            "dependencyConditions": ["Succeeded"]
        }
    ]
}
```

#### Issue 4: Variable Not Visible in Child Pipeline
**Symptom**: Cannot access parent pipeline variable in Execute Pipeline activity
**Solution**: Pass as parameter
```json
{
    "name": "ExecuteChildPipeline",
    "type": "ExecutePipeline",
    "typeProperties": {
        "pipeline": {"referenceName": "ChildPipeline"},
        "parameters": {
            "ParentVariable": "@variables('MyVariable')"
        }
    }
}
```

### Interview Questions You Might Face

**Q1: What's the difference between variables and parameters in ADF?**

**Answer**: "Parameters are input values defined when the pipeline is triggered and remain constant throughout the run—they're read-only. Variables, on the other hand, are mutable values that can be changed during pipeline execution using Set Variable or Append Variable activities. Parameters are great for configuration values like environment names or date ranges, while variables are used for runtime state management like counters, flags, or accumulating results. For example, I use a parameter for the source database name, but a variable to track how many records have been processed across multiple copy activities."

**Q2: Can you set variables in parallel activities like ForEach?**

**Answer**: "No, Set Variable activities must execute sequentially—you cannot set the same variable from parallel branches or within a ForEach loop running in parallel mode. However, you can use Append Variable for array variables in parallel execution. If I need to collect results from parallel activities, I use Append Variable to add each result to an array variable. Alternatively, I write results to a database table or storage and aggregate them afterward. This is a common gotcha that I've seen cause pipeline failures."

**Q3: How do you use variables for error handling?**

**Answer**: "I use variables to collect error information throughout the pipeline. I declare a Boolean variable like 'HasErrors' initialized to false, and an Array variable 'ErrorMessages' initialized to empty. For each critical activity, I add a dependency on failure that appends the error message to the array and sets HasErrors to true. At the end of the pipeline, I check if HasErrors is true and send a consolidated error report with all collected messages. This pattern allows the pipeline to continue processing other activities even if some fail, giving us a complete picture of all issues in one run."

**Q4: Can you give an example of complex variable usage?**

**Answer**: "In a recent project, we built a dynamic data quality framework. We used variables to track multiple metrics: 'TotalRecordsProcessed' (Integer), 'FailedValidations' (Array of objects), 'QualityScore' (Integer calculated as percentage), and 'ProcessingStatus' (String). As each validation rule executed, we incremented the total count, appended failed records to the array, and calculated the quality score. Based on the score, we set the status to 'Passed', 'Warning', or 'Failed'. At the end, we used these variables to decide whether to promote data to production and what notifications to send. This made the pipeline highly dynamic and maintainable."

**Q5: What are the limitations of variables in ADF?**

**Answer**: "First, variables are pipeline-scoped—you can't share them across pipelines without passing as parameters. Second, Set Variable must execute sequentially, which can create bottlenecks. Third, there's no direct way to create complex objects—you're limited to String, Integer, Boolean, and Array. Fourth, array manipulation is limited—you can append but not remove or update specific elements. Fifth, variables don't persist after pipeline completion—if you need historical tracking, you must write to external storage. Understanding these limitations helps me design better solutions, like using database tables for complex state management."

**Q6: How do you debug variable values?**

**Answer**: "I use several techniques. First, I add Set Variable activities at key points and check their values in the monitoring UI under activity outputs. Second, I use Lookup activities to write variable values to a debug table with timestamps. Third, I use the pipeline debug feature to step through and inspect variable values in real-time. Fourth, I temporarily add Web activities that post variable values to a webhook for immediate visibility. Fifth, I document expected values in activity descriptions. For complex expressions, I test them in isolation using a simple pipeline with just Set Variable activities."

### Pro Tips

1. **Initialize with meaningful defaults**: Makes debugging easier
2. **Use descriptive names**: `RecordsProcessed` not `var1`
3. **Document variable purpose**: Use pipeline annotations
4. **Avoid deep nesting**: If you need complex objects, use JSON strings
5. **Consider alternatives**: Sometimes parameters or datasets are better choices
6. **Use variables for readability**: Break complex expressions into multiple variable assignments
7. **Clean up unused variables**: Remove variables you're not using

### Common Mistakes to Avoid

❌ Trying to set variables in parallel ForEach loops
❌ Not initializing variables before use
❌ Type mismatches (string vs integer)
❌ Forgetting dependency conditions when using variables
❌ Using variables when parameters would be more appropriate
❌ Not handling null/empty values
❌ Overly complex variable logic (consider breaking into multiple pipelines)

### Interview Red Flags to Avoid

❌ "Variables and parameters are the same thing"
❌ Not knowing you can't set variables in parallel
❌ Never having used Append Variable
❌ Not understanding variable scope
❌ Unable to explain when to use variables vs parameters
❌ Not knowing how to debug variable values

---

## Summary

These three scenarios—**Timeouts**, **Conditional Execution**, and **Variables Usage**—are fundamental to building robust, production-ready ADF pipelines. They enable you to:

- **Control execution time** and handle long-running operations gracefully
- **Implement complex business logic** without writing code
- **Manage state and track progress** throughout pipeline execution

Mastering these patterns will significantly improve your ability to handle real-world data integration challenges and demonstrate deep ADF expertise in interviews.

### Key Takeaways

1. **Timeouts**: Always set explicit timeouts, handle timeout errors differently from other failures, and implement progressive retry strategies
2. **Conditional Execution**: Use If Condition for business logic, dependency conditions for error handling, and combine both for comprehensive control flow
3. **Variables**: Use for runtime state management, understand scope limitations, and know when to use variables vs parameters

### Practice Recommendations

1. Build a pipeline that implements all three patterns together
2. Create error scenarios and test your handling logic
3. Monitor variable values through complex execution paths
4. Practice explaining your design decisions in interview-style scenarios

By mastering these scenarios, you'll be well-equipped to handle the majority of real-world ADF pipeline requirements and confidently discuss your "experience" in interviews.

