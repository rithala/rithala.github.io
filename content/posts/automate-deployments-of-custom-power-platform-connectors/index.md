---
title: 'Automate Power Platform custom connectors deployment'
date: 2022-07-04T22:00:00+02:00
draft: false
cover:
    image: "images/connector-automation-cover.png"
    alt: "Article cover"
    caption: "Automate Power Platform custom connectors deployment"
images:
- "images/connector-automation-cover.png"
tags: ['Power Platform', 'Power Automate', 'DevOps']
---

Power Platform custom connectors enable using your API in Power Apps or Power Automate flows. When the API which you plan to integrate with your app is a stable and finished product, then creating a custom connector is a one-time task to do. However, when the API is a part of a developed solution or is under active development then keeping the connector in sync with the API is a cumbersome task.

If you are already familiar with [DevOps](https://www.atlassian.com/devops/what-is-devops) principles then "Continous Integration (CI)" and "Continous Deployment (CD)" terms should come to your mind. What we try to achieve here is to get the latest definition of the API after its update, and prepare the custom connector (CI). After that, push the changes to a Power Platform environment (CD).

## Building the deployment pipeline

How to do this? In this article, I use Azure DevOps pipelines and [Power Platform Connectors CLI (paconn)](https://docs.microsoft.com/en-us/connectors/custom-connectors/paconn-cli) to achieve this. The process should be similar for other build automation software like Jenkins or GitHub Actions.

### Create the custom connector in the Power Automate or Power Apps portal

Wait, we are talking about automation, and then I need to create something manually? Yes, you can create the connector from the scratch using CLI. However, making it in the UI is much more simple. The automation process will take care of updating the connector, and that is the most important for us.

Follow the [official guide](https://docs.microsoft.com/en-us/connectors/custom-connectors/define-blank) to create the custom connector.

After saving the connector, copy the ID from the URL. Look for a value similar to `shared_test-5f4540e4010cf8ad28-5fb7527efc5ed37b29` in between `/connections/available/custom/` and `/edit/`. This value is URL encoded. Please remember to **decode** it before future usage.

### Installing Power Platform Connectors CLI

Installation is very easy. The CLI is a Python app available in PIP.

```bash
pip install paconn
```

How to run it in Azure DevOps?

```yml
- script: pip install paconn
  displayName: Ensure PACONN
```

### Authentication

This is the tricky part. At the moment of writing this post, `paconn` does not support userless authentication flows. So we need to help it a little by creating a simple python script that obtains access token to Azure Service Management using a user/password combination (Resource Owner Password Credential).

Currently, there is no way to get the access token as an app (Client Credentials) which is why the script uses a service account for authentication. **Make sure the account has got an Environment Maker role in the target Power Platform environment**.

> You can use your own Azure AD app registration for the deployment by passing "Client ID" parameter to the script. If not provided the default app of CLI is used. Basicaly, this is the same as in Azure CLI 😉
> If you want to use your app, make sure it has got permissions to `https://management.azure.com/user_impersonation` scope.

```python
# authenticate-user.py

import argparse
import os

import adal
from msrestazure.azure_active_directory import AADTokenCredentials
from paconn.authentication.tokenmanager import TokenManager
from paconn.settings.settings import Settings
from paconn.common.util import get_config_dir

parser = argparse.ArgumentParser(
    description='Login to Azure Management using ROPC.')

parser.add_argument('--user', action='store', type=str, required=True)
parser.add_argument('--password', action='store', type=str, required=True)
parser.add_argument('--clientid', action='store', type=str)
parser.add_argument('--tenant', action='store', type=str)

args = parser.parse_args()

os.makedirs(get_config_dir(), exist_ok=True)

# Pass nones as it is not needed here
settings = Settings(
    connector_id=None,
    environment=None,
    api_properties=None,
    api_definition=None,
    icon=None,
    script=None,
    powerapps_url=None,
    powerapps_api_version=None
)


tenant = args.tenant or settings.tenant
client_id = args.clientid or settings.client_id

auth_context = adal.AuthenticationContext(
    authority=settings.authority_url + tenant,
    api_version=None
)

token = auth_context.acquire_token_with_username_password(
    resource=settings.resource,
    username=args.user,
    password=args.password,
    client_id=client_id
)

credentials = AADTokenCredentials(
    token=token,
    client_id=client_id)

tokenmanager = TokenManager()
tokenmanager.write(credentials.token)

```

Of course, you have to run it in the pipeline.

```yml
- task: PythonScript@0
  displayName: Login to Azure Management with ROPC
  inputs:
    scriptSource: 'filePath'
    scriptPath: '$(Pipeline.Workspace)/drop/connector/authenticate-user.py'
    arguments: '--user "$(pp_connector_user)" --password "$(pp_connector_password)" --tenant "$(az_ad_tenant_id)"'
```

### Connector deployment

Finally, it is time to deploy the connector. I used PowerShell, but you can use any scripting language. The steps are:

1. Download the connector definition from the Power Platform environment
2. Fetch the latest Open API definition of your API (only ver. 2 - Swagger 2.0 is supported currently)
3. Update the connector with a new API definition
4. Clean up the stored credentials

```PowerShell
# update-connector.ps1
param (
    [string]$ApiDefinitionUrl, # something like "https://webappname.azurewebsites.net/swagger/v1/swagger.json"
    [string]$EnvId, # Power Platform environment ID
    [string]$ConnectorId # The value copied from the URL after you created the connector
)


Write-Host "1/4 Getting connector definition"

paconn.exe download -e $EnvId -c $ConnectorId

Set-Location -Path ".\$ConnectorId"

Write-Host "2/4 Updating swagger file"

# Add a few retries here, the app needs some time to warm up straight after the deployment.
Invoke-RestMethod -Method Get -Uri $ApiDefinitionUrl -OutFile "apiDefinition.swagger.json" -MaximumRetryCount 5 -TimeoutSec 30 -RetryIntervalSec 15


Write-Host "3/4 Updating connector"

paconn.exe update -s .\settings.json

Write-Host "4/4 Cleaning creadentials"

Remove-Item "$env:USERPROFILE\.paconn" -Recurse -Force
```

Let's run it!

```yml
- task: PowerShell@2
  displayName: Update connector
  inputs:
    filePath: '$(Pipeline.Workspace)/drop/connector/update-connector.ps1'
    arguments: '-ApiDefinitionUrl $(az_api_swagger_url) -EnvId "$(pp_env_id)" -ConnectorId "$(pp_connector_id)"'
    workingDirectory: '$(Pipeline.Workspace)/drop/connector'
```

### Putting it all together

Now it is time to see what the final pipeline looks like.

```yml
# pipeline.yml
# Main API and custom connector CICD pipeline file


trigger:
- develop
- master

variables:
- group: pipeline-variables

stages:
  - stage: build
    displayName: Build Stage
    variables:
      - name: project_path
        value: $(Build.SourcesDirectory)/api/
      - name: connector_path
        value: $(Build.SourcesDirectory)/connector/
      - name: build_configuration
        value: Release
      - name: build_platform
        value: Any CPU
      - name: runtime
        value: linux-x64
      - name: vmName
        value: ubuntu-latest

    jobs:
      - job: build_api
        displayName: Build API and Connector
        pool:
          vmImage: '$(vmName)'
        steps:
          # Here are omitted tasks for building the API package
          - task: CopyFiles@2
            inputs:
              SourceFolder: '$(connector_path)'
              Contents: '**'
              TargetFolder: '$(Build.ArtifactStagingDirectory)/connector'
          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'drop'
              publishLocation: 'Container'

    - deployment: deployment_api
      displayName: Deploy API
      pool:
        vmImage: 'ubuntu-latest'
      environment: env-name
      strategy:
        runOnce:
          deploy:
            steps:
            - download: current
              artifact: drop
            - template: api-deployment.yml
              parameters:
                az_subscription: $(az_subcription_test)


    - deployment: deployment_connector
      displayName: Deploy PA Connector
      pool:
        vmImage: 'windows-latest'
      environment: env-name
      dependsOn: deployment_api
      condition: succeeded()
      strategy:
        runOnce:
          deploy:
            steps:
            - download: current
              artifact: drop
            - template: connector-deployment.yml
```

```yml
# connector-deployment.yml

steps:
  - task: UsePythonVersion@0
    displayName: Use Python 3
    inputs:
      versionSpec: '3.x'
      addToPath: true
      architecture: 'x64'

  - script: pip install paconn
    displayName: Ensure PACONN

  - task: PythonScript@0
    displayName: Login to Azure Management with ROPC
    inputs:
      scriptSource: 'filePath'
      scriptPath: '$(Pipeline.Workspace)/drop/connector/authenticate-user.py'
      arguments: '--user "$(pp_connector_user)" --password "$(pp_connector_password)" --tenant "$(az_ad_tenant_id)"'

  - task: PowerShell@2
    displayName: Update connector
    inputs:
      filePath: '$(Pipeline.Workspace)/drop/connector/update-connector.ps1'
      arguments: '-ApiDefinitionUrl $(az_api_swagger_url) -EnvId "$(pp_env_id)" -ConnectorId "$(pp_connector_id)"'
      workingDirectory: '$(Pipeline.Workspace)/drop/connector'
```

## Summary

Automating a custom connector deployment allows a development team to work more efficiently. Every new API endpoint is ready to use in your Power App or Power Automate just a few minutes later after pushed changes. There is a list of key things to remember:
* Make sure the service account has got an Environment Maker role in the target Power Platform environment
* Store the credentials securely e.g., variables group in Azure DevOps and make them "secret" type.
* You can use the Azure Pipelines Teams connector to notify the team about a new connector version.
* To make the changes available in a canvas app you need to remove and add again the connectorW
