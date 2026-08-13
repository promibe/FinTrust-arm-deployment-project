# Handover Document — FinTrust ARM Deployment

**Deliverables**
**For:** FinTrust Ltd IT Manager
**From:** Promise Ibediogwu Ekele, Azure Cloud Administrator

---

## What Was Delivered

A single ARM template (`scripts/azuredeploy.json`) that deploys a complete, production-ready web server stack — VNet, NSG, Public IP, NIC, Storage Account, Ubuntu Linux VM, and Nginx — in a single command, to any Azure region, with zero manual post-deployment steps.

| Resource | Purpose |
|----------|---------|
| Virtual Network (`vnet-{name}`) | Isolates the VM in a private address space (10.0.0.0/16) |
| Network Security Group (`nsg-{name}`) | Allows inbound HTTP (80) and SSH (22); blocks everything else |
| Public IP (`pip-{name}`) | Gives the VM a reachable internet address |
| Network Interface (`nic-{name}`) | Connects the VM to the network |
| Storage Account (`st{hash}`) | Boot diagnostics storage |
| Virtual Machine (`{name}`) | Ubuntu Linux, Standard_DS1_v2 |
| VM Extension (`{name}/install-nginx`) | Installs and starts Nginx automatically at provision time |

---

## How to Deploy to a New Region

**Prerequisites:**
1. Azure CLI installed — `az --version`
2. Logged in — `az login`
3. Resource group created in the target region:
   ```bash
   az group create --name rg-fintrust-{region} --location {region}
   ```

**Step 1 — Clone the repo:**
```bash
git clone https://github.com/promibe/fintrust-arm-deployment.git
cd fintrust-arm-deployment
```

**Step 2 — Review the parameters file:**
```bash
# Edit scripts/azuredeploy.parameters.json
# Change vmName and location only — everything else is derived
```

```json
{
  "parameters": {
    "vmName":             { "value": "vm-linux-westeu" },
    "adminUsername":      { "value": "azureuser" },
    "adminPasswordOrKey": { "value": "YOUR-SECURE-PASSWORD" },
    "location":           { "value": "westeurope" }
  }
}
```

> **Security:** Never commit a real password to source control.
> Pass it at deploy time instead:
> ```bash
> az deployment group create ... --parameters adminPasswordOrKey="YourPass123!"
> ```

**Step 3 — Preview what will be created:**
```bash
az deployment group what-if \
  --resource-group rg-fintrust-westeu \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```
All resources should show `+` (will be created). Review before proceeding.

**Step 4 — Deploy:**
```bash
az deployment group create \
  --resource-group rg-fintrust-westeu \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```

**Step 5 — Get the outputs:**
The CLI returns the outputs when deployment completes:
```
vmPublicIP:         20.x.x.x        ← browse to this
vmName:             vm-linux-westeu
vnetName:           vnet-vm-linux-westeu
storageAccountName: st{hash}
```

**Step 6 — Verify:**
Open a browser → navigate to the `vmPublicIP` value → you should see **Welcome to nginx!**

---

## What the Outputs Mean

| Output | What It Is | When You Need It |
|--------|-----------|-----------------|
| `vmPublicIP` | The VM's public internet address | Browse here to verify Nginx is running |
| `vmName` | The name of the deployed VM | To SSH in or manage the VM in the portal |
| `vnetName` | The virtual network name | To peer networks or add subnets |
| `storageAccountName` | The auto-generated storage account | Boot diagnostics — rarely needed directly |

---

## How to Re-Run arm-ttk

If the template is modified, re-run the ARM Template Test Toolkit before the next deployment:

```powershell
# PowerShell (Windows or PowerShell Core)
# Install arm-ttk (first time only):
Install-Module -Name arm-ttk -Force

# Run from the project root:
Test-AzTemplate -TemplatePath .\scripts\azuredeploy.json

# What to expect:
# Total: 36  Fail: 0  Pass: 36
```

**If a test fails:** Read the error message — it names the rule and the line number. Common fixes:
- `apiVersions Should Be Recent` → update the `apiVersion` field to within the last 2 years
- `CommandToExecute Must Use ProtectedSettings For Secrets` → move `commandToExecute` from `settings` to `protectedSettings`
- `Secure String Parameters Cannot Have Default` → remove any `defaultValue` from `secureString` parameters

---

## How to Update the Nginx Script

The `install-nginx.sh` script lives in the `scripts/` folder and is fetched from GitHub by the Custom Script Extension at deployment time.

To update what gets installed:
1. Edit `scripts/install-nginx.sh`
2. Commit and push to `main` branch on GitHub
3. The next deployment will automatically use the updated script

The GitHub raw URL in the template:
```
https://raw.githubusercontent.com/promibe/FinTrust-arm-deployment-project/refs/heads/main/scripts/install-nginx.sh
```

> **Important:** The script must be publicly accessible at this URL for the Custom Script Extension to fetch it. If you move the repo to private, you'll need to generate a SAS token for the script file or host it in Azure Blob Storage instead.

---

## Teardown

When a regional deployment is no longer needed:

```bash
# Deletes ALL resources in the resource group — irreversible
az group delete --name rg-fintrust-{region} --yes --no-wait
```

`--no-wait` returns immediately; deletion runs in the background (typically 2–5 minutes).

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Nginx page doesn't load | NSG or Custom Script Extension issue | Check VM extension status in portal → VM → Extensions |
| Deployment fails: "resource not found" | `dependsOn` issue in modified template | Re-run arm-ttk, check NIC dependsOn block |
| arm-ttk: `apiVersions Should Be Recent` | API version > 730 days old | Update `apiVersion` to latest listed by the toolkit |
| Storage account name conflict | `uniqueString` collision (extremely rare) | Change the resource group name to generate a different hash |
| Custom Script Extension: failed | Script fetch error or syntax error | Check `https://raw.githubusercontent.com/...install-nginx.sh` is accessible |

