# Building Custom Azure Images with CrowdStrike Falcon

This solution demonstrates how to create custom Azure images with CrowdStrike Falcon sensor pre-installed using Azure Image Builder. The images can be used across various Azure compute services, providing endpoint protection from the moment systems are deployed.

> [!IMPORTANT]
> Using VM Extensions and Azure Policy is the recommended approach for most scenarios. Image Builder provides an alternative when there is a specialized need for pre-installed security components or when VM Extensions cannot be used. See the [VM Extension documentation](https://github.com/CrowdStrike/azure-vm-extension) for details on using VM Extensions and Azure Policy for automated sensor deployment.

## Use Cases

Azure Image Builder with CrowdStrike Falcon integration supports multiple deployment scenarios:

- **Microsoft Dev Box**: Pre-configured development environments with security built-in
- **Virtual Machines (VMs)**: Standardized VM images with consistent security posture
- **Virtual Machine Scale Sets (VMSS)**: Auto-scaling workloads with pre-installed protection

## How It Works

The solution uses Azure Image Builder to:
1. Start with a base image
2. Install CrowdStrike Falcon sensor using [Falcon Scripts](https://github.com/CrowdStrike/falcon-scripts)
3. Configure the sensor with your CrowdStrike credentials
4. Output the customized image to Azure Compute Image Gallery
5. Make the image available for deployment across Azure services

## Prerequisites

- Azure Subscription with appropriate permissions
- Azure Compute Gallery configured. See https://learn.microsoft.com/en-us/azure/virtual-machines/shared-image-galleries?tabs=vmsource%2Cazure-cli
- CrowdStrike Falcon API credentials with appropriate permissions
  - See [Falcon Scripts API Permissions documentation](https://github.com/CrowdStrike/falcon-scripts/tree/main/powershell/install#falcon-api-permissions)
- User-assigned managed identity with permissions to the Azure Compute Gallery

## Template Features

The included example ARM template [azure-image-builder-template.json](azure-image-builder-template.json) provides:

- **Configurable source image**: Choose Windows 11, Windows 10, or Windows Server base images. You can change the image by modifying the `source` section. For example:
  ```json
  "source": {
    "type": "PlatformImage",
    "publisher": "MicrosoftWindowsServer",
    "offer": "WindowsServer",
    "sku": "2022-datacenter-azure-edition",
    "version": "latest"
  }
  ```
- **CrowdStrike Falcon integration**: Automated sensor installation using official scripts
- **Flexible sensor configuration**: Customize installation parameters (e.g., NO_START=1 for golden images)
- **Multi-region support**: Replicate images to multiple Azure regions. You can accomplish this by adding multiple distribution targets in the `distribute` section. For example:
  ```json
  "distribute": [
    {
      "type": "SharedImage",
      "galleryImageId": "[variables('imageDefinitionId')]",
      "runOutputName": "devBoxImage",
      "replicationRegions": ["eastus", "westus2"],
      "storageAccountType": "Standard_LRS"
    },
    {
      "type": "ManagedImage",
      "imageId": "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP/providers/Microsoft.Compute/images/falcon-vm-image",
      "runOutputName": "managedImage",
      "location": "eastus"
    }
  ]
  ```
- **Azure Compute Gallery output**: Images ready for enterprise deployment

## Deployment Guide

> [!IMPORTANT]
>
> This is an example implementation showing how to integrate CrowdStrike Falcon sensor installation into Azure Image Builder templates.
> Customize the template parameters and deployment steps according to your organization's requirements, security policies, and specific use cases.

### Windows Image Creation Guide

#### Understanding the Customization

The template uses Azure Image Builder's PowerShell customization to install CrowdStrike Falcon. The following code snippet shows how it downloads and executes the Falcon installation script:

```json
"customize": [
  {
    "type": "PowerShell",
    "name": "Install-CrowdStrikeFalcon",
    "inline": [
      "$scriptUrl = 'https://raw.githubusercontent.com/CrowdStrike/falcon-scripts/main/powershell/install/falcon_windows_install.ps1'; [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; $scriptPath = Join-Path $env:TEMP 'falcon_windows_install.ps1'",
      "Invoke-WebRequest -Uri $scriptUrl -OutFile $scriptPath -UseBasicParsing -Verbose",
      "[concat('& $scriptPath -FalconClientId ''', parameters('falconClientId'), ''' -FalconClientSecret ''', parameters('falconClientSecret'), ''' -FalconCloud ''', parameters('falconCloud'), ''' -InstallParams ''', parameters('falconInstallParams'), ''' -Verbose')]"
    ],
    "runElevated": true,
    "runAsSystem": true
  }
]
```

The template downloads the [Falcon Windows installation script](https://github.com/CrowdStrike/falcon-scripts/tree/main/powershell/install) and executes it with your API credentials during the image build process.

#### 1. Create the Image Definition

Before deploying the Image Builder template, you must create an image definition in your Azure Compute Gallery. The Image Builder template references this definition to store the built image.

**Using Azure CLI:**

```bash
az sig image-definition create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --gallery-name YOUR_GALLERY_NAME \
  --gallery-image-definition falcon-protected-windows \
  --publisher "CrowdStrike" \
  --offer "FalconProtected" \
  --sku "Windows-Enterprise" \
  --os-type "Windows" \
  --os-state "Generalized" \
  --hyper-v-generation "V2" \
  --features SecurityType=TrustedLaunch \
  --description "Windows with CrowdStrike Falcon Sensor pre-installed"
```

**Using Azure PowerShell:**

```powershell
New-AzGalleryImageDefinition `
  -ResourceGroupName YOUR_RESOURCE_GROUP_NAME `
  -GalleryName YOUR_GALLERY_NAME `
  -Name falcon-protected-windows `
  -Publisher "CrowdStrike" `
  -Offer "FalconProtected" `
  -Sku "Windows-Enterprise" `
  -OsType "Windows" `
  -OsState "Generalized" `
  -HyperVGeneration "V2" `
  -Feature @{Name='SecurityType';Value='TrustedLaunch'} `
  -Description "Windows with CrowdStrike Falcon Sensor pre-installed"
```

> [!NOTE]
> The image definition name `falcon-protected-windows` must match the `imageDefinitionName` parameter in your Image Builder template deployment.

#### 2. Deploy the Image Builder Template

**Using Azure CLI:**

```bash
az deployment group create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --template-file azure-image-builder-template.json \
  --parameters \
    osType="Windows" \
    userAssignedIdentityId="/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.ManagedIdentity/userAssignedIdentities/YOUR_IDENTITY_NAME" \
    falconClientId="YOUR_FALCON_CLIENT_ID" \
    falconClientSecret="YOUR_FALCON_CLIENT_SECRET" \
    falconCloud="us-1" \
    computeImageGalleryName="YOUR_GALLERY_NAME"
```

**Using Azure PowerShell:**

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName YOUR_RESOURCE_GROUP_NAME `
  -TemplateFile azure-image-builder-template.json `
  -osType "Windows" `
  -userAssignedIdentityId "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.ManagedIdentity/userAssignedIdentities/YOUR_IDENTITY_NAME" `
  -falconClientId "YOUR_FALCON_CLIENT_ID" `
  -falconClientSecret "YOUR_FALCON_CLIENT_SECRET" `
  -falconCloud "us-1" `
  -computeImageGalleryName "YOUR_GALLERY_NAME"
```

**Using Parameters File:**

```bash
az deployment group create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --template-file azure-image-builder-template.json \
  --parameters @parameters.json
```

#### 3. Build the Custom Image

**Trigger the image build:**

```bash
az image builder run \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name falcon-windows-image
```

**Monitor build progress:**

```bash
az image builder show \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name falcon-windows-image \
  --query "lastRunStatus"
```

The build process typically takes 60-90 minutes, depending on the customizations and base image size.

### Linux Image Creation Guide

#### Understanding the Customization

The template uses Azure Image Builder's Shell customization to install CrowdStrike Falcon on Linux. The installer uses environment variables for configuration.

**Configure the source for Ubuntu:**

```json
"source": {
  "type": "PlatformImage",
  "publisher": "Canonical",
  "offer": "0001-com-ubuntu-server-jammy",
  "sku": "22_04-lts-gen2",
  "version": "latest"
}
```

**Customize section for Linux:**

The Linux installer uses environment variables. Set them inline before running the script:

```json
"customize": [
  {
    "type": "Shell",
    "name": "Install-CrowdStrikeFalcon",
    "inline": [
      "curl -sSL https://raw.githubusercontent.com/CrowdStrike/falcon-scripts/main/bash/install/falcon-linux-install.sh -o /tmp/falcon_linux_install.sh",
      "chmod +x /tmp/falcon_linux_install.sh",
      "[concat('sudo FALCON_CLIENT_ID=\"', parameters('falconClientId'), '\" FALCON_CLIENT_SECRET=\"', parameters('falconClientSecret'), '\" FALCON_CLOUD=\"', parameters('falconCloud'), '\" PREP_GOLDEN_IMAGE=\"true\" /tmp/falcon_linux_install.sh')]"
    ]
  }
]
```

**Example with additional options:**

```json
"customize": [
  {
    "type": "Shell",
    "name": "Install-CrowdStrikeFalcon",
    "inline": [
      "curl -sSL https://raw.githubusercontent.com/CrowdStrike/falcon-scripts/main/bash/install/falcon-linux-install.sh -o /tmp/falcon_linux_install.sh",
      "chmod +x /tmp/falcon_linux_install.sh",
      "sudo FALCON_CLIENT_ID=\"YOUR_CLIENT_ID\" FALCON_CLIENT_SECRET=\"YOUR_CLIENT_SECRET\" FALCON_CLOUD=\"us-1\" FALCON_TAGS=\"ImageBuilder,Production\" PREP_GOLDEN_IMAGE=\"true\" /tmp/falcon_linux_install.sh"
    ]
  }
]
```

See the [Falcon Scripts bash installer documentation](https://github.com/CrowdStrike/falcon-scripts/tree/main/bash/install) for complete installation options.

#### 1. Create the Image Definition

Before deploying the Image Builder template, you must create an image definition in your Azure Compute Gallery. The Image Builder template references this definition to store the built image.

**Using Azure CLI:**

```bash
az sig image-definition create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --gallery-name YOUR_GALLERY_NAME \
  --gallery-image-definition falcon-protected-linux \
  --publisher "CrowdStrike" \
  --offer "FalconProtected" \
  --sku "Ubuntu-Server" \
  --os-type "Linux" \
  --os-state "Generalized" \
  --hyper-v-generation "V2" \
  --features SecurityType=TrustedLaunch \
  --description "Linux with CrowdStrike Falcon Sensor pre-installed"
```

**Using Azure PowerShell:**

```powershell
New-AzGalleryImageDefinition `
  -ResourceGroupName YOUR_RESOURCE_GROUP_NAME `
  -GalleryName YOUR_GALLERY_NAME `
  -Name falcon-protected-linux `
  -Publisher "CrowdStrike" `
  -Offer "FalconProtected" `
  -Sku "Ubuntu-Server" `
  -OsType "Linux" `
  -OsState "Generalized" `
  -HyperVGeneration "V2" `
  -Feature @{Name='SecurityType';Value='TrustedLaunch'} `
  -Description "Linux with CrowdStrike Falcon Sensor pre-installed"
```

> [!NOTE]
> The image definition name `falcon-protected-linux` must match the `imageDefinitionName` parameter in your Image Builder template deployment.

#### 2. Deploy the Image Builder Template

**Using Azure CLI:**

```bash
az deployment group create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --template-file azure-image-builder-template.json \
  --parameters \
    osType="Linux" \
    userAssignedIdentityId="/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.ManagedIdentity/userAssignedIdentities/YOUR_IDENTITY_NAME" \
    falconClientId="YOUR_FALCON_CLIENT_ID" \
    falconClientSecret="YOUR_FALCON_CLIENT_SECRET" \
    falconCloud="us-1" \
    computeImageGalleryName="YOUR_GALLERY_NAME" \
    falconTags="ImageBuilder,Production"
```

**Using Azure PowerShell:**

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName YOUR_RESOURCE_GROUP_NAME `
  -TemplateFile azure-image-builder-template.json `
  -osType "Linux" `
  -userAssignedIdentityId "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.ManagedIdentity/userAssignedIdentities/YOUR_IDENTITY_NAME" `
  -falconClientId "YOUR_FALCON_CLIENT_ID" `
  -falconClientSecret "YOUR_FALCON_CLIENT_SECRET" `
  -falconCloud "us-1" `
  -computeImageGalleryName "YOUR_GALLERY_NAME" `
  -falconTags "ImageBuilder,Production"
```

**Using Parameters File:**

```bash
az deployment group create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --template-file azure-image-builder-template.json \
  --parameters @parameters.json
```

#### 3. Build the Custom Image

**Trigger the image build:**

```bash
az image builder run \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name falcon-linux-image
```

**Monitor build progress:**

```bash
az image builder show \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name falcon-linux-image \
  --query "lastRunStatus"
```

The build process typically takes 60-90 minutes, depending on the customizations and base image size.

## Using the Custom Image

### For Virtual Machines

**Create a VM from the custom image:**

```bash
az vm create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name YOUR_VM_NAME \
  --image "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.Compute/galleries/YOUR_GALLERY_NAME/images/YOUR_IMAGE_DEFINITION_NAME/versions/latest" \
  --security-type TrustedLaunch \
  --admin-username YOUR_USERNAME \
  --admin-password YOUR_PASSWORD
```

### For Virtual Machine Scale Sets

**Create a VMSS from the custom image:**

```bash
az vmss create \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --name YOUR_VMSS_NAME \
  --image "/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.Compute/galleries/YOUR_GALLERY_NAME/images/YOUR_IMAGE_DEFINITION_NAME/versions/latest" \
  --security-type TrustedLaunch \
  --admin-username YOUR_USERNAME \
  --admin-password YOUR_PASSWORD \
  --instance-count 2
```
