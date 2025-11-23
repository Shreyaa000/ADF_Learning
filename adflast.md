# ADF-07: 30+ Real-World Scenarios

## Scenarios 32-34: DevOps and Environment Management

---

## Scenario 32: Git Integration (CI/CD)

### Business Requirement
Your organization requires version control for all ADF pipelines, proper code review processes, and automated deployment across environments. The development team needs to collaborate on ADF development using Git, track changes, and implement CI/CD pipelines for automated deployments.

### Implementation Steps

#### Step 1: Configure Git Repository in ADF

1. **Navigate to ADF Studio**
   - Go to Azure Data Factory Studio
   - Click on "Manage" hub (toolbox icon)
   - Select "Git configuration"

2. **Choose Git Provider**
   - Select either Azure DevOps Git or GitHub
   - For Azure DevOps:
     - Organization name: `your-org`
     - Project name: `adf-project`
     - Repository name: `adf-repo`
     - Collaboration branch: `main`
     - Publish branch: `adf_publish`
     - Root folder: `/`

3. **Configure Branch Settings**
   ```
   Collaboration Branch: main (where developers work)
   Publish Branch: adf_publish (ARM templates generated here)
   Import existing resources: Yes
   Branch to import from: main
   ```

#### Step 2: Development Workflow

**Feature Branch Development:**
```
1. Create feature branch from main
   - Branch name: feature/add-customer-pipeline
   
2. Develop in ADF Studio
   - Switch to feature branch in ADF
   - Create/modify pipelines
   - Save all changes (commits to feature branch)
   
3. Create Pull Request
   - Compare feature/add-customer-pipeline → main
   - Add reviewers
   - Include description of changes
   
4. Code Review Process
   - Reviewers check:
     * Pipeline logic correctness
     * Naming conventions
     * Parameter usage
     * Error handling
     * Performance considerations
   
5. Merge to Main
   - Complete PR after approval
   - Delete feature branch
```

#### Step 3: Set Up Azure DevOps CI/CD Pipeline

**Create Build Pipeline (azure-pipelines-build.yml):**
```yaml
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - pipeline/*
    - dataset/*
    - linkedService/*

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: NodeTool@0
  inputs:
    versionSpec: '14.x'
  displayName: 'Install Node.js'

- task: Npm@1
  inputs:
    command: 'install'
    workingDir: '$(Build.SourcesDirectory)'
  displayName: 'Install npm packages'

# Validate ADF artifacts
- task: PowerShell@2
  inputs:
    targetType: 'inline'
    script: |
      # Validate JSON files
      Get-ChildItem -Path . -Filter *.json -Recurse | ForEach-Object {
        Write-Host "Validating $($_.FullName)"
        $json = Get-Content $_.FullName -Raw | ConvertFrom-Json
        Write-Host "✓ Valid JSON: $($_.Name)"
      }
  displayName: 'Validate ADF JSON Files'

# Publish build artifacts
- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.SourcesDirectory)'
    ArtifactName: 'ADF-Artifacts'
  displayName: 'Publish ADF Artifacts'
```

**Create Release Pipeline (azure-pipelines-release.yml):**
```yaml
# Release to UAT
stages:
- stage: DeployToUAT
  displayName: 'Deploy to UAT'
  jobs:
  - deployment: DeployADF
    displayName: 'Deploy ADF to UAT'
    environment: 'UAT'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzurePowerShell@5
            inputs:
              azureSubscription: 'Azure-Subscription-UAT'
              ScriptType: 'FilePath'
              ScriptPath: '$(Pipeline.Workspace)/ADF-Artifacts/deployment/deploy-adf.ps1'
              ScriptArguments: >
                -ResourceGroupName "rg-adf-uat"
                -DataFactoryName "adf-uat-001"
                -ArmTemplateFile "$(Pipeline.Workspace)/ADF-Artifacts/ARMTemplateForFactory.json"
                -ArmParametersFile "$(Pipeline.Workspace)/ADF-Artifacts/ARMTemplateParametersForFactory-UAT.json"
              azurePowerShellVersion: 'LatestVersion'
            displayName: 'Deploy ADF ARM Template'

          # Stop triggers before deployment
          - task: AzurePowerShell@5
            inputs:
              azureSubscription: 'Azure-Subscription-UAT'
              ScriptType: 'InlineScript'
              Inline: |
                $triggers = Get-AzDataFactoryV2Trigger -ResourceGroupName "rg-adf-uat" -DataFactoryName "adf-uat-001"
                foreach ($trigger in $triggers) {
                  if ($trigger.RuntimeState -eq "Started") {
                    Stop-AzDataFactoryV2Trigger -ResourceGroupName "rg-adf-uat" -DataFactoryName "adf-uat-001" -Name $trigger.Name -Force
                  }
                }
              azurePowerShellVersion: 'LatestVersion'
            displayName: 'Stop ADF Triggers'

          # Start triggers after deployment
          - task: AzurePowerShell@5
            inputs:
              azureSubscription: 'Azure-Subscription-UAT'
              ScriptType: 'InlineScript'
              Inline: |
                $triggers = Get-AzDataFactoryV2Trigger -ResourceGroupName "rg-adf-uat" -DataFactoryName "adf-uat-001"
                foreach ($trigger in $triggers) {
                  Start-AzDataFactoryV2Trigger -ResourceGroupName "rg-adf-uat" -DataFactoryName "adf-uat-001" -Name $trigger.Name -Force
                }
              azurePowerShellVersion: 'LatestVersion'
            displayName: 'Start ADF Triggers'

# Release to PROD (with approval)
- stage: DeployToPROD
  displayName: 'Deploy to PROD'
  dependsOn: DeployToUAT
  jobs:
  - deployment: DeployADF
    displayName: 'Deploy ADF to PROD'
    environment: 'PROD'
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzurePowerShell@5
            inputs:
              azureSubscription: 'Azure-Subscription-PROD'
              ScriptType: 'FilePath'
              ScriptPath: '$(Pipeline.Workspace)/ADF-Artifacts/deployment/deploy-adf.ps1'
              ScriptArguments: >
                -ResourceGroupName "rg-adf-prod"
                -DataFactoryName "adf-prod-001"
                -ArmTemplateFile "$(Pipeline.Workspace)/ADF-Artifacts/ARMTemplateForFactory.json"
                -ArmParametersFile "$(Pipeline.Workspace)/ADF-Artifacts/ARMTemplateParametersForFactory-PROD.json"
              azurePowerShellVersion: 'LatestVersion'
            displayName: 'Deploy ADF ARM Template'
```

#### Step 4: PowerShell Deployment Script

**deploy-adf.ps1:**
```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,
    
    [Parameter(Mandatory=$true)]
    [string]$DataFactoryName,
    
    [Parameter(Mandatory=$true)]
    [string]$ArmTemplateFile,
    
    [Parameter(Mandatory=$true)]
    [string]$ArmParametersFile
)

Write-Host "Starting ADF Deployment..."
Write-Host "Resource Group: $ResourceGroupName"
Write-Host "Data Factory: $DataFactoryName"

# Stop all triggers
Write-Host "Stopping triggers..."
$triggers = Get-AzDataFactoryV2Trigger -ResourceGroupName $ResourceGroupName -DataFactoryName $DataFactoryName
foreach ($trigger in $triggers) {
    if ($trigger.RuntimeState -eq "Started") {
        Write-Host "Stopping trigger: $($trigger.Name)"
        Stop-AzDataFactoryV2Trigger -ResourceGroupName $ResourceGroupName `
                                     -DataFactoryName $DataFactoryName `
                                     -Name $trigger.Name -Force
    }
}

# Deploy ARM template
Write-Host "Deploying ARM template..."
New-AzResourceGroupDeployment -ResourceGroupName $ResourceGroupName `
                               -TemplateFile $ArmTemplateFile `
                               -TemplateParameterFile $ArmParametersFile `
                               -Mode Incremental `
                               -Verbose

# Start triggers
Write-Host "Starting triggers..."
$triggers = Get-AzDataFactoryV2Trigger -ResourceGroupName $ResourceGroupName -DataFactoryName $DataFactoryName
foreach ($trigger in $triggers) {
    Write-Host "Starting trigger: $($trigger.Name)"
    Start-AzDataFactoryV2Trigger -ResourceGroupName $ResourceGroupName `
                                  -DataFactoryName $DataFactoryName `
                                  -Name $trigger.Name -Force
}

Write-Host "Deployment completed successfully!"
```

### Expected Behavior

1. **Version Control:**
   - All ADF changes tracked in Git
   - Complete history of who changed what and when
   - Ability to rollback to previous versions

2. **Code Review:**
   - No direct commits to main branch
   - All changes reviewed before merge
   - Quality gates enforced

3. **Automated Deployment:**
   - Build pipeline validates artifacts
   - Release pipeline deploys to UAT automatically
   - PROD deployment requires manual approval
   - Triggers stopped during deployment, restarted after

4. **Audit Trail:**
   - Git commit history
   - Azure DevOps deployment logs
   - Approval records

### Common Issues and Resolutions

**Issue 1: Merge Conflicts**
```
Problem: Two developers modified the same pipeline
Resolution:
1. Pull latest changes from main
2. Resolve conflicts in JSON files carefully
3. Test pipeline after merge
4. Create PR for review
```

**Issue 2: Publish Branch Out of Sync**
```
Problem: adf_publish branch has outdated ARM templates
Resolution:
1. In ADF Studio, switch to main branch
2. Click "Publish" button
3. This regenerates ARM templates in adf_publish
4. Use latest adf_publish for deployments
```

**Issue 3: Deployment Fails Due to Dependencies**
```
Problem: Pipeline references dataset that doesn't exist yet
Resolution:
1. Ensure deployment order: Linked Services → Datasets → Pipelines → Triggers
2. Use ARM template deployment which handles dependencies
3. Check parameter files have correct values
```

**Issue 4: Triggers Not Starting After Deployment**
```
Problem: Triggers remain stopped after deployment
Resolution:
1. Check service principal has Contributor role on ADF
2. Verify trigger start script executed successfully
3. Manually start triggers if needed:
   Get-AzDataFactoryV2Trigger | Start-AzDataFactoryV2Trigger -Force
```

### Best Practices

1. **Branching Strategy:**
   - Use feature branches for development
   - Keep main branch stable and deployable
   - Delete feature branches after merge

2. **Commit Messages:**
   - Use descriptive commit messages
   - Format: `[Type] Description`
   - Example: `[Feature] Add incremental load for customers table`

3. **Code Review Checklist:**
   - Naming conventions followed
   - Parameters used instead of hardcoded values
   - Error handling implemented
   - Logging added for troubleshooting
   - Performance considerations addressed

4. **Deployment Strategy:**
   - Always deploy to UAT first
   - Require approval for PROD deployments
   - Keep rollback plan ready
   - Deploy during maintenance windows

5. **Environment Isolation:**
   - Separate service principals per environment
   - Different Key Vaults per environment
   - Environment-specific parameter files

### Interview Questions

**Q1: How do you implement CI/CD for Azure Data Factory?**

**Answer:** "In my projects, I've implemented CI/CD for ADF using Azure DevOps. First, I configure Git integration in ADF Studio, connecting it to an Azure Repos Git repository. We use a feature branch workflow where developers create feature branches for changes, and all changes go through pull requests before merging to main.

For CI, I create a build pipeline that validates the JSON artifacts and publishes them as build artifacts. For CD, I use release pipelines with multiple stages - UAT and PROD. The deployment process involves stopping triggers, deploying the ARM template from the adf_publish branch, and restarting triggers. We use environment-specific parameter files to override connection strings and other environment-specific values.

I also implement approval gates for PROD deployments and maintain separate service principals for each environment to ensure proper security isolation."

**Q2: What's the difference between collaboration branch and publish branch?**

**Answer:** "The collaboration branch (typically 'main' or 'develop') is where developers work and save their changes. When you save a pipeline in ADF Studio, it commits to this branch. The publish branch (typically 'adf_publish') is where ARM templates are generated when you click the Publish button in ADF Studio.

The ARM templates in the publish branch are what you use for deploying to other environments. The collaboration branch contains the ADF JSON artifacts, while the publish branch contains the deployment-ready ARM templates. You should never manually edit the publish branch - it's automatically managed by ADF."

**Q3: How do you handle hotfixes in production?**

**Answer:** "For hotfixes, I follow this process: Create a hotfix branch from main, make the minimal necessary changes, get expedited review and approval, merge to main, publish to generate new ARM templates, and deploy to PROD using the release pipeline with proper change management approval.

After deployment, I verify the fix in PROD and monitor for any issues. The key is to keep hotfixes minimal and focused on the specific issue, and ensure they still go through the CI/CD pipeline rather than making manual changes directly in PROD ADF."

**Q4: What challenges have you faced with ADF Git integration?**

**Answer:** "One challenge I faced was merge conflicts when multiple developers worked on the same pipeline. ADF JSON files can be complex, and resolving conflicts requires careful attention. I addressed this by implementing clear ownership of pipelines and better communication within the team.

Another challenge was the learning curve for team members unfamiliar with Git workflows. I created documentation and conducted training sessions on branching strategy and pull request processes.

I also encountered issues with the publish branch getting out of sync, which I resolved by ensuring developers always publish from the main branch after merging their changes, and documenting this as a mandatory step in our process."

---

## Scenario 33: Environment Promotion (DEV → UAT → PROD)

### Business Requirement
Your organization has three environments (DEV, UAT, PROD) with different configurations for connection strings, storage accounts, and compute resources. You need a systematic approach to promote ADF pipelines across environments while maintaining environment-specific configurations and ensuring data doesn't leak between environments.

### Implementation Steps

#### Step 1: Environment Setup

**Environment Structure:**
```
DEV Environment:
- Resource Group: rg-adf-dev
- Data Factory: adf-dev-001
- Storage Account: stadlsdev001
- SQL Database: sqldb-dev-001
- Key Vault: kv-adf-dev-001

UAT Environment:
- Resource Group: rg-adf-uat
- Data Factory: adf-uat-001
- Storage Account: stadlsuat001
- SQL Database: sqldb-uat-001
- Key Vault: kv-adf-uat-001

PROD Environment:
- Resource Group: rg-adf-prod
- Data Factory: adf-prod-001
- Storage Account: stadlsprod001
- SQL Database: sqldb-prod-001
- Key Vault: kv-adf-prod-001
```

#### Step 2: Parameterize Linked Services

**Create Global Parameters in ADF:**

In DEV ADF, create global parameters:
```json
{
  "environment": "DEV",
  "storageAccountName": "stadlsdev001",
  "sqlServerName": "sqlserver-dev-001.database.windows.net",
  "sqlDatabaseName": "sqldb-dev-001",
  "keyVaultName": "kv-adf-dev-001",
  "dataLakePath": "https://stadlsdev001.dfs.core.windows.net"
}
```

**Parameterized Linked Service (Azure SQL):**
```json
{
  "name": "LS_AzureSQL_Parameterized",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "annotations": [],
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "LS_KeyVault",
          "type": "LinkedServiceReference"
        },
        "secretName": "sql-connection-string"
      }
    }
  }
}
```

**Parameterized Linked Service (ADLS Gen2):**
```json
{
  "name": "LS_ADLS_Parameterized",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "annotations": [],
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "@{concat('https://', pipeline().globalParameters.storageAccountName, '.dfs.core.windows.net')}",
      "accountKey": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "LS_KeyVault",
          "type": "LinkedServiceReference"
        },
        "secretName": "storage-account-key"
      }
    }
  }
}
```

#### Step 3: Create Environment-Specific Parameter Files

**ARMTemplateParametersForFactory-DEV.json:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "factoryName": {
      "value": "adf-dev-001"
    },
    "environment": {
      "value": "DEV"
    },
    "storageAccountName": {
      "value": "stadlsdev001"
    },
    "sqlServerName": {
      "value": "sqlserver-dev-001.database.windows.net"
    },
    "sqlDatabaseName": {
      "value": "sqldb-dev-001"
    },
    "keyVaultName": {
      "value": "kv-adf-dev-001"
    },
    "LS_KeyVault_properties_typeProperties_baseUrl": {
      "value": "https://kv-adf-dev-001.vault.azure.net/"
    }
  }
}
```

**ARMTemplateParametersForFactory-UAT.json:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "factoryName": {
      "value": "adf-uat-001"
    },
    "environment": {
      "value": "UAT"
    },
    "storageAccountName": {
      "value": "stadlsuat001"
    },
    "sqlServerName": {
      "value": "sqlserver-uat-001.database.windows.net"
    },
    "sqlDatabaseName": {
      "value": "sqldb-uat-001"
    },
    "keyVaultName": {
      "value": "kv-adf-uat-001"
    },
    "LS_KeyVault_properties_typeProperties_baseUrl": {
      "value": "https://kv-adf-uat-001.vault.azure.net/"
    }
  }
}
```

**ARMTemplateParametersForFactory-PROD.json:**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "factoryName": {
      "value": "adf-prod-001"
    },
    "environment": {
      "value": "PROD"
    },
    "storageAccountName": {
      "value": "stadlsprod001"
    },
    "sqlServerName": {
      "value": "sqlserver-prod-001.database.windows.net"
    },
    "sqlDatabaseName": {
      "value": "sqldb-prod-001"
    },
    "keyVaultName": {
      "value": "kv-adf-prod-001"
    },
    "LS_KeyVault_properties_typeProperties_baseUrl": {
      "value": "https://kv-adf-prod-001.vault.azure.net/"
    }
  }
}
```

#### Step 4: Promotion Process

**Manual Promotion Steps:**

1. **DEV to UAT Promotion:**
```powershell
# Step 1: Verify in DEV
Write-Host "Testing pipeline in DEV..."
$devPipeline = Invoke-AzDataFactoryV2Pipeline `
    -ResourceGroupName "rg-adf-dev" `
    -DataFactoryName "adf-dev-001" `
    -PipelineName "PL_Customer_Incremental_Load"

# Wait for completion
Start-Sleep -Seconds 60

# Check status
$status = Get-AzDataFactoryV2PipelineRun `
    -ResourceGroupName "rg-adf-dev" `
    -DataFactoryName "adf-dev-001" `
    -PipelineRunId $devPipeline

if ($status.Status -eq "Succeeded") {
    Write-Host "✓ DEV testing successful"
    
    # Step 2: Deploy to UAT
    Write-Host "Deploying to UAT..."
    New-AzResourceGroupDeployment `
        -ResourceGroupName "rg-adf-uat" `
        -TemplateFile "./ARMTemplateForFactory.json" `
        -TemplateParameterFile "./ARMTemplateParametersForFactory-UAT.json" `
        -Mode Incremental
    
    Write-Host "✓ Deployed to UAT"
    
    # Step 3: Test in UAT
    Write-Host "Testing in UAT..."
    $uatPipeline = Invoke-AzDataFactoryV2Pipeline `
        -ResourceGroupName "rg-adf-uat" `
        -DataFactoryName "adf-uat-001" `
        -PipelineName "PL_Customer_Incremental_Load"
    
    Write-Host "✓ UAT testing initiated"
} else {
    Write-Host "✗ DEV testing failed. Promotion aborted."
}
```

2. **UAT to PROD Promotion (with approvals):**
```powershell
# Automated via Azure DevOps Release Pipeline
# Requires manual approval gate

# Pre-deployment checks
function Test-PreDeployment {
    param($Environment)
    
    Write-Host "Running pre-deployment checks for $Environment..."
    
    # Check 1: Verify UAT success
    $uatRuns = Get-AzDataFactoryV2PipelineRun `
        -ResourceGroupName "rg-adf-uat" `
        -DataFactoryName "adf-uat-001" `
        -LastUpdatedAfter (Get-Date).AddDays(-7)
    
    $failedRuns = $uatRuns | Where-Object { $_.Status -eq "Failed" }
    if ($failedRuns.Count -gt 0) {
        throw "UAT has failed pipeline runs. Cannot promote to PROD."
    }
    
    # Check 2: Verify PROD backup
    Write-Host "Backing up PROD configuration..."
    # Export current PROD ADF configuration
    
    # Check 3: Verify maintenance window
    $currentHour = (Get-Date).Hour
    if ($currentHour -lt 22 -and $currentHour -gt 6) {
        Write-Warning "Deployment outside maintenance window (10 PM - 6 AM)"
        # Require additional approval
    }
    
    Write-Host "✓ Pre-deployment checks passed"
}

# Execute deployment
Test-PreDeployment -Environment "PROD"

# Stop PROD triggers
$triggers = Get-AzDataFactoryV2Trigger `
    -ResourceGroupName "rg-adf-prod" `
    -DataFactoryName "adf-prod-001"

foreach ($trigger in $triggers) {
    if ($trigger.RuntimeState -eq "Started") {
        Stop-AzDataFactoryV2Trigger `
            -ResourceGroupName "rg-adf-prod" `
            -DataFactoryName "adf-prod-001" `
            -Name $trigger.Name -Force
    }
}

# Deploy to PROD
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg-adf-prod" `
    -TemplateFile "./ARMTemplateForFactory.json" `
    -TemplateParameterFile "./ARMTemplateParametersForFactory-PROD.json" `
    -Mode Incremental

# Start PROD triggers
foreach ($trigger in $triggers) {
    Start-AzDataFactoryV2Trigger `
        -ResourceGroupName "rg-adf-prod" `
        -DataFactoryName "adf-prod-001" `
        -Name $trigger.Name -Force
}

Write-Host "✓ PROD deployment completed"
```

#### Step 5: Environment-Specific Pipeline Behavior

**Use Global Parameters in Pipelines:**
```json
{
  "name": "PL_Customer_Load",
  "properties": {
    "activities": [
      {
        "name": "Copy Customer Data",
        "type": "Copy",
        "inputs": [
          {
            "referenceName": "DS_Source_Customer",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "DS_Sink_Customer",
            "type": "DatasetReference",
            "parameters": {
              "folderPath": "@concat('data/', pipeline().globalParameters.environment, '/customers')"
            }
          }
        ]
      },
      {
        "name": "Log Environment",
        "type": "SetVariable",
        "dependsOn": [
          {
            "activity": "Copy Customer Data",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "variableName": "logMessage",
          "value": "@concat('Loaded in ', pipeline().globalParameters.environment, ' environment')"
        }
      }
    ]
  }
}
```

### Expected Behavior

1. **Environment Isolation:**
   - DEV uses DEV resources only
   - UAT uses UAT resources only
   - PROD uses PROD resources only
   - No cross-environment data access

2. **Consistent Behavior:**
   - Same pipeline logic across environments
   - Different configurations per environment
   - Predictable promotion process

3. **Promotion Flow:**
   - Develop and test in DEV
   - Promote to UAT for user acceptance testing
   - Promote to PROD after UAT approval
   - Each promotion uses environment-specific parameters

4. **Audit and Compliance:**
   - All promotions logged
   - Approval records maintained
   - Rollback capability available

### Common Issues and Resolutions

**Issue 1: Hardcoded Values Not Parameterized**
```
Problem: Pipeline works in DEV but fails in UAT due to hardcoded DEV storage path
Resolution:
1. Identify all hardcoded values
2. Replace with global parameters or pipeline parameters
3. Test in DEV with UAT parameter values
4. Re-promote to UAT
```

**Issue 2: Missing Secrets in Target Environment**
```
Problem: Deployment succeeds but pipeline fails with "Secret not found"
Resolution:
1. Ensure all secrets exist in target Key Vault
2. Verify secret names match across environments
3. Check ADF Managed Identity has Get/List permissions on Key Vault
4. Use consistent secret naming convention
```

**Issue 3: Different Data Volumes Cause Timeouts**
```
Problem: Pipeline runs fine in DEV (small data) but times out in PROD (large data)
Resolution:
1. Use environment-specific timeout values
2. Increase DIU for PROD environment
3. Implement partitioning for large datasets
4. Use global parameters for DIU configuration:
   @pipeline().globalParameters.copyDataIntegrationUnits
```

**Issue 4: Trigger Schedules Differ Across Environments**
```
Problem: Need different trigger schedules (DEV: hourly, PROD: every 15 min)
Resolution:
1. Parameterize trigger schedules in ARM template
2. Use environment-specific parameter files
3. Example parameter:
   "TR_Schedule_recurrence": {
     "value": {
       "frequency": "Minute",
       "interval": 15
     }
   }
```

### Best Practices

1. **Parameterization Strategy:**
   - Use global parameters for environment-wide settings
   - Use pipeline parameters for runtime values
   - Store secrets in Key Vault, never in parameters
   - Document all parameters and their purpose

2. **Testing Strategy:**
   - Unit test in DEV with various scenarios
   - Integration test in UAT with production-like data volumes
   - Performance test in UAT before PROD promotion
   - Smoke test in PROD immediately after deployment

3. **Deployment Strategy:**
   - Always promote DEV → UAT → PROD, never skip
   - Deploy during maintenance windows for PROD
   - Keep rollback plan ready
   - Notify stakeholders before PROD deployments

4. **Environment Parity:**
   - Keep UAT as close to PROD as possible
   - Use similar data volumes in UAT
   - Match compute resources between UAT and PROD
   - Test with production-like scenarios

5. **Change Management:**
   - Document all changes in release notes
   - Maintain change log with deployment dates
   - Track which version is in each environment
   - Require approvals for PROD promotions

### Interview Questions

**Q1: How do you manage environment-specific configurations in ADF?**

**Answer:** "I use a combination of global parameters and ARM template parameters to manage environment-specific configurations. In ADF, I create global parameters for values like environment name, storage account names, and database server names. These are referenced throughout pipelines and datasets using expressions like `pipeline().globalParameters.storageAccountName`.

For secrets like connection strings and access keys, I store them in Azure Key Vault and use Key Vault-backed linked services. Each environment has its own Key Vault with the same secret names but different values.

During deployment, I use environment-specific ARM parameter files that override the global parameters and linked service configurations. This ensures the same pipeline code runs across all environments but connects to environment-specific resources."

**Q2: Describe your environment promotion process.**

**Answer:** "My promotion process follows a strict DEV → UAT → PROD flow. In DEV, developers build and test pipelines with sample data. Once tested, we publish the changes which generates ARM templates in the adf_publish branch.

For UAT promotion, I deploy the ARM templates using the UAT parameter file through an Azure DevOps release pipeline. The UAT team performs user acceptance testing with production-like data volumes. If issues are found, we fix them in DEV and re-promote.

For PROD promotion, we require approval from stakeholders. The deployment happens during a maintenance window, typically after business hours. We stop all triggers, deploy the ARM template with PROD parameters, verify the deployment, and restart triggers. We monitor the first few pipeline runs closely and have a rollback plan ready.

Throughout this process, we maintain environment-specific parameter files and ensure secrets are properly configured in each environment's Key Vault."

**Q3: What challenges have you faced with environment promotion?**

**Answer:** "One major challenge was ensuring complete parameterization. Initially, we had some hardcoded values that caused failures in UAT and PROD. I addressed this by implementing a checklist for code reviews that specifically checks for hardcoded values and enforces the use of parameters.

Another challenge was data volume differences. Pipelines that ran quickly in DEV would timeout in PROD due to larger data volumes. I solved this by implementing environment-specific DIU settings and timeout values, and by conducting performance testing in UAT with production-like data volumes.

I also faced issues with missing or misnamed secrets in Key Vaults across environments. I created a standardized naming convention for secrets and a deployment checklist that verifies all required secrets exist before promoting to a new environment."

**Q4: How do you handle rollback if a PROD deployment fails?**

**Answer:** "I maintain a rollback strategy with multiple layers. First, before any PROD deployment, I export the current ADF configuration as a backup. This can be done by keeping the previous version of ARM templates.

If a deployment fails during the ARM template deployment phase, Azure automatically rolls back to the previous state. If the deployment succeeds but pipelines fail during execution, I have two options: either redeploy the previous version using the backed-up ARM templates, or make a hotfix if the issue is minor and can be quickly resolved.

I also implement a smoke test immediately after PROD deployment - running a simple pipeline to verify basic functionality before enabling all triggers. This gives us an early warning if something is wrong. Additionally, I maintain detailed deployment logs and have a communication plan to notify stakeholders if a rollback is needed."

---

## Scenario 34: Parameterized Linked Services

### Business Requirement
Your organization needs to create reusable ADF pipelines that can connect to different data sources dynamically based on runtime parameters. You want to avoid creating multiple linked services for similar connection types and need the flexibility to change connection details without modifying the linked service itself.

### Implementation Steps

#### Step 1: Understanding Parameterization Levels

**Parameterization Hierarchy:**
```
1. Global Parameters (Factory-level)
   ↓
2. Linked Service Parameters
   ↓
3. Dataset Parameters
   ↓
4. Pipeline Parameters
```

#### Step 2: Create Parameterized Linked Service for Azure SQL

**LS_AzureSQL_Parameterized:**
```json
{
  "name": "LS_AzureSQL_Parameterized",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "parameters": {
      "ServerName": {
        "type": "string",
        "defaultValue": "sqlserver-dev-001.database.windows.net"
      },
      "DatabaseName": {
        "type": "string",
        "defaultValue": "defaultDB"
      },
      "KeyVaultSecretName": {
        "type": "string",
        "defaultValue": "sql-connection-string"
      }
    },
    "annotations": [],
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "LS_KeyVault",
          "type": "LinkedServiceReference"
        },
        "secretName": "@{linkedService().KeyVaultSecretName}"
      }
    }
  }
}
```

**Alternative with Connection String Parameterization:**
```json
{
  "name": "LS_AzureSQL_FullyParameterized",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "parameters": {
      "ServerName": {
        "type": "string"
      },
      "DatabaseName": {
        "type": "string"
      },
      "UserName": {
        "type": "string"
      }
    },
    "annotations": [],
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "@concat('Server=tcp:',linkedService().ServerName,',1433;Database=',linkedService().DatabaseName,';User ID=',linkedService().UserName,';Encrypt=True;Connection Timeout=30;')",
      "password": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "LS_KeyVault",
          "type": "LinkedServiceReference"
        },
        "secretName": "sql-password"
      }
    }
  }
}
```

#### Step 3: Create Parameterized Linked Service for ADLS Gen2

**LS_ADLS_Parameterized:**
```json
{
  "name": "LS_ADLS_Parameterized",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "parameters": {
      "StorageAccountName": {
        "type": "string",
        "defaultValue": "stadlsdev001"
      },
      "ContainerName": {
        "type": "string",
        "defaultValue": "raw"
      }
    },
    "annotations": [],
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "@concat('https://',linkedService().StorageAccountName,'.dfs.core.windows.net/',linkedService().ContainerName)",
      "accountKey": {
        "type": "AzureKeyVaultSecret",
        "store": {
          "referenceName": "LS_KeyVault",
          "type": "LinkedServiceReference"
        },
        "secretName": "@concat(linkedService().StorageAccountName,'-key')"
      }
    }
  }
}
```

**With Managed Identity (Recommended):**
```json
{
  "name": "LS_ADLS_Parameterized_MI",
  "type": "Microsoft.DataFactory/factories/linkedservices",
  "properties": {
    "parameters": {
      "StorageAccountName": {
        "type": "string"
      }
    },
    "annotations": [],
    "type": "AzureBlobFS",
    "typeProperties": {
      "url": "@concat('https://',linkedService().StorageAccountName,'.dfs.core.windows.net')",
      "authenticationType": "ManagedIdentity"
    }
  }
}
```

#### Step 4: Create Parameterized Dataset Using Parameterized Linked Service

**DS_AzureSQL_Parameterized:**
```json
{
  "name": "DS_AzureSQL_Parameterized",
  "properties": {
    "linkedServiceName": {
      "referenceName": "LS_AzureSQL_Parameterized",
      "type": "LinkedServiceReference",
      "parameters": {
        "ServerName": "@dataset().ServerName",
        "DatabaseName": "@dataset().DatabaseName",
        "KeyVaultSecretName": "@dataset().SecretName"
      }
    },
    "parameters": {
      "ServerName": {
        "type": "string"
      },
      "DatabaseName": {
        "type": "string"
      },
      "SchemaName": {
        "type": "string"
      },
      "TableName": {
        "type": "string"
      },
      "SecretName": {
        "type": "string",
        "defaultValue": "sql-connection-string"
      }
    },
    "annotations": [],
    "type": "AzureSqlTable",
    "schema": [],
    "typeProperties": {
      "schema": "@dataset().SchemaName",
      "table": "@dataset().TableName"
    }
  }
}
```

**DS_ADLS_Parameterized:**
```json
{
  "name": "DS_ADLS_Parameterized",
  "properties": {
    "linkedServiceName": {
      "referenceName": "LS_ADLS_Parameterized_MI",
      "type": "LinkedServiceReference",
      "parameters": {
        "StorageAccountName": "@dataset().StorageAccountName"
      }
    },
    "parameters": {
      "StorageAccountName": {
        "type": "string"
      },
      "ContainerName": {
        "type": "string"
      },
      "FolderPath": {
        "type": "string"
      },
      "FileName": {
        "type": "string"
      }
    },
    "annotations": [],
    "type": "DelimitedText",
    "typeProperties": {
      "location": {
        "type": "AzureBlobFSLocation",
        "fileName": "@dataset().FileName",
        "folderPath": "@dataset().FolderPath",
        "fileSystem": "@dataset().ContainerName"
      },
      "columnDelimiter": ",",
      "escapeChar": "\\",
      "firstRowAsHeader": true,
      "quoteChar": "\""
    },
    "schema": []
  }
}
```

#### Step 5: Create Pipeline Using Parameterized Components

**PL_Dynamic_Copy_Metadata_Driven:**
```json
{
  "name": "PL_Dynamic_Copy_Metadata_Driven",
  "properties": {
    "parameters": {
      "SourceServerName": {
        "type": "string"
      },
      "SourceDatabaseName": {
        "type": "string"
      },
      "TargetStorageAccount": {
        "type": "string"
      },
      "TargetContainer": {
        "type": "string"
      }
    },
    "activities": [
      {
        "name": "Get Table List",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT SchemaName, TableName FROM dbo.TableMetadata WHERE IsActive = 1"
          },
          "dataset": {
            "referenceName": "DS_AzureSQL_Parameterized",
            "type": "DatasetReference",
            "parameters": {
              "ServerName": "@pipeline().parameters.SourceServerName",
              "DatabaseName": "@pipeline().parameters.SourceDatabaseName",
              "SchemaName": "dbo",
              "TableName": "TableMetadata"
            }
          },
          "firstRowOnly": false
        }
      },
      {
        "name": "ForEach Table",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "Get Table List",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "items": "@activity('Get Table List').output.value",
          "isSequential": false,
          "batchCount": 4,
          "activities": [
            {
              "name": "Copy Table Data",
              "type": "Copy",
              "typeProperties": {
                "source": {
                  "type": "AzureSqlSource",
                  "queryTimeout": "02:00:00"
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
                  "referenceName": "DS_AzureSQL_Parameterized",
                  "type": "DatasetReference",
                  "parameters": {
                    "ServerName": "@pipeline().parameters.SourceServerName",
                    "DatabaseName": "@pipeline().parameters.SourceDatabaseName",
                    "SchemaName": "@item().SchemaName",
                    "TableName": "@item().TableName"
                  }
                }
              ],
              "outputs": [
                {
                  "referenceName": "DS_ADLS_Parameterized",
                  "type": "DatasetReference",
                  "parameters": {
                    "StorageAccountName": "@pipeline().parameters.TargetStorageAccount",
                    "ContainerName": "@pipeline().parameters.TargetContainer",
                    "FolderPath": "@concat('data/', item().SchemaName)",
                    "FileName": "@concat(item().TableName, '.csv')"
                  }
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

#### Step 6: Advanced Pattern - Multi-Environment with Control Table

**Control Table (dbo.SourceSystemConfig):**
```sql
CREATE TABLE dbo.SourceSystemConfig (
    ConfigID INT PRIMARY KEY IDENTITY(1,1),
    SourceSystemName VARCHAR(100),
    ServerName VARCHAR(255),
    DatabaseName VARCHAR(100),
    Environment VARCHAR(20),
    IsActive BIT,
    KeyVaultSecretName VARCHAR(100),
    LastLoadDate DATETIME
);

-- Sample data
INSERT INTO dbo.SourceSystemConfig VALUES
('CRM_System', 'crm-sql-server.database.windows.net', 'CRM_DB', 'PROD', 1, 'crm-sql-connection', NULL),
('ERP_System', 'erp-sql-server.database.windows.net', 'ERP_DB', 'PROD', 1, 'erp-sql-connection', NULL),
('Finance_System', 'finance-sql-server.database.windows.net', 'Finance_DB', 'PROD', 1, 'finance-sql-connection', NULL);
```

**Master Pipeline (PL_Master_Multi_Source_Load):**
```json
{
  "name": "PL_Master_Multi_Source_Load",
  "properties": {
    "parameters": {
      "Environment": {
        "type": "string",
        "defaultValue": "PROD"
      }
    },
    "activities": [
      {
        "name": "Get Source Systems",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": {
              "value": "@concat('SELECT * FROM dbo.SourceSystemConfig WHERE Environment = ''', pipeline().parameters.Environment, ''' AND IsActive = 1')",
              "type": "Expression"
            }
          },
          "dataset": {
            "referenceName": "DS_ControlDB",
            "type": "DatasetReference"
          },
          "firstRowOnly": false
        }
      },
      {
        "name": "ForEach Source System",
        "type": "ForEach",
        "dependsOn": [
          {
            "activity": "Get Source Systems",
            "dependencyConditions": ["Succeeded"]
          }
        ],
        "typeProperties": {
          "items": "@activity('Get Source Systems').output.value",
          "isSequential": false,
          "batchCount": 3,
          "activities": [
            {
              "name": "Execute Child Pipeline",
              "type": "ExecutePipeline",
              "typeProperties": {
                "pipeline": {
                  "referenceName": "PL_Dynamic_Copy_Metadata_Driven",
                  "type": "PipelineReference"
                },
                "waitOnCompletion": true,
                "parameters": {
                  "SourceServerName": "@item().ServerName",
                  "SourceDatabaseName": "@item().DatabaseName",
                  "TargetStorageAccount": "@pipeline().globalParameters.storageAccountName",
                  "TargetContainer": "@concat('data-', toLower(item().SourceSystemName))"
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

#### Step 7: REST API Parameterized Linked Service

**LS_RestAPI_Parameterized:**
```json
{
  "name": "LS_RestAPI_Parameterized",
  "properties": {
    "parameters": {
      "BaseURL": {
        "type": "string"
      },
      "AuthTokenSecretName": {
        "type": "string",
        "defaultValue": "api-auth-token"
      }
    },
    "annotations": [],
    "type": "RestService",
    "typeProperties": {
      "url": "@{linkedService().BaseURL}",
      "enableServerCertificateValidation": true,
      "authenticationType": "Anonymous"
    }
  }
}
```

**DS_RestAPI_Parameterized:**
```json
{
  "name": "DS_RestAPI_Parameterized",
  "properties": {
    "linkedServiceName": {
      "referenceName": "LS_RestAPI_Parameterized",
      "type": "LinkedServiceReference",
      "parameters": {
        "BaseURL": "@dataset().BaseURL"
      }
    },
    "parameters": {
      "BaseURL": {
        "type": "string"
      },
      "RelativeURL": {
        "type": "string"
      }
    },
    "annotations": [],
    "type": "RestResource",
    "typeProperties": {
      "relativeUrl": "@dataset().RelativeURL"
    },
    "schema": []
  }
}
```

### Expected Behavior

1. **Dynamic Connections:**
   - Single linked service connects to multiple servers/databases
   - Runtime determination of connection details
   - Reduced number of linked services to maintain

2. **Reusability:**
   - Same pipeline works across different environments
   - Same pipeline handles multiple source systems
   - Metadata-driven approach reduces code duplication

3. **Flexibility:**
   - Easy to add new source systems (just add to control table)
   - Connection details can be changed without modifying pipelines
   - Environment-specific behavior controlled by parameters

4. **Maintainability:**
   - Centralized configuration in control tables
   - Single point of change for connection details
   - Easier testing and debugging

### Common Issues and Resolutions

**Issue 1: Test Connection Fails for Parameterized Linked Service**
```
Problem: Cannot test connection when linked service has parameters
Resolution:
This is expected behavior. You can only test connection when:
1. All parameters have default values, OR
2. You test at the dataset level where parameters are provided

Workaround:
- Provide default values for all linked service parameters
- Test using a pipeline with actual parameter values
- Use dataset test connection instead
```

**Issue 2: Key Vault Secret Not Found**
```
Problem: Pipeline fails with "Secret 'xyz' not found in Key Vault"
Resolution:
1. Verify secret name is correct
2. Check ADF Managed Identity has Get/List permissions on Key Vault
3. Ensure secret name parameter is passed correctly
4. Debug by logging the secret name being used:
   @concat('Looking for secret: ', linkedService().KeyVaultSecretName)
```

**Issue 3: Expression Evaluation Errors**
```
Problem: "Invalid expression" error when using nested parameters
Resolution:
1. Check expression syntax carefully
2. Ensure proper escaping of quotes in concat functions
3. Use @{} syntax for expressions
4. Test expressions in Set Variable activity first

Example fix:
Wrong: @concat('Server=',linkedService().ServerName,';Database=',linkedService().DatabaseName)
Right: @concat('Server=',linkedService().ServerName,';Database=',linkedService().DatabaseName,';')
```

**Issue 4: Performance Issues with Dynamic Connections**
```
Problem: Pipeline runs slower when using parameterized linked services
Resolution:
1. This is normal - dynamic connection establishment takes time
2. Use connection pooling where possible
3. Consider caching connection details if used repeatedly
4. Balance between reusability and performance
5. For high-frequency pipelines, consider dedicated linked services
```

### Best Practices

1. **When to Parameterize:**
   - Multiple similar connections (different databases on same server)
   - Multi-environment deployments (DEV/UAT/PROD)
   - Metadata-driven patterns
   - Dynamic source/sink scenarios

2. **When NOT to Parameterize:**
   - Single, static connection
   - High-frequency, performance-critical pipelines
   - When connection details never change
   - Simple scenarios where parameterization adds complexity

3. **Security Best Practices:**
   - Always store credentials in Key Vault
   - Use Managed Identity when possible
   - Never pass passwords as pipeline parameters
   - Parameterize Key Vault secret names, not secret values

4. **Naming Conventions:**
   - Linked Service: `LS_<Type>_Parameterized`
   - Parameters: PascalCase (e.g., `ServerName`, `DatabaseName`)
   - Default values for optional parameters
   - Clear parameter descriptions

5. **Testing Strategy:**
   - Test with default parameter values first
   - Test with various parameter combinations
   - Validate error handling for invalid parameters
   - Test connection failures gracefully

6. **Documentation:**
   - Document all parameters and their purpose
   - Provide example values for each parameter
   - Document which parameters are required vs optional
   - Maintain list of valid values for each parameter

### Interview Questions

**Q1: What are parameterized linked services and when would you use them?**

**Answer:** "Parameterized linked services allow you to create a single linked service that can connect to multiple instances of the same type of data source by passing parameters at runtime. Instead of creating separate linked services for each database or storage account, you create one parameterized linked service and pass the specific connection details when you use it.

I use parameterized linked services in several scenarios: First, for multi-environment deployments where the same pipeline needs to connect to different servers in DEV, UAT, and PROD. Second, for metadata-driven patterns where a single pipeline loads data from multiple source systems. Third, when implementing dynamic pipelines that need to connect to different databases based on runtime conditions.

For example, in a recent project, I had to load data from 15 different SQL databases. Instead of creating 15 linked services, I created one parameterized linked service with ServerName and DatabaseName parameters. This made the solution much more maintainable and scalable."

**Q2: How do you pass parameters to a linked service?**

**Answer:** "Parameters are passed to linked services through datasets. The flow is: Pipeline Parameters → Dataset Parameters → Linked Service Parameters.

In the dataset definition, I reference the parameterized linked service and map dataset parameters to linked service parameters. For example:

```json
'linkedServiceName': {
  'referenceName': 'LS_AzureSQL_Parameterized',
  'parameters': {
    'ServerName': '@dataset().ServerName',
    'DatabaseName': '@dataset().DatabaseName'
  }
}
```

Then in the pipeline, when I reference the dataset, I pass the actual values:

```json
'parameters': {
  'ServerName': '@pipeline().parameters.SourceServer',
  'DatabaseName': '@pipeline().parameters.SourceDB'
}
```

You cannot pass parameters directly from a pipeline to a linked service - it must go through a dataset. This is by design to maintain the separation of concerns between connection logic and data structure."

**Q3: What are the limitations of parameterized linked services?**

**Answer:** "There are several important limitations to be aware of. First, you cannot test the connection directly in a parameterized linked service unless all parameters have default values. You need to test at the dataset or pipeline level where actual parameter values are provided.

Second, not all linked service properties can be parameterized. For example, you cannot parameterize the authentication type - it must be fixed when you create the linked service. You can parameterize connection strings, URLs, and some authentication properties like usernames, but the overall authentication method is fixed.

Third, there's a slight performance overhead because the connection is established dynamically at runtime rather than being pre-validated. For high-frequency, performance-critical pipelines, this might be a consideration.

Fourth, debugging can be more complex because you need to trace parameter values through multiple levels - pipeline to dataset to linked service. I address this by using Set Variable activities to log parameter values during development.

Finally, you cannot use parameterized linked services with some integration runtime configurations, particularly with certain on-premises data sources that require pre-configured connections."

**Q4: How do you handle secrets in parameterized linked services?**

**Answer:** "Security is critical when working with parameterized linked services. I always store secrets in Azure Key Vault and use Key Vault-backed linked services. The key is to parameterize the secret name, not the secret value itself.

For example, I create a Key Vault linked service first, then reference it in my parameterized linked service:

```json
'password': {
  'type': 'AzureKeyVaultSecret',
  'store': {
    'referenceName': 'LS_KeyVault',
    'type': 'LinkedServiceReference'
  },
  'secretName': '@linkedService().KeyVaultSecretName'
}
```

I pass the secret name as a parameter (like 'sql-prod-password' or 'sql-dev-password'), and ADF retrieves the actual secret value from Key Vault at runtime. This way, the secret value never appears in pipeline code or parameters.

I also use Managed Identity authentication wherever possible, especially for Azure services like ADLS Gen2 and Azure SQL. This eliminates the need to manage credentials entirely. The ADF Managed Identity is granted appropriate permissions on the target resources, and authentication happens automatically.

For multi-environment scenarios, I maintain consistent secret naming conventions across Key Vaults. For example, 'sql-connection-string' exists in all environment Key Vaults but with environment-specific values."

---

## Summary

These three scenarios (32-34) cover critical DevOps and deployment aspects of Azure Data Factory:

1. **Git Integration (CI/CD)**: Version control, code review, and automated deployment pipelines
2. **Environment Promotion**: Systematic approach to moving pipelines across DEV → UAT → PROD
3. **Parameterized Linked Services**: Creating flexible, reusable connection components

Together, these scenarios enable enterprise-grade ADF implementations with proper governance, security, and maintainability. They represent real-world practices used in production environments and are frequently discussed in senior-level ADF interviews.

### Key Takeaways

- **DevOps is Essential**: Modern ADF implementations require proper CI/CD practices
- **Parameterization is Power**: Reduces duplication and increases flexibility
- **Security First**: Always use Key Vault and Managed Identity
- **Environment Isolation**: Strict separation between DEV, UAT, and PROD
- **Automation**: Minimize manual interventions in deployment processes
- **Documentation**: Maintain clear documentation of parameters and processes

These patterns form the foundation of scalable, maintainable ADF solutions in enterprise environments.

