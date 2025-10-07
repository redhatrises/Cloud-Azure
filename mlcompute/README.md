# Installing CrowdStrike Falcon Sensor on Azure ML Compute Instances

This guide provides instructions for installing the CrowdStrike Falcon sensor on Azure Machine Learning compute instances using [CrowdStrike falcon-scripts](https://github.com/CrowdStrike/falcon-scripts) and Azure ML setup script capabilities.

## Prerequisites

Before starting, ensure you have:

- **CrowdStrike Falcon Credentials**
  - Crowdstrike Oauth Client ID and Client Secret scoped to be able to download sensors

- **Azure Resources Required:**

  **1. User-Assigned Managed Identity**
  - Create a user-assigned managed identity (e.g., `falcon-compute-identity`)
  - Note the following values from the identity:
    - **Principal ID**: Used for granting Key Vault access
    - **Client ID**: Used in the installation script command arguments
    - **Resource ID**: Used when attaching the identity to compute instances

  **2. Azure Key Vault**
  - Create or use an existing Key Vault
  - Store your Falcon credentials as secrets:
    - Secret name: `falcon-client-id` (value: your Falcon API Client ID)
    - Secret name: `falcon-client-secret` (value: your Falcon API Client Secret)

  **3. Key Vault Access for Managed Identity**
  - Grant the user-assigned managed identity access to read Key Vault secrets
  - Use either:
    - **RBAC**: Assign "Key Vault Secrets User" role to the managed identity

- **Permissions**
  - Ability to create compute instances in Azure ML
  - Access to manage Key Vault secrets
  - Permission to assign managed identities

## Step 1: Create the Installation Script

Create a bash script that retrieves credentials from Key Vault using the compute instance's managed identity and installs Falcon using the official CrowdStrike installation script.

> [!IMPORTANT]
> You must use the Azure SDK (azure-identity library) with the managed identity's explicit `client_id`. This script uses Python inline to retrieve the secrets, then continues with bash for the installation.

**File: `install-falcon.sh`**

> [!IMPORTANT]
> This Bash script is not meant to contain all functionality provided in Falcon Scripts. Customize it as needed.

```bash
#!/bin/bash

################################################################################
# Azure ML Compute Instance - Falcon Sensor Installation
# Uses official CrowdStrike falcon-linux-install.sh
# Retrieves credentials from Azure Key Vault using managed identity via Python
#
# Usage: install-falcon.sh <key-vault-name> <identity-client-id> [cloud-region]
# Example: install-falcon.sh your-keyvault-name your_identity_client_id us-1
################################################################################

set -e

# Get command arguments
KEY_VAULT_NAME="${1:?Error: Key Vault name required as first argument}"
IDENTITY_CLIENT_ID="${2:?Error: Identity client ID required as second argument}"
FALCON_CLOUD="${3:-us-1}"

# Logging setup
LOG_FILE="/var/log/falcon-installation.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "============================================================"
echo "Falcon Sensor Installation - Azure ML Compute Instance"
echo "Started: $(date)"
echo "Hostname: $(hostname)"
echo "Key Vault: $KEY_VAULT_NAME"
echo "Identity Client ID: $IDENTITY_CLIENT_ID"
echo "Falcon Cloud: $FALCON_CLOUD"
echo "============================================================"

# Install dependencies
echo "[1/5] Installing dependencies..."
apt-get update -qq
apt-get install -y -qq python3 python3-pip curl wget

# Install Python packages for Key Vault access
echo "[2/5] Installing Python packages for Azure SDK..."
python3 -m pip install -q azure-identity azure-keyvault-secrets

# Retrieve credentials from Key Vault using Python
echo "[3/5] Retrieving credentials from Key Vault..."
SECRETS_JSON=$(python3 -c "
import sys
import json
from azure.identity import ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

key_vault_name = sys.argv[1]
identity_client_id = sys.argv[2]

try:
    credential = ManagedIdentityCredential(client_id=identity_client_id)
    vault_url = f'https://{key_vault_name}.vault.azure.net/'
    kv_client = SecretClient(vault_url=vault_url, credential=credential)

    falcon_client_id = kv_client.get_secret('falcon-client-id').value
    falcon_client_secret = kv_client.get_secret('falcon-client-secret').value

    output = {
        'FALCON_CLIENT_ID': falcon_client_id,
        'FALCON_CLIENT_SECRET': falcon_client_secret
    }
    print(json.dumps(output))
except Exception as e:
    print(f'Error: {e}', file=sys.stderr)
    sys.exit(1)
" "$KEY_VAULT_NAME" "$IDENTITY_CLIENT_ID")

RETRIEVE_RESULT=$?

if [ $RETRIEVE_RESULT -ne 0 ]; then
    echo ""
    echo "============================================================"
    echo "ERROR: Failed to retrieve credentials from Key Vault"
    echo "============================================================"
    echo ""
    echo "Possible causes:"
    echo "1. The managed identity doesn't have access to Key Vault"
    echo "2. The secrets don't exist in Key Vault"
    echo "3. The identity client ID is incorrect"
    echo ""
    echo "To verify:"
    echo "  - Check Key Vault access policy or RBAC permissions"
    echo "  - Verify secrets exist: falcon-client-id, falcon-client-secret"
    echo "  - Confirm identity client ID: $IDENTITY_CLIENT_ID"
    echo "============================================================"
    exit 1
fi

# Parse JSON output
FALCON_CLIENT_ID=$(echo "$SECRETS_JSON" | python3 -c "import sys, json; print(json.load(sys.stdin)['FALCON_CLIENT_ID'])")
FALCON_CLIENT_SECRET=$(echo "$SECRETS_JSON" | python3 -c "import sys, json; print(json.load(sys.stdin)['FALCON_CLIENT_SECRET'])")

if [ -z "$FALCON_CLIENT_ID" ] || [ -z "$FALCON_CLIENT_SECRET" ]; then
    echo "ERROR: Failed to parse credentials from Key Vault response"
    exit 1
fi

echo "✓ Credentials retrieved successfully"

# Download the official CrowdStrike installation script
echo "[4/5] Downloading official CrowdStrike installation script..."
SCRIPT_URL="https://raw.githubusercontent.com/CrowdStrike/falcon-scripts/main/bash/install/falcon-linux-install.sh"
INSTALL_SCRIPT="/tmp/falcon-linux-install.sh"

curl -L -s -o "$INSTALL_SCRIPT" "$SCRIPT_URL"

if [ ! -f "$INSTALL_SCRIPT" ]; then
    echo "ERROR: Failed to download installation script from $SCRIPT_URL"
    exit 1
fi

chmod +x "$INSTALL_SCRIPT"
echo "✓ Installation script downloaded successfully"

# Export credentials for falcon-linux-install.sh
export FALCON_CLIENT_ID
export FALCON_CLIENT_SECRET
export FALCON_CLOUD

# Run the official installation script
echo "[5/5] Running Falcon installation..."
echo "----------------------------------------"

"$INSTALL_SCRIPT"

INSTALL_RESULT=$?
if [ $INSTALL_RESULT -ne 0 ]; then
    echo "ERROR: Installation script failed with exit code $INSTALL_RESULT"
    exit $INSTALL_RESULT
fi

echo "✓ Falcon installation completed successfully"

# Cleanup
rm -f "$INSTALL_SCRIPT"
unset FALCON_CLIENT_ID FALCON_CLIENT_SECRET

echo "============================================================"
echo "Falcon Sensor Installation Complete"
echo "Completed: $(date)"
echo "============================================================"
echo ""
echo "The agent will appear in the Falcon console within 5-10 minutes."
echo "Log file: $LOG_FILE"
```

## Step 2: Upload Script to Azure ML Notebooks

1. Sign into [Azure ML Studio](https://ml.azure.com) and select your workspace
2. Navigate to **Notebooks** in the left menu
3. Click **Add files** → **Create new file**
4. Name it `install-falcon.sh`
5. Change **File type** to **bash (.sh)**
6. Paste the bash script content from Step 1 and save

## Step 3: Create Compute Instance with Falcon

### Using Azure ML Studio (Portal)

1. Navigate to [Azure ML Studio](https://ml.azure.com)
2. Go to **Compute** → **Compute instances**
3. Click **+ New**
4. Configure the required settings as desired for your environment

5. Under **Assign managed identity**:
   - Select **User assigned**
   - Click **Add user assigned managed identity**
   - Select the user assigned managed identity to use for authentication e.g. `falcon-compute-identity`
   - Click **Select**

6. Under **Applications**:
   - Toggle on **Provision with a creation script**
   - Click **Browse**
   - **Script location**: Select **Notebook file**
   - **Select script**: Browse and select `install-falcon.sh`
   - **Command arguments**: Enter your configuration (example below)
     ```
     your-keyvault-name your_identity_client_id us-1
     ```
     * First argument: Your Key Vault name
     * Second argument: Identity client ID
     * Third argument: Falcon cloud region (us-1, us-2, eu-1, etc.)
   - **Timeout (minutes)**: Set to `20`

7. Click **Create**

**That's it!** The compute instance will be created with the user-assigned identity that has access to Key Vault. The Falcon sensor will be installed automatically during provisioning.

### Using the Azure Machine Learning Python SDK v2

> [!IMPORTANT]
> This Python code is not meant to contain all functionality provided in Falcon Scripts. Customize it as needed.

```python
from azure.ai.ml import MLClient
from azure.ai.ml.entities import (
    ComputeInstance,
    SetupScripts,
    ScriptReference,
    ManagedIdentityConfiguration,
    IdentityConfiguration
)
from azure.identity import DefaultAzureCredential

# Configuration variables - customize these
SUBSCRIPTION_ID = "00000000-0000-0000-0000-000000000000"  # Replace with your subscription ID
RESOURCE_GROUP = "your-resource-group"  # Replace with your resource group name
WORKSPACE = "your-workspace-name"  # Replace with your workspace name
IDENTITY_CLIENT_ID = "your_identity_client_id"  # Replace with identity client ID from prerequisites
IDENTITY_NAME = "falcon-compute-identity"  # Replace with your identity name
KEY_VAULT_NAME = "your-keyvault-name"  # Replace with your Key Vault name
FALCON_CLOUD = "us-1"  # Replace with your Falcon cloud region (us-1, us-2, eu-1, etc.)
SCRIPT_PATH = "Users/your-username/install-falcon.sh"  # Replace 'your-username' with your Azure ML username

# Construct the identity resource ID from variables
IDENTITY_RESOURCE_ID = f"/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/{IDENTITY_NAME}"

# Initialize client
credential = DefaultAzureCredential()
ml_client = MLClient(
    credential=credential,
    subscription_id=SUBSCRIPTION_ID,
    resource_group_name=RESOURCE_GROUP,
    workspace_name=WORKSPACE
)

# Define setup script with command arguments
setup_scripts = SetupScripts(
    creation_script=ScriptReference(
        path=SCRIPT_PATH,
        command=f"bash install-falcon.sh {KEY_VAULT_NAME} {IDENTITY_CLIENT_ID} {FALCON_CLOUD}",
        timeout_minutes=20
    )
)

# Configure user-assigned managed identity using resource_id (required by Azure ML API)
identity_config = IdentityConfiguration(
    type="UserAssigned",
    user_assigned_identities=[
        ManagedIdentityConfiguration(
            resource_id=IDENTITY_RESOURCE_ID
        )
    ]
)

# Create compute instance with Falcon setup
compute_instance = ComputeInstance(
    name="falcon-dev-instance",
    size="Standard_DS12_v2",
    setup_scripts=setup_scripts,
    identity=identity_config
)

# Create the instance
print(f"Creating compute instance '{compute_instance.name}'...")
result = ml_client.compute.begin_create_or_update(compute_instance).result()
print(f"✓ Compute instance created: {result.name}")
print("✓ User-assigned identity already has Key Vault access")
```

The compute instance will be created with the user-assigned identity that has access to Key Vault. The Falcon sensor will be installed automatically during provisioning.
