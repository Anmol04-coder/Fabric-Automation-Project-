# 🚀 Microsoft Fabric & Azure DevOps Git Provisioning Automation

This Python notebook automates the creation and Git-integration of a Microsoft Fabric Workspace using the Fabric REST APIs (via `sempy.fabric`) and Service Principal authentication.

---

## 🛠️ Key Technologies & Dependencies

* **Primary Tool:** `sempy.fabric.FabricRestClient` (Fabric REST API client)
* **Authentication:** `azure.identity.ClientSecretCredential`
* **Secret Management:** `notebookutils.mssparkutils.credentials` (for Azure Key Vault integration)
* **External API:** `msgraph-sdk` (Microsoft Graph API)
* **Git API:** `requests` with `HTTPBasicAuth` (for Azure DevOps Repo ID lookup)

---

## ⚙️ Configuration Parameters (config dictionary)

These are the main variables defining the deployment environment:

| Parameter | Description | Example Value |
| :--- | :--- | :--- |
| `workspaceName` | Name of the Fabric workspace to create. | `AutoProject` |
| `fabricCapacityName` | Display name of the Fabric capacity to host the workspace. | `mcaps` |
| `devOpsOrgName` | Azure DevOps organization name. | `codereashu` |
| `devOpsProjectName` | Azure DevOps project containing the repository. | `Automation` |
| `devOpsRepoName` | Name of the Git repository. | `Auto_repo1` |
| `devOpsBranchName` | Git branch to connect the workspace to. | `main` |
| `adminGroups` | Azure AD group display name to assign Admin role. | `Fabric_Automation` |
| `devOpsPAT` | Personal Access Token (PAT) for Azure DevOps. | `<PATToken>` |
| `gitProviderType` | Must be `AzureDevOps` for this setup. | `AzureDevOps` |
| `directoryName` | Sub-directory in the repository for workspace items. | `GitAuto` |

---

## 💻 Code Flow Breakdown (Cell by Cell)

| Section | Cell Logic | Purpose |
| :--- | :--- | :--- |
| **Setup** | `%pip install -q msgraph-sdk` | Installs the required Microsoft Graph SDK package. |
| **Libs** | `import json, requests, sempy.fabric as F, ClientSecretCredential, etc.` | Imports all necessary Python modules. |
| **Key Vault** | `client_id = mssparkutils.credentials.getSecret(...)` | Fetches Service Principal credentials (`client_id`, `client_secret`, `tenant_id`) from Azure Key Vault. |
| **Config** | Defines the main `config` dictionary with all project parameters. | Sets all environment and DevOps variables. |
| **Graph API Auth** | `credential = ClientSecretCredential(...)` and `graph_client = GraphServiceClient(...)` | Initializes the credential and the Microsoft Graph API client for potential future group/user lookups. |
| **Fabric API Auth** | Defines `ServicePrincipalTokenProvider` class. | Helper class to get the Fabric API access token using the Service Principal secret. |
| **Fabric API Init** | `fabric_client = F.FabricRestClient(token_provider=token_provider)` | Initializes the main Fabric client used for all subsequent API calls. |
| **Capacity Details** | `fabric_client.get("/v1/capacities")` | Retrieves the ID of the target Fabric capacity specified in the `config`. |
| **Workspace Creation** | `fabric_client.get("/v1/workspaces")` followed by `fabric_client.post("/v1/workspaces", ...)` if not found. | Checks for the existing workspace and creates a new one on the specified capacity if it doesn't exist. Stores the resulting `workspace_id`. |
| **Role Assignment** | `fabric_client.post(f"/v1/workspaces/{workspace_id}/roleAssignments", ...)` | Assigns the **Admin** role to a specified Azure AD Group ID. |
| **Get DevOps Repo ID** | `requests.get("https://dev.azure.com/.../_apis/git/repositories/{repo}")` | Calls the Azure DevOps API (authenticating with PAT) to retrieve the required internal `repositoryId`. |
| **Connect Git** | `fabric_client.post("/v1/connections", ...)` then `fabric_client.post(f"/v1/workspaces/{workspace_id}/git/connect", ...)` | **1.** Creates a reusable Git connection credential using the Service Principal. **2.** Links the Fabric workspace to the Azure DevOps repository and branch using the new connection ID. |
| **Get Git Connection** | `fabric_client.get(f"/v1/workspaces/{workspace_id}/git/connection")` | Confirms the Git connection status. |
| **Initialize Git** | `fabric_client.post(f"/v1/workspaces/{workspace_id}/git/initializeConnection")` | Initializes the Git connection for the first time. |
| **Get Git Status** | `fabric_client.get(f"/v1/workspaces/{workspace_id}/git/status")` | Retrieves pending changes (new items like Lakehouses) and extracts the `workspace_head` for the commit. |
| **Commit to Git** | `fabric_client.post(f"/v1/workspaces/{workspace_id}/git/commitToGit", ...)` | Executes the first Git commit of all new workspace items. |
