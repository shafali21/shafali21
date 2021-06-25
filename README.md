$ t
rigger:
stages:
- stage: Terraform_init_plan

  pool:
    vmImage: ubuntu-latest

  jobs:

  - job: terraform_init_plan

    steps:

    - task: TerraformInstaller@0
      displayName: Install TF 0.14.8
      inputs:
        terraformVersion: '0.14.8'

    - task: TerraformTaskV1@0
      displayName: $ terraform init
      inputs:
        provider: 'azurerm'
        command: 'init'
        workingDirectory: '$(System.DefaultWorkingDirectory)/01_ResourceGroup'
        backendServiceArm: 'FAB-SPN'
        backendAzureRmResourceGroupName: 'terraform-state-fab-rg'
        backendAzureRmStorageAccountName: 'terraformstatefab056'
        backendAzureRmContainerName: 'tfstate'
        backendAzureRmKey: 'terraform.state'

    - task: TerraformTaskV1@0
      displayName: $ terraform plan
      inputs:
        provider: 'azurerm'
        command: 'plan'
        workingDirectory: '$(System.DefaultWorkingDirectory)/01_ResourceGroup'
        commandOptions: '-out tfplan'
        environmentServiceNameAzureRM: 'FAB-SPN'

    - script: |
          cd $(System.DefaultWorkingDirectory)/01_ResourceGroup
          terraform show -json tfplan >> tfplan.json
          # Format tfplan.json file
          terraform show -json tfplan | jq '.' > tfplan.json
          # show only the changes
          cat tfplan.json | jq '[.resource_changes[] | {type: .type, name: .change.after.name, actions: .change.actions[]}]'
      displayName: Create tfplan.json
    - task: PublishBuildArtifacts@1
      displayName: Upload tfplan
      inputs:
        PathtoPublish: '$(System.DefaultWorkingDirectory)/01_ResourceGroup/'
        ArtifactName: 'drop'
        publishLocation: 'Container'

  - job: waitForValidation
    displayName: Wait for external validation
    dependsOn: terraform_init_plan
    pool: server
    timeoutInMinutes: 4320 # job times out in 3 days
    steps:

    - task: ManualValidation@0
      inputs:
        notifyUsers: 'houssem.dellai@live.com'
        instructions: 'you should validate the Terraform Plan file'

- stage: Terraform_apply

  pool:
    vmImage: ubuntu-latest

  jobs:

  - job: terraform_apply

    steps:

    - task: DownloadBuildArtifacts@0
      displayName: Download tfplan
      inputs:
        buildType: 'current'
        downloadType: 'specific'
        itemPattern: 'drop/tfplan'
        downloadPath: '$(System.ArtifactsDirectory)'

    - task: CopyFiles@2
      displayName: Copy tfplan
      inputs:
        SourceFolder: '$(System.ArtifactsDirectory)/drop'
        Contents: 'tfplan'
        TargetFolder: '$(System.DefaultWorkingDirectory)/01_ResourceGroup'

    - task: TerraformInstaller@0
bash: trigger:: command not found
      displayName: Install TF 0.14.8
      inputs:
        terraformVersion: '0.14.8'

    - task: TerraformTaskV1@0
      displayName: $ terraform init
      inputs:
        provider: 'azurerm'
        command: 'init'
        workingDirectory: '$(System.DefaultWorkingDirectory)/01_ResourceGroup'
        backendServiceArm: 'FAB-SPN'
        backendAzureRmResourceGroupName: 'terraform-state-fab-rg'
        backendAzureRmStorageAccountName: 'terraformstatefab056'
        backendAzureRmContainerName: 'tfstate'
        backendAzureRmKey: 'terraform.state'

    - task: TerraformTaskV1@0
      displayName: $ terraform apply
      inputs:
        provider: 'azurerm'
        command: 'apply'
        workingDirectory: '$(System.DefaultWorkingDirectory)/01_ResourceGroup'
        commandOptions: 'tfplan'
        environmentServiceNameAzureRM: 'FAB-SPN'
