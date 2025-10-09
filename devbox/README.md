# Deploy CrowdStrike Falcon for Microsoft Dev Box using Azure Image Builder

This solution demonstrates how to create custom Microsoft Dev Box images with CrowdStrike Falcon sensor pre-installed using Azure Image Builder. Dev Boxes provisioned from these images will have endpoint protection built-in from the moment they are created, providing security from day one for your development teams.

## Prerequisites

- Microsoft Dev Center configured in your subscription
- Follow the [Azure Image Builder with CrowdStrike Falcon guide](https://github.com/CrowdStrike/cloud-azure/tree/main/imagebuilder) to create your custom Windows image
  - This includes creating the Azure Compute Gallery, image definition, deploying the Image Builder template, and building the custom image
  - See [Microsoft Dev Box Image Builder documentation](https://learn.microsoft.com/en-us/azure/dev-box/how-to-customize-devbox-azure-image-builder)
- Windows 11 Enterprise as the base image

## Dev Box Configuration

**Workflow:**
```
Azure Image Builder → Custom Image with Falcon → Azure Compute Gallery → Dev Box Definition → Developer Dev Boxes
```

Once you have completed the [Azure Image Builder setup](https://github.com/CrowdStrike/cloud-azure/tree/main/imagebuilder) and have a custom image available in your Azure Compute Gallery, follow these steps to configure Dev Box.

### Step 1: Create a Dev Box Definition

**Using Azure CLI:**

```bash
az devcenter admin devbox-definition create \
  --name "FalconProtectedDevBox" \
  --dev-center-name YOUR_DEV_CENTER_NAME \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --image-reference id="/subscriptions/YOUR_SUBSCRIPTION_ID/resourceGroups/YOUR_RESOURCE_GROUP_NAME/providers/Microsoft.DevCenter/devcenters/testdev/galleries/YOUR_GALLERY_NAME/images/falcon-protected-windows/versions/latest" \
  --os-storage-type "ssd_256gb" \
  --sku name="general_i_8c32gb256ssd_v2"
```

**Using Azure Portal:**

1. Navigate to your Dev Center
2. Select "Dev box definitions" under "Dev box configuration"
3. Click "+ Create"
4. Provide a name (e.g., "FalconProtectedDevBox")
5. Under "Image", select the name of image from your Azure Compute Gallery
6. Choose either `latest` or a specific version number for the "Image" version
7. Select appropriate compute size and storage options
8. Click "Create"

### Step 2: Assign the Definition to a Dev Box Pool

Create or update a Dev Box pool to use the new definition:

```bash
az devcenter admin pool create \
  --name "SecureDevPool" \
  --project-name YOUR_PROJECT_NAME \
  --resource-group YOUR_RESOURCE_GROUP_NAME \
  --devbox-definition-name "FalconProtectedDevBox" \
  --local-administrator "Enabled" \
  --network-connection-name YOUR_NETWORK_CONNECTION_NAME
```

### Step 3: Provision Dev Boxes

Developers can now provision Dev Boxes through:

- **Microsoft Dev Box Developer Portal**: https://devbox.microsoft.com
- **Azure Portal**: Navigate to Dev Center → Projects → Dev boxes
- **Azure CLI**:
  ```bash
  az devcenter dev dev-box create \
    --name "my-secure-devbox" \
    --project-name YOUR_PROJECT_NAME \
    --pool-name "SecureDevPool" \
    --dev-center-name YOUR_DEV_CENTER_NAME
  ```

The Dev Box will provision with CrowdStrike Falcon already installed and configured.
