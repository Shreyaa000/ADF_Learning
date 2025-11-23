# ADF-06: Building Your First Pipeline

## Introduction

**Welcome to your first hands-on ADF experience!** This guide will walk you through building a complete, working pipeline from scratch. By the end of this chapter, you'll have:

✅ Created your first Azure Data Factory
✅ Built a simple copy pipeline
✅ Debugged and tested your pipeline
✅ Monitored pipeline execution
✅ Understood every component involved

**Think of this like:** Learning to drive. We'll start in an empty parking lot (simple copy), not on a highway (complex transformations). Once you master the basics, everything else builds on this foundation.

---

## Prerequisites

Before we begin, make sure you have:

1. **Azure Subscription** (free trial works fine)
2. **Azure Storage Account** (we'll use this as source and destination)
3. **Sample data** (we'll create this together)
4. **Contributor access** to a resource group

**Cost Note:** This tutorial will cost less than $1 if you clean up resources afterward.

---

## PART 1: Environment Setup

### Step 1: Create a Storage Account

**Why:** We need somewhere to store our sample data (source) and somewhere to copy it to (destination).

**How to Create:**

1. **Go to Azure Portal** (portal.azure.com)

2. **Click "Create a resource"**

3. **Search for "Storage Account"** and click Create

4. **Fill in details:**
   - **Subscription:** Your subscription
   - **Resource Group:** Create new → "rg-adf-tutorial"
   - **Storage account name:** "stadftutorial" + random numbers (must be globally unique)
   - **Region:** East US (or your preferred region)
   - **Performance:** Standard
   - **Redundancy:** LRS (Locally Redundant Storage - cheapest)

5. **Click "Review + Create"** → **Create**

6. **Wait 1-2 minutes** for deployment to complete

**What You Created:**
- A storage account that can hold blobs (files), tables, and queues
- We'll use blob storage to store CSV files

### Step 2: Create Containers

**Why:** Containers are like folders in blob storage. We need one for source data and one for destination.

**How to Create:**

1. **Go to your storage account** (stadftutorial...)

2. **Click "Containers"** in the left menu

3. **Click "+ Container"**
   - **Name:** "source-data"
   - **Public access level:** Private
   - Click **Create**

4. **Create another container:**
   - **Name:** "destination-data"
   - **Public access level:** Private
   - Click **Create**

**What You Created:**
```
Storage Account: stadftutorial12345
├── source-data (container)
└── destination-data (container)
```

### Step 3: Upload Sample Data

**Why:** We need some data to copy!

**Create Sample Data:**

1. **Open Notepad or any text editor**

2. **Copy this CSV content:**
```csv
EmployeeID,FirstName,LastName,Department,Salary,HireDate
1,John,Doe,IT,75000,2020-01-15
2,Jane,Smith,HR,65000,2019-03-22
3,Mike,Johnson,Sales,80000,2021-06-10
4,Sarah,Williams,IT,85000,2018-11-05
5,Tom,Brown,Finance,70000,2022-02-14
6,Emily,Davis,Sales,78000,2020-09-30
7,David,Miller,HR,62000,2021-04-18
8,Lisa,Wilson,IT,90000,2017-12-01
9,James,Moore,Finance,73000,2019-08-25
10,Maria,Taylor,Sales,82000,2020-05-12
```

3. **Save as:** `employees.csv` (make sure it's .csv, not .txt)

**Upload to Azure:**

1. **Go to your storage account** → **Containers** → **source-data**

2. **Click "Upload"**

3. **Select your** `employees.csv` file

4. **Click "Upload"**

5. **Verify:** You should see `employees.csv` in the source-data container

**What You Created:**
```
Storage Account: stadftutorial12345
├── source-data
│   └── employees.csv (10 employee records)
└── destination-data (empty)
```

---

## PART 2: Create Azure Data Factory

### Step 1: Create ADF Resource

**How to Create:**

1. **Go to Azure Portal** → **Create a resource**

2. **Search for "Data Factory"** → Click **Create**

3. **Fill in details:**
   - **Subscription:** Your subscription
   - **Resource Group:** rg-adf-tutorial (same as storage account)
   - **Region:** East US (same as storage account)
   - **Name:** "adf-tutorial-" + your initials + random numbers
     - Example: "adf-tutorial-jd-001"
     - Must be globally unique
   - **Version:** V2 (should be default)

4. **Git configuration:** Skip for now (click Next)

5. **Networking:** Default (Public endpoint)

6. **Click "Review + Create"** → **Create**

7. **Wait 2-3 minutes** for deployment

**What You Created:**
- An Azure Data Factory instance
- This is your orchestration engine for data pipelines

### Step 2: Launch ADF Studio

**How to Access:**

1. **Go to your Data Factory** resource in Azure Portal

2. **Click "Launch Studio"** (big button in the center)

3. **New tab opens:** This is the ADF Studio interface

**ADF Studio Interface Overview:**

```
┌─────────────────────────────────────────────────┐
│  🏠 Home  ✏️ Author  🔍 Monitor  🔧 Manage      │  ← Top Navigation
├─────────────────────────────────────────────────┤
│                                                 │
│  Left Panel:          Main Canvas              │
│  ├── Pipelines                                 │
│  ├── Datasets                                  │
│  ├── Data flows                                │
│  └── Power Query                               │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Four Main Sections:**

1. **Home** 🏠
   - Quick start guides
   - Recent items
   - Learning resources

2. **Author** ✏️
   - Create pipelines
   - Create datasets
   - Create data flows
   - Where you'll spend most time

3. **Monitor** 🔍
   - View pipeline runs
   - Debug runs
   - Trigger runs
   - Activity runs

4. **Manage** 🔧
   - Linked services
   - Integration runtimes
   - Triggers
   - Git configuration

---

## PART 3: Create Linked Services

**What are Linked Services?** Think of them as connection strings. They tell ADF how to connect to your data sources.

### Step 1: Create Source Linked Service

**Purpose:** Connect to the storage account where our source data lives.

**How to Create:**

1. **Click "Manage"** (toolbox icon) in the left sidebar

2. **Click "Linked services"** under Connections

3. **Click "+ New"**

4. **Search for "Azure Blob Storage"** → Click it → Click **Continue**

5. **Fill in details:**
   - **Name:** `LS_AzureBlob_Source`
   - **Description:** "Connection to source storage account"
   - **Connect via integration runtime:** AutoResolveIntegrationRuntime
   - **Authentication method:** Account key
   - **Account selection method:** From Azure subscription
   - **Azure subscription:** Your subscription
   - **Storage account name:** Select your storage account (stadftutorial...)

6. **Click "Test connection"** → Should show "Connection successful"

7. **Click "Create"**

**What You Created:**
- A connection to your storage account
- ADF can now read from this storage account
- Uses account key authentication (Azure manages the key)

### Step 2: Create Destination Linked Service

**Purpose:** Connect to the storage account where we'll copy data to.

**Option 1: Create New (if using different storage account)**

Follow the same steps as above, but name it `LS_AzureBlob_Destination`

**Option 2: Reuse Existing (if using same storage account)**

We'll use `LS_AzureBlob_Source` for both source and destination since they're in the same storage account. We'll specify different containers in the datasets.

**For this tutorial, we'll use Option 2** (reuse the same linked service).

**What We Have Now:**
```
Linked Services:
└── LS_AzureBlob_Source (connects to stadftutorial12345)
```

---

## PART 4: Create Datasets

**What are Datasets?** Think of them as pointers to specific data. They use a linked service (connection) and specify exactly where the data is (container, folder, file).

### Step 1: Create Source Dataset

**Purpose:** Point to the employees.csv file in the source-data container.

**How to Create:**

1. **Click "Author"** (pencil icon) in the left sidebar

2. **Click the "+" button** next to "Datasets" → **New dataset**

3. **Search for "Azure Blob Storage"** → Click it → Click **Continue**

4. **Select format:** "DelimitedText" (for CSV) → Click **Continue**

5. **Fill in details:**
   - **Name:** `DS_Source_EmployeesCSV`
   - **Linked service:** Select `LS_AzureBlob_Source`
   - **File path:**
     - **Container:** `source-data`
     - **Directory:** (leave empty)
     - **File:** `employees.csv`
   - **First row as header:** ✅ Check this box
   - **Import schema:** From connection/store

6. **Click "OK"**

7. **Review the schema:**
   - You should see columns: EmployeeID, FirstName, LastName, Department, Salary, HireDate
   - If you see them, schema import worked!

8. **Click "Publish all"** at the top → **Publish**

**What You Created:**
```
Dataset: DS_Source_EmployeesCSV
├── Uses: LS_AzureBlob_Source
├── Container: source-data
├── File: employees.csv
├── Format: CSV with headers
└── Schema: 6 columns (EmployeeID, FirstName, LastName, Department, Salary, HireDate)
```

### Step 2: Create Destination Dataset

**Purpose:** Point to where we want to copy the data.

**How to Create:**

1. **Click the "+" button** next to "Datasets" → **New dataset**

2. **Search for "Azure Blob Storage"** → Click it → Click **Continue**

3. **Select format:** "DelimitedText" → Click **Continue**

4. **Fill in details:**
   - **Name:** `DS_Destination_EmployeesCSV`
   - **Linked service:** Select `LS_AzureBlob_Source` (yes, same one!)
   - **File path:**
     - **Container:** `destination-data`
     - **Directory:** (leave empty)
     - **File:** `employees_copy.csv`
   - **First row as header:** ✅ Check this box
   - **Import schema:** From connection/store → Select `DS_Source_EmployeesCSV`

5. **Click "OK"**

6. **Click "Publish all"** → **Publish**

**What You Created:**
```
Dataset: DS_Destination_EmployeesCSV
├── Uses: LS_AzureBlob_Source (same connection, different container)
├── Container: destination-data
├── File: employees_copy.csv
├── Format: CSV with headers
└── Schema: Same as source (6 columns)
```

**Current State:**
```
Linked Services:
└── LS_AzureBlob_Source

Datasets:
├── DS_Source_EmployeesCSV (points to source-data/employees.csv)
└── DS_Destination_EmployeesCSV (points to destination-data/employees_copy.csv)
```

---

## PART 5: Create Your First Pipeline

**What is a Pipeline?** A pipeline is a logical grouping of activities that together perform a task. Our task: Copy data from source to destination.

### Step 1: Create Pipeline

**How to Create:**

1. **Click "Author"** (pencil icon)

2. **Click the "+" button** next to "Pipelines" → **Pipeline**

3. **Name your pipeline:**
   - Click on "Pipeline1" at the top
   - Rename to: `PL_Copy_Employees`
   - Press Enter

**What You See:**
```
┌─────────────────────────────────────────────────┐
│  PL_Copy_Employees                         [×]  │  ← Pipeline name
├─────────────────────────────────────────────────┤
│  Activities:                                    │
│  ├── Move & transform                          │  ← Expand this
│  ├── Azure Function                            │
│  ├── Batch Service                             │
│  └── ... more categories                       │
│                                                 │
│  [Empty Canvas]                                │  ← Drag activities here
│                                                 │
└─────────────────────────────────────────────────┘
```

### Step 2: Add Copy Activity

**How to Add:**

1. **Expand "Move & transform"** in the Activities panel

2. **Drag "Copy data"** activity onto the canvas

3. **Name the activity:**
   - Click on the activity (it should be selected)
   - At the bottom, you'll see tabs: General, Source, Sink, Mapping, Settings
   - In **General** tab:
     - **Name:** `Copy_Employees_Data`
     - **Description:** "Copy employee data from source to destination"

**What You See:**
```
Canvas:
┌─────────────────────────────────────────────────┐
│                                                 │
│       ┌─────────────────────────┐              │
│       │  Copy_Employees_Data    │              │
│       │  [Copy data activity]   │              │
│       └─────────────────────────┘              │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Step 3: Configure Source

**Purpose:** Tell the Copy activity where to read data from.

**How to Configure:**

1. **Click on the Copy activity** (if not already selected)

2. **Click the "Source" tab** at the bottom

3. **Configure:**
   - **Source dataset:** Select `DS_Source_EmployeesCSV`
   - **File path type:** File path in dataset (default)
   - **Compression type:** None (default)
   - **Additional columns:** (leave empty)

**What This Means:**
- Read from: source-data/employees.csv
- Format: CSV with headers
- No compression
- No additional columns

### Step 4: Configure Sink (Destination)

**Purpose:** Tell the Copy activity where to write data to.

**How to Configure:**

1. **Click the "Sink" tab** at the bottom

2. **Configure:**
   - **Sink dataset:** Select `DS_Destination_EmployeesCSV`
   - **Copy behavior:** None (default)
   - **Max concurrent connections:** (leave empty - uses default)
   - **Write method:** (leave empty - uses default)

**What This Means:**
- Write to: destination-data/employees_copy.csv
- Format: CSV with headers
- Overwrite if file exists

### Step 5: Configure Mapping (Optional)

**Purpose:** Map source columns to destination columns. Since our source and destination have the same schema, auto-mapping works fine.

**How to Configure:**

1. **Click the "Mapping" tab** at the bottom

2. **Click "Import schemas"**

3. **Review the mapping:**
   - You should see all 6 columns mapped automatically
   - Source → Destination
   - EmployeeID → EmployeeID
   - FirstName → FirstName
   - etc.

**What This Means:**
- Each source column maps to the corresponding destination column
- No transformations applied
- Data copied as-is

### Step 6: Save and Publish

**How to Save:**

1. **Click "Validate"** at the top (next to Debug)
   - This checks for errors
   - Should show "Your Pipeline has been validated. No errors were found."

2. **Click "Publish all"** at the top
   - This saves your pipeline to the ADF service
   - Click **Publish** in the dialog

3. **Wait for "Publishing succeeded"** message

**What You Created:**
```
Pipeline: PL_Copy_Employees
└── Activity: Copy_Employees_Data
    ├── Source: DS_Source_EmployeesCSV (source-data/employees.csv)
    ├── Sink: DS_Destination_EmployeesCSV (destination-data/employees_copy.csv)
    └── Mapping: Auto-mapped (6 columns)
```

---

## PART 6: Debug Your Pipeline

**What is Debug?** Debug mode lets you test your pipeline without creating a trigger. It's like a test drive before the real thing.

### Step 1: Start Debug

**How to Debug:**

1. **Make sure your pipeline is open** (PL_Copy_Employees)

2. **Click "Debug"** at the top of the canvas

3. **If prompted for parameters:** Click **OK** (we don't have parameters yet)

4. **Watch the "Output" panel** at the bottom

**What You See:**

```
Output:
┌─────────────────────────────────────────────────────────────────┐
│  Pipeline run ID: 12345678-1234-1234-1234-123456789012         │
│                                                                 │
│  Activity Name          Type        Status      Duration       │
│  ─────────────────────────────────────────────────────────────│
│  Copy_Employees_Data    Copy        In Progress  00:00:05      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Status Progression:**
1. **Queued** → Waiting to start
2. **In Progress** → Currently running
3. **Succeeded** ✅ → Completed successfully
4. **Failed** ❌ → Something went wrong

### Step 2: Monitor Execution

**While Running:**

1. **Watch the activity icon** on the canvas
   - Yellow spinner = In Progress
   - Green checkmark = Succeeded
   - Red X = Failed

2. **Check the Output panel:**
   - Shows real-time status
   - Shows duration
   - Shows data read/written

**When Complete:**

```
Output:
┌─────────────────────────────────────────────────────────────────┐
│  Activity Name          Type    Status      Duration  Details   │
│  ─────────────────────────────────────────────────────────────│
│  Copy_Employees_Data    Copy    Succeeded   00:00:12  [👁️]     │
│                                                                 │
│  ✅ Pipeline run finished successfully                          │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: View Details

**How to View:**

1. **Click the "eyeglasses" icon** 👁️ next to the activity in the Output panel

2. **Details window opens:**

```
Copy Activity Details:
┌─────────────────────────────────────────────────┐
│  Status: Succeeded                              │
│  Duration: 12 seconds                           │
│                                                 │
│  Data Read:                                     │
│    Rows: 10                                     │
│    Size: 512 bytes                              │
│                                                 │
│  Data Written:                                  │
│    Rows: 10                                     │
│    Size: 512 bytes                              │
│                                                 │
│  Throughput: 42 bytes/s                         │
│  Data Integration Units: 4                      │
│  Effective Integration Runtime: AutoResolveIR   │
│  Region: East US                                │
│                                                 │
│  Source:                                        │
│    Type: AzureBlobStorage                       │
│    Path: source-data/employees.csv              │
│                                                 │
│  Sink:                                          │
│    Type: AzureBlobStorage                       │
│    Path: destination-data/employees_copy.csv    │
└─────────────────────────────────────────────────┘
```

**Key Metrics:**
- **Rows:** 10 rows read, 10 rows written ✅
- **Duration:** 12 seconds (includes startup time)
- **Throughput:** How fast data was copied
- **DIU:** Data Integration Units used (affects cost)

### Step 4: Verify the Copy

**Check Destination:**

1. **Go to Azure Portal**

2. **Navigate to your storage account** → **Containers** → **destination-data**

3. **You should see:** `employees_copy.csv`

4. **Click on the file** → **Download**

5. **Open in Excel or Notepad:**
   - Should have 11 lines (1 header + 10 data rows)
   - Should match the source file exactly

**Success! You've copied data!** 🎉

---

## PART 7: Understanding What Happened

### Behind the Scenes

When you clicked Debug, here's what happened:

**1. Pipeline Execution Started**
```
ADF Service: "Starting pipeline PL_Copy_Employees"
```

**2. Integration Runtime Allocated**
```
ADF Service: "Allocating AutoResolveIntegrationRuntime in East US"
ADF Service: "Allocating 4 Data Integration Units"
```

**3. Source Connection Established**
```
Integration Runtime: "Connecting to stadftutorial12345.blob.core.windows.net"
Integration Runtime: "Authenticating with account key"
Integration Runtime: "Connection successful"
```

**4. Source Data Read**
```
Integration Runtime: "Reading source-data/employees.csv"
Integration Runtime: "Detected CSV format with headers"
Integration Runtime: "Reading 10 rows, 512 bytes"
Integration Runtime: "Read complete"
```

**5. Destination Connection Established**
```
Integration Runtime: "Connecting to stadftutorial12345.blob.core.windows.net"
Integration Runtime: "Authenticating with account key"
Integration Runtime: "Connection successful"
```

**6. Data Written**
```
Integration Runtime: "Writing to destination-data/employees_copy.csv"
Integration Runtime: "Writing 10 rows, 512 bytes"
Integration Runtime: "Write complete"
```

**7. Cleanup and Reporting**
```
Integration Runtime: "Releasing resources"
ADF Service: "Pipeline completed successfully"
ADF Service: "Logging metrics: 10 rows, 12 seconds, 4 DIU"
```

### Cost Breakdown

**What You Were Charged For:**

1. **Pipeline Activity Run:** 1 activity run
   - Cost: ~$0.001 (less than a penny)

2. **Data Integration Units (DIU):** 4 DIU for 12 seconds
   - Cost: ~$0.0001 (fraction of a penny)

3. **Data Movement:** 512 bytes
   - Cost: Free (first 10 GB free per month)

**Total Cost:** Less than $0.01 (one cent)

---

## PART 8: Add Monitoring and Logging

### Step 1: View Pipeline Run History

**How to View:**

1. **Click "Monitor"** (bar chart icon) in the left sidebar

2. **Click "Pipeline runs"**

3. **You should see your debug run:**

```
Pipeline Runs:
┌────────────────────────────────────────────────────────────────────────┐
│  Pipeline Name         Run Start           Status      Triggered By   │
│  ────────────────────────────────────────────────────────────────────│
│  PL_Copy_Employees     2024-01-15 14:30    Succeeded   Debug          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Click on the pipeline name** to see details.

### Step 2: View Activity Runs

**What You See:**

```
Activity Runs for Pipeline: PL_Copy_Employees
┌────────────────────────────────────────────────────────────────────────┐
│  Activity Name          Type    Status      Start Time    Duration    │
│  ────────────────────────────────────────────────────────────────────│
│  Copy_Employees_Data    Copy    Succeeded   14:30:05      00:00:12    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Click the eyeglasses icon** to see detailed metrics (same as before).

### Step 3: View Input and Output

**Click the "Input" icon** (document with arrow in):
```json
{
  "source": {
    "type": "DelimitedTextSource",
    "storeSettings": {
      "type": "AzureBlobStorageReadSettings",
      "recursive": false,
      "enablePartitionDiscovery": false
    },
    "formatSettings": {
      "type": "DelimitedTextReadSettings"
    }
  },
  "sink": {
    "type": "DelimitedTextSink",
    "storeSettings": {
      "type": "AzureBlobStorageWriteSettings"
    },
    "formatSettings": {
      "type": "DelimitedTextWriteSettings",
      "quoteAllText": true,
      "fileExtension": ".csv"
    }
  },
  "enableStaging": false,
  "dataIntegrationUnits": 4
}
```

**Click the "Output" icon** (document with arrow out):
```json
{
  "dataRead": 512,
  "dataWritten": 512,
  "filesRead": 1,
  "filesWritten": 1,
  "rowsRead": 10,
  "rowsCopied": 10,
  "copyDuration": 12,
  "throughput": 42.67,
  "errors": [],
  "effectiveIntegrationRuntime": "AutoResolveIntegrationRuntime (East US)",
  "usedDataIntegrationUnits": 4,
  "billedDuration": 12
}
```

**What This Tells You:**
- Exactly what settings were used
- Exactly what happened
- Any errors (none in this case!)
- Performance metrics

---

## PART 9: Add a Trigger

**So far:** We've manually run the pipeline with Debug. Let's automate it!

### Step 1: Create a Schedule Trigger

**How to Create:**

1. **Go back to Author** → Open **PL_Copy_Employees**

2. **Click "Add trigger"** at the top → **New/Edit**

3. **Click "Choose trigger..."** dropdown → **+ New**

4. **Fill in details:**
   - **Name:** `TR_Daily_Copy_Employees`
   - **Description:** "Run employee copy daily at 2 AM"
   - **Type:** Schedule
   - **Start date:** Today's date, 02:00:00 (2 AM)
   - **Time zone:** Your time zone
   - **Recurrence:** Every 1 Day
   - **End:** No end

5. **Click "OK"**

6. **Click "OK"** again (trigger parameters dialog)

7. **Click "Publish all"** → **Publish**

**What You Created:**
- A trigger that runs your pipeline every day at 2 AM
- Fully automated - no manual intervention needed

### Step 2: Activate the Trigger

**Important:** Creating a trigger doesn't start it!

**How to Activate:**

1. **Go to Manage** → **Triggers**

2. **Find your trigger:** `TR_Daily_Copy_Employees`

3. **Toggle the switch** to **Started**

4. **Confirm** in the dialog

**What This Means:**
- Starting tomorrow at 2 AM, your pipeline will run automatically
- It will run every day at 2 AM
- You can monitor runs in the Monitor section

### Step 3: Test the Trigger (Optional)

**Don't want to wait until 2 AM?** Trigger it manually!

**How to Trigger Now:**

1. **Go to Author** → Open **PL_Copy_Employees**

2. **Click "Add trigger"** → **Trigger now**

3. **Click "OK"**

4. **Go to Monitor** → **Pipeline runs**

5. **You should see a new run:**
   - **Triggered by:** TR_Daily_Copy_Employees (Manual)

**What This Means:**
- You manually triggered the scheduled trigger
- This doesn't affect the 2 AM schedule
- Good for testing!

---

## PART 10: Troubleshooting Common Issues

### Issue 1: "Test connection failed"

**Symptoms:**
- When creating linked service, test connection fails

**Possible Causes:**

1. **Wrong storage account selected**
   - **Solution:** Double-check you selected the correct storage account

2. **Firewall blocking connection**
   - **Solution:** Go to storage account → Networking → Add your IP address

3. **Storage account key rotated**
   - **Solution:** Linked service will auto-update if using "From Azure subscription"

### Issue 2: "Dataset schema not found"

**Symptoms:**
- When creating dataset, schema doesn't import

**Possible Causes:**

1. **File doesn't exist**
   - **Solution:** Verify file exists in the container

2. **Wrong file path**
   - **Solution:** Check container name, folder path, file name

3. **File is empty**
   - **Solution:** Verify file has content

**How to Fix:**

1. **Go to the dataset** (DS_Source_EmployeesCSV)

2. **Click "Connection" tab**

3. **Verify:**
   - Container: source-data
   - File: employees.csv

4. **Click "Preview data"**
   - Should show your 10 employee records
   - If it doesn't, file path is wrong

### Issue 3: "Copy activity failed"

**Symptoms:**
- Debug run fails with red X

**Possible Causes:**

1. **Source file not found**
   - **Error:** "The specified blob does not exist"
   - **Solution:** Verify file exists in source-data container

2. **Permission denied**
   - **Error:** "Authorization permission mismatch"
   - **Solution:** Check linked service authentication

3. **Invalid format**
   - **Error:** "Failed to parse CSV"
   - **Solution:** Verify CSV is properly formatted

**How to Debug:**

1. **Click the eyeglasses icon** on the failed activity

2. **Read the error message carefully**

3. **Common errors:**

**Error:** "The specified blob does not exist"
```
Solution:
1. Go to Azure Portal → Storage Account → Containers → source-data
2. Verify employees.csv exists
3. If not, re-upload the file
```

**Error:** "Authorization permission mismatch"
```
Solution:
1. Go to Manage → Linked Services → LS_AzureBlob_Source
2. Click "Test connection"
3. If fails, recreate the linked service
```

**Error:** "Failed to parse CSV"
```
Solution:
1. Download the source file
2. Open in Notepad
3. Verify:
   - First line is headers
   - No extra commas
   - No special characters
   - Proper line endings
```

### Issue 4: "Pipeline runs but no data copied"

**Symptoms:**
- Pipeline shows "Succeeded"
- But destination file is empty or doesn't exist

**Possible Causes:**

1. **Source file is empty**
   - **Solution:** Check source file has data

2. **Mapping is incorrect**
   - **Solution:** Check Mapping tab in Copy activity

3. **Filter is excluding all rows**
   - **Solution:** Check if you added any filters in Source tab

**How to Debug:**

1. **Check the activity output:**
   - Click eyeglasses icon
   - Look at "rowsRead" and "rowsCopied"
   - If rowsRead = 0, source file is empty or not found
   - If rowsRead > 0 but rowsCopied = 0, check mapping/filters

2. **Preview source data:**
   - Go to DS_Source_EmployeesCSV
   - Click "Preview data"
   - Should show 10 rows
   - If not, source file issue

3. **Check destination:**
   - Go to Azure Portal → Storage Account → destination-data
   - Verify employees_copy.csv exists
   - Download and verify content

---

## PART 11: Enhance Your Pipeline

### Enhancement 1: Add Parameters

**Why:** Make your pipeline reusable for different files.

**How to Add:**

1. **Open PL_Copy_Employees**

2. **Click on empty canvas** (not on the activity)

3. **Click "Parameters" tab** at the bottom

4. **Click "+ New"**

5. **Add parameter:**
   - **Name:** `sourceFileName`
   - **Type:** String
   - **Default value:** `employees.csv`

6. **Add another parameter:**
   - **Name:** `destinationFileName`
   - **Type:** String
   - **Default value:** `employees_copy.csv`

**Update Source Dataset:**

1. **Go to DS_Source_EmployeesCSV**

2. **Click "Parameters" tab**

3. **Add parameter:**
   - **Name:** `fileName`
   - **Type:** String

4. **Click "Connection" tab**

5. **In "File" field, click "Add dynamic content"**

6. **Enter:** `@dataset().fileName`

7. **Click "OK"**

8. **Publish all**

**Update Sink Dataset:**

1. **Go to DS_Destination_EmployeesCSV**

2. **Repeat the same steps as source**

**Update Pipeline:**

1. **Open PL_Copy_Employees**

2. **Click on Copy_Employees_Data activity**

3. **In Source tab:**
   - **Source dataset:** DS_Source_EmployeesCSV
   - **Dataset properties:**
     - **fileName:** `@pipeline().parameters.sourceFileName`

4. **In Sink tab:**
   - **Sink dataset:** DS_Destination_EmployeesCSV
   - **Dataset properties:**
     - **fileName:** `@pipeline().parameters.destinationFileName`

5. **Publish all**

**Test with Parameters:**

1. **Click "Debug"**

2. **In the parameters dialog:**
   - **sourceFileName:** `employees.csv`
   - **destinationFileName:** `employees_backup.csv`

3. **Click "OK"**

4. **Check destination-data container:**
   - Should see `employees_backup.csv`

**What You Achieved:**
- Pipeline now accepts file names as parameters
- Can copy any file, not just employees.csv
- More reusable and flexible

### Enhancement 2: Add Error Handling

**Why:** Know when something goes wrong.

**How to Add:**

1. **Open PL_Copy_Employees**

2. **Drag a "Web" activity** onto the canvas

3. **Name it:** `Send_Failure_Notification`

4. **Click on Copy_Employees_Data activity**

5. **Click the red "X" icon** on the activity (failure path)

6. **Drag to Send_Failure_Notification** activity

**Configure Web Activity:**

1. **Click on Send_Failure_Notification**

2. **In Settings tab:**
   - **URL:** (your webhook URL - we'll use a test URL)
   - **Method:** POST
   - **Body:**
```json
{
  "pipelineName": "@{pipeline().Pipeline}",
  "activityName": "Copy_Employees_Data",
  "errorMessage": "@{activity('Copy_Employees_Data').error.message}",
  "runId": "@{pipeline().RunId}",
  "time": "@{utcnow()}"
}
```

**What This Does:**
- If Copy activity fails, Web activity runs
- Sends error details to a webhook (could be Logic App, Teams, email)
- You get notified immediately

### Enhancement 3: Add Success Logging

**How to Add:**

1. **Drag another "Web" activity** onto the canvas

2. **Name it:** `Log_Success`

3. **Click on Copy_Employees_Data activity**

4. **Click the green checkmark icon** (success path)

5. **Drag to Log_Success** activity

**Configure:**

1. **Click on Log_Success**

2. **In Settings tab:**
   - **URL:** (your logging endpoint)
   - **Method:** POST
   - **Body:**
```json
{
  "pipelineName": "@{pipeline().Pipeline}",
  "activityName": "Copy_Employees_Data",
  "rowsCopied": "@{activity('Copy_Employees_Data').output.rowsCopied}",
  "duration": "@{activity('Copy_Employees_Data').output.copyDuration}",
  "runId": "@{pipeline().RunId}",
  "time": "@{utcnow()}"
}
```

**What This Does:**
- If Copy activity succeeds, Log_Success runs
- Sends success metrics to a logging system
- Track performance over time

**Your Enhanced Pipeline:**

```
┌─────────────────────────────────────────────────┐
│                                                 │
│       ┌─────────────────────────┐              │
│       │  Copy_Employees_Data    │              │
│       └──────────┬──────────────┘              │
│                  │                              │
│         Success  │  Failure                     │
│                  │                              │
│       ┌──────────▼──────────┐                  │
│       │  Log_Success        │                  │
│       └─────────────────────┘                  │
│                                                 │
│       ┌──────────────────────────┐             │
│       │  Send_Failure_Notification│            │
│       └──────────────────────────┘             │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

## PART 12: Best Practices You Just Learned

### 1. Naming Conventions

**What You Did:**
- Linked Service: `LS_AzureBlob_Source`
- Dataset: `DS_Source_EmployeesCSV`
- Pipeline: `PL_Copy_Employees`
- Activity: `Copy_Employees_Data`
- Trigger: `TR_Daily_Copy_Employees`

**Pattern:**
- **Linked Service:** `LS_{Type}_{Purpose}`
- **Dataset:** `DS_{SourceOrSink}_{Description}`
- **Pipeline:** `PL_{Action}_{Entity}`
- **Activity:** `{Action}_{Entity}_{Details}`
- **Trigger:** `TR_{Frequency}_{Purpose}`

**Why This Matters:**
- Easy to understand at a glance
- Consistent across your ADF instance
- Easy to find what you need
- Professional and maintainable

### 2. Always Test Connection

**What You Did:**
- Tested linked service connection before using it

**Why This Matters:**
- Catches authentication issues early
- Verifies network connectivity
- Saves debugging time later

### 3. Import Schema

**What You Did:**
- Imported schema when creating datasets

**Why This Matters:**
- Validates file format
- Enables column mapping
- Catches format issues early

### 4. Use Debug Before Trigger

**What You Did:**
- Debugged pipeline before creating trigger

**Why This Matters:**
- Test without scheduling
- Iterate quickly
- Don't waste time on scheduled failures

### 5. Monitor and Verify

**What You Did:**
- Checked output metrics
- Verified destination file
- Reviewed activity details

**Why This Matters:**
- Confirm data copied correctly
- Catch silent failures
- Understand performance

### 6. Parameterize Early

**What You Did:**
- Added parameters to make pipeline reusable

**Why This Matters:**
- One pipeline for multiple scenarios
- Easier maintenance
- More flexible

### 7. Add Error Handling

**What You Did:**
- Added failure path to Web activity

**Why This Matters:**
- Know when things go wrong
- Faster incident response
- Better reliability

---

## PART 13: What You've Accomplished

### Skills Acquired

✅ **Created Azure Data Factory** from scratch
✅ **Created Linked Services** to connect to data sources
✅ **Created Datasets** to define data structures
✅ **Built a Pipeline** with Copy activity
✅ **Debugged and tested** your pipeline
✅ **Monitored execution** and analyzed metrics
✅ **Created a Trigger** for automation
✅ **Added parameters** for flexibility
✅ **Implemented error handling** for reliability

### Real-World Equivalents

**What you just did is equivalent to:**

1. **Data Engineer Task:** "Copy daily sales files from vendor SFTP to data lake"
   - You: Created automated copy pipeline ✅

2. **Interview Question:** "How do you create a simple copy pipeline in ADF?"
   - You: Can explain and demonstrate step-by-step ✅

3. **Production Scenario:** "We need to copy employee data daily at 2 AM"
   - You: Implemented exactly this ✅

### Talking Points for Interviews

**Interviewer:** "Tell me about a pipeline you've built."

**You:** "I built a data ingestion pipeline that copies employee data from Azure Blob Storage to a destination container daily. The pipeline uses:

- **Linked Services** for connection management with account key authentication
- **Parameterized datasets** so the same pipeline can copy different files
- **Copy activity** configured with schema mapping
- **Schedule trigger** that runs daily at 2 AM
- **Error handling** with Web activity to send notifications on failure
- **Success logging** to track metrics like rows copied and duration

I used debug mode for testing before scheduling, and monitored execution through the ADF Monitor interface. The pipeline processes about 10,000 employee records daily with 99.9% success rate."

**Interviewer:** "Impressive! Tell me more about the error handling..."

**You:** "Sure! I implemented a failure path using the Copy activity's failure output. When the copy fails, it triggers a Web activity that calls a webhook (could be Logic App or Teams) with error details including:
- Pipeline name and run ID for tracking
- Error message from the failed activity
- Timestamp for incident logging

This reduced our mean time to resolution from hours to minutes because we know immediately when something fails and have all the context we need to troubleshoot."

---

## PART 14: Next Steps

### Immediate Practice

**Exercise 1: Copy Different File**

1. Create a new CSV file: `departments.csv`
```csv
DepartmentID,DepartmentName,Location,Budget
1,IT,New York,500000
2,HR,Chicago,300000
3,Sales,Los Angeles,800000
4,Finance,Boston,400000
```

2. Upload to source-data container

3. Use your parameterized pipeline to copy it:
   - sourceFileName: `departments.csv`
   - destinationFileName: `departments_copy.csv`

4. Verify the copy worked

**Exercise 2: Schedule at Different Time**

1. Create a new trigger
2. Schedule for every hour
3. Test with "Trigger now"
4. Stop the trigger (so you don't get charged!)

**Exercise 3: Add Validation**

1. Add a "Get Metadata" activity before Copy
2. Get file size
3. Add "If Condition" activity
4. Only copy if file size > 0

### Advanced Challenges

**Challenge 1: Copy Multiple Files**

1. Create 3 different CSV files
2. Use ForEach activity to loop through file names
3. Copy all files in one pipeline run

**Challenge 2: Incremental Copy**

1. Add a "Lookup" activity to read last processed date
2. Filter Copy activity to only copy new records
3. Update last processed date after successful copy

**Challenge 3: Transform During Copy**

1. Add a "Data Flow" activity instead of Copy
2. Transform data (e.g., uppercase names, calculate age from hire date)
3. Write to destination

### Learning Path

**You've completed:** Foundation ✅

**Next:**
- **ADF-07:** Real-World Scenarios (30+ practical examples)
- **ADF-08:** Data Flows (transformations)
- **ADF-09:** Advanced Patterns (metadata-driven, frameworks)
- **ADF-10:** Monitoring and Troubleshooting (deep dive)

---

## PART 15: Cleanup (Optional)

**To avoid charges, delete resources:**

### Option 1: Delete Resource Group (Deletes Everything)

1. **Go to Azure Portal**
2. **Search for "Resource groups"**
3. **Find "rg-adf-tutorial"**
4. **Click "Delete resource group"**
5. **Type the resource group name to confirm**
6. **Click "Delete"**

**This deletes:**
- Data Factory
- Storage Account
- All data

### Option 2: Keep for Practice (Minimal Cost)

**To minimize costs:**

1. **Stop all triggers:**
   - Go to Manage → Triggers
   - Stop all triggers

2. **Delete large files:**
   - Go to Storage Account
   - Delete any large files (our tutorial files are tiny, so negligible cost)

**Monthly cost if you keep everything:**
- Storage: ~$0.50 (for a few KB of data)
- Data Factory: $0 (no pipeline runs = no cost)
- **Total: ~$0.50/month**

---

## Summary

### What You Built

```
Azure Data Factory: adf-tutorial-jd-001
│
├── Linked Services
│   └── LS_AzureBlob_Source (connects to storage account)
│
├── Datasets
│   ├── DS_Source_EmployeesCSV (source-data/employees.csv)
│   └── DS_Destination_EmployeesCSV (destination-data/employees_copy.csv)
│
├── Pipelines
│   └── PL_Copy_Employees
│       ├── Copy_Employees_Data (copy activity)
│       ├── Log_Success (success path)
│       └── Send_Failure_Notification (failure path)
│
└── Triggers
    └── TR_Daily_Copy_Employees (runs daily at 2 AM)
```

### Key Concepts Mastered

1. **Linked Services** = Connections
2. **Datasets** = Data pointers
3. **Pipelines** = Workflows
4. **Activities** = Tasks
5. **Triggers** = Scheduling
6. **Debug** = Testing
7. **Monitor** = Observability
8. **Parameters** = Flexibility
9. **Error Handling** = Reliability

### Interview Readiness

You can now confidently answer:
- ✅ "How do you create a pipeline in ADF?"
- ✅ "What's the difference between linked service and dataset?"
- ✅ "How do you debug a pipeline?"
- ✅ "How do you schedule a pipeline?"
- ✅ "How do you monitor pipeline execution?"
- ✅ "How do you handle errors in ADF?"
- ✅ "How do you make pipelines reusable?"

### Real-World Readiness

You can now:
- ✅ Create data ingestion pipelines
- ✅ Connect to Azure storage
- ✅ Copy data between locations
- ✅ Schedule automated runs
- ✅ Monitor and troubleshoot
- ✅ Implement error handling
- ✅ Use parameters for flexibility

---

**Congratulations!** You've built your first Azure Data Factory pipeline from scratch. This is the foundation for everything else you'll learn. Every complex pipeline you'll ever build uses these same concepts - they just combine them in more sophisticated ways.

**Next:** Move on to ADF-07 to see 30+ real-world scenarios that build on what you just learned!

