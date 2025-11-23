# ADF-03: Linked Services and Datasets Deep Dive

## Understanding the Connection Architecture

**Think of it like a phone system:**
- **Linked Service** = Phone company account (connection settings, authentication)
- **Dataset** = Specific phone number (exact location of data)
- **Activity** = The actual phone call (the operation you perform)

### The Relationship

```
Linked Service (Connection)
    ↓
Dataset (Data Location)
    ↓
Activity (Operation)
```

**Example:**
```
Azure Blob Storage Linked Service (account: mystorageaccount)
    ↓
CSV Dataset (container: raw-data, folder: sales, file: sales.csv)
    ↓
Copy Activity (read the CSV file)
```

---

## LINKED SERVICES IN DEPTH

### What is a Linked Service?

A Linked Service defines the connection information needed for ADF to connect to external resources. It's like a connection string, but more powerful.

**Components of a Linked Service:**
1. **Type:** What kind of data store (Azure SQL, Blob Storage, REST API, etc.)
2. **Connection Properties:** Server name, account name, endpoint URL
3. **Authentication:** How to authenticate (key, MSI, service principal, SQL auth)
4. **Integration Runtime:** Which IR to use for connection

### Linked Service Lifecycle

```
Create → Test Connection → Use in Datasets → Monitor → Update/Rotate Credentials
```

---

## AZURE STORAGE LINKED SERVICES

### Azure Blob Storage Linked Service

**When to Use:**
- Storing raw files (CSV, JSON, Parquet)
- Staging area for data loads
- Archive storage
- Log files

#### Configuration Options:

**Option 1: Account Key Authentication**
```json
{
  "name": "AzureBlobStorageLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureBlobStorage",
    "typeProperties": {
      "connectionString": "DefaultEndpointsProtocol=https;AccountName=mystorageaccount;AccountKey=<key>;EndpointSuffix=core.windows.net"
    },
    "connectVia": {
      "referenceName": "AutoResolveIntegrationRuntime",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

**Option 2: SAS Token Authentication**
```json
{
  "name": "AzureBlobStorageSASLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureBlobStorage",
    "typeProperties": {
      "sasUri": "https://mystorageaccount.blob.core.windows.net/?sv=2021-06-08&ss=bfqt&srt=sco&sp=rwdlacupiytfx&se=2024-12-31T23:59:59Z&st=2024-01-01T00:00:00Z&spr=https&sig=<signature>",
      "sasToken": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "BlobSASToken"
      }
    }
  }
}
```

**Option 3: Managed Identity (Recommended)**
```json
{
  "name": "AzureBlobStorageMSILS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureBlobStorage",
    "typeProperties": {
      "serviceEndpoint": "https://mystorageaccount.blob.core.windows.net/",
      "authenticationType": "ManagedIdentity"
    },
    "connectVia": {
      "referenceName": "AutoResolveIntegrationRuntime",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

**Setup for Managed Identity:**
1. Enable Managed Identity on ADF
2. Go to Storage Account → Access Control (IAM)
3. Add role assignment: "Storage Blob Data Contributor" to ADF Managed Identity
4. Create linked service with MSI authentication

**Pro Tip:** Always use Managed Identity when possible. No credentials to manage or rotate!

#### Real-World Example: Multi-Environment Setup

**Scenario:** Same pipeline works in DEV, UAT, PROD with different storage accounts.

**Solution: Parameterized Linked Service**

```json
{
  "name": "AzureBlobStorageParameterizedLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureBlobStorage",
    "parameters": {
      "storageAccountName": {
        "type": "string"
      }
    },
    "typeProperties": {
      "serviceEndpoint": {
        "value": "@concat('https://', linkedService().storageAccountName, '.blob.core.windows.net/')",
        "type": "Expression"
      },
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

**Pipeline Usage:**
```json
{
  "linkedServiceName": {
    "referenceName": "AzureBlobStorageParameterizedLS",
    "type": "LinkedServiceReference",
    "parameters": {
      "storageAccountName": {
        "value": "@pipeline().globalParameters.StorageAccountName",
        "type": "Expression"
      }
    }
  }
}
```

**Global Parameters by Environment:**
- DEV: StorageAccountName = "devstorageaccount"
- UAT: StorageAccountName = "uatstorageaccount"
- PROD: StorageAccountName = "prodstorageaccount"

### Azure Data Lake Storage Gen2 Linked Service

**When to Use:**
- Data lake architecture
- Big data analytics
- Hierarchical namespace needed
- Better performance for analytics workloads

**Key Difference from Blob Storage:**
- Hierarchical namespace (real directories, not just prefixes)
- Better performance for analytics
- ACL support at file/folder level
- Optimized for big data workloads

#### Configuration:

```json
{
  "name": "AzureDataLakeStorageGen2LS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "https://mydatalake.dfs.core.windows.net/",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

**Note:** URL uses `dfs.core.windows.net` not `blob.core.windows.net`

#### Real-World Example: Data Lake Layers

**Scenario:** Implement Bronze/Silver/Gold data lake architecture.

**Storage Structure:**
```
mydatalake
├── bronze (raw data)
│   ├── sales/
│   ├── customers/
│   └── products/
├── silver (cleansed data)
│   ├── sales/
│   ├── customers/
│   └── products/
└── gold (aggregated data)
    ├── sales-summary/
    └── customer-360/
```

**Linked Service:** Single ADLS Gen2 linked service
**Datasets:** Separate datasets for each layer with parameterized paths

---

## DATABASE LINKED SERVICES

### Azure SQL Database Linked Service

**When to Use:**
- Relational data storage
- OLTP workloads
- Control tables
- Metadata storage
- Small to medium data warehouses

#### Authentication Methods:

**1. SQL Authentication**
```json
{
  "name": "AzureSqlDatabaseLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;User ID=sqladmin;Password=<password>;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
    }
  }
}
```

**Better: SQL Authentication with Key Vault**
```json
{
  "name": "AzureSqlDatabaseKVLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;User ID=sqladmin;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "SqlPassword"
      }
    }
  }
}
```

**2. Managed Identity (Recommended)**
```json
{
  "name": "AzureSqlDatabaseMSILS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

**Setup for Managed Identity:**
```sql
-- Run in Azure SQL Database
CREATE USER [your-adf-name] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [your-adf-name];
ALTER ROLE db_datawriter ADD MEMBER [your-adf-name];
ALTER ROLE db_ddladmin ADD MEMBER [your-adf-name]; -- if creating tables
```

**3. Service Principal**
```json
{
  "name": "AzureSqlDatabaseSPLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net,1433;Database=mydb;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
      "authenticationType": "ServicePrincipal",
      "servicePrincipalId": "your-sp-app-id",
      "servicePrincipalKey": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "ServicePrincipalKey"
      },
      "tenant": "your-tenant-id"
    }
  }
}
```

#### Real-World Example: Control Table Pattern

**Scenario:** Use Azure SQL Database for pipeline orchestration metadata.

**Control Tables:**
```sql
-- Table configuration
CREATE TABLE TableConfig (
    TableID INT PRIMARY KEY,
    SourceSystem NVARCHAR(50),
    SchemaName NVARCHAR(50),
    TableName NVARCHAR(100),
    SourceQuery NVARCHAR(MAX),
    DestinationPath NVARCHAR(500),
    LoadFrequency NVARCHAR(20),
    IsActive BIT,
    LastLoadDate DATETIME,
    LastLoadStatus NVARCHAR(20)
);

-- Watermark table
CREATE TABLE WatermarkTable (
    TableName NVARCHAR(100) PRIMARY KEY,
    WatermarkColumn NVARCHAR(100),
    WatermarkValue DATETIME,
    LastUpdated DATETIME
);

-- Execution log
CREATE TABLE PipelineExecutionLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    PipelineRunID NVARCHAR(100),
    PipelineName NVARCHAR(100),
    ActivityName NVARCHAR(100),
    StartTime DATETIME,
    EndTime DATETIME,
    Status NVARCHAR(20),
    RowsRead BIGINT,
    RowsWritten BIGINT,
    ErrorMessage NVARCHAR(MAX)
);
```

**Pipeline Flow:**
```
1. Lookup: Get active tables from TableConfig
2. ForEach: Loop through tables
   ├─ Lookup: Get watermark from WatermarkTable
   ├─ Copy: Incremental load
   ├─ Stored Procedure: Update watermark
   └─ Stored Procedure: Log execution to PipelineExecutionLog
```

### Azure Synapse Analytics Linked Service

**When to Use:**
- Large-scale data warehousing (TB to PB)
- Complex analytics queries
- Star schema / dimensional modeling
- Integration with Power BI

#### Configuration:

```json
{
  "name": "AzureSynapseAnalyticsLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureSqlDW",
    "typeProperties": {
      "connectionString": "Server=tcp:mysynapse.sql.azuresynapse.net,1433;Database=mydw;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

#### Loading Data to Synapse: Best Practices

**1. Use PolyBase/COPY Command (Fastest)**

Enable staging in Copy Activity:
```json
{
  "enableStaging": true,
  "stagingSettings": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLS",
      "type": "LinkedServiceReference"
    },
    "path": "staging-container/staging-folder",
    "enableCompression": false
  },
  "polyBaseSettings": {
    "rejectType": "percentage",
    "rejectValue": 10,
    "rejectSampleValue": 100,
    "useTypeDefault": true
  }
}
```

**2. Use Appropriate Distribution**

```sql
-- Hash distribution for large fact tables
CREATE TABLE FactSales (
    SalesID BIGINT,
    CustomerID INT,
    Amount DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = HASH(CustomerID),
    CLUSTERED COLUMNSTORE INDEX
);

-- Round robin for staging tables
CREATE TABLE StagingSales (
    SalesID BIGINT,
    CustomerID INT,
    Amount DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = ROUND_ROBIN,
    HEAP
);

-- Replicated for small dimension tables
CREATE TABLE DimDate (
    DateKey INT,
    FullDate DATE,
    Year INT,
    Month INT
)
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED COLUMNSTORE INDEX
);
```

**3. Load Pattern: Staging → Production**

```
Pipeline:
1. Copy Activity: Load to staging table (HEAP, ROUND_ROBIN)
2. Stored Procedure: Transform and load to production
   - Data cleansing
   - Business rules
   - Merge/Upsert
3. Stored Procedure: Update statistics
4. Stored Procedure: Rebuild indexes if needed
```

### SQL Server (On-Premises) Linked Service

**When to Use:**
- On-premises SQL Server databases
- Hybrid cloud scenarios
- Legacy systems

**Requires:** Self-Hosted Integration Runtime

#### Configuration:

```json
{
  "name": "SqlServerLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "SqlServer",
    "typeProperties": {
      "connectionString": "Server=myserver;Database=mydb;User ID=sqladmin;Password=<password>;Encrypt=True;TrustServerCertificate=True;",
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
      "referenceName": "SelfHostedIR",
      "type": "IntegrationRuntimeReference"
    }
  }
}
```

**Windows Authentication:**
```json
{
  "typeProperties": {
    "connectionString": "Server=myserver;Database=mydb;Integrated Security=True;",
    "userName": "domain\\username",
    "password": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVaultLS",
        "type": "LinkedServiceReference"
      },
      "secretName": "WindowsPassword"
    }
  }
}
```

#### Troubleshooting On-Premises Connections:

**Issue 1: Connection Timeout**
```
Error: "A network-related or instance-specific error occurred"

Solutions:
1. Check firewall rules on SQL Server
2. Enable TCP/IP in SQL Server Configuration Manager
3. Verify SQL Server Browser service is running
4. Check if port 1433 is open
5. Test connection from Self-Hosted IR machine using SSMS
```

**Issue 2: Authentication Failed**
```
Error: "Login failed for user"

Solutions:
1. Verify credentials in Key Vault
2. Check SQL Server authentication mode (mixed mode)
3. Verify user has necessary permissions
4. Check if user account is locked
```

**Issue 3: Self-Hosted IR Offline**
```
Error: "The Integration Runtime is offline"

Solutions:
1. Check if IR service is running (Services.msc)
2. Verify network connectivity from IR machine
3. Check if IR machine has internet access (outbound 443)
4. Review IR logs: C:\ProgramData\Microsoft\Integration Runtime\
```

---

## CLOUD DATA PLATFORM LINKED SERVICES

### Snowflake Linked Service

**When to Use:**
- Snowflake as data warehouse
- Multi-cloud strategy
- Need Snowflake-specific features

#### Configuration:

```json
{
  "name": "SnowflakeLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "Snowflake",
    "typeProperties": {
      "connectionString": "jdbc:snowflake://myaccount.snowflakecomputing.com/?user=myuser&db=mydb&warehouse=mywh&schema=myschema",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "SnowflakePassword"
      }
    }
  }
}
```

**Key-Pair Authentication (More Secure):**
```json
{
  "typeProperties": {
    "connectionString": "jdbc:snowflake://myaccount.snowflakecomputing.com/?user=myuser&db=mydb&warehouse=mywh&schema=myschema&authenticator=snowflake_jwt",
    "privateKey": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVaultLS",
        "type": "LinkedServiceReference"
      },
      "secretName": "SnowflakePrivateKey"
    },
    "privateKeyPassphrase": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVaultLS",
        "type": "LinkedServiceReference"
      },
      "secretName": "SnowflakeKeyPassphrase"
    }
  }
}
```

#### Real-World Example: Azure to Snowflake

**Scenario:** Load data from Azure SQL Database to Snowflake.

**Pipeline:**
```
1. Copy Activity: Azure SQL → ADLS (Parquet)
   - Extract data to staging

2. Copy Activity: ADLS → Snowflake
   - Use Snowflake COPY INTO command
   - Leverage Snowflake's cloud storage integration

Sink Settings:
{
  "type": "SnowflakeSink",
  "preCopyScript": "TRUNCATE TABLE staging.orders",
  "importSettings": {
    "type": "SnowflakeImportCopyCommand",
    "additionalCopyOptions": {
      "ON_ERROR": "CONTINUE",
      "FORCE": "TRUE"
    }
  }
}
```

### Amazon S3 Linked Service

**When to Use:**
- Multi-cloud data integration
- Migrate data from AWS to Azure
- Access data stored in S3

#### Configuration:

```json
{
  "name": "AmazonS3LS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AmazonS3",
    "typeProperties": {
      "serviceUrl": "https://s3.amazonaws.com",
      "accessKeyId": "your-access-key-id",
      "secretAccessKey": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "AWSS3SecretKey"
      }
    }
  }
}
```

**IAM Role-Based (More Secure):**
```json
{
  "typeProperties": {
    "serviceUrl": "https://s3.amazonaws.com",
    "authenticationType": "TemporarySecurityCredentials",
    "sessionToken": {
      "type": "AzureKeyVaultSecret",
      "store": {
        "referenceName": "AzureKeyVaultLS",
        "type": "LinkedServiceReference"
      },
      "secretName": "AWSSessionToken"
    }
  }
}
```

---

## REST API AND WEB SERVICES

### REST API Linked Service

**When to Use:**
- Integrate with external APIs
- SaaS applications (Salesforce, Dynamics, ServiceNow)
- Custom web services
- Microservices integration

#### Configuration:

**Basic Authentication:**
```json
{
  "name": "RestAPILS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "RestService",
    "typeProperties": {
      "url": "https://api.example.com",
      "enableServerCertificateValidation": true,
      "authenticationType": "Basic",
      "userName": "apiuser",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "APIPassword"
      }
    }
  }
}
```

**OAuth2 Authentication:**
```json
{
  "name": "RestAPIOAuth2LS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "RestService",
    "typeProperties": {
      "url": "https://api.example.com",
      "authenticationType": "OAuth2ClientCredential",
      "clientId": "your-client-id",
      "clientSecret": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "OAuthClientSecret"
      },
      "tokenEndpoint": "https://login.example.com/oauth2/token",
      "scope": "api.read api.write"
    }
  }
}
```

**API Key Authentication:**
```json
{
  "name": "RestAPIKeyLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "RestService",
    "typeProperties": {
      "url": "https://api.example.com",
      "authenticationType": "Anonymous"
    }
  }
}
```

Note: For API Key, pass it in headers via Web Activity or dataset.

#### Real-World Example: Salesforce Integration

**Scenario:** Extract data from Salesforce using REST API.

**Linked Service:**
```json
{
  "name": "SalesforceRestLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "RestService",
    "typeProperties": {
      "url": "https://myinstance.salesforce.com",
      "authenticationType": "OAuth2ClientCredential",
      "clientId": "salesforce-connected-app-id",
      "clientSecret": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "AzureKeyVaultLS",
          "type": "LinkedServiceReference"
        },
        "secretName": "SalesforceClientSecret"
      },
      "tokenEndpoint": "https://login.salesforce.com/services/oauth2/token",
      "scope": "api"
    }
  }
}
```

**Pipeline:**
```
1. Web Activity: Get OAuth token
   URL: https://login.salesforce.com/services/oauth2/token
   Method: POST
   Body: {
     "grant_type": "client_credentials",
     "client_id": "...",
     "client_secret": "..."
   }

2. Copy Activity: Extract Accounts
   Source: REST API
   URL: /services/data/v57.0/query/?q=SELECT Id, Name, Industry FROM Account
   Headers: {
     "Authorization": "Bearer @{activity('GetToken').output.access_token}"
   }
   Sink: ADLS (JSON)

3. Repeat for other objects (Contacts, Opportunities, etc.)
```

---

## AZURE KEY VAULT LINKED SERVICE

**Critical for Security:** Store all secrets, connection strings, passwords in Key Vault.

### Configuration:

```json
{
  "name": "AzureKeyVaultLS",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "type": "AzureKeyVault",
    "typeProperties": {
      "baseUrl": "https://mykeyvault.vault.azure.net/"
    }
  }
}
```

**Setup:**
1. Create Azure Key Vault
2. Enable ADF Managed Identity
3. Grant ADF access to Key Vault:
   - Go to Key Vault → Access policies
   - Add access policy
   - Secret permissions: Get, List
   - Select principal: Your ADF name
   - Save

### Using Key Vault Secrets:

**In Linked Services:**
```json
{
  "password": {
    "type": "AzureKeyVaultSecret",
    "store": {
      "referenceName": "AzureKeyVaultLS",
      "type": "LinkedServiceReference"
    },
    "secretName": "SqlPassword"
  }
}
```

**In Pipeline (via Web Activity):**
```json
{
  "name": "GetSecretFromKV",
  "type": "WebActivity",
  "typeProperties": {
    "url": "https://mykeyvault.vault.azure.net/secrets/MySecret?api-version=7.0",
    "method": "GET",
    "authentication": {
      "type": "MSI",
      "resource": "https://vault.azure.net"
    }
  }
}
```

---

## DATASETS IN DEPTH

### What is a Dataset?

A Dataset is a named view of data that points to or references the data you want to use in activities. It's the "what" and "where" of your data.

**Dataset = Linked Service + Location + Structure**

### Dataset Components:

1. **Linked Service Reference:** Which connection to use
2. **Location:** Path, table name, file name
3. **Structure (Schema):** Column definitions (optional)
4. **Format:** CSV, JSON, Parquet, Avro, etc.
5. **Parameters:** Make dataset dynamic

---

## FILE-BASED DATASETS

### Delimited Text (CSV) Dataset

**Use Cases:**
- CSV files from vendors
- Export files
- Legacy system outputs

#### Configuration:

```json
{
  "name": "CsvDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLS",
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
      "rowDelimiter": "\n",
      "escapeChar": "\\",
      "quoteChar": "\"",
      "firstRowAsHeader": true,
      "compressionCodec": "gzip",
      "compressionLevel": "Optimal",
      "nullValue": "NULL"
    },
    "schema": [
      {
        "name": "OrderID",
        "type": "String"
      },
      {
        "name": "OrderDate",
        "type": "String"
      },
      {
        "name": "Amount",
        "type": "String"
      }
    ]
  }
}
```

**Key Properties:**
- **columnDelimiter:** Comma, pipe, tab (\t), etc.
- **rowDelimiter:** \n, \r\n
- **firstRowAsHeader:** true if first row contains column names
- **quoteChar:** Character used to quote values containing delimiter
- **escapeChar:** Character used to escape special characters
- **compressionCodec:** None, gzip, deflate, bzip2, ZipDeflate
- **nullValue:** String representing NULL

#### Handling Special Cases:

**Case 1: Pipe-Delimited with Header**
```json
{
  "columnDelimiter": "|",
  "firstRowAsHeader": true
}
```

**Case 2: Tab-Delimited, No Header**
```json
{
  "columnDelimiter": "\\t",
  "firstRowAsHeader": false
}
```

**Case 3: Custom Delimiter**
```json
{
  "columnDelimiter": "~|~",
  "quoteChar": "'"
}
```

**Case 4: Compressed Files**
```json
{
  "compressionCodec": "gzip",
  "compressionLevel": "Optimal"
}
```

### JSON Dataset

**Use Cases:**
- REST API responses
- NoSQL exports
- Log files
- Semi-structured data

#### Configuration:

**Array of JSON Objects:**
```json
{
  "name": "JsonDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureBlobStorageLS",
      "type": "LinkedServiceReference"
    },
    "type": "Json",
    "typeProperties": {
      "location": {
        "type": "AzureBlobStorageLocation",
        "fileName": "data.json",
        "folderPath": "raw",
        "container": "json-data"
      }
    },
    "schema": {
      "type": "object",
      "properties": {
        "id": {
          "type": "integer"
        },
        "name": {
          "type": "string"
        },
        "orders": {
          "type": "array"
        }
      }
    }
  }
}
```

**JSON Lines (JSONL):**
```json
{
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "fileName": "data.jsonl",
      "folderPath": "raw",
      "container": "json-data"
    },
    "encodingName": "UTF-8"
  }
}
```

**Nested JSON:**
Use Data Flow to flatten nested structures.

### Parquet Dataset

**Use Cases:**
- Data lake storage (recommended format)
- Big data analytics
- Columnar storage for analytics
- Schema evolution support

**Why Parquet?**
- Columnar format (faster analytics queries)
- Efficient compression
- Schema embedded in file
- Supports complex nested structures
- Industry standard for data lakes

#### Configuration:

```json
{
  "name": "ParquetDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureDataLakeStorageGen2LS",
      "type": "LinkedServiceReference"
    },
    "parameters": {
      "folderPath": {
        "type": "string"
      }
    },
    "type": "Parquet",
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "folderPath": {
          "value": "@dataset().folderPath",
          "type": "Expression"
        },
        "fileSystem": "datalake"
      },
      "compressionCodec": "snappy"
    }
  }
}
```

**Compression Options:**
- **snappy:** Fast, moderate compression (recommended for most cases)
- **gzip:** Slower, better compression
- **lzo:** Fast, less compression
- **none:** No compression

#### Real-World Example: Data Lake Partitioning

**Scenario:** Store sales data partitioned by year/month/day.

**Folder Structure:**
```
datalake/
└── sales/
    └── year=2024/
        └── month=01/
            ├── day=01/
            │   └── sales_20240101.parquet
            ├── day=02/
            │   └── sales_20240102.parquet
            └── day=03/
                └── sales_20240103.parquet
```

**Parameterized Dataset:**
```json
{
  "name": "PartitionedParquetDataset",
  "properties": {
    "parameters": {
      "year": {
        "type": "string"
      },
      "month": {
        "type": "string"
      },
      "day": {
        "type": "string"
      }
    },
    "type": "Parquet",
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "folderPath": {
          "value": "@concat('sales/year=', dataset().year, '/month=', dataset().month, '/day=', dataset().day)",
          "type": "Expression"
        },
        "fileSystem": "datalake"
      }
    }
  }
}
```

**Pipeline Usage:**
```json
{
  "outputs": [
    {
      "referenceName": "PartitionedParquetDataset",
      "type": "DatasetReference",
      "parameters": {
        "year": {
          "value": "@formatDateTime(pipeline().parameters.ProcessDate, 'yyyy')",
          "type": "Expression"
        },
        "month": {
          "value": "@formatDateTime(pipeline().parameters.ProcessDate, 'MM')",
          "type": "Expression"
        },
        "day": {
          "value": "@formatDateTime(pipeline().parameters.ProcessDate, 'dd')",
          "type": "Expression"
        }
      }
    }
  ]
}
```

### Avro Dataset

**Use Cases:**
- Schema evolution
- Kafka integration
- Event streaming
- Data serialization

**Avro vs Parquet:**
- Avro: Row-based, better for write-heavy workloads
- Parquet: Column-based, better for read-heavy analytics

---

## DATABASE DATASETS

### Azure SQL Table Dataset

**Use Cases:**
- Relational tables
- Control tables
- Metadata storage

#### Configuration:

**Static Table:**
```json
{
  "name": "SqlTableDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureSqlDatabaseLS",
      "type": "LinkedServiceReference"
    },
    "type": "AzureSqlTable",
    "typeProperties": {
      "schema": "dbo",
      "table": "Orders"
    },
    "schema": [
      {
        "name": "OrderID",
        "type": "int"
      },
      {
        "name": "OrderDate",
        "type": "datetime"
      },
      {
        "name": "Amount",
        "type": "decimal",
        "precision": 18,
        "scale": 2
      }
    ]
  }
}
```

**Parameterized Table:**
```json
{
  "name": "SqlTableParameterizedDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureSqlDatabaseLS",
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

**Query-Based (No Table):**
```json
{
  "name": "SqlQueryDataset",
  "properties": {
    "linkedServiceName": {
      "referenceName": "AzureSqlDatabaseLS",
      "type": "LinkedServiceReference"
    },
    "type": "AzureSqlTable"
  }
}
```

Use with sqlReaderQuery in Copy Activity source.

---

## DATASET PARAMETERIZATION STRATEGIES

### Strategy 1: File Name Parameterization

**Use Case:** Process different files with same structure.

**Dataset:**
```json
{
  "parameters": {
    "fileName": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "fileName": {
        "value": "@dataset().fileName",
        "type": "Expression"
      }
    }
  }
}
```

**Usage:**
```json
{
  "inputs": [
    {
      "referenceName": "CsvDataset",
      "type": "DatasetReference",
      "parameters": {
        "fileName": "sales_20240101.csv"
      }
    }
  ]
}
```

### Strategy 2: Dynamic Folder Path

**Use Case:** Date-based folder structure.

**Dataset:**
```json
{
  "parameters": {
    "year": {
      "type": "string"
    },
    "month": {
      "type": "string"
    }
  },
  "typeProperties": {
    "location": {
      "folderPath": {
        "value": "@concat('data/', dataset().year, '/', dataset().month)",
        "type": "Expression"
      }
    }
  }
}
```

### Strategy 3: Complete Path Parameterization

**Use Case:** Maximum flexibility.

**Dataset:**
```json
{
  "parameters": {
    "container": {
      "type": "string"
    },
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
      "container": {
        "value": "@dataset().container",
        "type": "Expression"
      },
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
```

### Strategy 4: Schema and Table Parameterization

**Use Case:** Metadata-driven table loads.

**Dataset:**
```json
{
  "parameters": {
    "schemaName": {
      "type": "string"
    },
    "tableName": {
      "type": "string"
    }
  },
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
```

---

## TROUBLESHOOTING GUIDE

### Common Linked Service Issues

**Issue 1: Test Connection Fails**

**Error:** "Cannot connect to database"

**Troubleshooting Steps:**
1. Check firewall rules
   - Azure SQL: Add ADF IP ranges or enable "Allow Azure services"
   - On-premises: Check firewall on server and network
2. Verify credentials in Key Vault
3. Check Integration Runtime status
4. Test connection from IR machine (for Self-Hosted IR)
5. Verify DNS resolution
6. Check if service is running

**Issue 2: Managed Identity Authentication Fails**

**Error:** "Login failed for user 'NT AUTHORITY\\ANONYMOUS LOGON'"

**Solution:**
```sql
-- Ensure user is created in database
CREATE USER [your-adf-name] FROM EXTERNAL PROVIDER;

-- Grant necessary permissions
ALTER ROLE db_datareader ADD MEMBER [your-adf-name];
ALTER ROLE db_datawriter ADD MEMBER [your-adf-name];

-- Verify
SELECT * FROM sys.database_principals WHERE name = 'your-adf-name';
```

**Issue 3: Key Vault Access Denied**

**Error:** "The user, group or application does not have secrets get permission"

**Solution:**
1. Go to Key Vault → Access policies
2. Click "Add Access Policy"
3. Secret permissions: Get, List
4. Select principal: Your ADF Managed Identity
5. Save

### Common Dataset Issues

**Issue 1: Schema Mismatch**

**Error:** "Column 'X' does not exist in the target"

**Solutions:**
1. Enable "Allow schema drift" in Data Flow
2. Use column mapping in Copy Activity
3. Update dataset schema definition
4. Use "Auto-map" in Copy Activity

**Issue 2: File Not Found**

**Error:** "The specified blob does not exist"

**Troubleshooting:**
1. Verify file path (case-sensitive in Linux-based systems)
2. Check if file exists using Get Metadata
3. Verify container/folder names
4. Check if using correct linked service
5. Verify permissions

**Issue 3: Encoding Issues**

**Error:** "Invalid character in file"

**Solutions:**
```json
{
  "typeProperties": {
    "encodingName": "UTF-8"  // or "UTF-16", "ISO-8859-1", etc.
  }
}
```

Common encodings:
- UTF-8 (most common)
- UTF-16
- ISO-8859-1 (Latin-1)
- Windows-1252

---

## BEST PRACTICES SUMMARY

### Linked Services:

1. **Use Managed Identity** wherever possible
2. **Store secrets in Key Vault**, never hardcode
3. **Parameterize for environments** (DEV/UAT/PROD)
4. **Use descriptive names** (AzureSqlDB_DW_Prod, not LinkedService1)
5. **Document connection details** in descriptions
6. **Test connections** before deploying
7. **Use Self-Hosted IR** for on-premises securely
8. **Monitor connection health** regularly

### Datasets:

1. **Parameterize** for reusability
2. **Define schema** when possible (better performance)
3. **Use appropriate file formats**:
   - Parquet for data lakes
   - CSV for interoperability
   - JSON for semi-structured
4. **Implement partitioning** for large datasets
5. **Use compression** to reduce storage and transfer costs
6. **Naming conventions**: Include source, format, purpose
7. **Document** parameters and usage

### Security:

1. **Never hardcode credentials**
2. **Use Key Vault** for all secrets
3. **Enable Managed Identity** on ADF
4. **Grant least privilege** access
5. **Use Private Endpoints** for sensitive data
6. **Enable diagnostic logging**
7. **Regular access reviews**

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: Explain the difference between Linked Service and Dataset.**

**A:** "A Linked Service is the connection to a data store - it contains authentication and connection information. A Dataset is a specific location within that data store - it points to a particular file, folder, or table. Think of it like a phone system: the Linked Service is your phone account, and the Dataset is a specific phone number you're calling.

For example, I might have one Azure Blob Storage Linked Service, but multiple datasets pointing to different containers or files within that storage account. This promotes reusability - I don't need separate linked services for each file."

**Q2: How do you implement environment-specific configurations (DEV/UAT/PROD)?**

**A:** "I use parameterized linked services with global parameters:

1. Create parameterized linked service with parameters for server name, database name, etc.
2. Define global parameters in ADF for each environment
3. Reference global parameters in linked service parameters
4. Use ARM template parameter files for deployment

Example:
```json
Linked Service parameter: @linkedService().serverName
Global parameter: SqlServerName
DEV: SqlServerName = "dev-sql-server"
PROD: SqlServerName = "prod-sql-server"
```

During deployment, I use different parameter files for each environment. This way, the same pipeline code works across all environments with zero changes."

**Q3: What's your approach to handling schema drift?**

**A:** "Schema drift is when source schema changes (new columns added, columns removed). My approach depends on the scenario:

**For Copy Activity:**
- Use 'Auto map' to automatically map matching columns
- Use 'translator' for explicit column mapping
- Enable 'Skip incompatible rows' to handle mismatches

**For Data Flows (Better):**
- Enable 'Allow schema drift' in source
- Use 'Auto-map drifted columns' in sink
- Use 'Rule-based mapping' for pattern-based mapping

**Best Practice:**
In my last project, we had weekly schema changes from the source system. We used Data Flow with schema drift enabled and rule-based mapping: 'All columns except X' or 'All columns matching pattern'. This handled new columns automatically without pipeline changes."

**Q4: How do you secure sensitive data in ADF?**

**A:** "Multi-layered approach:

1. **Credentials:** Store in Azure Key Vault, reference in linked services
2. **Authentication:** Use Managed Identity instead of keys/passwords
3. **Network:** Use Private Endpoints for data stores, Self-Hosted IR for on-premises
4. **Data:** Enable encryption at rest and in transit
5. **Access:** RBAC for ADF access, least privilege principle
6. **Monitoring:** Enable diagnostic logs, monitor for suspicious activity
7. **Compliance:** Use secureInput and secureOutput in activities to prevent sensitive data in logs

In my experience, I never store credentials in ADF directly. Everything goes through Key Vault. For database connections, I use Managed Identity. For APIs, I use Key Vault secrets with automatic rotation."

**Q5: Explain your strategy for metadata-driven pipelines.**

**A:** "Metadata-driven means configuration drives execution, not hardcoded pipelines. My pattern:

**Control Table:**
```sql
CREATE TABLE TableConfig (
    TableID INT,
    SourceSystem NVARCHAR(50),
    SourceSchema NVARCHAR(50),
    SourceTable NVARCHAR(100),
    DestinationPath NVARCHAR(500),
    IsActive BIT
)
```

**Pipeline:**
1. Lookup: Get active tables from control table
2. ForEach: Loop through tables
   - Copy Activity with parameterized datasets
   - Source: @item().SourceSchema.@item().SourceTable
   - Destination: @item().DestinationPath

**Benefits:**
- Add new tables without code changes
- Enable/disable tables via configuration
- Centralized control
- Easy to maintain

In my last project, we loaded 100+ tables this way. Adding a new table was just inserting a row in the control table - no deployment needed."

---

## Summary

**Key Takeaways:**

1. **Linked Services** = Connection (authentication, endpoint)
2. **Datasets** = Location (specific file, table, or folder)
3. **Parameterization** = Reusability and flexibility
4. **Security** = Key Vault + Managed Identity
5. **Best Format** = Parquet for data lakes, CSV for interoperability
6. **Metadata-Driven** = Configuration over code

**Next Steps:**
- Master Integration Runtime configurations
- Learn triggers and scheduling
- Practice building real-world pipelines
- Understand monitoring and troubleshooting

Remember: In interviews, always explain WHY you made a choice, not just WHAT you did!

