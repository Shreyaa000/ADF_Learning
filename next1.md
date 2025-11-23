# ADF-07: Real-World Scenarios - Additional Patterns

This document covers additional real-world scenarios for Azure Data Factory implementation.

---

## Scenario 18: Lookup Enrichment

### Business Requirement
A retail company needs to enrich their sales transaction data with product details (category, brand, price) from a reference table before loading into the data warehouse. The transaction files contain only product IDs, and we need to add descriptive information for reporting purposes.

### Implementation Steps

#### Step 1: Create Linked Services
- **Source Linked Service**: Azure Blob Storage (for transaction files)
- **Reference Linked Service**: Azure SQL Database (for product master data)
- **Destination Linked Service**: Azure Synapse Analytics

#### Step 2: Create Datasets
1. **TransactionFiles Dataset**:
```json
{
  "name": "DS_TransactionFiles",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "container": "transactions"
    },
    "columnDelimiter": ",",
    "firstRowAsHeader": true
  }
}
```

2. **ProductReference Dataset**:
```json
{
  "name": "DS_ProductReference",
  "type": "AzureSqlTable",
  "linkedServiceName": "LS_AzureSQL",
  "typeProperties": {
    "schema": "dbo",
    "table": "ProductMaster"
  }
}
```

3. **EnrichedTransactions Dataset**:
```json
{
  "name": "DS_EnrichedTransactions",
  "type": "AzureSqlTable",
  "linkedServiceName": "LS_Synapse",
  "typeProperties": {
    "schema": "staging",
    "table": "EnrichedTransactions"
  }
}
```

#### Step 3: Build the Pipeline

**Pipeline: PL_EnrichTransactions**

1. **Lookup Activity** - Get Product Reference Data:
```json
{
  "name": "LKP_GetProductData",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT ProductID, ProductName, Category, Brand, UnitPrice FROM dbo.ProductMaster WHERE IsActive = 1"
    },
    "dataset": {
      "referenceName": "DS_ProductReference"
    },
    "firstRowOnly": false
  }
}
```

2. **Data Flow Activity** - Enrich and Load:
```json
{
  "name": "DF_EnrichTransactions",
  "type": "ExecuteDataFlow",
  "dependsOn": [
    {
      "activity": "LKP_GetProductData",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "dataflow": {
      "referenceName": "DF_EnrichmentFlow"
    },
    "compute": {
      "coreCount": 8,
      "computeType": "General"
    }
  }
}
```

#### Step 4: Create Data Flow (Alternative Approach)

**Data Flow: DF_EnrichmentFlow**

1. **Source Transformation** - Read Transactions:
   - Dataset: DS_TransactionFiles
   - Columns: TransactionID, TransactionDate, ProductID, Quantity, Amount

2. **Lookup Transformation** - Enrich with Product Data:
   - Lookup stream: DS_ProductReference
   - Lookup conditions: `TransactionSource.ProductID == ProductReference.ProductID`
   - Select columns: All from source + ProductName, Category, Brand, UnitPrice from lookup

3. **Derived Column Transformation** - Calculate Extended Amount:
   - Add column: `ExtendedAmount = Quantity * UnitPrice`
   - Add column: `LoadDate = currentTimestamp()`

4. **Sink Transformation** - Write to Synapse:
   - Dataset: DS_EnrichedTransactions
   - Write method: Upsert
   - Key columns: TransactionID

### Alternative: Copy Activity with Lookup

For simpler scenarios without complex transformations:

```json
{
  "name": "Copy_EnrichTransactions",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "additionalColumns": [
        {
          "name": "ProductName",
          "value": "$$COLUMN:ProductName"
        },
        {
          "name": "Category",
          "value": "$$COLUMN:Category"
        }
      ]
    },
    "sink": {
      "type": "SqlDWSink",
      "preCopyScript": "TRUNCATE TABLE staging.EnrichedTransactions"
    },
    "translator": {
      "type": "TabularTranslator",
      "mappings": [
        {
          "source": { "name": "TransactionID" },
          "sink": { "name": "TransactionID" }
        },
        {
          "source": { "name": "ProductID" },
          "sink": { "name": "ProductID" }
        }
      ]
    }
  }
}
```

### Expected Behavior
- Transaction file with 10,000 rows gets enriched with product details
- Each transaction now has ProductName, Category, Brand, and UnitPrice
- Data loaded into Synapse staging table with all enriched columns
- Processing time: ~2-3 minutes for 10K rows

### Common Issues and Resolutions

**Issue 1: Lookup Returns No Data**
- **Error**: "Lookup activity returned no results"
- **Cause**: Reference table is empty or filter condition is too restrictive
- **Solution**: 
  - Verify reference table has data: `SELECT COUNT(*) FROM dbo.ProductMaster`
  - Check IsActive filter or remove it for testing
  - Use `firstRowOnly: false` to get all rows

**Issue 2: Memory Issues with Large Lookup Tables**
- **Error**: "OutOfMemoryException in Data Flow"
- **Cause**: Loading entire reference table (millions of rows) into memory
- **Solution**: 
  - Use Data Flow Lookup transformation instead of activity-level lookup
  - Enable broadcast optimization for smaller lookup tables (<100MB)
  - For large tables, use Join transformation with partitioning

**Issue 3: Null Values After Lookup**
- **Error**: ProductName is NULL for some transactions
- **Cause**: ProductID in transaction doesn't exist in reference table
- **Solution**:
  - Use Left Outer Join in Data Flow
  - Add Conditional Split to separate matched/unmatched records
  - Log unmatched records for investigation
  ```json
  {
    "name": "Split_MatchedUnmatched",
    "type": "ConditionalSplit",
    "conditions": [
      {
        "name": "Matched",
        "expression": "!isNull(ProductName)"
      },
      {
        "name": "Unmatched",
        "expression": "isNull(ProductName)"
      }
    ]
  }
  ```

**Issue 4: Performance Degradation**
- **Symptom**: Pipeline takes 30+ minutes for 100K rows
- **Cause**: Inefficient join strategy or no partitioning
- **Solution**:
  - Enable partition optimization in Data Flow
  - Use Hash partitioning on ProductID
  - Increase DIU (Data Integration Units) from 4 to 8 or 16
  - Cache reference data if used multiple times

### Interview Questions

**Q1: What's the difference between Lookup activity and Lookup transformation in Data Flow?**
**Answer**: 
- **Lookup Activity**: Runs at pipeline level, retrieves data before data flow execution, stores results in pipeline variables. Best for small reference data (<5000 rows) that fits in memory. Limited to 5000 rows output.
- **Lookup Transformation**: Runs within data flow, streams data, can handle millions of rows, supports broadcast optimization, more scalable for large datasets.

**Q2: How do you handle missing reference data during enrichment?**
**Answer**: Multiple approaches:
1. Use Left Outer Join to keep all source records
2. Add Conditional Split to separate matched/unmatched
3. Route unmatched records to error handling sink
4. Use Derived Column to set default values: `iif(isNull(ProductName), 'Unknown', ProductName)`
5. Log unmatched ProductIDs for data quality investigation

**Q3: What if the reference table is too large (10 million rows)?**
**Answer**: 
- Don't use Lookup activity (5000 row limit)
- Use Data Flow with Join transformation
- Partition both streams on join key (ProductID)
- Use Hash distribution strategy
- Consider incremental enrichment (only new/changed records)
- Cache reference data in Azure Redis if frequently accessed
- Use Synapse dedicated SQL pool for large-scale joins

**Q4: How do you optimize lookup performance?**
**Answer**:
1. **Broadcast optimization**: For small lookup tables (<100MB), enable broadcast to send to all nodes
2. **Partitioning**: Use Hash partition on join key
3. **Filtering**: Load only required columns and active records
4. **Caching**: Enable data flow debug cache for development
5. **Indexing**: Ensure reference table has index on join column
6. **Compute sizing**: Use appropriate DIU based on data volume

**Q5: Can you enrich data during Copy activity without Data Flow?**
**Answer**: Limited options:
- Use `additionalColumns` to add static values
- Use stored procedures in sink to perform enrichment
- Use mapping data flows for complex enrichment
- For simple lookups, consider pre-joining in source query if both are in same database
- Cannot perform dynamic lookups directly in Copy activity

### Pro Tips
1. **Cache Reference Data**: If reference table changes infrequently, cache it in pipeline variable and reuse across multiple activities
2. **Incremental Enrichment**: For large datasets, enrich only new/changed records using watermark pattern
3. **Broadcast Small Tables**: Enable broadcast for lookup tables <100MB for better performance
4. **Monitor Data Quality**: Track percentage of unmatched records to identify data quality issues
5. **Use Surrogate Keys**: If possible, use integer surrogate keys instead of string keys for faster joins

### Common Mistakes to Avoid
1. ❌ Loading entire reference table when only subset is needed
2. ❌ Not handling NULL values from failed lookups
3. ❌ Using Lookup activity for large reference tables (>5000 rows)
4. ❌ Not partitioning data in Data Flow for large-scale enrichment
5. ❌ Ignoring unmatched records without logging or alerting

### Real-World Anecdote
"In my previous project for a retail client, we had to enrich 50 million transaction records daily with product and customer data. Initially, we used Lookup activity which failed due to the 5000-row limit. We switched to Data Flow with Hash partitioning on CustomerID and ProductID, enabled broadcast for the smaller product table (50K rows), and the pipeline completed in 15 minutes instead of timing out. The key was understanding when to use Lookup activity vs Lookup transformation."

---

## Scenario 19: File Format Conversion

### Business Requirement
A financial services company receives data in multiple formats (CSV, JSON, XML) from different vendors and needs to convert everything to Parquet format for efficient storage and querying in Azure Data Lake. They also need to convert Parquet files to CSV for legacy systems that cannot read Parquet.

### Implementation Steps

#### Step 1: Create Linked Services
- **Source**: Azure Blob Storage (multiple format files)
- **Destination**: Azure Data Lake Gen2 (Parquet files)
- **Legacy Output**: Azure Blob Storage (CSV files)

#### Step 2: Create Datasets

**1. CSV Source Dataset**:
```json
{
  "name": "DS_CSV_Source",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "folderPath": {
      "type": "string"
    },
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "@dataset().folderPath",
      "container": "raw-data"
    },
    "columnDelimiter": ",",
    "rowDelimiter": "\n",
    "escapeChar": "\\",
    "quoteChar": "\"",
    "firstRowAsHeader": true,
    "encoding": "UTF-8"
  }
}
```

**2. JSON Source Dataset**:
```json
{
  "name": "DS_JSON_Source",
  "type": "Json",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "folderPath": {
      "type": "string"
    },
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "@dataset().folderPath",
      "container": "raw-data"
    }
  }
}
```

**3. Parquet Sink Dataset**:
```json
{
  "name": "DS_Parquet_Sink",
  "type": "Parquet",
  "linkedServiceName": "LS_ADLS_Gen2",
  "parameters": {
    "folderPath": {
      "type": "string"
    },
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobFSLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "@dataset().folderPath",
      "fileSystem": "processed-data"
    },
    "compressionCodec": "snappy"
  }
}
```

**4. CSV Sink Dataset (for legacy systems)**:
```json
{
  "name": "DS_CSV_Sink",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "folderPath": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "folderPath": "@dataset().folderPath",
      "container": "legacy-output"
    },
    "columnDelimiter": ",",
    "rowDelimiter": "\n",
    "firstRowAsHeader": true,
    "encoding": "UTF-8"
  }
}
```

#### Step 3: Build the Pipeline

**Pipeline: PL_FileFormatConversion**

**1. Get Metadata Activity** - List Files:
```json
{
  "name": "GetMetadata_ListFiles",
  "type": "GetMetadata",
  "typeProperties": {
    "dataset": {
      "referenceName": "DS_CSV_Source",
      "parameters": {
        "folderPath": "incoming",
        "fileName": "*"
      }
    },
    "fieldList": ["childItems"],
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": true,
      "wildcardFileName": "*.*"
    }
  }
}
```

**2. ForEach Activity** - Process Each File:
```json
{
  "name": "ForEach_File",
  "type": "ForEach",
  "dependsOn": [
    {
      "activity": "GetMetadata_ListFiles",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "items": "@activity('GetMetadata_ListFiles').output.childItems",
    "isSequential": false,
    "batchCount": 10,
    "activities": [
      {
        "name": "Switch_FileType",
        "type": "Switch",
        "typeProperties": {
          "on": "@substring(item().name, add(lastIndexOf(item().name, '.'), 1), sub(length(item().name), add(lastIndexOf(item().name, '.'), 1)))",
          "cases": [
            {
              "value": "csv",
              "activities": [
                {
                  "name": "Copy_CSV_to_Parquet",
                  "type": "Copy"
                }
              ]
            },
            {
              "value": "json",
              "activities": [
                {
                  "name": "Copy_JSON_to_Parquet",
                  "type": "Copy"
                }
              ]
            }
          ],
          "defaultActivities": [
            {
              "name": "Log_UnsupportedFormat",
              "type": "Wait",
              "typeProperties": {
                "waitTimeInSeconds": 1
              }
            }
          ]
        }
      }
    ]
  }
}
```

**3. Copy Activity - CSV to Parquet**:
```json
{
  "name": "Copy_CSV_to_Parquet",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings",
        "recursive": false
      },
      "formatSettings": {
        "type": "DelimitedTextReadSettings",
        "skipLineCount": 0
      }
    },
    "sink": {
      "type": "ParquetSink",
      "storeSettings": {
        "type": "AzureBlobFSWriteSettings",
        "copyBehavior": "PreserveHierarchy"
      },
      "formatSettings": {
        "type": "ParquetWriteSettings"
      }
    },
    "enableStaging": false,
    "validateDataConsistency": true,
    "translator": {
      "type": "TabularTranslator",
      "typeConversion": true,
      "typeConversionSettings": {
        "allowDataTruncation": false,
        "treatBooleanAsNumber": false
      }
    }
  },
  "inputs": [
    {
      "referenceName": "DS_CSV_Source",
      "parameters": {
        "folderPath": "incoming",
        "fileName": "@item().name"
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_Parquet_Sink",
      "parameters": {
        "folderPath": "@concat('processed/', formatDateTime(utcnow(), 'yyyy-MM-dd'))",
        "fileName": "@concat(substring(item().name, 0, lastIndexOf(item().name, '.')), '.parquet')"
      }
    }
  ]
}
```

**4. Copy Activity - JSON to Parquet**:
```json
{
  "name": "Copy_JSON_to_Parquet",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "JsonSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings",
        "recursive": false
      },
      "formatSettings": {
        "type": "JsonReadSettings"
      }
    },
    "sink": {
      "type": "ParquetSink",
      "storeSettings": {
        "type": "AzureBlobFSWriteSettings"
      },
      "formatSettings": {
        "type": "ParquetWriteSettings"
      }
    },
    "enableStaging": false,
    "translator": {
      "type": "TabularTranslator",
      "typeConversion": true
    }
  },
  "inputs": [
    {
      "referenceName": "DS_JSON_Source",
      "parameters": {
        "folderPath": "incoming",
        "fileName": "@item().name"
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_Parquet_Sink",
      "parameters": {
        "folderPath": "@concat('processed/', formatDateTime(utcnow(), 'yyyy-MM-dd'))",
        "fileName": "@concat(substring(item().name, 0, lastIndexOf(item().name, '.')), '.parquet')"
      }
    }
  ]
}
```

**5. Copy Activity - Parquet to CSV (for legacy systems)**:
```json
{
  "name": "Copy_Parquet_to_CSV",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "ParquetSource",
      "storeSettings": {
        "type": "AzureBlobFSReadSettings",
        "recursive": true
      }
    },
    "sink": {
      "type": "DelimitedTextSink",
      "storeSettings": {
        "type": "AzureBlobStorageWriteSettings",
        "copyBehavior": "FlattenHierarchy"
      },
      "formatSettings": {
        "type": "DelimitedTextWriteSettings",
        "quoteAllText": true,
        "fileExtension": ".csv"
      }
    },
    "enableStaging": false
  }
}
```

### Advanced: XML to Parquet Conversion

For XML files, use Data Flow:

**Data Flow: DF_XML_to_Parquet**

1. **Source Transformation**:
   - Dataset: DS_XML_Source
   - Parse settings: Define XML structure

2. **Flatten Transformation**:
   - Unroll arrays from XML hierarchy
   - Create flat structure

3. **Select Transformation**:
   - Choose required columns
   - Rename columns to match target schema

4. **Sink Transformation**:
   - Dataset: DS_Parquet_Sink
   - Partition: Single partition for small files, Hash for large files

### Compression Options for Parquet

```json
{
  "compressionCodec": "snappy",  // Options: none, snappy, gzip, lzo, brotli, lz4, zstd
  "compressionLevel": "optimal"   // For gzip: fastest, optimal, or custom level 1-9
}
```

**Compression Comparison**:
- **Snappy**: Fast compression/decompression, moderate compression ratio (default, recommended)
- **Gzip**: Slower but better compression ratio
- **LZ4**: Fastest, lower compression ratio
- **Zstd**: Best balance of speed and compression (if supported)

### Expected Behavior
- 100 CSV files (10MB each) converted to Parquet in ~5 minutes
- File size reduction: 70-80% (CSV 1GB → Parquet 200-300MB)
- Parquet files maintain schema and data types
- Parallel processing of 10 files at a time

### Common Issues and Resolutions

**Issue 1: Schema Mismatch During Conversion**
- **Error**: "Column 'Amount' has type 'String' but expected 'Decimal'"
- **Cause**: CSV treats all data as strings, Parquet enforces types
- **Solution**:
  ```json
  {
    "translator": {
      "type": "TabularTranslator",
      "typeConversion": true,
      "typeConversionSettings": {
        "allowDataTruncation": false,
        "treatBooleanAsNumber": false,
        "dateTimeFormat": "yyyy-MM-dd HH:mm:ss",
        "dateTimeOffsetFormat": "yyyy-MM-dd HH:mm:ss.fff"
      },
      "mappings": [
        {
          "source": { "name": "Amount", "type": "String" },
          "sink": { "name": "Amount", "type": "Decimal" }
        }
      ]
    }
  }
  ```

**Issue 2: Special Characters in CSV Breaking Parquet Conversion**
- **Error**: "Failed to parse CSV: Unexpected character"
- **Cause**: CSV contains special characters, quotes, or delimiters in data
- **Solution**:
  - Set proper escape character: `"escapeChar": "\\"`
  - Set quote character: `"quoteChar": "\""`
  - Enable quote all text in sink: `"quoteAllText": true`
  - Use Data Flow for complex parsing scenarios

**Issue 3: Large Files Causing Memory Issues**
- **Error**: "OutOfMemoryException during conversion"
- **Cause**: Trying to convert very large files (>5GB) in single operation
- **Solution**:
  - Enable staging: `"enableStaging": true` with Azure Blob staging
  - Use Data Flow with partitioning
  - Split large files before conversion
  - Increase DIU from 4 to 8 or 16

**Issue 4: Parquet Files Not Readable by External Tools**
- **Symptom**: Parquet files created but cannot be read by Spark/Databricks
- **Cause**: Incompatible Parquet version or compression codec
- **Solution**:
  - Use Snappy compression (most compatible)
  - Verify Parquet version compatibility
  - Test with small sample file first
  - Check file permissions in Data Lake

**Issue 5: Date/Time Format Issues**
- **Error**: "Cannot convert '2024-01-15' to DateTime"
- **Cause**: CSV has dates as strings, Parquet expects DateTime type
- **Solution**:
  ```json
  {
    "typeConversionSettings": {
      "dateTimeFormat": "yyyy-MM-dd",
      "culture": "en-US",
      "timeZone": "UTC"
    }
  }
  ```

### Performance Optimization

**1. Parallel Processing**:
```json
{
  "typeProperties": {
    "isSequential": false,
    "batchCount": 20  // Process 20 files in parallel
  }
}
```

**2. DIU Optimization**:
- Small files (<100MB): 4 DIU
- Medium files (100MB-1GB): 8 DIU
- Large files (>1GB): 16-32 DIU

**3. Partitioning Strategy**:
For large files, partition Parquet output:
```json
{
  "partitionOption": "PartitionByKey",
  "partitionSettings": {
    "partitionColumnNames": ["Year", "Month"]
  }
}
```

### Interview Questions

**Q1: Why convert CSV to Parquet? What are the benefits?**
**Answer**:
1. **Storage Efficiency**: 70-80% size reduction due to columnar storage and compression
2. **Query Performance**: Columnar format allows reading only required columns (column pruning)
3. **Schema Enforcement**: Parquet stores schema with data, ensures data type consistency
4. **Compression**: Built-in compression (Snappy, Gzip) without sacrificing read performance
5. **Compatibility**: Works seamlessly with Spark, Databricks, Synapse, Athena
6. **Predicate Pushdown**: Supports filtering at file level, skips unnecessary data

**Q2: What challenges do you face when converting JSON to Parquet?**
**Answer**:
1. **Nested Structures**: JSON can have nested objects/arrays, Parquet is flat - need to flatten or use struct types
2. **Schema Inference**: JSON schema may vary between files - need schema validation
3. **Data Types**: JSON doesn't enforce types, Parquet does - need type conversion
4. **Array Handling**: JSON arrays need to be exploded or stored as complex types
5. **Null Handling**: JSON null vs missing fields - need consistent handling

**Q3: How do you handle schema evolution when converting files?**
**Answer**:
1. **Schema Drift**: Enable "Allow schema drift" in Data Flow to handle new columns
2. **Schema Validation**: Use GetMetadata to validate schema before conversion
3. **Default Values**: Use Derived Column to add missing columns with defaults
4. **Version Control**: Store schema versions and handle backward compatibility
5. **Schema Registry**: Maintain central schema repository for validation

**Q4: What's the difference between Snappy and Gzip compression for Parquet?**
**Answer**:
- **Snappy**: 
  - Faster compression/decompression (2-3x faster than Gzip)
  - Moderate compression ratio (~2:1)
  - Default for Parquet, best for query performance
  - Recommended for frequently accessed data
- **Gzip**: 
  - Slower but better compression ratio (~3:1)
  - Higher CPU usage
  - Good for archival/cold storage
  - Use when storage cost > compute cost

**Q5: How do you convert Parquet back to CSV for legacy systems?**
**Answer**:
- Use Copy activity with Parquet source and DelimitedText sink
- Challenges: Large Parquet files may create huge CSV files
- Solution: Partition CSV output by date/region
- Consider: Add header row, handle special characters, set proper encoding
- Alternative: Provide API access instead of CSV files

**Q6: Can you convert files in place or do you need staging?**
**Answer**:
- Cannot convert in place - need separate source and sink
- For large files (>1GB), enable staging for better performance
- Staging uses Azure Blob as intermediate storage
- After conversion, archive or delete source files
- Use Copy activity's "deleteFilesAfterCompletion" for cleanup

### Pro Tips
1. **Test with Small Files First**: Validate conversion logic with small samples before processing production data
2. **Monitor File Sizes**: Track compression ratios to identify data quality issues
3. **Partition Large Parquet Files**: Use date-based partitioning for better query performance
4. **Use Snappy for Hot Data**: Default to Snappy compression for frequently queried data
5. **Validate Row Counts**: Always compare source and sink row counts after conversion
6. **Handle Schema Changes**: Implement schema validation before conversion to catch issues early

### Common Mistakes to Avoid
1. ❌ Not handling special characters in CSV (quotes, delimiters, newlines)
2. ❌ Ignoring data type conversions (everything becomes string)
3. ❌ Converting very large files without staging or partitioning
4. ❌ Not testing Parquet compatibility with downstream systems
5. ❌ Using wrong compression codec for use case
6. ❌ Not archiving source files after successful conversion

### Real-World Anecdote
"In my previous project for a financial services client, we received daily transaction files from 50+ vendors in various formats - CSV, JSON, pipe-delimited, fixed-width. We built a metadata-driven conversion framework that automatically detected file format, applied appropriate schema, and converted to Parquet. The key was using GetMetadata to identify file type, Switch activity to route to appropriate conversion logic, and Data Flow for complex JSON/XML parsing. We reduced storage costs by 75% and query performance improved by 10x after moving to Parquet."

---

## Scenario 20: Compression Handling

### Business Requirement
A healthcare company receives compressed patient data files (GZip, Zip, BZip2) from various hospitals and needs to extract, process, and load the data into Azure SQL Database. They also need to compress processed files before archiving to reduce storage costs.

### Implementation Steps

#### Step 1: Create Linked Services
- **Source**: Azure Blob Storage (compressed files)
- **Processing**: Azure Data Lake Gen2 (extracted files)
- **Destination**: Azure SQL Database
- **Archive**: Azure Blob Storage Cool Tier (compressed archives)

#### Step 2: Create Datasets

**1. Compressed Source Dataset (GZip)**:
```json
{
  "name": "DS_CompressedCSV_GZip",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "incoming/compressed",
      "container": "raw-data"
    },
    "columnDelimiter": ",",
    "rowDelimiter": "\n",
    "firstRowAsHeader": true,
    "compressionCodec": "gzip",
    "compressionLevel": "optimal"
  }
}
```

**2. Compressed Source Dataset (Zip)**:
```json
{
  "name": "DS_CompressedCSV_Zip",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage",
  "parameters": {
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "incoming/compressed",
      "container": "raw-data"
    },
    "columnDelimiter": ",",
    "firstRowAsHeader": true,
    "compressionCodec": "zipDeflate"
  }
}
```

**3. Extracted Dataset (Uncompressed)**:
```json
{
  "name": "DS_ExtractedCSV",
  "type": "DelimitedText",
  "linkedServiceName": "LS_ADLS_Gen2",
  "parameters": {
    "folderPath": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobFSLocation",
      "folderPath": "@dataset().folderPath",
      "fileSystem": "extracted-data"
    },
    "columnDelimiter": ",",
    "firstRowAsHeader": true,
    "compressionCodec": "none"
  }
}
```

**4. Compressed Archive Dataset**:
```json
{
  "name": "DS_CompressedArchive",
  "type": "DelimitedText",
  "linkedServiceName": "LS_BlobStorage_CoolTier",
  "parameters": {
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "@dataset().fileName",
      "folderPath": "archive",
      "container": "processed-data"
    },
    "columnDelimiter": ",",
    "firstRowAsHeader": true,
    "compressionCodec": "gzip",
    "compressionLevel": "optimal"
  }
}
```

**5. SQL Database Dataset**:
```json
{
  "name": "DS_PatientData",
  "type": "AzureSqlTable",
  "linkedServiceName": "LS_AzureSQL",
  "typeProperties": {
    "schema": "dbo",
    "table": "PatientData"
  }
}
```

#### Step 3: Build the Pipeline

**Pipeline: PL_ProcessCompressedFiles**

**1. Get Metadata Activity** - List Compressed Files:
```json
{
  "name": "GetMetadata_CompressedFiles",
  "type": "GetMetadata",
  "typeProperties": {
    "dataset": {
      "referenceName": "DS_CompressedCSV_GZip",
      "parameters": {
        "fileName": "*"
      }
    },
    "fieldList": ["childItems", "itemName", "itemType"],
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": true,
      "wildcardFileName": "*.gz"
    }
  }
}
```

**2. ForEach Activity** - Process Each Compressed File:
```json
{
  "name": "ForEach_CompressedFile",
  "type": "ForEach",
  "dependsOn": [
    {
      "activity": "GetMetadata_CompressedFiles",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "items": "@activity('GetMetadata_CompressedFiles').output.childItems",
    "isSequential": false,
    "batchCount": 5,
    "activities": []
  }
}
```

**3. Copy Activity - Extract and Load (Automatic Decompression)**:
```json
{
  "name": "Copy_ExtractAndLoad",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings",
        "recursive": false
      },
      "formatSettings": {
        "type": "DelimitedTextReadSettings",
        "compressionProperties": {
          "type": "GZipReadSettings"
        }
      }
    },
    "sink": {
      "type": "AzureSqlSink",
      "preCopyScript": "TRUNCATE TABLE staging.PatientData",
      "writeBehavior": "insert",
      "sqlWriterUseTableLock": false,
      "tableOption": "autoCreate"
    },
    "enableStaging": false,
    "validateDataConsistency": true,
    "dataIntegrationUnits": 4
  },
  "inputs": [
    {
      "referenceName": "DS_CompressedCSV_GZip",
      "parameters": {
        "fileName": "@item().name"
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_PatientData"
    }
  ]
}
```

**4. Copy Activity - Extract to Intermediate Storage**:
```json
{
  "name": "Copy_ExtractToADLS",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings"
      },
      "formatSettings": {
        "type": "DelimitedTextReadSettings"
      }
    },
    "sink": {
      "type": "DelimitedTextSink",
      "storeSettings": {
        "type": "AzureBlobFSWriteSettings"
      },
      "formatSettings": {
        "type": "DelimitedTextWriteSettings",
        "quoteAllText": true,
        "fileExtension": ".csv"
      }
    },
    "enableStaging": false
  },
  "inputs": [
    {
      "referenceName": "DS_CompressedCSV_GZip",
      "parameters": {
        "fileName": "@item().name"
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_ExtractedCSV",
      "parameters": {
        "folderPath": "@concat('extracted/', formatDateTime(utcnow(), 'yyyy-MM-dd'))"
      }
    }
  ]
}
```

**5. Copy Activity - Compress and Archive**:
```json
{
  "name": "Copy_CompressAndArchive",
  "type": "Copy",
  "dependsOn": [
    {
      "activity": "Copy_ExtractAndLoad",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobFSReadSettings"
      }
    },
    "sink": {
      "type": "DelimitedTextSink",
      "storeSettings": {
        "type": "AzureBlobStorageWriteSettings"
      },
      "formatSettings": {
        "type": "DelimitedTextWriteSettings",
        "fileExtension": ".csv.gz"
      }
    }
  },
  "inputs": [
    {
      "referenceName": "DS_ExtractedCSV",
      "parameters": {
        "folderPath": "@concat('extracted/', formatDateTime(utcnow(), 'yyyy-MM-dd'))"
      }
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_CompressedArchive",
      "parameters": {
        "fileName": "@concat(item().name, '_', formatDateTime(utcnow(), 'yyyyMMddHHmmss'), '.gz')"
      }
    }
  ]
}
```

### Supported Compression Formats in ADF

| Format | Read | Write | Extension | Use Case |
|--------|------|-------|-----------|----------|
| **GZip** | ✅ | ✅ | .gz | General purpose, good compression ratio |
| **Deflate** | ✅ | ✅ | .deflate | Similar to GZip, slightly faster |
| **BZip2** | ✅ | ✅ | .bz2 | Better compression than GZip, slower |
| **ZipDeflate** | ✅ | ✅ | .zip | Multiple files in single archive |
| **Snappy** | ✅ | ✅ | .snappy | Fast compression, for Parquet/ORC |
| **LZ4** | ✅ | ✅ | .lz4 | Fastest compression/decompression |
| **TarGZip** | ✅ | ❌ | .tar.gz | Multiple files, read-only |

### Compression Configuration Examples

**1. GZip with Compression Level**:
```json
{
  "compressionCodec": "gzip",
  "compressionLevel": "optimal"  // Options: fastest, optimal, or 1-9
}
```

**2. Zip with Multiple Files**:
```json
{
  "compressionCodec": "zipDeflate",
  "compressionLevel": "optimal",
  "preserveZipFileNameAsFolder": false
}
```

**3. BZip2 for Maximum Compression**:
```json
{
  "compressionCodec": "bzip2"
}
```

### Advanced Scenario: Multi-File Zip Handling

For Zip files containing multiple CSV files:

```json
{
  "name": "Copy_MultiFileZip",
  "type": "Copy",
  "typeProperties": {
    "source": {
      "type": "DelimitedTextSource",
      "storeSettings": {
        "type": "AzureBlobStorageReadSettings",
        "recursive": true,
        "wildcardFileName": "*.csv"
      },
      "formatSettings": {
        "type": "DelimitedTextReadSettings",
        "compressionProperties": {
          "type": "ZipDeflateReadSettings",
          "preserveZipFileNameAsFolder": true
        }
      }
    }
  }
}
```

### Expected Behavior
- GZip compressed file (100MB) extracts to 500MB CSV in ~2 minutes
- Automatic decompression during read (no intermediate storage needed)
- Compression ratio: 5:1 for text files, 2:1 for binary files
- Parallel processing of 5 compressed files simultaneously

### Common Issues and Resolutions

**Issue 1: Corrupted Compressed File**
- **Error**: "Failed to decompress file: Unexpected end of stream"
- **Cause**: Incomplete file upload, network interruption, or file corruption
- **Solution**:
  - Validate file integrity using GetMetadata (check file size)
  - Implement retry logic with exponential backoff
  - Use MD5 checksum validation
  - Add error handling to skip corrupted files and log them
  ```json
  {
    "validateDataConsistency": true,
    "skipErrorFile": {
      "enabled": true,
      "fileMissing": true,
      "dataInconsistency": true
    }
  }
  ```

**Issue 2: Zip File with Multiple Files**
- **Error**: "Cannot determine which file to read from Zip archive"
- **Cause**: Zip contains multiple files, ADF needs to know which one to read
- **Solution**:
  - Use wildcard pattern: `"wildcardFileName": "*.csv"`
  - Set `preserveZipFileNameAsFolder: true` to extract all files
  - Use recursive read to process all files in Zip
  - Alternative: Use Azure Function to extract Zip first

**Issue 3: Memory Issues with Large Compressed Files**
- **Error**: "OutOfMemoryException during decompression"
- **Cause**: Compressed file is small (100MB) but extracts to very large file (5GB)
- **Solution**:
  - Enable staging for large files
  - Increase DIU from 4 to 8 or 16
  - Use Data Flow instead of Copy activity for better memory management
  - Split large files before compression

**Issue 4: Unsupported Compression Format**
- **Error**: "Compression format 'rar' is not supported"
- **Cause**: File is in RAR, 7z, or other unsupported format
- **Solution**:
  - Use Azure Function to extract unsupported formats
  - Request source system to use supported formats (GZip, Zip)
  - Use custom activity with extraction tool
  - Pre-process files using Azure Batch

**Issue 5: Compression Level Not Applied**
- **Symptom**: Compressed output file is larger than expected
- **Cause**: Compression level set to "fastest" or data is already compressed
- **Solution**:
  - Change compression level to "optimal" for better ratio
  - Check if source data is already compressed (images, videos)
  - Use BZip2 for maximum compression (slower)
  - Verify compression codec is correctly set

**Issue 6: Performance Degradation with Compression**
- **Symptom**: Pipeline takes 3x longer when writing compressed files
- **Cause**: Compression is CPU-intensive, especially for large files
- **Solution**:
  - Use faster compression (Snappy, LZ4) instead of GZip
  - Increase DIU to provide more CPU resources
  - Compress files in parallel using ForEach
  - Consider: Is compression worth the time? (for frequently accessed data, maybe not)

### Performance Comparison

**Decompression Performance** (100MB GZip → 500MB CSV):
- **DIU 4**: ~3 minutes
- **DIU 8**: ~1.5 minutes
- **DIU 16**: ~1 minute

**Compression Performance** (500MB CSV → 100MB GZip):
- **Fastest**: ~2 minutes (120MB output)
- **Optimal**: ~4 minutes (100MB output)
- **Maximum (BZip2)**: ~8 minutes (80MB output)

### Interview Questions

**Q1: How does ADF handle compressed files? Do you need to extract them first?**
**Answer**: 
No, ADF automatically handles decompression during read operations. You just specify the compression codec in the dataset, and ADF extracts on-the-fly. For example:
- Set `"compressionCodec": "gzip"` in source dataset
- Copy activity reads and decompresses automatically
- No intermediate storage needed for extraction
- Supports GZip, Zip, BZip2, Snappy, LZ4, Deflate

However, for complex scenarios (Zip with multiple files, unsupported formats), you may need explicit extraction steps.

**Q2: What's the difference between GZip and Zip compression in ADF?**
**Answer**:
- **GZip (.gz)**:
  - Single file compression
  - Better for streaming/pipeline processing
  - Faster decompression
  - Commonly used for log files, CSV files
  - Example: `data.csv.gz`
  
- **Zip (.zip)**:
  - Can contain multiple files
  - Archive format with directory structure
  - Slightly slower processing
  - Need to specify which file to extract
  - Example: `data.zip` containing multiple CSVs

**Q3: When should you compress files and when should you leave them uncompressed?**
**Answer**:
**Compress when**:
- Storage cost is a concern (Cool/Archive tier)
- Transferring data over network (reduce bandwidth)
- Archiving processed data
- Source system provides compressed files
- Data is text-based (CSV, JSON, XML) - high compression ratio

**Don't compress when**:
- Frequently accessed data (hot tier) - decompression overhead
- Data is already compressed (images, videos, Parquet with Snappy)
- Real-time processing requirements - compression adds latency
- Small files (<10MB) - overhead not worth it
- Using formats with built-in compression (Parquet, ORC)

**Q4: How do you handle a Zip file containing multiple CSV files?**
**Answer**:
```json
{
  "source": {
    "type": "DelimitedTextSource",
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": true,
      "wildcardFileName": "*.csv"
    },
    "formatSettings": {
      "type": "DelimitedTextReadSettings",
      "compressionProperties": {
        "type": "ZipDeflateReadSettings",
        "preserveZipFileNameAsFolder": true
      }
    }
  }
}
```
- Set `recursive: true` to read all files
- Use wildcard pattern to filter file types
- ADF extracts and processes each file separately
- Alternative: Extract to ADLS first, then process

**Q5: What compression format gives the best performance vs compression ratio?**
**Answer**:
**For storage (archive)**:
- **BZip2**: Best compression ratio (~30% smaller than GZip), but slowest
- Use for long-term archive, rarely accessed data

**For processing (hot data)**:
- **Snappy**: Best performance, moderate compression (~50% of GZip ratio)
- Use for Parquet files, frequently accessed data

**Balanced**:
- **GZip (optimal)**: Good balance of compression and speed
- Default choice for most scenarios

**Fastest**:
- **LZ4**: Fastest compression/decompression, lowest compression ratio
- Use for real-time pipelines with latency requirements

**Q6: How do you optimize compression performance in ADF?**
**Answer**:
1. **Increase DIU**: More CPU resources for compression/decompression
2. **Parallel Processing**: Use ForEach to compress multiple files simultaneously
3. **Choose Right Codec**: Snappy for speed, GZip for balance, BZip2 for size
4. **Compression Level**: Use "fastest" for speed, "optimal" for balance
5. **File Size**: Compress files in 100-500MB chunks for optimal performance
6. **Enable Staging**: For very large files, use staging to avoid memory issues

### Pro Tips
1. **Test Compression Ratios**: Different data types compress differently - test with real data
2. **Monitor Costs**: Compression reduces storage cost but increases compute cost
3. **Use Snappy for Parquet**: Snappy is the default and best choice for Parquet files
4. **Archive with BZip2**: Use BZip2 for long-term archives to maximize storage savings
5. **Validate After Compression**: Always compare row counts before and after compression
6. **Consider Network**: For cross-region transfers, compression saves bandwidth costs

### Common Mistakes to Avoid
1. ❌ Compressing already compressed data (Parquet, images, videos)
2. ❌ Using maximum compression for frequently accessed data
3. ❌ Not handling corrupted compressed files
4. ❌ Ignoring compression overhead in pipeline timing
5. ❌ Using wrong compression format for use case
6. ❌ Not testing decompression with downstream systems

### Real-World Anecdote
"In my previous project for a healthcare client, we received daily patient data files (5GB each) in GZip format from 20+ hospitals. Initially, we extracted all files to ADLS before processing, which consumed 100GB daily storage. We optimized by reading compressed files directly in Copy activity, which eliminated intermediate storage and reduced pipeline time from 45 minutes to 15 minutes. We also implemented BZip2 compression for archived data, reducing archive storage costs by 40% compared to GZip. The key was understanding when to decompress (for processing) and when to keep compressed (for archive)."

---

## Scenario 21: Transaction Log Processing (CDC - Change Data Capture)

### Business Requirement
An e-commerce company needs to capture and process changes (inserts, updates, deletes) from their SQL Server transactional database in real-time and replicate them to Azure Synapse Analytics for reporting. They want to track only changed records instead of doing full loads every time, reducing processing time and costs.

### Implementation Steps

#### Step 1: Enable CDC on Source Database

**On SQL Server (Source)**:
```sql
-- Enable CDC on database
USE [EcommerceDB];
GO
EXEC sys.sp_cdc_enable_db;
GO

-- Enable CDC on specific table
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Orders',
    @role_name = NULL,
    @supports_net_changes = 1;
GO

-- Verify CDC is enabled
SELECT name, is_cdc_enabled 
FROM sys.databases 
WHERE name = 'EcommerceDB';

SELECT name, is_tracked_by_cdc 
FROM sys.tables 
WHERE name = 'Orders';
```

**CDC System Tables Created**:
- `cdc.dbo_Orders_CT` - Change table storing all changes
- `cdc.change_tables` - Metadata about CDC tables
- `cdc.captured_columns` - Columns being tracked

#### Step 2: Create Linked Services

- **Source**: SQL Server (on-premises or Azure SQL with CDC enabled)
- **Destination**: Azure Synapse Analytics
- **Control**: Azure SQL Database (for watermark tracking)

#### Step 3: Create Datasets

**1. CDC Source Dataset**:
```json
{
  "name": "DS_CDC_Orders",
  "type": "SqlServerTable",
  "linkedServiceName": "LS_SQLServer",
  "typeProperties": {
    "schema": "cdc",
    "table": "dbo_Orders_CT"
  }
}
```

**2. Watermark Control Dataset**:
```json
{
  "name": "DS_WatermarkTable",
  "type": "AzureSqlTable",
  "linkedServiceName": "LS_ControlDB",
  "typeProperties": {
    "schema": "control",
    "table": "Watermark"
  }
}
```

**3. Synapse Destination Dataset**:
```json
{
  "name": "DS_Synapse_Orders",
  "type": "AzureSqlDWTable",
  "linkedServiceName": "LS_Synapse",
  "typeProperties": {
    "schema": "dbo",
    "table": "Orders"
  }
}
```

#### Step 4: Create Control Table

```sql
-- Create watermark control table
CREATE TABLE control.Watermark (
    TableName NVARCHAR(100) PRIMARY KEY,
    WatermarkValue DATETIME2,
    LastUpdateTime DATETIME2 DEFAULT GETDATE()
);

-- Initialize watermark for Orders table
INSERT INTO control.Watermark (TableName, WatermarkValue)
VALUES ('Orders', '1900-01-01 00:00:00');
```

#### Step 5: Build the Pipeline

**Pipeline: PL_CDC_Orders_Incremental**

**1. Lookup Activity - Get Old Watermark**:
```json
{
  "name": "LKP_GetOldWatermark",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "AzureSqlSource",
      "sqlReaderQuery": "SELECT WatermarkValue FROM control.Watermark WHERE TableName = 'Orders'"
    },
    "dataset": {
      "referenceName": "DS_WatermarkTable"
    },
    "firstRowOnly": true
  }
}
```

**2. Lookup Activity - Get New Watermark**:
```json
{
  "name": "LKP_GetNewWatermark",
  "type": "Lookup",
  "typeProperties": {
    "source": {
      "type": "SqlServerSource",
      "sqlReaderQuery": "SELECT MAX(__$start_lsn) as NewWatermarkValue FROM cdc.dbo_Orders_CT"
    },
    "dataset": {
      "referenceName": "DS_CDC_Orders"
    },
    "firstRowOnly": true
  }
}
```

**3. Copy Activity - Extract CDC Changes**:
```json
{
  "name": "Copy_CDCChanges",
  "type": "Copy",
  "dependsOn": [
    {
      "activity": "LKP_GetOldWatermark",
      "dependencyConditions": ["Succeeded"]
    },
    {
      "activity": "LKP_GetNewWatermark",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "source": {
      "type": "SqlServerSource",
      "sqlReaderQuery": {
        "value": "SELECT \n    __$start_lsn,\n    __$operation,\n    OrderID,\n    CustomerID,\n    OrderDate,\n    TotalAmount,\n    Status\nFROM cdc.dbo_Orders_CT\nWHERE __$start_lsn > @{activity('LKP_GetOldWatermark').output.firstRow.WatermarkValue}\n  AND __$start_lsn <= @{activity('LKP_GetNewWatermark').output.firstRow.NewWatermarkValue}",
        "type": "Expression"
      }
    },
    "sink": {
      "type": "SqlDWSink",
      "preCopyScript": "TRUNCATE TABLE staging.Orders_CDC",
      "writeBehavior": "insert"
    },
    "enableStaging": true,
    "stagingSettings": {
      "linkedServiceName": "LS_BlobStorage",
      "path": "staging"
    }
  },
  "inputs": [
    {
      "referenceName": "DS_CDC_Orders"
    }
  ],
  "outputs": [
    {
      "referenceName": "DS_Synapse_Orders_Staging"
    }
  ]
}
```

**4. Stored Procedure Activity - Process CDC Changes**:
```json
{
  "name": "SP_ProcessCDCChanges",
  "type": "SqlServerStoredProcedure",
  "dependsOn": [
    {
      "activity": "Copy_CDCChanges",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "storedProcedureName": "[dbo].[usp_ProcessCDCChanges]"
  },
  "linkedServiceName": {
    "referenceName": "LS_Synapse"
  }
}
```

**5. Stored Procedure Activity - Update Watermark**:
```json
{
  "name": "SP_UpdateWatermark",
  "type": "SqlServerStoredProcedure",
  "dependsOn": [
    {
      "activity": "SP_ProcessCDCChanges",
      "dependencyConditions": ["Succeeded"]
    }
  ],
  "typeProperties": {
    "storedProcedureName": "[control].[usp_UpdateWatermark]",
    "storedProcedureParameters": {
      "TableName": {
        "value": "Orders",
        "type": "String"
      },
      "WatermarkValue": {
        "value": {
          "value": "@activity('LKP_GetNewWatermark').output.firstRow.NewWatermarkValue",
          "type": "Expression"
        },
        "type": "DateTime"
      }
    }
  },
  "linkedServiceName": {
    "referenceName": "LS_ControlDB"
  }
}
```

#### Step 6: Create Stored Procedures

**Stored Procedure to Process CDC Changes** (in Synapse):
```sql
CREATE PROCEDURE [dbo].[usp_ProcessCDCChanges]
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Process Inserts (__$operation = 2)
    INSERT INTO dbo.Orders (OrderID, CustomerID, OrderDate, TotalAmount, Status, LastModified)
    SELECT 
        OrderID, 
        CustomerID, 
        OrderDate, 
        TotalAmount, 
        Status,
        GETDATE()
    FROM staging.Orders_CDC
    WHERE __$operation = 2  -- Insert
      AND NOT EXISTS (SELECT 1 FROM dbo.Orders WHERE OrderID = staging.Orders_CDC.OrderID);
    
    -- Process Updates (__$operation = 4)
    UPDATE t
    SET 
        t.CustomerID = s.CustomerID,
        t.OrderDate = s.OrderDate,
        t.TotalAmount = s.TotalAmount,
        t.Status = s.Status,
        t.LastModified = GETDATE()
    FROM dbo.Orders t
    INNER JOIN staging.Orders_CDC s ON t.OrderID = s.OrderID
    WHERE s.__$operation = 4;  -- Update
    
    -- Process Deletes (__$operation = 1)
    DELETE t
    FROM dbo.Orders t
    INNER JOIN staging.Orders_CDC s ON t.OrderID = s.OrderID
    WHERE s.__$operation = 1;  -- Delete
    
    -- Alternative: Soft delete instead of hard delete
    -- UPDATE t
    -- SET t.IsDeleted = 1, t.DeletedDate = GETDATE()
    -- FROM dbo.Orders t
    -- INNER JOIN staging.Orders_CDC s ON t.OrderID = s.OrderID
    -- WHERE s.__$operation = 1;
    
END;
GO
```

**Stored Procedure to Update Watermark** (in Control DB):
```sql
CREATE PROCEDURE [control].[usp_UpdateWatermark]
    @TableName NVARCHAR(100),
    @WatermarkValue DATETIME2
AS
BEGIN
    UPDATE control.Watermark
    SET WatermarkValue = @WatermarkValue,
        LastUpdateTime = GETDATE()
    WHERE TableName = @TableName;
END;
GO
```

### Understanding CDC Operation Codes

| __$operation | Operation Type | Description |
|--------------|----------------|-------------|
| 1 | Delete | Row was deleted |
| 2 | Insert | Row was inserted |
| 3 | Before Update | Row values before update |
| 4 | After Update | Row values after update |

### Alternative Approach: Using Data Flow for CDC Processing

**Data Flow: DF_ProcessCDC**

1. **Source Transformation** - Read CDC Changes:
   - Dataset: DS_CDC_Orders
   - Filter: `__$start_lsn > $oldWatermark AND __$start_lsn <= $newWatermark`

2. **Conditional Split** - Separate Operations:
   - **Inserts**: `__$operation == 2`
   - **Updates**: `__$operation == 4`
   - **Deletes**: `__$operation == 1`

3. **Alter Row Transformation**:
   - **Insert Stream**: Set insert condition
   - **Update Stream**: Set update condition on OrderID
   - **Delete Stream**: Set delete condition on OrderID

4. **Sink Transformation** - Write to Synapse:
   - Dataset: DS_Synapse_Orders
   - Update method: Allow insert, Allow update, Allow delete
   - Key columns: OrderID

### Advanced: Net Changes Only

If you only want the final state (net changes) instead of all intermediate changes:

```sql
-- Use CDC function to get net changes
DECLARE @from_lsn binary(10), @to_lsn binary(10);

SET @from_lsn = @OldWatermark;
SET @to_lsn = @NewWatermark;

SELECT *
FROM cdc.fn_cdc_get_net_changes_dbo_Orders(@from_lsn, @to_lsn, 'all')
ORDER BY __$start_lsn;
```

### Expected Behavior
- CDC captures 1,000 changes (500 inserts, 300 updates, 200 deletes)
- Pipeline processes only changed records (not full 10 million rows)
- Processing time: 2 minutes vs 30 minutes for full load
- Watermark updated to latest LSN for next run
- Near real-time replication (run every 5-15 minutes)

### Common Issues and Resolutions

**Issue 1: CDC Not Capturing Changes**
- **Symptom**: CDC table is empty even though source table has changes
- **Cause**: CDC not enabled or SQL Server Agent not running
- **Solution**:
  - Verify CDC is enabled: `SELECT is_cdc_enabled FROM sys.databases`
  - Check SQL Server Agent is running (CDC requires Agent)
  - Verify capture job is running: `EXEC sys.sp_cdc_help_jobs`
  - Check CDC job errors: `SELECT * FROM msdb.dbo.cdc_errors`

**Issue 2: Watermark Not Updating**
- **Error**: Pipeline keeps processing same changes repeatedly
- **Cause**: Watermark update step failing or not executing
- **Solution**:
  - Add dependency: Update watermark only after successful processing
  - Add error handling: Use Try-Catch in stored procedure
  - Log watermark values for debugging
  - Use transaction to ensure atomicity

**Issue 3: LSN Out of Range**
- **Error**: "LSN value is not within the valid range"
- **Cause**: CDC retention period expired, old LSN cleaned up
- **Solution**:
  - Increase CDC retention: `EXEC sys.sp_cdc_change_job @job_type = 'cleanup', @retention = 4320` (3 days)
  - Reset watermark to current LSN if too old
  - Run pipeline more frequently to avoid retention issues
  - Monitor CDC cleanup job schedule

**Issue 4: Performance Issues with Large CDC Tables**
- **Symptom**: Pipeline takes very long to read CDC changes
- **Cause**: CDC table has millions of rows, no proper filtering
- **Solution**:
  - Always use LSN range filtering in query
  - Create index on __$start_lsn column
  - Increase pipeline frequency to process smaller batches
  - Use Data Flow with partitioning for large volumes
  - Enable staging for Copy activity

**Issue 5: Handling Schema Changes**
- **Error**: "Column 'NewColumn' does not exist in target table"
- **Cause**: Source table schema changed but CDC/target not updated
- **Solution**:
  - Disable and re-enable CDC after schema changes
  - Update target table schema before enabling CDC
  - Use schema drift handling in Data Flow
  - Implement schema validation before processing

**Issue 6: Duplicate Processing**
- **Symptom**: Same changes processed multiple times
- **Cause**: Pipeline failed after processing but before updating watermark
- **Solution**:
  - Use idempotent operations (MERGE instead of INSERT)
  - Add unique constraint on target table
  - Use transaction log to track processed LSNs
  - Implement deduplication logic in stored procedure

### Performance Optimization

**1. Increase Pipeline Frequency**:
- Run every 5-15 minutes instead of hourly
- Smaller batches process faster
- Reduces CDC table size

**2. Use Staging**:
```json
{
  "enableStaging": true,
  "stagingSettings": {
    "linkedServiceName": "LS_BlobStorage",
    "path": "staging"
  }
}
```

**3. Partition CDC Processing**:
For very large CDC volumes, partition by date or ID range:
```sql
WHERE __$start_lsn > @OldWatermark 
  AND __$start_lsn <= @NewWatermark
  AND OrderID BETWEEN @MinID AND @MaxID
```

**4. Use MERGE for Upsert**:
```sql
MERGE dbo.Orders AS target
USING staging.Orders_CDC AS source
ON target.OrderID = source.OrderID
WHEN MATCHED AND source.__$operation = 4 THEN
    UPDATE SET 
        target.CustomerID = source.CustomerID,
        target.TotalAmount = source.TotalAmount
WHEN NOT MATCHED AND source.__$operation = 2 THEN
    INSERT (OrderID, CustomerID, TotalAmount)
    VALUES (source.OrderID, source.CustomerID, source.TotalAmount)
WHEN MATCHED AND source.__$operation = 1 THEN
    DELETE;
```

### Interview Questions

**Q1: What is CDC and why use it instead of full loads?**
**Answer**:
CDC (Change Data Capture) is a SQL Server feature that tracks and captures insert, update, and delete operations on tables. Benefits:
1. **Efficiency**: Process only changed records (1,000 rows) vs full table (10 million rows)
2. **Performance**: Reduces load on source system and network bandwidth
3. **Near Real-Time**: Can run every few minutes for near real-time sync
4. **Cost**: Lower compute and storage costs
5. **Audit Trail**: Maintains history of all changes with timestamps

**Q2: How does CDC work internally in SQL Server?**
**Answer**:
1. **Transaction Log Reader**: CDC reads SQL Server transaction log
2. **Capture Process**: SQL Server Agent job captures changes asynchronously
3. **Change Tables**: Changes stored in `cdc.<schema>_<table>_CT` tables
4. **LSN Tracking**: Uses Log Sequence Number (LSN) to track changes chronologically
5. **Cleanup Job**: Another Agent job cleans up old changes based on retention period
6. **Minimal Overhead**: ~5-10% performance impact on source system

**Q3: What's the difference between CDC and Change Tracking?**
**Answer**:
| Feature | CDC | Change Tracking |
|---------|-----|-----------------|
| **Data Captured** | All column values | Only primary key + version |
| **Historical Data** | Yes, stores all changes | No, only latest version |
| **Overhead** | Higher (~5-10%) | Lower (~2-5%) |
| **Use Case** | Full replication, audit | Sync primary keys only |
| **Requires Agent** | Yes | No |
| **Retention** | Configurable | Auto-cleanup |

**Q4: How do you handle CDC in ADF for multiple tables?**
**Answer**:
Metadata-driven approach:
1. Create control table with list of CDC-enabled tables
2. Use Lookup to get table list
3. ForEach activity to process each table
4. Parameterize pipeline with table name, schema, watermark
5. Single pipeline handles all tables dynamically
6. Separate watermark for each table

```json
{
  "name": "ForEach_CDCTables",
  "type": "ForEach",
  "typeProperties": {
    "items": "@activity('LKP_GetCDCTables').output.value",
    "activities": [
      {
        "name": "Execute_CDCPipeline",
        "type": "ExecutePipeline",
        "typeProperties": {
          "pipeline": {
            "referenceName": "PL_CDC_Generic"
          },
          "parameters": {
            "TableName": "@item().TableName",
            "Schema": "@item().Schema"
          }
        }
      }
    ]
  }
}
```

**Q5: What happens if pipeline fails mid-processing?**
**Answer**:
**Problem**: Changes copied to staging but not processed, watermark not updated
**Solutions**:
1. **Idempotent Processing**: Use MERGE instead of INSERT - can rerun safely
2. **Transaction Control**: Wrap processing and watermark update in transaction
3. **Checkpoint Pattern**: Track processed LSN ranges separately
4. **Retry Logic**: Configure automatic retry with exponential backoff
5. **Alerting**: Set up monitoring to detect failures immediately

**Q6: How do you handle deletes in CDC?**
**Answer**:
Two approaches:
1. **Hard Delete**: Actually delete from target table
   - Pros: Target matches source exactly
   - Cons: Lose historical data, cannot track when deleted
   
2. **Soft Delete**: Add IsDeleted flag and DeletedDate columns
   - Pros: Maintain history, can restore if needed
   - Cons: Queries need to filter IsDeleted = 0
   - Recommended for data warehouses

```sql
-- Soft delete approach
UPDATE target
SET IsDeleted = 1, 
    DeletedDate = GETDATE(),
    DeletedBy = SYSTEM_USER
WHERE OrderID IN (
    SELECT OrderID FROM staging.Orders_CDC WHERE __$operation = 1
);
```

### Pro Tips
1. **Monitor CDC Table Size**: Set appropriate retention period to avoid bloat
2. **Index LSN Column**: Create index on `__$start_lsn` for faster queries
3. **Use Net Changes**: For high-frequency updates, use `fn_cdc_get_net_changes` to get final state only
4. **Frequent Runs**: Run pipeline every 5-15 minutes for near real-time sync
5. **Backup Strategy**: CDC doesn't replace backups - maintain separate backup strategy
6. **Test Failover**: Regularly test pipeline failure and recovery scenarios

### Common Mistakes to Avoid
1. ❌ Not checking if SQL Server Agent is running (CDC requires it)
2. ❌ Using CDC without proper LSN filtering (processing all changes every time)
3. ❌ Not handling schema changes (CDC breaks when columns added/removed)
4. ❌ Setting retention too short (LSN out of range errors)
5. ❌ Not updating watermark after successful processing
6. ❌ Hard deleting records without maintaining history

### Real-World Anecdote
"In my previous project for an e-commerce client, we had to replicate 50 tables from on-premises SQL Server to Azure Synapse for real-time reporting. Initially, we did full loads every hour which took 45 minutes and impacted production. We implemented CDC with ADF, running every 10 minutes, and processing time dropped to 2-3 minutes. The key challenges were handling schema changes (we added schema validation step), managing CDC retention (increased to 3 days), and ensuring idempotent processing (used MERGE). We also implemented soft deletes to maintain historical data for audit purposes. The solution reduced replication lag from 1 hour to 10 minutes and eliminated production impact."

---

## Summary

These four scenarios cover critical real-world patterns:

1. **Lookup Enrichment**: Adding reference data to transactions using Lookup activity and Data Flow transformations
2. **File Format Conversion**: Converting between CSV, JSON, Parquet formats with proper compression handling
3. **Compression Handling**: Reading and writing compressed files (GZip, Zip, BZip2) with performance optimization
4. **Transaction Log Processing (CDC)**: Capturing and processing incremental changes using Change Data Capture for near real-time replication

Each scenario includes:
- ✅ Complete implementation steps with JSON configurations
- ✅ Common issues and detailed resolutions
- ✅ Performance optimization techniques
- ✅ Interview questions with comprehensive answers
- ✅ Pro tips and common mistakes to avoid
- ✅ Real-world anecdotes demonstrating practical experience

These patterns are essential for any ADF practitioner and frequently come up in interviews and production implementations.

