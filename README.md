# FinTrust ARM Deployment — Automated Web Server Infrastructure

**Engagement ref:** FT-AZ-003 | **Role:** Azure Cloud Administrator | **Track:** AZ-104 Project Series #3

> One ARM template. One parameters file. One command. Seven resources. Zero manual steps.

Automated, consistent, repeatable web server deployment for FinTrust Ltd using Azure Resource Manager Templates — replacing a day of manual portal clicking with a single CLI command that provisions the full network + compute stack and installs Nginx automatically.

---

## The Problem

FinTrust was standing up identical web server environments in multiple Azure regions by hand. Every region came out slightly different. A VM once failed mid-deployment because it tried to attach to a NIC that hadn't been created yet. A junior engineer hard-coded `eastus` into a script reused for South Africa. There were no documented steps and no validation before deployments hit production.

---

## Architecture

```
Parameters (vmName, location, adminUsername, adminPasswordOrKey, vmSize)
        │
        ▼
Variables (vnetName, nsgName, publicIPName, nicName, storageAccountName, subnetName)
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Resource Group                               │
│                                                                 │
│  VNet ──────────────────────────────────┐                       │
│  NSG  ──────────────────────────────── ─┼─→ NIC ──→ VM          │
│  Public IP ─────────────────────────────┘         │             │
│  Storage Account ─────────────────────────────────┘             │
│                                         VM ──→ Custom Script    │
│                                                Extension        │
│                                                (Nginx)          │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
Outputs (vmName, vmPublicIP, vnetName, storageAccountName)
```

---

## Requirements → ARM Feature Mapping

| Ref | Requirement | ARM Feature Used | Alternative Rejected |
|-----|-------------|-----------------|---------------------|
| R1 | Consistent deployments across regions | Parameters + variables — no hardcoded values | Portal deployment — no repeatability guarantee |
| R2 | No 40-step manual portal process | Single `az deployment group create` command | Bash scripts — not idempotent, order-dependent |
| R3 | Validated against Microsoft best practices | ARM Template Test Toolkit (arm-ttk) | Manual review — misses structural issues |
| R4 | Resources deploy in the correct order | Explicit `dependsOn` on NIC (3 deps) and VM (2 deps) | Implicit references alone — insufficient for parallel engine |
| R5 | Server address returned at deployment end | `outputs` block — vmPublicIP, vmName, vnetName, storageAccountName | Portal browsing — slow, error-prone |
| R6 | Nginx installed without manual SSH | Custom Script Extension — runs `install-nginx.sh` at provision time | Post-deployment script — requires separate step and login |
| R7 | No hard-coded regions or names | All names derived from `parameters('vmName')` via `concat()` | Variables with literal values — breaks reuse |
| R8 | Proof of correctness before spending money | `az deployment group what-if` + `az deployment group validate` | Deploy and hope — no preview |

---

## Repo Structure

```
fintrust-arm-deployment/
├── README.md
├── scripts/
│   ├── azuredeploy.json              # Main ARM template (7 resources)
│   ├── azuredeploy.parameters.json   # Parameters file — fill before deploying
│   └── install-nginx.sh              # Custom Script Extension — installs Nginx
└── evidence/
    ├── Shot_1_-_Parameter_block.png
    ├── Shot_2_-_Full_skeleton_without_resource_block_content.png
    ├── Shot_3_-_NIC_resource_block.png
    ├── Shot_4_-_resources.png
    ├── Shot_4a_-_resources.png
    ├── Shot_4b_-_resources.png
    ├── Shot_5_-_what-if.png
    ├── Shot_6_-_failed_ARM_Test_Took_kit.png
    ├── Shot_6a_-_failed_ARM_Test_Took_kit.png
    ├── Shot_7_-_passed_ARM_Test_Took_kit.png
    ├── Shot_9_-_Deployment_via_CLI.png
    ├── Shot_9a_-_Deployment_via_CLI.png
    ├── Shot_9c_-_Deployment_template_via_CLI_with_Vm_extension.png
    ├── Shot_10_-_default_nginx_browser.png
    ├── Shot_11_-_Deployment_blade_overview.png
    └── Shot_11_-_Deployment_via_CLI.png
```

---

## Template Overview

### Resources (7 total, deployed in parallel where safe)

| # | Resource Type | Name | dependsOn |
|---|--------------|------|-----------|
| 0 | `Microsoft.Network/virtualNetworks` | `vnet-{vmName}` | None |
| 1 | `Microsoft.Network/networkSecurityGroups` | `nsg-{vmName}` | None |
| 2 | `Microsoft.Network/publicIPAddresses` | `pip-{vmName}` | None |
| 3 | `Microsoft.Network/networkInterfaces` | `nic-{vmName}` | VNet, NSG, Public IP |
| 4 | `Microsoft.Storage/storageAccounts` | `st{uniqueString}` | None |
| 5 | `Microsoft.Compute/virtualMachines` | `{vmName}` | NIC, Storage |
| 6 | `Microsoft.Compute/virtualMachines/extensions` | `{vmName}/install-nginx` | VM |

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `vmName` | string | — | VM name — also seeds all other resource names |
| `location` | string | `[resourceGroup().location]` | Region — never hardcoded |
| `adminUsername` | string | — | VM admin username |
| `adminPasswordOrKey` | secureString | — | Password or SSH key — never logged or stored in plain text |
| `vmSize` | string | `Standard_DS1_v2` | VM SKU |

### Outputs

| Output | Value |
|--------|-------|
| `vmName` | Name of the deployed VM |
| `vmPublicIP` | Public IP address — browse to this for Nginx |
| `vnetName` | Virtual network name |
| `storageAccountName` | Storage account name (auto-generated, unique) |

---

## How to Deploy

### Prerequisites

- Azure CLI installed and authenticated (`az login`)
- Resource group already created
- arm-ttk installed (PowerShell module) — optional but recommended

### Step 1 — Update the parameters file

Edit `scripts/azuredeploy.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName":             { "value": "vm-linux-demo" },
    "adminUsername":      { "value": "azureuser" },
    "adminPasswordOrKey": { "value": "YourSecurePassword123!" },
    "location":           { "value": "southafricanorth" }
  }
}
```

> **Security note:** Never commit `adminPasswordOrKey` with a real value to source control. Use Azure Key Vault reference or pass it at deploy time:
> ```bash
> --parameters adminPasswordOrKey="$(read -s p && echo $p)"
> ```

### Step 2 — Validate (optional but recommended)

```bash
az deployment group validate \
  --resource-group rg-fintrust-arm \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```

### Step 3 — What-if preview (show what will change)

```bash
az deployment group what-if \
  --resource-group rg-fintrust-arm \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```

### Step 4 — Run arm-ttk (PowerShell)

```powershell
# Install arm-ttk (first time only)
Install-Module -Name arm-ttk -Force

# Run from project root
Test-AzTemplate -TemplatePath .\scripts\azuredeploy.json

# Expected: Total: 36  Fail: 0  Pass: 36
```

### Step 5 — Deploy

```bash
az deployment group create \
  --resource-group rg-fintrust-arm \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```

Deployment takes approximately 5–8 minutes. When complete, the outputs block returns:

```json
{
  "vmName":             "vm-linux-demo",
  "vmPublicIP":         "20.164.38.116",
  "vnetName":           "vnet-vm-linux-demo",
  "storageAccountName": "st6vgc74ras4nci"
}
```

### Step 6 — Verify Nginx

Open a browser and navigate to the `vmPublicIP` value from the outputs.
You should see the **Welcome to nginx!** page — installed automatically
by the Custom Script Extension with no manual login required.

### Step 7 — Deploy to a new region

Change only the `location` parameter:

```bash
az deployment group create \
  --resource-group rg-fintrust-westeurope \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json \
  --parameters location="westeurope" vmName="vm-linux-weu"
```

Everything else is derived. Same template, different region, consistent result.

---

## Teardown

```bash
az group delete --name rg-fintrust-arm --yes --no-wait
```

---

## Key Design Decisions

**Why `secureString` for the admin credential?**
A `secureString` parameter is never logged, never returned in deployment history, and never visible in the portal after deployment. A plain `string` parameter would appear in every deployment log Azure keeps. Even if the password is strong, logging it defeats its purpose.

**Why explicit `dependsOn` on the NIC?**
ARM's deployment engine runs resources in parallel by default. The NIC references the VNet subnet, NSG, and public IP by `resourceId()`. ARM resolves those references at runtime — but the referenced resources must exist first. Without explicit `dependsOn`, ARM might start the NIC before any of its three dependencies finish, producing a "resource not found" error. The three `dependsOn` entries make the ordering deterministic.

**Why the Custom Script Extension over post-deployment scripting?**
A Custom Script Extension runs as part of the ARM deployment — it appears in the deployment details as resource #6, depends on the VM, and its success or failure is tracked in the deployment result. A post-deployment script is a separate step outside the template, breaks the "one command, complete environment" guarantee, and is easily forgotten in multi-region rollouts.

**Why `uniqueString(resourceGroup().id)` for the storage account name?**
Storage account names must be globally unique across all of Azure. `uniqueString()` generates a deterministic 13-character hash from the resource group ID — same resource group always produces the same name, but different resource groups produce different names. This avoids name collision without hardcoding.

---

## arm-ttk Results

| Run | Total | Fail | Pass | Key Issue |
|-----|-------|------|------|-----------|
| First | 36 | 1 | 35 | `apiVersions Should Be Recent` — API versions 1071 days old |
| After fix | 36 | 0 | 36 | All tests passing ✅ |

**What the failing test was protecting against:** Older API versions may lack security features introduced in newer releases, and Azure occasionally deprecates functionality in old versions without warning. The toolkit flags anything older than 730 days as a risk signal — not necessarily broken, but worth updating before production use.

---

## Future Improvements

- **Azure Key Vault reference** — replace the plaintext `adminPasswordOrKey` parameter with a Key Vault secret reference so the credential never appears in any command or file
- **Bicep conversion** — rewrite in Bicep for cleaner syntax, native module support, and better IDE tooling
- **HTTPS/TLS** — add Let's Encrypt certificate or Azure Front Door for HTTPS on the Nginx endpoint
- **Availability Zones** — deploy the VM across zones for higher SLA (99.99% vs 99.9%)
- **Idempotency testing** — run the template twice against the same resource group to verify it updates correctly rather than failing on existing resources

---

## Related Projects

- **Project 1 — FinTrust Secure Document Storage** (Azure Blob Storage, RBAC, SAS, stored access policies): [[GitHub link]](https://github.com/promibe/fintrust-secure-storage)
- **Project 2 — TradeCore Secure Infrastructure** (VNets, NSGs, Azure Bastion, VMs, Azure Backup, Monitor): [[GitHub link]](https://github.com/promibe/TradeCore-Secure-Infrastructure)

---

## Full Write-Up

📝 Medium article: https://medium.com/@promiseibediogwu1/one-template-one-command-seven-resources-zero-manual-steps-255318c02af6
🎬 YouTube walkthrough: 

---

*Fictitious company scenario modelled on real operational patterns. All names, figures, and identifiers are invented.*

*Promise Ibediogwu Ekele — https://github.com/promibe*
