# GitHub Actions Azure OIDC - Resource Group & Storage Container

This repository contains a GitHub Actions workflow that securely connects to Microsoft Azure using **OpenID Connect (OIDC)** without long-lived client secrets, creates/manages an **Azure Resource Group**, **Storage Account**, and **Blob Storage Container** via manual triggering (`workflow_dispatch`), supports clean-up/deletion, and generates rich **GitHub Job Step Summaries**.

---

## 📋 Features

- 🔐 **Keyless Azure Authentication**: Uses GitHub OIDC (`azure/login@v2`) with `id-token: write` permission—no service principal passwords or client secrets needed.
- ⚙️ **Manual Dispatch Input**: Customizable inputs for `action` (`create` or `delete`), `resource_group_name`, `storage_account_name`, `container_name`, `azure_region`, and `delete_target`.
- 🗑️ **Granular Cleanup & Deletion**: Delete only the blob container, delete the entire storage account, or tear down the entire resource group.
- 📊 **Step Summary Output**: Appends Markdown status tables and operation details directly to `$GITHUB_STEP_SUMMARY` for visibility in the GitHub Actions UI.
- 🛡️ **Built-in Validation & Idempotency**: Pre-validates storage account name constraints (3-24 lowercase alphanumeric chars) and safely handles existing resources.

---

## 🛠️ Prerequisites & Azure Setup

To allow GitHub Actions to authenticate with Azure using OIDC, you need:
1. A Microsoft Entra ID (Azure AD) **App Registration**
2. A **Federated Identity Credential** linked to your GitHub repository
3. An **RBAC Role Assignment** (e.g., `Contributor`) on your Azure Subscription or Resource Group
4. Three GitHub Repository Secrets: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID`

---

### Option 1: Quick Setup via Azure CLI (Recommended)

Run the following script in Azure CLI / Cloud Shell (replace placeholders with your GitHub org/repo details):

```bash
# Set your variables
GH_ORG="your-github-username-or-org"
GH_REPO="github-oidc-azure"
APP_NAME="GitHub-Actions-OIDC-App"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# 1. Create the Entra ID Application
APP_ID=$(az ad app create --display-name "${APP_NAME}" --query appId -o tsv)

# 2. Create the Service Principal
SP_ID=$(az ad sp create --id "${APP_ID}" --query id -o tsv)

# 3. Add Federated Identity Credential for your repo's main branch
az ad app federated-credential create \
  --id "${APP_ID}" \
  --parameters '{
    "name": "github-actions-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"${GH_ORG}"'/'"${GH_REPO}"':ref:refs/heads/main",
    "description": "GitHub Actions OIDC for main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Assign Contributor role to the Service Principal
az role assignment create \
  --role "Contributor" \
  --assignee-object-id "${SP_ID}" \
  --assignee-principal-type "ServicePrincipal" \
  --scope "/subscriptions/${SUBSCRIPTION_ID}"

# 5. Display the secrets needed for GitHub
echo "------------------------------------------------------"
echo "AZURE_CLIENT_ID:       ${APP_ID}"
echo "AZURE_TENANT_ID:       $(az account show --query tenantId -o tsv)"
echo "AZURE_SUBSCRIPTION_ID: ${SUBSCRIPTION_ID}"
echo "------------------------------------------------------"
```

---

### Option 2: Manual Setup via Azure Portal

#### Step A: Create the App Registration
1. Navigate to **Microsoft Entra ID** > **App registrations** > **New registration**.
2. Name: `GitHub-Actions-OIDC-App` (or your preferred name).
3. Supported account types: **Accounts in this organizational directory only (Single tenant)**.
4. Click **Register**. Note down the **Application (client) ID** and **Directory (tenant) ID**.

#### Step B: Add Federated Credential (OIDC Trust)
1. In your App Registration, navigate to **Certificates & secrets** > **Federated credentials**.
2. Click **+ Add credential**.
3. Under **Federated credential scenario**, select **GitHub Actions deploying Azure resources**.
4. Configure the parameters:
   - **Organization**: Your GitHub username or organization (e.g., `octocat`).
   - **Repository**: Your repository name (e.g., `github-oidc-azure`).
   - **Entity type**: Select **Branch** and specify `main` (or select **Pull request** / **Environment**).
   - **Credential Name**: `github-actions-oidc`.
   - **Audience**: `api://AzureADTokenExchange` (default).
5. Click **Add**.

#### Step C: Assign RBAC Permissions
1. Navigate to **Subscriptions** > Select your Subscription (or target Resource Group).
2. Go to **Access control (IAM)** > **Add** > **Add role assignment**.
3. Role: Select **Contributor** (allows managing Resource Groups and Storage Accounts).
4. Members: Select **User, group, or service principal** > Click **+ Select members**.
5. Search for your application name (`GitHub-Actions-OIDC-App`) and select it.
6. Click **Review + assign**.

> [!TIP]
> If your organization enforces strict data plane RBAC, also assign the **Storage Blob Data Contributor** role.

---

### Step D: Add GitHub Repository Secrets

In your GitHub repository:
1. Navigate to **Settings** > **Secrets and variables** > **Actions**.
2. Click **New repository secret** and add the following 3 secrets:

| Secret Name | Value | Description |
| :--- | :--- | :--- |
| `AZURE_CLIENT_ID` | `00000000-0000-0000-0000-000000000000` | Application (client) ID from App Registration |
| `AZURE_TENANT_ID` | `00000000-0000-0000-0000-000000000000` | Directory (tenant) ID of your Entra tenant |
| `AZURE_SUBSCRIPTION_ID` | `00000000-0000-0000-0000-000000000000` | Target Azure Subscription ID |

---

## 🚀 Running the Workflow

### 1. Create Resources
1. Go to the **Actions** tab in your GitHub repository.
2. Select the **Azure OIDC - Resource Group & Storage Container** workflow.
3. Click **Run workflow**.
4. Set:
   - `action`: `create`
   - `resource_group_name`: e.g. `rg-github-oidc-demo`
   - `storage_account_name`: e.g. `mystorage12345` (3-24 alphanumeric lowercase characters, globally unique)
   - `container_name`: e.g. `demo-container`
   - `azure_region`: e.g. `eastus` (or `centralindia`, `westeurope`, etc.)
5. Click **Run workflow**.

### 2. Delete / Clean-up Resources
1. Click **Run workflow**.
2. Set `action` to `delete`.
3. Choose the `delete_target`:
   - `container_only`: Deletes only the Blob Container.
   - `storage_account`: Deletes the Storage Account and all its containers.
   - `entire_resource_group`: Deletes the entire Resource Group and all contained resources.
4. Click **Run workflow**.

---

## 📝 GitHub Step Summary Examples

### ✅ Create Success Example:
| Parameter | Value |
| --- | --- |
| **Action** | `create` |
| **Resource Group** | `rg-github-oidc-demo` |
| **RG Status** | ✅ Created Successfully |
| **Storage Account** | `mystorage12345` |
| **Account Status** | ✅ Created Successfully |
| **Blob Container** | `demo-container` |
| **Container Status** | ✅ Created Successfully |
| **Azure Region** | `eastus` |
| **Overall Status** | ✅ Created / Verified Successfully |

> Resource Group `rg-github-oidc-demo`, Storage Account `mystorage12345`, and Blob Container `demo-container` are ready.

---

### 🗑️ Delete Success Example:
| Parameter | Value |
| --- | --- |
| **Action** | `delete` |
| **Delete Target** | `entire_resource_group` |
| **Resource Group** | `rg-github-oidc-demo` |
| **Storage Account** | `mystorage12345` |
| **Blob Container** | `demo-container` |
| **Result** | 🗑️ Resource Group Deleted |
| **Overall Status** | ✅ Completed |

> Resource Group `rg-github-oidc-demo` and all contained resources (including storage accounts & containers) were deleted.

---

### ❌ Failure Example:
| Parameter | Value |
| --- | --- |
| **Action** | `create` |
| **Failed Step** | Create Storage Account |
| **Resource Group** | `rg-github-oidc-demo` |
| **Storage Account** | `alreadytakenaccount` |
| **Container Name** | `demo-container` |
| **Azure Region** | `eastus` |
| **Status** | ❌ Failed |

```text
The storage account name 'alreadytakenaccount' is already taken.
```
