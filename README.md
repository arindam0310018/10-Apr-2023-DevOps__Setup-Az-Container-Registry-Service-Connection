# Setup ACR Service Connection using Devops

Greetings to my fellow Technology Advocates and Specialists.

In this Session, I will demonstrate __How to Automate Azure Container Registry (ACR) Service Connection using Devops.__

| __AUTOMATION OBJECTIVES:-__ |
| --------- |

| __#__ | __TOPICS__ |
| --------- | --------- |
| 1. | Install Azure Devops CLI Extension in the Build Agent. |
| 2. | Validate Azure Devops CLI Extension Installation by running the Help option in the Build Agent. |
| 3. | Download Key Vault Secrets. |
| 4. | Create Azure Container Registry Service Connection. |
| 5. | Grant Access Permission to Azure Container Registry Service Connection for all Pipelines. |

| IMPORTANT NOTE:- |
| --------- |

The YAML Pipeline is tested on __WINDOWS BUILD AGENT__ Only!!!

| __REQUIREMENTS:-__ |
| --------- |

1. Azure Subscription.
2. Azure DevOps Organisation and Project.
3. Full Access PAT (Personal Access Token).
4. Service Principal with Required RBAC ( Contributor) applied on Subscription or Resource Group(s).
5. Azure Resource Manager Service Connection in Azure DevOps.
6. Azure Container Registry with "Admin" User Enabled.
7. Key Vault with 3 Secrets stored - 1) Azure Container Registry password, 2) Azure DevOps Project ID, and 3) Azure Devops Personal Access Token (PAT).

| __LIST OF AZURE RESOURCES DEPLOYED AND CONFIGURED FOR THIS AUTOMATION:-__ |
| --------- |
| Azure Container Registry with "Admin" user enabled |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5thc71xbmd2gor3xxlt7.jpg) |
| Key Vault with 3 Secrets stored - 1) Azure Container Registry password, 2) Azure DevOps Project ID, and 3) Azure Devops Personal Access Token (PAT). |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qupbua5z8u33uc1fj0s3.jpg) |

| HOW DOES MY CODE PLACEHOLDER LOOKS LIKE:- |
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p9tnuis632bz14194ejd.jpg) |

| PIPELINE CODE SNIPPET:- | 
| --------- |

| AZURE DEVOPS YAML PIPELINE (azure-pipelines-acr-service-connection-v1.0.yml):- |
| --------- |

```
trigger:
  none

######################
# Declare Parameters:-
######################
parameters: 
- name: DevOpsOrganisation
  type: string
  default: https://dev.azure.com/ArindamMitra0251
  values:
  - https://dev.azure.com/ArindamMitra0251

- name: DevOpsProjName
  type: string
  default: AMCLOUD
  values:
  - AMCLOUD

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: ampockv

- name: ACRName
  type: object
  default: ampocapplacr

- name: ACRSrvConnectionName
  type: object
  default: AM-ACR-Srv-Connection

#######################
# Declare Variables:-
#######################
variables:
  ServiceConnection: 'amcloud-cicd-service-connection'
  BuildAgent: 'windows-latest'
  emailID: "mail2arindam2003@yahoo.com"
  
######################
# Declare Build Agent:-
######################
pool:
  vmImage: '$(BuildAgent)'

###################
# Declare Stages:-
###################
stages:

- stage: Az_DevOps_ACR_Service_Connection
  jobs:
  - job: Setup_ACR_Service_Connection
    displayName: Setup ACR Service Connection
    steps:

########################################################
# Install Az DevOps CLI Extension in the Build Agent:-
#######################################################
    - task: AzureCLI@1
      displayName: Install Devops CLI Extension
      inputs:
        azureSubscription: '$(ServiceConnection)'
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az extension add --name azure-devops
          az extension show --name azure-devops --output table

###############################################################
# Help Option of Az DevOps CLI Extension in the Build Agent:-
###############################################################
    - task: PowerShell@2
      displayName: Help Option of Az Devops CLI
      inputs:
        targetType: 'inline'
        script: |
          az devops -h

################################        
# Download Keyvault Secrets:-
################################
    - task: AzureKeyVault@2
      displayName: Fetch all Secrets from Keyvault
      inputs:
        azureSubscription: '$(ServiceConnection)'
        KeyVaultName: '${{ parameters.KVNAME }}'
        SecretsFilter: '*'
        RunAsPreJob: false

############################################################
# Create ACR Service Connection in Azure DevOps Project:-
############################################################
    - task: PowerShell@2
      displayName: Create ACR Service Connection
      inputs:
        targetType: 'inline'
        script: |
         $B64Pat = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes(":$(PAT)"))

         $header = @{
          'Authorization' = 'Basic ' + $B64Pat
          'Content-Type' = 'application/json'
          }
         
         $body = '{
            "data": {},
            "name": "${{ parameters.ACRSrvConnectionName }}",
            "type": "dockerregistry",
            "authorization": {
              "parameters": {
                "username": "${{ parameters.ACRName }}",
                "password": "$(ACRPasswd)",
              "email": "$(emailID)",
              "registry": "https://${{ parameters.ACRName }}.azurecr.io"
              },
              "scheme": "UsernamePassword"
            },
            "isShared": false,
            "isReady": true,
            "serviceEndpointProjectReferences": [
              {
                "projectReference": {
                  "id": "$(AMCLOUD-DevOps-Prj-ID)",
                  "name": "${{ parameters.DevOpsProjName }}"   
                },
                "name": "${{ parameters.ACRSrvConnectionName }}" 
              }
            ]
          }'
         
         $srvEndpointID = Invoke-RestMethod -Method Post -Uri ${{ parameters.DevOpsOrganisation }}/_apis/serviceendpoint/endpoints?api-version=6.0-preview.4 -Headers $header -Body $body | Select -ExpandProperty id

         echo "#####################################################################################"
         echo "ACR Service Connection ${{ parameters.ACRSrvConnectionName }} created successfully."
         echo "#####################################################################################" 
         
         $patchbody = '{
            "allPipelines": {
            "authorized": true,
            "authorizedBy": null,
            "authorizedOn": null
            },
            "pipelines": null,
            "resource": {
                "id": "$srvEndpointID",
                "type": "endpoint"
            }
          }'
        
          Invoke-RestMethod -Method PATCH -Uri ${{ parameters.DevOpsOrganisation }}/${{ parameters.DevOpsProjName }}/_apis/pipelines/pipelinepermissions/endpoint/"$srvEndpointID"?api-version=7.0-preview.1 -Headers $header -Body $patchbody

         echo "###############################################################################################################"
         echo "ACR Service Connection ${{ parameters.ACRSrvConnectionName }} was granted access permission to all Pipelines."
         echo "###############################################################################################################"
```

Now, let me explain each part of YAML Pipeline for better understanding.

| PART #1:- | 
| --------- |

| BELOW FOLLOWS PIPELINE RUNTIME VARIABLES CODE SNIPPET:- | 
| --------- |

```
######################
# Declare Parameters:-
######################
parameters: 
- name: DevOpsOrganisation
  type: string
  default: https://dev.azure.com/ArindamMitra0251
  values:
  - https://dev.azure.com/ArindamMitra0251

- name: DevOpsProjName
  type: string
  default: AMCLOUD
  values:
  - AMCLOUD

- name: KVNAME
  displayName: Please Provide the Keyvault Name:-
  type: object
  default: ampockv

- name: ACRName
  type: object
  default: ampocapplacr

- name: ACRSrvConnectionName
  type: object
  default: AM-ACR-Srv-Connection
```

| THIS IS HOW IT LOOKS:- | 
| --------- |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4sflbdz394i19jdva5yt.jpg) |

| PART #2:- | 
| --------- |

| BELOW FOLLOWS PIPELINE VARIABLES CODE SNIPPET:- | 
| --------- |

```
#######################
# Declare Variables:-
#######################
variables:
  ServiceConnection: 'amcloud-cicd-service-connection'
  BuildAgent: 'windows-latest'
  emailID: "mail2arindam2003@yahoo.com"
```

| NOTE:- | 
| --------- |
| Please change the values of the variables accordingly. |
| The entire YAML pipeline is build using Runtime Parameters and Variables. No Values are Hardcoded. |

| PART #3:- | 
| --------- |

| This is a Single Stage Pipeline - Az_DevOps_ACR_Service_Connection (with 4 Pipeline Tasks):- | 
| --------- |

```
###################
# Declare Stages:-
###################
stages:

- stage: Az_DevOps_ACR_Service_Connection
  jobs:
  - job: Setup_ACR_Service_Connection
    displayName: Setup ACR Service Connection
    steps:
```

| PIPELINE TASK #1:-:- | 
| --------- |
| Install Az DevOps CLI Extension in the Build Agent. |

```
########################################################
# Install Az DevOps CLI Extension in the Build Agent:-
#######################################################
    - task: AzureCLI@1
      displayName: Install Devops CLI Extension
      inputs:
        azureSubscription: '$(ServiceConnection)'
        scriptType: ps
        scriptLocation: inlineScript
        inlineScript: |
          az extension add --name azure-devops
          az extension show --name azure-devops --output table
```

| PIPELINE TASK #2:-:- | 
| --------- |
| Validate installation of Az DevOps CLI Extension in the Build Agent by running the Help option. |

```
###############################################################
# Help Option of Az DevOps CLI Extension in the Build Agent:-
###############################################################
    - task: PowerShell@2
      displayName: Help Option of Az Devops CLI
      inputs:
        targetType: 'inline'
        script: |
          az devops -h
```

| PIPELINE TASK #3:-:- | 
| --------- |
| Download all secrets from Keyvault. |

```
################################        
# Download Keyvault Secrets:-
################################
    - task: AzureKeyVault@2
      displayName: Fetch all Secrets from Keyvault
      inputs:
        azureSubscription: '$(ServiceConnection)'
        KeyVaultName: '${{ parameters.KVNAME }}'
        SecretsFilter: '*'
        RunAsPreJob: false
```

| PIPELINE TASK #4:-:- | 
| --------- |
| Create ACR Service Connection in Azure DevOps Project. |
| Grant the newly created ACR Service Connection in Azure DevOps Project access permission to all Pipelines. |

```
############################################################
# Create ACR Service Connection in Azure DevOps Project:-
############################################################
    - task: PowerShell@2
      displayName: Create ACR Service Connection
      inputs:
        targetType: 'inline'
        script: |
         $B64Pat = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes(":$(PAT)"))

         $header = @{
          'Authorization' = 'Basic ' + $B64Pat
          'Content-Type' = 'application/json'
          }
         
         $body = '{
            "data": {},
            "name": "${{ parameters.ACRSrvConnectionName }}",
            "type": "dockerregistry",
            "authorization": {
              "parameters": {
                "username": "${{ parameters.ACRName }}",
                "password": "$(ACRPasswd)",
              "email": "$(emailID)",
              "registry": "https://${{ parameters.ACRName }}.azurecr.io"
              },
              "scheme": "UsernamePassword"
            },
            "isShared": false,
            "isReady": true,
            "serviceEndpointProjectReferences": [
              {
                "projectReference": {
                  "id": "$(AMCLOUD-DevOps-Prj-ID)",
                  "name": "${{ parameters.DevOpsProjName }}"   
                },
                "name": "${{ parameters.ACRSrvConnectionName }}" 
              }
            ]
          }'
         
         $srvEndpointID = Invoke-RestMethod -Method Post -Uri ${{ parameters.DevOpsOrganisation }}/_apis/serviceendpoint/endpoints?api-version=6.0-preview.4 -Headers $header -Body $body | Select -ExpandProperty id

         echo "#####################################################################################"
         echo "ACR Service Connection ${{ parameters.ACRSrvConnectionName }} created successfully."
         echo "#####################################################################################" 
         
         $patchbody = '{
            "allPipelines": {
            "authorized": true,
            "authorizedBy": null,
            "authorizedOn": null
            },
            "pipelines": null,
            "resource": {
                "id": "$srvEndpointID",
                "type": "endpoint"
            }
          }'
        
          Invoke-RestMethod -Method PATCH -Uri ${{ parameters.DevOpsOrganisation }}/${{ parameters.DevOpsProjName }}/_apis/pipelines/pipelinepermissions/endpoint/"$srvEndpointID"?api-version=7.0-preview.1 -Headers $header -Body $patchbody

         echo "###############################################################################################################"
         echo "ACR Service Connection ${{ parameters.ACRSrvConnectionName }} was granted access permission to all Pipelines."
         echo "###############################################################################################################"
```

| NOTE:- | 
| --------- |
| Passing PAT as Pipeline Runtime variable or Fetching it from Key Vault and use it as Environmental variables will not work and it will throw below error. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ouf4yrfkjqbhmt97b19l.jpg) |
| Refer the [Github Issue: 24108](https://github.com/Azure/azure-cli/issues/24108) |
| Sample format of the JSON Config file is used to prepare the "$body" variable section. It can be found [here](https://learn.microsoft.com/en-us/azure/devops/cli/service-endpoint?view=azure-devops) |

__NOW ITS TIME TO TEST!!!__

| TEST CASES:- | 
| --------- |
| 1. Pipeline executed successfully. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xyxhf6c2hgmtgvqbcrio.jpg) |
| 2. ACR Service connection created successfully in the Devops project. |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9oi0j0v2hnfn1jn8xdck.jpg) |
| ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/al2da4yc6jllegd9izs3.jpg) |

__Hope You Enjoyed the Session!!!__

__Stay Safe | Keep Learning | Spread Knowledge__
