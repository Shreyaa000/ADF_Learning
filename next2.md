# ADF-07: Real-World Scenarios - Part 4

## Scenario 22: Cross-Region Replication

### Business Requirement
A multinational company needs to replicate critical sales data from their primary Azure region (East US) to a disaster recovery region (West Europe) for business continuity. Data must be synchronized daily, and the solution should handle network failures gracefully.

### Implementation Steps

#### Step 1: Set Up Source Linked Service (East US)
```json
{
  "name": "LS_ADLS_EastUS",
  "type": "AzureBlobFS",
  "typeProperties": {
    "url": "https://storageeastus.dfs.core.windows.net",
    "accountKey": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AKV_EastUS",
        "type": "LinkedServiceReference"
      },
      "secretName": "storage-account-key"
    }
  }
}
```

#### Step 2: Set Up Destination Linked Service (West Europe)
```json
{
  "name": "LS_ADLS_WestEurope",
  "type": "AzureBlobFS",
  "typeProperties": {
    "url": "https://storagewesteurope.dfs.core.windows.net",
    "accountKey": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AKV_WestEurope",
        "type": "LinkedServiceReference"
      },
      "secretName": "storage-account-key"
    }
  }
}
```

#### Step 3: Create Parameterized Datasets
**Source Dataset:**
```json
{
  "name": "DS_Source_CrossRegion",
  "properties": {
    "linkedServiceName": {
      "referenceName": "LS_ADLS_EastUS",
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
        "type": "AzureBlobFSLocation",
        "folderPath": {
          "value": "@dataset().folderPath",
          "type": "Expression"
        },
        "fileName": {
          "value": "@dataset().fileName",
          "type": "Expression"
        }
      }
    }
  }
}
```

**Destination Dataset:**
```json
{
  "name": "DS_Destination_CrossRegion",
  "properties": {
    "linkedServiceName": {
      "referenceName": "LS_ADLS_WestEurope",
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
        "type": "AzureBlobFSLocation",
        "folderPath": {
          "value": "@dataset().folderPath",
          "type": "Expression"
        },
        "fileName": {
          "value": "@dataset().fileName",
          "type": "Expression"
        }
      }
    }
  }
}
```

#### Step 4: Build the Cross-Region Replication Pipeline
```json
{
  "name": "PL_CrossRegion_Replication",
  "properties": {
    "activities": [
      {
        "name": "GetMetadata_SourceFiles",
        "type": "GetMetadata",
        "typeProperties": {
          "dataset": {
            "referenceName": "DS_Source_CrossRegion",
            "type": "DatasetReference",
            "parameters": {
              "folderPath": "sales-data",
              "fileName": "*"
            }
          },
          "fieldList": ["childItems"],
          "storeSettings": {
            "type": "AzureBlobFSReadSettings",
            "recursive": true
          }
        }
      },
      {
        "name": "ForEach_File",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "GetMetadata_SourceFiles",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "items": {
            "value": "@activity('GetMetadata_SourceFiles').output.childItems",
            "type": "Expression"
          },
          "isSequential": false,
          "batchCount": 20,
          "activities": [
            {
              "name": "Copy_ToWestEurope",
              "type": "Copy",
              "policy": {
                "timeout": "0.12:00:00",
                "retry": 3,
                "retryIntervalInSeconds": 30
              },
              "typeProperties": {
                "source": {
                  "type": "DelimitedTextSource"
                },
                "sink": {
                  "type": "DelimitedTextSink",
                  "storeSettings": {
                    "type": "AzureBlobFSWriteSettings",
                    "copyBehavior": "PreserveHierarchy"
                  }
                },
                "enableStaging": false,
                "dataIntegrationUnits": 4
              },
              "inputs": [
                {
                  "referenceName": "DS_Source_CrossRegion",
                  "type": "DatasetReference",
                  "parameters": {
                    "folderPath": "sales-data",
                    "fileName": {
                      "value": "@item().name",
                      "type": "Expression"
                    }
                  }
                }
              ],
              "outputs": [
                {
                  "referenceName": "DS_Destination_CrossRegion",
                  "type": "DatasetReference",
                  "parameters": {
                    "folderPath": "sales-data-replica",
                    "fileName": {
                      "value": "@item().name",
                      "type": "Expression"
                    }
                  }
                }
              ]
            }
          ]
        }
      },
      {
        "name": "LogSuccess",
        "type": "WebActivity",
        "dependsOn": [
          {
            "activity": "ForEach_File",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "url": "https://your-logging-endpoint.com/log",
          "method": "POST",
          "body": {
            "value": "@concat('Cross-region replication completed at ', utcnow())",
            "type": "Expression"
          }
        }
      }
    ]
  }
}
```

### Expected Behavior
- Pipeline discovers all files in the source region
- Files are copied in parallel (batch of 20) to destination region
- Each copy activity retries 3 times on failure
- Success is logged after all files are replicated
- Network latency between regions is handled by retry policy

### Common Issues and Resolutions

**Issue 1: Network Timeout During Large File Transfer**
- **Error**: "Copy activity failed with error: Connection timeout"
- **Solution**: Increase timeout value and reduce DIU if network is unstable
```json
"policy": {
  "timeout": "1.00:00:00",
  "retry": 5,
  "retryIntervalInSeconds": 60
},
"dataIntegrationUnits": 2
```

**Issue 2: Inconsistent File Versions**
- **Problem**: Source file changes during copy operation
- **Solution**: Implement snapshot-based copying or use GetMetadata to check lastModified timestamp before and after copy

**Issue 3: Cost Concerns**
- **Problem**: High egress charges for cross-region data transfer
- **Solution**: 
  - Use Azure Private Link for inter-region transfer
  - Schedule replication during off-peak hours
  - Implement incremental replication instead of full copy

### Pro Tips
💡 **Use Azure IR in the source region** for optimal performance
💡 **Enable logging to track replication status** and create audit trails
💡 **Consider using AzCopy for very large files** (TB+) and trigger via ADF
💡 **Implement checksum validation** after copy to ensure data integrity

### Interview Questions You Might Face

**Q1: How do you handle network failures during cross-region replication?**
**Answer**: "I implement a multi-layered approach: First, I configure retry policies with exponential backoff (3-5 retries with 30-60 second intervals). Second, I use Azure IR's built-in resilience features. Third, I implement a checkpoint mechanism using a control table that tracks which files were successfully replicated, so we can resume from where we left off. Finally, I set up Azure Monitor alerts to notify the team of persistent failures."

**Q2: What are the cost implications of cross-region replication?**
**Answer**: "Cross-region data transfer incurs egress charges from the source region, typically $0.05-$0.08 per GB depending on regions. To optimize costs, I use several strategies: compress data before transfer, implement incremental replication to minimize data volume, use Azure Private Link to reduce egress costs, and schedule replication during off-peak hours. I also use lifecycle policies to archive old replicated data to cool/archive tiers."

**Q3: How do you ensure data consistency across regions?**
**Answer**: "I implement a multi-step validation process: First, I use GetMetadata to capture file size and lastModified timestamp before copy. After copy, I validate the destination file has the same size. For critical data, I implement checksum validation using MD5 hashes. I also maintain a replication log table that records source and destination file metadata, copy timestamps, and validation status. For transactional consistency, I use a staging-commit pattern where files are copied to a staging folder first, validated, then moved to the final location."

---

## Scenario 23: Data Quality Checks

### Business Requirement
An insurance company needs to validate incoming claims data before loading it into their data warehouse. Data must pass multiple quality checks including completeness, accuracy, consistency, and business rule validation. Invalid records should be quarantined for manual review.

### Implementation Steps

#### Step 1: Create Quality Check Configuration Table
```sql
CREATE TABLE [dbo].[DataQualityRules] (
    RuleID INT PRIMARY KEY,
    RuleName VARCHAR(100),
    RuleType VARCHAR(50), -- NULL_CHECK, RANGE_CHECK, FORMAT_CHECK, BUSINESS_RULE
    TableName VARCHAR(100),
    ColumnName VARCHAR(100),
    RuleExpression VARCHAR(500),
    IsActive BIT,
    Severity VARCHAR(20) -- ERROR, WARNING
);

-- Sample rules
INSERT INTO [dbo].[DataQualityRules] VALUES
(1, 'ClaimAmount_NotNull', 'NULL_CHECK', 'Claims', 'ClaimAmount', 'ClaimAmount IS NOT NULL', 1, 'ERROR'),
(2, 'ClaimAmount_Positive', 'RANGE_CHECK', 'Claims', 'ClaimAmount', 'ClaimAmount > 0', 1, 'ERROR'),
(3, 'ClaimDate_Valid', 'RANGE_CHECK', 'Claims', 'ClaimDate', 'ClaimDate >= ''2020-01-01'' AND ClaimDate <= GETDATE()', 1, 'ERROR'),
(4, 'Email_Format', 'FORMAT_CHECK', 'Claims', 'CustomerEmail', 'CustomerEmail LIKE ''%@%.%''', 1, 'WARNING'),
(5, 'PolicyNumber_Length', 'FORMAT_CHECK', 'Claims', 'PolicyNumber', 'LEN(PolicyNumber) = 10', 1, 'ERROR');
```

#### Step 2: Create Quality Check Results Table
```sql
CREATE TABLE [dbo].[DataQualityResults] (
    CheckID INT IDENTITY(1,1) PRIMARY KEY,
    PipelineRunID VARCHAR(100),
    RuleID INT,
    CheckTimestamp DATETIME,
    TotalRecords INT,
    FailedRecords INT,
    PassRate DECIMAL(5,2),
    Status VARCHAR(20), -- PASSED, FAILED, WARNING
    ErrorDetails VARCHAR(MAX)
);
```

#### Step 3: Create Quarantine Table
```sql
CREATE TABLE [dbo].[Claims_Quarantine] (
    QuarantineID INT IDENTITY(1,1) PRIMARY KEY,
    OriginalRecordID INT,
    ClaimData VARCHAR(MAX), -- JSON representation of the record
    FailedRules VARCHAR(500),
    QuarantineDate DATETIME,
    ReviewStatus VARCHAR(20), -- PENDING, APPROVED, REJECTED
    ReviewedBy VARCHAR(100),
    ReviewDate DATETIME
);
```

#### Step 4: Build Data Quality Pipeline

**Pipeline Structure:**
```json
{
  "name": "PL_DataQuality_Claims",
  "properties": {
    "parameters": {
      "SourceFileName": {
        "type": "string"
      }
    },
    "variables": {
      "TotalRecords": {
        "type": "Integer"
      },
      "ValidRecords": {
        "type": "Integer"
      },
      "InvalidRecords": {
        "type": "Integer"
      }
    },
    "activities": [
      {
        "name": "Copy_ToStaging",
        "type": "Copy",
        "typeProperties": {
          "source": {
            "type": "DelimitedTextSource"
          },
          "sink": {
            "type": "AzureSqlSink",
            "preCopyScript": "TRUNCATE TABLE [staging].[Claims_Raw]"
          }
        }
      },
      {
        "name": "GetMetadata_RecordCount",
        "type": "Lookup",
        "dependsOn": [
          {
            "activity": "Copy_ToStaging",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT COUNT(*) as RecordCount FROM [staging].[Claims_Raw]"
          }
        }
      },
      {
        "name": "Set_TotalRecords",
        "type": "SetVariable",
        "dependsOn": [
          {
            "activity": "GetMetadata_RecordCount",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "variableName": "TotalRecords",
          "value": {
            "value": "@activity('GetMetadata_RecordCount').output.firstRow.RecordCount",
            "type": "Expression"
          }
        }
      },
      {
        "name": "Lookup_QualityRules",
        "type": "Lookup",
        "dependsOn": [
          {
            "activity": "Set_TotalRecords",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT * FROM [dbo].[DataQualityRules] WHERE IsActive = 1 AND TableName = 'Claims'"
          },
          "firstRowOnly": false
        }
      },
      {
        "name": "ForEach_QualityRule",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "Lookup_QualityRules",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "items": {
            "value": "@activity('Lookup_QualityRules').output.value",
            "type": "Expression"
          },
          "isSequential": true,
          "activities": [
            {
              "name": "Execute_QualityCheck",
              "type": "SqlServerStoredProcedure",
              "typeProperties": {
                "storedProcedureName": "[dbo].[sp_ExecuteQualityCheck]",
                "storedProcedureParameters": {
                  "RuleID": {
                    "value": {
                      "value": "@item().RuleID",
                      "type": "Expression"
                    },
                    "type": "Int32"
                  },
                  "RuleExpression": {
                    "value": {
                      "value": "@item().RuleExpression",
                      "type": "Expression"
                    },
                    "type": "String"
                  },
                  "PipelineRunID": {
                    "value": {
                      "value": "@pipeline().RunId",
                      "type": "Expression"
                    },
                    "type": "String"
                  }
                }
              }
            }
          ]
        }
      },
      {
        "name": "Check_QualityResults",
        "type": "Lookup",
        "dependsOn": [
          {
            "activity": "ForEach_QualityRule",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
              "value": "SELECT COUNT(*) as FailedChecks FROM [dbo].[DataQualityResults] WHERE PipelineRunID = '@{pipeline().RunId}' AND Status = 'FAILED' AND Severity = 'ERROR'",
              "type": "Expression"
            }
          }
        }
      },
      {
        "name": "If_QualityPassed",
        "type": "IfCondition",
        "dependsOn": [
          {
            "activity": "Check_QualityResults",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "expression": {
            "value": "@equals(activity('Check_QualityResults').output.firstRow.FailedChecks, 0)",
            "type": "Expression"
          },
          "ifTrueActivities": [
            {
              "name": "Load_ToProduction",
              "type": "SqlServerStoredProcedure",
              "typeProperties": {
                "storedProcedureName": "[dbo].[sp_LoadClaimsToProduction]"
              }
            }
          ],
          "ifFalseActivities": [
            {
              "name": "Quarantine_InvalidRecords",
              "type": "SqlServerStoredProcedure",
              "typeProperties": {
                "storedProcedureName": "[dbo].[sp_QuarantineInvalidRecords]",
                "storedProcedureParameters": {
                  "PipelineRunID": {
                    "value": {
                      "value": "@pipeline().RunId",
                      "type": "Expression"
                    },
                    "type": "String"
                  }
                }
              }
            },
            {
              "name": "Send_QualityAlert",
              "type": "WebActivity",
              "dependsOn": [
                {
                  "activity": "Quarantine_InvalidRecords",
                  "dependencyConditions": ["Succeeded"]
                }
              ],
              "typeProperties": {
                "url": "https://prod-xx.eastus.logic.azure.com:443/workflows/xxx",
                "method": "POST",
                "body": {
                  "value": "@concat('Data quality check failed. Pipeline: ', pipeline().Pipeline, ', Run ID: ', pipeline().RunId)",
                  "type": "Expression"
                }
              }
            }
          ]
        }
      }
    ]
  }
}
```

#### Step 5: Create Quality Check Stored Procedure
```sql
CREATE PROCEDURE [dbo].[sp_ExecuteQualityCheck]
    @RuleID INT,
    @RuleExpression VARCHAR(500),
    @PipelineRunID VARCHAR(100)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @TotalRecords INT;
    DECLARE @FailedRecords INT;
    DECLARE @PassRate DECIMAL(5,2);
    DECLARE @Status VARCHAR(20);
    
    -- Get total records
    SELECT @TotalRecords = COUNT(*) FROM [staging].[Claims_Raw];
    
    -- Build dynamic SQL to check rule
    SET @SQL = N'SELECT @FailedRecords = COUNT(*) FROM [staging].[Claims_Raw] WHERE NOT (' + @RuleExpression + ')';
    
    EXEC sp_executesql @SQL, N'@FailedRecords INT OUTPUT', @FailedRecords OUTPUT;
    
    -- Calculate pass rate
    SET @PassRate = CASE WHEN @TotalRecords > 0 
                         THEN ((@TotalRecords - @FailedRecords) * 100.0 / @TotalRecords)
                         ELSE 100 END;
    
    -- Determine status
    SET @Status = CASE 
                    WHEN @FailedRecords = 0 THEN 'PASSED'
                    WHEN @PassRate >= 95 THEN 'WARNING'
                    ELSE 'FAILED'
                  END;
    
    -- Log results
    INSERT INTO [dbo].[DataQualityResults] 
    (PipelineRunID, RuleID, CheckTimestamp, TotalRecords, FailedRecords, PassRate, Status)
    VALUES 
    (@PipelineRunID, @RuleID, GETDATE(), @TotalRecords, @FailedRecords, @PassRate, @Status);
END;
```

#### Step 6: Create Quarantine Procedure
```sql
CREATE PROCEDURE [dbo].[sp_QuarantineInvalidRecords]
    @PipelineRunID VARCHAR(100)
AS
BEGIN
    -- Identify failed rules for this run
    DECLARE @FailedRules TABLE (RuleID INT, RuleExpression VARCHAR(500));
    
    INSERT INTO @FailedRules
    SELECT r.RuleID, r.RuleExpression
    FROM [dbo].[DataQualityResults] qr
    INNER JOIN [dbo].[DataQualityRules] r ON qr.RuleID = r.RuleID
    WHERE qr.PipelineRunID = @PipelineRunID 
      AND qr.Status = 'FAILED'
      AND r.Severity = 'ERROR';
    
    -- Move invalid records to quarantine
    INSERT INTO [dbo].[Claims_Quarantine] 
    (OriginalRecordID, ClaimData, FailedRules, QuarantineDate, ReviewStatus)
    SELECT 
        c.ClaimID,
        (SELECT * FROM [staging].[Claims_Raw] WHERE ClaimID = c.ClaimID FOR JSON PATH) as ClaimData,
        STRING_AGG(fr.RuleID, ',') as FailedRules,
        GETDATE(),
        'PENDING'
    FROM [staging].[Claims_Raw] c
    CROSS APPLY @FailedRules fr
    WHERE NOT EXISTS (
        SELECT 1 
        FROM [staging].[Claims_Raw] 
        WHERE ClaimID = c.ClaimID 
        -- Check if record passes all rules
    )
    GROUP BY c.ClaimID;
END;
```

### Expected Behavior
1. Data is copied to staging table
2. Record count is captured
3. All active quality rules are retrieved
4. Each rule is executed sequentially
5. Results are logged to DataQualityResults table
6. If all ERROR-level checks pass, data loads to production
7. If any ERROR-level checks fail, invalid records are quarantined
8. Alert is sent for quality failures

### Common Issues and Resolutions

**Issue 1: Performance Degradation with Many Rules**
- **Problem**: Pipeline takes too long when checking 50+ rules
- **Solution**: 
  - Combine similar rules into single SQL statements
  - Use parallel execution for independent rule categories
  - Create indexed views for complex validations
  - Implement sampling for very large datasets (check 10% of records)

**Issue 2: False Positives in Quality Checks**
- **Problem**: Valid records flagged as invalid due to overly strict rules
- **Solution**: 
  - Implement rule severity levels (ERROR vs WARNING)
  - Add business rule exceptions table
  - Use confidence scores instead of binary pass/fail
  - Implement human-in-the-loop review for edge cases

**Issue 3: Handling Null vs Empty String**
- **Problem**: NULL checks don't catch empty strings
- **Solution**:
```sql
-- Better null check
'(ColumnName IS NOT NULL AND LTRIM(RTRIM(ColumnName)) <> '''')'
```

### Pro Tips
💡 **Use metadata-driven approach** for scalable quality framework
💡 **Implement quality score dashboards** using Power BI connected to DataQualityResults
💡 **Create data quality SLAs** (e.g., 99.5% pass rate required)
💡 **Use Data Flow for complex transformations** during quality checks
💡 **Implement auto-correction** for common issues (trim spaces, fix date formats)

### Interview Questions You Might Face

**Q1: How do you design a scalable data quality framework in ADF?**
**Answer**: "I design a metadata-driven framework with three key components: First, a configuration table that stores all quality rules with their expressions, severity levels, and target tables. Second, a reusable pipeline that reads these rules and executes them dynamically using stored procedures or Data Flow. Third, a logging and monitoring layer that tracks quality metrics over time. This approach allows business users to add new rules without modifying pipelines. I also implement different rule types: schema validation, null checks, range checks, referential integrity, and business rules. For performance, I use parallel execution for independent rules and sampling for very large datasets."

**Q2: What's your approach to handling data quality failures?**
**Answer**: "I implement a multi-tier approach based on severity. For ERROR-level failures, I quarantine the invalid records and halt the load to production, sending immediate alerts to the data team. For WARNING-level issues, I load the data but flag it for review and notify stakeholders. I maintain a quarantine table where invalid records are stored with details about which rules they failed. This allows business users to review and either correct the data or approve exceptions. I also track quality metrics over time to identify systemic issues. For example, if a particular rule fails consistently, it might indicate a problem with the source system or the rule itself needs adjustment."

**Q3: How do you balance data quality checks with pipeline performance?**
**Answer**: "It's always a trade-off. I use several optimization strategies: First, I implement checks at the right stage - basic format validation during copy activity using schema validation, complex business rules after loading to staging. Second, I use sampling for very large datasets where checking 100% isn't necessary. Third, I combine multiple checks into single SQL statements to reduce round trips. Fourth, I use parallel execution for independent rule categories. Fifth, I implement incremental checking - only validate new/changed records. I also use Data Flow for complex transformations as it's optimized for large-scale data processing. The key is to define quality SLAs with business stakeholders and optimize to meet those SLAs without over-engineering."

---

## Scenario 24: Orchestrating Multiple Pipelines

### Business Requirement
A retail company has separate pipelines for loading customer data, product data, sales transactions, and inventory data. These pipelines must execute in a specific order with dependencies, error handling, and the ability to run certain pipelines in parallel. The orchestration should handle partial failures and provide comprehensive logging.

### Implementation Steps

#### Step 1: Create Control Table for Pipeline Orchestration
```sql
CREATE TABLE [dbo].[PipelineOrchestration] (
    PipelineID INT PRIMARY KEY,
    PipelineName VARCHAR(100),
    ExecutionOrder INT,
    DependsOnPipelineID INT NULL, -- NULL means no dependency
    CanRunInParallel BIT,
    IsActive BIT,
    RetryCount INT DEFAULT 3,
    TimeoutMinutes INT DEFAULT 60,
    IsCritical BIT -- If true, failure stops entire orchestration
);

-- Sample configuration
INSERT INTO [dbo].[PipelineOrchestration] VALUES
(1, 'PL_Load_Customers', 1, NULL, 1, 1, 3, 30, 1),
(2, 'PL_Load_Products', 1, NULL, 1, 1, 3, 30, 1),
(3, 'PL_Load_Stores', 1, NULL, 1, 1, 3, 20, 1),
(4, 'PL_Load_Inventory', 2, 2, 1, 1, 3, 45, 1), -- Depends on Products
(5, 'PL_Load_Sales', 2, 1, 1, 1, 3, 60, 1),     -- Depends on Customers
(6, 'PL_Load_SalesDetails', 3, 5, 0, 1, 3, 90, 1), -- Depends on Sales
(7, 'PL_Aggregate_DailySales', 4, 6, 0, 1, 2, 30, 0),
(8, 'PL_Update_DataMart', 5, 7, 0, 1, 2, 45, 0);
```

#### Step 2: Create Execution Log Table
```sql
CREATE TABLE [dbo].[PipelineExecutionLog] (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    OrchestrationRunID VARCHAR(100),
    PipelineID INT,
    PipelineName VARCHAR(100),
    StartTime DATETIME,
    EndTime DATETIME,
    Status VARCHAR(20), -- RUNNING, SUCCEEDED, FAILED, SKIPPED
    ErrorMessage VARCHAR(MAX),
    RowsProcessed INT,
    DurationSeconds INT
);
```

#### Step 3: Build Master Orchestration Pipeline

**Master Pipeline JSON:**
```json
{
  "name": "PL_Master_Orchestration",
  "properties": {
    "parameters": {
      "ExecutionDate": {
        "type": "string",
        "defaultValue": "@formatDateTime(utcnow(), 'yyyy-MM-dd')"
      }
    },
    "variables": {
      "OrchestrationRunID": {
        "type": "String"
      },
      "CurrentExecutionOrder": {
        "type": "Integer"
      },
      "HasFailures": {
        "type": "Boolean",
        "defaultValue": false
      }
    },
    "activities": [
      {
        "name": "Set_OrchestrationRunID",
        "type": "SetVariable",
        "typeProperties": {
          "variableName": "OrchestrationRunID",
          "value": {
            "value": "@pipeline().RunId",
            "type": "Expression"
          }
        }
      },
      {
        "name": "Get_MaxExecutionOrder",
        "type": "Lookup",
        "dependsOn": [
          {
            "activity": "Set_OrchestrationRunID",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT MAX(ExecutionOrder) as MaxOrder FROM [dbo].[PipelineOrchestration] WHERE IsActive = 1"
          }
        }
      },
      {
        "name": "Until_AllOrdersComplete",
        "type": "Until",
        "dependsOn": [
          {
            "activity": "Get_MaxExecutionOrder",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "expression": {
            "value": "@or(greater(variables('CurrentExecutionOrder'), activity('Get_MaxExecutionOrder').output.firstRow.MaxOrder), variables('HasFailures'))",
            "type": "Expression"
          },
          "activities": [
            {
              "name": "Increment_ExecutionOrder",
              "type": "SetVariable",
              "typeProperties": {
                "variableName": "CurrentExecutionOrder",
                "value": {
                  "value": "@add(variables('CurrentExecutionOrder'), 1)",
                  "type": "Expression"
                }
              }
            },
            {
              "name": "Get_PipelinesForCurrentOrder",
              "type": "Lookup",
              "dependsOn": [
                {
                  "activity": "Increment_ExecutionOrder",
                  "dependencyConditions": ["Succeeded"]
                }
              ],
              "typeProperties": {
                "source": {
                  "type": "AzureSqlSource",
                  "sqlReaderQuery": {
                    "value": "SELECT * FROM [dbo].[PipelineOrchestration] WHERE ExecutionOrder = @{variables('CurrentExecutionOrder')} AND IsActive = 1",
                    "type": "Expression"
                  }
                },
                "firstRowOnly": false
              }
            },
            {
              "name": "Execute_PipelinesInOrder",
              "type": "ForEach",
              "dependsOn": [
                {
                  "activity": "Get_PipelinesForCurrentOrder",
                  "dependencyConditions": ["Succeeded"]
                }
              ],
              "typeProperties": {
                "items": {
                  "value": "@activity('Get_PipelinesForCurrentOrder').output.value",
                  "type": "Expression"
                },
                "isSequential": {
                  "value": "@not(activity('Get_PipelinesForCurrentOrder').output.value[0].CanRunInParallel)",
                  "type": "Expression"
                },
                "batchCount": 10,
                "activities": [
                  {
                    "name": "Check_Dependencies",
                    "type": "IfCondition",
                    "typeProperties": {
                      "expression": {
                        "value": "@or(equals(item().DependsOnPipelineID, null), equals(activity('Check_DependencyStatus').output.firstRow.Status, 'SUCCEEDED'))",
                        "type": "Expression"
                      },
                      "ifTrueActivities": [
                        {
                          "name": "Execute_ChildPipeline",
                          "type": "ExecutePipeline",
                          "typeProperties": {
                            "pipeline": {
                              "referenceName": {
                                "value": "@item().PipelineName",
                                "type": "Expression"
                              },
                              "type": "PipelineReference"
                            },
                            "waitOnCompletion": true,
                            "parameters": {
                              "ExecutionDate": {
                                "value": "@pipeline().parameters.ExecutionDate",
                                "type": "Expression"
                              }
                            }
                          },
                          "policy": {
                            "timeout": {
                              "value": "@concat('0.00:', item().TimeoutMinutes, ':00')",
                              "type": "Expression"
                            },
                            "retry": {
                              "value": "@item().RetryCount",
                              "type": "Expression"
                            },
                            "retryIntervalInSeconds": 30
                          }
                        },
                        {
                          "name": "Log_Success",
                          "type": "SqlServerStoredProcedure",
                          "dependsOn": [
                            {
                              "activity": "Execute_ChildPipeline",
                              "dependencyConditions": ["Succeeded"]
                            }
                          ],
                          "typeProperties": {
                            "storedProcedureName": "[dbo].[sp_LogPipelineExecution]",
                            "storedProcedureParameters": {
                              "OrchestrationRunID": {
                                "value": {
                                  "value": "@variables('OrchestrationRunID')",
                                  "type": "Expression"
                                },
                                "type": "String"
                              },
                              "PipelineID": {
                                "value": {
                                  "value": "@item().PipelineID",
                                  "type": "Expression"
                                },
                                "type": "Int32"
                              },
                              "Status": {
                                "value": "SUCCEEDED",
                                "type": "String"
                              },
                              "ErrorMessage": {
                                "value": null,
                                "type": "String"
                              }
                            }
                          }
                        }
                      ],
                      "ifFalseActivities": [
                        {
                          "name": "Log_Skipped",
                          "type": "SqlServerStoredProcedure",
                          "typeProperties": {
                            "storedProcedureName": "[dbo].[sp_LogPipelineExecution]",
                            "storedProcedureParameters": {
                              "OrchestrationRunID": {
                                "value": {
                                  "value": "@variables('OrchestrationRunID')",
                                  "type": "Expression"
                                },
                                "type": "String"
                              },
                              "PipelineID": {
                                "value": {
                                  "value": "@item().PipelineID",
                                  "type": "Expression"
                                },
                                "type": "Int32"
                              },
                              "Status": {
                                "value": "SKIPPED",
                                "type": "String"
                              },
                              "ErrorMessage": {
                                "value": "Dependency pipeline failed",
                                "type": "String"
                              }
                            }
                          }
                        }
                      ]
                    }
                  }
                ]
              }
            }
          ],
          "timeout": "0.12:00:00"
        }
      },
      {
        "name": "Send_CompletionNotification",
        "type": "WebActivity",
        "dependsOn": [
          {
            "activity": "Until_AllOrdersComplete",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "url": "https://prod-xx.eastus.logic.azure.com:443/workflows/xxx",
          "method": "POST",
          "body": {
            "value": "@concat('Orchestration completed. Run ID: ', variables('OrchestrationRunID'))",
            "type": "Expression"
          }
        }
      }
    ]
  }
}
```

#### Step 4: Create Individual Child Pipelines (Example)

**Child Pipeline Template:**
```json
{
  "name": "PL_Load_Customers",
  "properties": {
    "parameters": {
      "ExecutionDate": {
        "type": "string"
      }
    },
    "activities": [
      {
        "name": "Copy_Customers",
        "type": "Copy",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
              "value": "SELECT * FROM [source].[Customers] WHERE ModifiedDate >= '@{pipeline().parameters.ExecutionDate}'",
              "type": "Expression"
            }
          },
          "sink": {
            "type": "AzureSqlSink",
            "preCopyScript": "TRUNCATE TABLE [staging].[Customers]"
          }
        }
      },
      {
        "name": "Transform_Customers",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [
          {
            "activity": "Copy_Customers",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "storedProcedureName": "[dbo].[sp_TransformCustomers]"
        }
      }
    ]
  }
}
```

#### Step 5: Create Logging Stored Procedure
```sql
CREATE PROCEDURE [dbo].[sp_LogPipelineExecution]
    @OrchestrationRunID VARCHAR(100),
    @PipelineID INT,
    @Status VARCHAR(20),
    @ErrorMessage VARCHAR(MAX) = NULL
AS
BEGIN
    DECLARE @PipelineName VARCHAR(100);
    DECLARE @StartTime DATETIME;
    DECLARE @DurationSeconds INT;
    
    SELECT @PipelineName = PipelineName FROM [dbo].[PipelineOrchestration] WHERE PipelineID = @PipelineID;
    
    -- Check if already logged (for retry scenarios)
    IF EXISTS (SELECT 1 FROM [dbo].[PipelineExecutionLog] 
               WHERE OrchestrationRunID = @OrchestrationRunID 
               AND PipelineID = @PipelineID 
               AND Status = 'RUNNING')
    BEGIN
        -- Update existing record
        UPDATE [dbo].[PipelineExecutionLog]
        SET EndTime = GETDATE(),
            Status = @Status,
            ErrorMessage = @ErrorMessage,
            DurationSeconds = DATEDIFF(SECOND, StartTime, GETDATE())
        WHERE OrchestrationRunID = @OrchestrationRunID 
        AND PipelineID = @PipelineID 
        AND Status = 'RUNNING';
    END
    ELSE
    BEGIN
        -- Insert new record
        INSERT INTO [dbo].[PipelineExecutionLog]
        (OrchestrationRunID, PipelineID, PipelineName, StartTime, Status)
        VALUES
        (@OrchestrationRunID, @PipelineID, @PipelineName, GETDATE(), @Status);
    END
END;
```

### Expected Behavior
1. Master pipeline starts and generates unique orchestration run ID
2. Pipelines are executed in order based on ExecutionOrder column
3. Pipelines with same ExecutionOrder and CanRunInParallel=1 execute simultaneously
4. Dependencies are checked before execution
5. Failed pipelines retry based on RetryCount configuration
6. If critical pipeline fails, entire orchestration stops
7. Non-critical pipeline failures are logged but don't stop orchestration
8. Completion notification sent at the end

### Common Issues and Resolutions

**Issue 1: Circular Dependencies**
- **Problem**: Pipeline A depends on B, B depends on C, C depends on A
- **Solution**: 
  - Implement dependency validation in control table
  - Create SQL check constraint or trigger
  ```sql
  CREATE TRIGGER trg_CheckCircularDependency
  ON [dbo].[PipelineOrchestration]
  AFTER INSERT, UPDATE
  AS
  BEGIN
      -- Recursive CTE to detect circular dependencies
      WITH DependencyChain AS (
          SELECT PipelineID, DependsOnPipelineID, 1 as Level
          FROM inserted
          UNION ALL
          SELECT dc.PipelineID, po.DependsOnPipelineID, dc.Level + 1
          FROM DependencyChain dc
          INNER JOIN [dbo].[PipelineOrchestration] po 
              ON dc.DependsOnPipelineID = po.PipelineID
          WHERE dc.Level < 100
      )
      IF EXISTS (SELECT 1 FROM DependencyChain WHERE PipelineID = DependsOnPipelineID)
      BEGIN
          RAISERROR('Circular dependency detected', 16, 1);
          ROLLBACK TRANSACTION;
      END
  END;
  ```

**Issue 2: Long-Running Orchestration Timeout**
- **Problem**: Until activity times out after 12 hours
- **Solution**: 
  - Break orchestration into multiple master pipelines
  - Use tumbling window triggers for each execution order
  - Implement checkpoint/resume mechanism

**Issue 3: Partial Failure Handling**
- **Problem**: Some pipelines succeed, some fail - how to resume?
- **Solution**: 
  - Implement idempotent pipelines (can be re-run safely)
  - Add "Resume from failure" parameter
  - Check execution log before starting each pipeline
  ```sql
  -- Only execute if not already succeeded
  WHERE NOT EXISTS (
      SELECT 1 FROM [dbo].[PipelineExecutionLog]
      WHERE OrchestrationRunID = @OrchestrationRunID
      AND PipelineID = @PipelineID
      AND Status = 'SUCCEEDED'
  )
  ```

### Pro Tips
💡 **Use tumbling window triggers** for time-based orchestration instead of Until loops
💡 **Implement pipeline versioning** in control table for A/B testing
💡 **Create orchestration dashboard** showing real-time pipeline status
💡 **Use Azure Monitor** to track orchestration metrics and set up alerts
💡 **Implement circuit breaker pattern** - if too many failures, stop orchestration
💡 **Use parent pipeline parameters** to pass context to all child pipelines

### Interview Questions You Might Face

**Q1: How do you design a scalable pipeline orchestration framework?**
**Answer**: "I design a metadata-driven orchestration framework with several key components. First, a control table that defines all pipelines, their execution order, dependencies, and configurations like retry count and timeout. Second, a master orchestration pipeline that reads this metadata and executes child pipelines dynamically using Execute Pipeline activity. Third, a comprehensive logging system that tracks each pipeline execution with start time, end time, status, and error details. For scalability, I implement parallel execution for independent pipelines using ForEach with sequential=false. I also use execution order levels - pipelines at the same level can run in parallel if they have no dependencies. For very complex orchestrations, I break them into multiple master pipelines triggered by tumbling window triggers with dependencies."

**Q2: How do you handle failures in a multi-pipeline orchestration?**
**Answer**: "I implement a multi-layered failure handling strategy. First, I classify pipelines as critical or non-critical in the control table. Critical pipeline failures stop the entire orchestration, while non-critical failures are logged but don't block downstream pipelines. Second, I configure retry policies at the Execute Pipeline activity level - typically 2-3 retries with 30-second intervals. Third, I implement dependency checking - before executing a pipeline, I verify its dependencies succeeded. Fourth, I maintain detailed execution logs that enable resume-from-failure scenarios. Fifth, I send immediate alerts for critical failures using Logic Apps or Azure Monitor. Finally, I design all pipelines to be idempotent so they can be safely re-run without causing data duplication or corruption."

**Q3: What are the performance considerations for orchestrating many pipelines?**
**Answer**: "Several performance factors to consider: First, maximize parallelization - identify independent pipelines that can run simultaneously. I use the CanRunInParallel flag and ForEach with sequential=false. Second, avoid the Until activity for long-running orchestrations as it has a 12-hour timeout - instead use tumbling window triggers with dependencies. Third, minimize the overhead of Execute Pipeline activities by batching related operations within child pipelines rather than having too many small pipelines. Fourth, use appropriate timeout values - don't set unnecessarily long timeouts that delay failure detection. Fifth, implement smart dependency checking - cache dependency status rather than querying the database repeatedly. Finally, consider the ADF limits - maximum 40 concurrent pipeline runs per subscription, so plan your parallelization accordingly."

---

## Scenario 25: Retry Logic

### Business Requirement
A logistics company integrates with multiple external APIs that occasionally experience transient failures (network timeouts, rate limiting, temporary service unavailability). The data pipeline must implement intelligent retry logic with exponential backoff, circuit breaker patterns, and different retry strategies based on error types.

### Implementation Steps

#### Step 1: Create Retry Configuration Table
```sql
CREATE TABLE [dbo].[RetryConfiguration] (
    ConfigID INT PRIMARY KEY,
    ActivityType VARCHAR(50), -- API_CALL, COPY_ACTIVITY, STORED_PROC
    ErrorType VARCHAR(50), -- TIMEOUT, RATE_LIMIT, SERVICE_UNAVAILABLE, AUTHENTICATION
    MaxRetries INT,
    InitialRetryIntervalSeconds INT,
    MaxRetryIntervalSeconds INT,
    UseExponentialBackoff BIT,
    CircuitBreakerThreshold INT, -- Number of consecutive failures before circuit opens
    CircuitBreakerResetMinutes INT
);

-- Sample configurations
INSERT INTO [dbo].[RetryConfiguration] VALUES
(1, 'API_CALL', 'TIMEOUT', 5, 10, 300, 1, 10, 30),
(2, 'API_CALL', 'RATE_LIMIT', 10, 60, 600, 1, 5, 60),
(3, 'API_CALL', 'SERVICE_UNAVAILABLE', 3, 30, 180, 1, 15, 60),
(4, 'COPY_ACTIVITY', 'TIMEOUT', 3, 30, 120, 1, 5, 15),
(5, 'STORED_PROC', 'DEADLOCK', 5, 5, 30, 1, 0, 0);
```

#### Step 2: Create Retry Log Table
```sql
CREATE TABLE [dbo].[RetryLog] (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    PipelineRunID VARCHAR(100),
    ActivityName VARCHAR(100),
    AttemptNumber INT,
    AttemptTime DATETIME,
    Status VARCHAR(20), -- SUCCESS, FAILED, RETRYING
    ErrorCode VARCHAR(50),
    ErrorMessage VARCHAR(MAX),
    RetryIntervalSeconds INT,
    NextRetryTime DATETIME
);
```

#### Step 3: Create Circuit Breaker State Table
```sql
CREATE TABLE [dbo].[CircuitBreakerState] (
    ServiceName VARCHAR(100) PRIMARY KEY,
    State VARCHAR(20), -- CLOSED, OPEN, HALF_OPEN
    ConsecutiveFailures INT,
    LastFailureTime DATETIME,
    CircuitOpenedTime DATETIME,
    NextRetryTime DATETIME
);
```

#### Step 4: Build Pipeline with Retry Logic

**Pipeline with Until Activity for Custom Retry:**
```json
{
  "name": "PL_API_Call_WithRetry",
  "properties": {
    "parameters": {
      "ApiEndpoint": {
        "type": "string"
      },
      "MaxRetries": {
        "type": "int",
        "defaultValue": 5
      }
    },
    "variables": {
      "AttemptCount": {
        "type": "Integer",
        "defaultValue": 0
      },
      "IsSuccess": {
        "type": "Boolean",
        "defaultValue": false
      },
      "RetryInterval": {
        "type": "Integer",
        "defaultValue": 10
      },
      "LastErrorMessage": {
        "type": "String"
      }
    },
    "activities": [
      {
        "name": "Check_CircuitBreaker",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
              "value": "SELECT State, NextRetryTime FROM [dbo].[CircuitBreakerState] WHERE ServiceName = '@{pipeline().parameters.ApiEndpoint}'",
              "type": "Expression"
            }
          }
        }
      },
      {
        "name": "If_CircuitOpen",
        "type": "IfCondition",
        "dependsOn": [
          {
            "activity": "Check_CircuitBreaker",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "expression": {
            "value": "@and(equals(activity('Check_CircuitBreaker').output.firstRow.State, 'OPEN'), less(utcnow(), activity('Check_CircuitBreaker').output.firstRow.NextRetryTime))",
            "type": "Expression"
          },
          "ifTrueActivities": [
            {
              "name": "Fail_CircuitOpen",
              "type": "Fail",
              "typeProperties": {
                "message": "Circuit breaker is OPEN. Service is temporarily unavailable.",
                "errorCode": "CIRCUIT_OPEN"
              }
            }
          ],
          "ifFalseActivities": [
            {
              "name": "Until_Success_Or_MaxRetries",
              "type": "Until",
              "typeProperties": {
                "expression": {
                  "value": "@or(variables('IsSuccess'), greaterOrEquals(variables('AttemptCount'), pipeline().parameters.MaxRetries))",
                  "type": "Expression"
                },
                "activities": [
                  {
                    "name": "Increment_AttemptCount",
                    "type": "SetVariable",
                    "typeProperties": {
                      "variableName": "AttemptCount",
                      "value": {
                        "value": "@add(variables('AttemptCount'), 1)",
                        "type": "Expression"
                      }
                    }
                  },
                  {
                    "name": "Log_Attempt",
                    "type": "SqlServerStoredProcedure",
                    "dependsOn": [
                      {
                        "activity": "Increment_AttemptCount",
                        "dependencyConditions": ["Succeeded"]
                      }
                    ],
                    "typeProperties": {
                      "storedProcedureName": "[dbo].[sp_LogRetryAttempt]",
                      "storedProcedureParameters": {
                        "PipelineRunID": {
                          "value": {
                            "value": "@pipeline().RunId",
                            "type": "Expression"
                          },
                          "type": "String"
                        },
                        "ActivityName": {
                          "value": "API_Call",
                          "type": "String"
                        },
                        "AttemptNumber": {
                          "value": {
                            "value": "@variables('AttemptCount')",
                            "type": "Expression"
                          },
                          "type": "Int32"
                        },
                        "Status": {
                          "value": "RETRYING",
                          "type": "String"
                        }
                      }
                    }
                  },
                  {
                    "name": "Try_API_Call",
                    "type": "WebActivity",
                    "dependsOn": [
                      {
                        "activity": "Log_Attempt",
                        "dependencyConditions": ["Succeeded"]
                      }
                    ],
                    "policy": {
                      "timeout": "0.00:05:00",
                      "retry": 0,
                      "retryIntervalInSeconds": 30
                    },
                    "typeProperties": {
                      "url": {
                        "value": "@pipeline().parameters.ApiEndpoint",
                        "type": "Expression"
                      },
                      "method": "GET",
                      "headers": {
                        "Authorization": "Bearer @{activity('Get_Token').output.token}"
                      }
                    }
                  },
                  {
                    "name": "Set_Success",
                    "type": "SetVariable",
                    "dependsOn": [
                      {
                        "activity": "Try_API_Call",
                        "dependencyConditions": ["Succeeded"]
                      }
                    ],
                    "typeProperties": {
                      "variableName": "IsSuccess",
                      "value": true
                    }
                  },
                  {
                    "name": "Log_Success",
                    "type": "SqlServerStoredProcedure",
                    "dependsOn": [
                      {
                        "activity": "Set_Success",
                        "dependencyConditions": ["Succeeded"]
                      }
                    ],
                    "typeProperties": {
                      "storedProcedureName": "[dbo].[sp_LogRetryAttempt]",
                      "storedProcedureParameters": {
                        "PipelineRunID": {
                          "value": {
                            "value": "@pipeline().RunId",
                            "type": "Expression"
                          },
                          "type": "String"
                        },
                        "ActivityName": {
                          "value": "API_Call",
                          "type": "String"
                        },
                        "AttemptNumber": {
                          "value": {
                            "value": "@variables('AttemptCount')",
                            "type": "Expression"
                          },
                          "type": "Int32"
                        },
                        "Status": {
                          "value": "SUCCESS",
                          "type": "String"
                        }
                      }
                    }
                  },
                  {
                    "name": "Reset_CircuitBreaker",
                    "type": "SqlServerStoredProcedure",
                    "dependsOn": [
                      {
                        "activity": "Log_Success",
                        "dependencyConditions": ["Succeeded"]
                      }
                    ],
                    "typeProperties": {
                      "storedProcedureName": "[dbo].[sp_ResetCircuitBreaker]",
                      "storedProcedureParameters": {
                        "ServiceName": {
                          "value": {
                            "value": "@pipeline().parameters.ApiEndpoint",
                            "type": "Expression"
                          },
                          "type": "String"
                        }
                      }
                    }
                  },
                  {
                    "name": "Handle_Failure",
                    "type": "IfCondition",
                    "dependsOn": [
                      {
                        "activity": "Try_API_Call",
                        "dependencyConditions": ["Failed"]
                      }
                    ],
                    "typeProperties": {
                      "expression": {
                        "value": "@less(variables('AttemptCount'), pipeline().parameters.MaxRetries)",
                        "type": "Expression"
                      },
                      "ifTrueActivities": [
                        {
                          "name": "Calculate_BackoffInterval",
                          "type": "SetVariable",
                          "typeProperties": {
                            "variableName": "RetryInterval",
                            "value": {
                              "value": "@min(mul(variables('RetryInterval'), 2), 300)",
                              "type": "Expression"
                            }
                          }
                        },
                        {
                          "name": "Wait_BeforeRetry",
                          "type": "Wait",
                          "dependsOn": [
                            {
                              "activity": "Calculate_BackoffInterval",
                              "dependencyConditions": ["Succeeded"]
                            }
                          ],
                          "typeProperties": {
                            "waitTimeInSeconds": {
                              "value": "@variables('RetryInterval')",
                              "type": "Expression"
                            }
                          }
                        },
                        {
                          "name": "Update_CircuitBreaker",
                          "type": "SqlServerStoredProcedure",
                          "dependsOn": [
                            {
                              "activity": "Wait_BeforeRetry",
                              "dependencyConditions": ["Succeeded"]
                            }
                          ],
                          "typeProperties": {
                            "storedProcedureName": "[dbo].[sp_UpdateCircuitBreaker]",
                            "storedProcedureParameters": {
                              "ServiceName": {
                                "value": {
                                  "value": "@pipeline().parameters.ApiEndpoint",
                                  "type": "Expression"
                                },
                                "type": "String"
                              },
                              "IsSuccess": {
                                "value": false,
                                "type": "Boolean"
                              }
                            }
                          }
                        }
                      ],
                      "ifFalseActivities": [
                        {
                          "name": "Open_CircuitBreaker",
                          "type": "SqlServerStoredProcedure",
                          "typeProperties": {
                            "storedProcedureName": "[dbo].[sp_OpenCircuitBreaker]",
                            "storedProcedureParameters": {
                              "ServiceName": {
                                "value": {
                                  "value": "@pipeline().parameters.ApiEndpoint",
                                  "type": "Expression"
                                },
                                "type": "String"
                              }
                            }
                          }
                        }
                      ]
                    }
                  }
                ],
                "timeout": "0.01:00:00"
              }
            }
          ]
        }
      },
      {
        "name": "If_FinalFailure",
        "type": "IfCondition",
        "dependsOn": [
          {
            "activity": "If_CircuitOpen",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "expression": {
            "value": "@not(variables('IsSuccess'))",
            "type": "Expression"
          },
          "ifTrueActivities": [
            {
              "name": "Send_Alert",
              "type": "WebActivity",
              "typeProperties": {
                "url": "https://prod-xx.eastus.logic.azure.com:443/workflows/xxx",
                "method": "POST",
                "body": {
                  "value": "@concat('API call failed after ', variables('AttemptCount'), ' attempts. Endpoint: ', pipeline().parameters.ApiEndpoint)",
                  "type": "Expression"
                }
              }
            },
            {
              "name": "Fail_Pipeline",
              "type": "Fail",
              "dependsOn": [
                {
                  "activity": "Send_Alert",
                  "dependencyConditions": ["Succeeded"]
                }
              ],
              "typeProperties": {
                "message": {
                  "value": "@concat('Max retries exceeded. Last error: ', variables('LastErrorMessage'))",
                  "type": "Expression"
                },
                "errorCode": "MAX_RETRIES_EXCEEDED"
              }
            }
          ]
        }
      }
    ]
  }
}
```

#### Step 5: Create Supporting Stored Procedures

**Log Retry Attempt:**
```sql
CREATE PROCEDURE [dbo].[sp_LogRetryAttempt]
    @PipelineRunID VARCHAR(100),
    @ActivityName VARCHAR(100),
    @AttemptNumber INT,
    @Status VARCHAR(20),
    @ErrorCode VARCHAR(50) = NULL,
    @ErrorMessage VARCHAR(MAX) = NULL
AS
BEGIN
    DECLARE @RetryInterval INT;
    DECLARE @NextRetryTime DATETIME;
    
    -- Calculate next retry time with exponential backoff
    SET @RetryInterval = POWER(2, @AttemptNumber - 1) * 10; -- 10, 20, 40, 80, 160 seconds
    SET @RetryInterval = CASE WHEN @RetryInterval > 300 THEN 300 ELSE @RetryInterval END; -- Cap at 5 minutes
    SET @NextRetryTime = DATEADD(SECOND, @RetryInterval, GETDATE());
    
    INSERT INTO [dbo].[RetryLog]
    (PipelineRunID, ActivityName, AttemptNumber, AttemptTime, Status, ErrorCode, ErrorMessage, RetryIntervalSeconds, NextRetryTime)
    VALUES
    (@PipelineRunID, @ActivityName, @AttemptNumber, GETDATE(), @Status, @ErrorCode, @ErrorMessage, @RetryInterval, @NextRetryTime);
END;
```

**Update Circuit Breaker:**
```sql
CREATE PROCEDURE [dbo].[sp_UpdateCircuitBreaker]
    @ServiceName VARCHAR(100),
    @IsSuccess BIT
AS
BEGIN
    DECLARE @ConsecutiveFailures INT;
    DECLARE @Threshold INT;
    
    -- Get circuit breaker threshold from configuration
    SELECT @Threshold = CircuitBreakerThreshold 
    FROM [dbo].[RetryConfiguration] 
    WHERE ActivityType = 'API_CALL' 
    AND ErrorType = 'SERVICE_UNAVAILABLE';
    
    IF @IsSuccess = 1
    BEGIN
        -- Reset on success
        UPDATE [dbo].[CircuitBreakerState]
        SET ConsecutiveFailures = 0,
            State = 'CLOSED',
            LastFailureTime = NULL
        WHERE ServiceName = @ServiceName;
    END
    ELSE
    BEGIN
        -- Increment failure count
        IF EXISTS (SELECT 1 FROM [dbo].[CircuitBreakerState] WHERE ServiceName = @ServiceName)
        BEGIN
            UPDATE [dbo].[CircuitBreakerState]
            SET ConsecutiveFailures = ConsecutiveFailures + 1,
                LastFailureTime = GETDATE()
            WHERE ServiceName = @ServiceName;
            
            SELECT @ConsecutiveFailures = ConsecutiveFailures 
            FROM [dbo].[CircuitBreakerState] 
            WHERE ServiceName = @ServiceName;
            
            -- Open circuit if threshold exceeded
            IF @ConsecutiveFailures >= @Threshold
            BEGIN
                EXEC [dbo].[sp_OpenCircuitBreaker] @ServiceName;
            END
        END
        ELSE
        BEGIN
            INSERT INTO [dbo].[CircuitBreakerState]
            (ServiceName, State, ConsecutiveFailures, LastFailureTime)
            VALUES
            (@ServiceName, 'CLOSED', 1, GETDATE());
        END
    END
END;
```

**Open Circuit Breaker:**
```sql
CREATE PROCEDURE [dbo].[sp_OpenCircuitBreaker]
    @ServiceName VARCHAR(100)
AS
BEGIN
    DECLARE @ResetMinutes INT;
    
    SELECT @ResetMinutes = CircuitBreakerResetMinutes 
    FROM [dbo].[RetryConfiguration] 
    WHERE ActivityType = 'API_CALL' 
    AND ErrorType = 'SERVICE_UNAVAILABLE';
    
    UPDATE [dbo].[CircuitBreakerState]
    SET State = 'OPEN',
        CircuitOpenedTime = GETDATE(),
        NextRetryTime = DATEADD(MINUTE, @ResetMinutes, GETDATE())
    WHERE ServiceName = @ServiceName;
    
    -- Send alert that circuit is open
    -- (Could trigger Logic App or send to monitoring system)
END;
```

**Reset Circuit Breaker:**
```sql
CREATE PROCEDURE [dbo].[sp_ResetCircuitBreaker]
    @ServiceName VARCHAR(100)
AS
BEGIN
    UPDATE [dbo].[CircuitBreakerState]
    SET State = 'CLOSED',
        ConsecutiveFailures = 0,
        LastFailureTime = NULL,
        CircuitOpenedTime = NULL,
        NextRetryTime = NULL
    WHERE ServiceName = @ServiceName;
END;
```

### Expected Behavior
1. Pipeline checks circuit breaker state before attempting API call
2. If circuit is OPEN and reset time hasn't passed, pipeline fails immediately
3. If circuit is CLOSED or HALF_OPEN, pipeline attempts API call
4. On failure, pipeline waits with exponential backoff (10s, 20s, 40s, 80s, 160s, capped at 300s)
5. Each attempt is logged with timestamp and error details
6. After max retries, circuit breaker opens if threshold is exceeded
7. Circuit breaker automatically resets after configured time period
8. On success, circuit breaker resets immediately

### Common Issues and Resolutions

**Issue 1: Until Activity Timeout**
- **Problem**: Until activity has 24-hour timeout, but long retry intervals can exceed this
- **Solution**: 
  - Use tumbling window trigger to retry at pipeline level instead
  - Reduce max retry interval
  - Implement external retry orchestration using Azure Functions

**Issue 2: Rate Limiting Not Detected**
- **Problem**: API returns 200 OK but with rate limit message in body
- **Solution**:
```json
{
  "name": "Check_RateLimit",
  "type": "IfCondition",
  "typeProperties": {
    "expression": {
      "value": "@contains(activity('Try_API_Call').output.response, 'rate limit exceeded')",
      "type": "Expression"
    },
    "ifTrueActivities": [
      {
        "name": "Wait_RateLimit",
        "type": "Wait",
        "typeProperties": {
          "waitTimeInSeconds": 60
        }
      }
    ]
  }
}
```

**Issue 3: Different Error Types Need Different Strategies**
- **Problem**: Timeout errors should retry quickly, authentication errors shouldn't retry
- **Solution**: Implement error classification
```json
{
  "name": "Classify_Error",
  "type": "Switch",
  "typeProperties": {
    "on": {
      "value": "@activity('Try_API_Call').error.errorCode",
      "type": "Expression"
    },
    "cases": [
      {
        "value": "TIMEOUT",
        "activities": [
          {
            "name": "Retry_Timeout",
            "type": "Wait",
            "typeProperties": {
              "waitTimeInSeconds": 10
            }
          }
        ]
      },
      {
        "value": "AUTHENTICATION",
        "activities": [
          {
            "name": "Fail_Auth",
            "type": "Fail",
            "typeProperties": {
              "message": "Authentication error - manual intervention required",
              "errorCode": "AUTH_ERROR"
            }
          }
        ]
      },
      {
        "value": "RATE_LIMIT",
        "activities": [
          {
            "name": "Wait_RateLimit",
            "type": "Wait",
            "typeProperties": {
              "waitTimeInSeconds": 60
            }
          }
        ]
      }
    ],
    "defaultActivities": [
      {
        "name": "Generic_Retry",
        "type": "Wait",
        "typeProperties": {
          "waitTimeInSeconds": 30
        }
      }
    ]
  }
}
```

### Pro Tips
💡 **Use built-in retry policies** for simple scenarios - only implement custom retry for complex cases
💡 **Implement jitter** in retry intervals to avoid thundering herd problem
💡 **Monitor retry metrics** - high retry rates indicate systemic issues
💡 **Set maximum total retry time** to prevent indefinite waiting
💡 **Use dead letter queue** for messages that fail after all retries
💡 **Implement idempotency tokens** for API calls to prevent duplicate operations

### Interview Questions You Might Face

**Q1: Explain the difference between retry policies and circuit breaker patterns.**
**Answer**: "Retry policies and circuit breakers serve different purposes. Retry policies handle transient failures by attempting the operation multiple times with delays between attempts - they're optimistic and assume the failure is temporary. Circuit breakers, on the other hand, prevent cascading failures by stopping attempts after a threshold of consecutive failures - they're protective and assume the service is down. In my implementations, I use both together: retry policies for individual request failures with exponential backoff, and circuit breakers to stop all requests when a service is clearly unavailable. For example, if an API call fails 3 times, I retry with increasing delays. But if 10 consecutive calls fail, the circuit breaker opens and all subsequent requests fail immediately for 30 minutes, giving the service time to recover without overwhelming it."

**Q2: How do you implement exponential backoff in ADF?**
**Answer**: "ADF doesn't have built-in exponential backoff, so I implement it using variables and the Wait activity within an Until loop. I maintain a RetryInterval variable that doubles after each failure, capped at a maximum (usually 5 minutes). The formula I use is: min(initialInterval * 2^attemptNumber, maxInterval). For example: 10s, 20s, 40s, 80s, 160s, 300s (capped). I also add jitter - a random component to prevent synchronized retries from multiple pipelines. The implementation uses SetVariable to calculate the next interval, then Wait activity to pause, then retry the operation. For very sophisticated scenarios, I use Azure Functions triggered by ADF to implement more complex backoff algorithms like decorrelated jitter."

**Q3: What are the different types of errors and how do you handle each?**
**Answer**: "I classify errors into several categories, each requiring different handling: First, transient errors (network timeouts, temporary service unavailability) - these should be retried with exponential backoff, 3-5 attempts. Second, rate limiting errors - these need longer wait times (60+ seconds) and potentially reduced request rate. Third, authentication errors - these shouldn't be retried automatically as they indicate configuration issues requiring manual intervention. Fourth, validation errors (bad request, invalid data) - these are permanent failures that shouldn't be retried. Fifth, server errors (500-level) - retry a few times but fail quickly if persistent. In my pipelines, I use the error.errorCode from failed activities to classify errors and route to appropriate handling logic using Switch or If Condition activities. I also log all errors with their classifications for monitoring and alerting."

---

## Summary

These four scenarios cover critical enterprise data integration patterns:

1. **Cross-Region Replication**: Handling geo-distributed data with network resilience
2. **Data Quality Checks**: Implementing comprehensive validation frameworks
3. **Orchestrating Multiple Pipelines**: Managing complex dependencies and parallel execution
4. **Retry Logic**: Building resilient pipelines with intelligent failure handling

Each scenario demonstrates production-ready patterns that you'll encounter in real-world ADF implementations. Master these, and you'll be prepared for the most challenging interview questions and production scenarios.

