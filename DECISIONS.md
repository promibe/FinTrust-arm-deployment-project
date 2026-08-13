# Design Decisions Document — FinTrust ARM Deployment

**Deliverable**

For each requirement R1–R8, the ARM feature chosen, why, and one alternative rejected.

---

## R1 — Consistent Deployments Across Regions

**Requirement:** "We deploy the same web server stack in several regions and every one turns out slightly different."

**ARM Feature Used:** Parameters + variables

- All values that vary between regions are parameters (`vmName`, `location`, `vmSize`)
- All values derived from those are variables (`vnetName`, `nsgName`, `publicIPName`, `nicName`, `storageAccountName`)
- Nothing is a literal string anywhere in the `resources` block

**Alternative Rejected:** Portal deployment with a documented checklist
- A checklist relies on the engineer following it perfectly every time
- ARM templates are enforced by the Azure API — deviations are impossible

---

## R2 — No 40-Step Manual Portal Process

**Requirement:** "We don't want engineers clicking through 40 portal screens every time."

**ARM Feature Used:** Single `az deployment group create` command

```bash
az deployment group create \
  --resource-group rg-fintrust-arm \
  --template-file scripts/azuredeploy.json \
  --parameters scripts/azuredeploy.parameters.json
```

One command provisions all 7 resources in the correct order.

**Alternative Rejected:** Bash scripts with individual `az resource create` calls
- Bash scripts are not idempotent — running twice creates duplicate resources or fails
- Order must be managed manually by the script author
- ARM handles idempotency and ordering natively

---

## R3 — Validated Against Microsoft Best Practices

**Requirement:** "Every deployment must be checked against Microsoft's own best practices before it goes near production."

**ARM Feature Used:** ARM Template Test Toolkit (arm-ttk)

```powershell
Test-AzTemplate -TemplatePath .\scripts\azuredeploy.json
# Target: Total: 36  Fail: 0  Pass: 36
```

First run produced 1 failure: `apiVersions Should Be Recent`. API versions `2023-09-01` (networking) and `2023-05-01` (storage) were flagged as 1071 days old — older than the 730-day threshold. Updated to `2024-10-01` and `2025-06-01` respectively. Second run: 36/36.

**Alternative Rejected:** `az deployment group validate` only
- Validation checks JSON syntax and whether Azure's API will accept the deployment
- arm-ttk checks best practices: API version freshness, security posture, naming conventions, no secrets in wrong places
- They answer different questions — both are necessary

---

## R4 — Resources Deploy in the Correct Order

**Requirement:** "Nothing should deploy out of order and break — the network has to exist before anything tries to attach to it."

**ARM Feature Used:** Explicit `dependsOn`

The NIC depends on three resources simultaneously:
```json
"dependsOn": [
  "[resourceId('Microsoft.Network/virtualNetworks', variables('vnetName'))]",
  "[resourceId('Microsoft.Network/networkSecurityGroups', variables('nsgName'))]",
  "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPName'))]"
]
```

The VM depends on the NIC and storage account.
The Custom Script Extension depends on the VM.

**Alternative Rejected:** Relying on implicit references alone
- ARM resolves `resourceId()` references but does not guarantee creation order from references alone
- The parallel deployment engine will start all resources simultaneously unless `dependsOn` is specified
- Removing the NIC's `dependsOn` causes a reproducible "resource not found" failure

---

## R5 — Server Address Returned at Deployment End

**Requirement:** "We need the server's address and name the moment deployment finishes, without hunting through the portal."

**ARM Feature Used:** Outputs block

```json
"outputs": {
  "vmPublicIP": {
    "type": "string",
    "value": "[reference(variables('publicIPName')).ipAddress]"
  }
}
```

The public IP, VM name, VNet name, and storage account name are all returned immediately in the deployment result — both in the CLI output and the portal's Outputs blade.

**Alternative Rejected:** Post-deployment `az network public-ip show`
- Requires a second command after deployment
- The operator must know which resource to query
- Outputs make the information contract explicit and part of the template specification

---

## R6 — Web Server Usable Without Manual Login

**Requirement:** "The web server should be usable the second deployment finishes — nobody logs in by hand to install anything."

**ARM Feature Used:** Custom Script Extension + `install-nginx.sh`

```json
{
  "type": "Microsoft.Compute/virtualMachines/extensions",
  "properties": {
    "publisher": "Microsoft.Azure.Extensions",
    "type": "CustomScript",
    "settings": {
      "fileUris": ["https://raw.githubusercontent.com/promibe/...install-nginx.sh"],
      "commandToExecute": "bash install-nginx.sh"
    }
  }
}
```

The extension pulls `install-nginx.sh` from GitHub at provisioning time and executes it on the VM. The script runs `apt update` and `apt install nginx -y`.

**Alternative Rejected:** cloud-init
- cloud-init is valid and commonly used, but requires embedding configuration in the template as a base64-encoded string
- The Custom Script Extension pulls from an external URL, keeping the shell script as a separate, readable, version-controlled file
- Easier to update the script without modifying the template

---

## R7 — No Hard-Coded Regions or Resource Names

**Requirement:** "No hard-coded region or resource names — we reuse this template in the next region next quarter."

**ARM Feature Used:** Parameterized location, all names derived via `concat()`

```json
"variables": {
  "vnetName": "[concat('vnet-', parameters('vmName'))]",
  "nsgName":  "[concat('nsg-', parameters('vmName'))]"
}
```

The storage account uses `uniqueString(resourceGroup().id)` for global uniqueness without hardcoding.

Deploying to a new region:
```bash
--parameters location="westeurope" vmName="vm-linux-weu"
```
No template edits required.

**Alternative Rejected:** Template copy per region
- Copying the template per region defeats the entire purpose — inconsistencies creep back in as engineers modify their local copies
- One template, parameterized for all regions, is the IaC principle

---

## R8 — Proof the Template Will Work Before Spending Money

**Requirement:** "We want proof, before we spend money, that the template will actually deploy successfully."

**ARM Feature Used:** `az deployment group validate` + `az deployment group what-if`

- **Validate:** checks that the template JSON is syntactically valid and the Azure API will accept the deployment request. Returns immediately, no resources created.
- **What-if:** shows exactly which resources will be created (`+`), modified (`~`), or deleted (`-`). Contacts Azure, resolves references, returns a diff. No resources created.

**Practical difference:** Validate confirms the request is acceptable. What-if shows the actual delta against existing state — critical when redeploying to a resource group that already has resources.

**Alternative Rejected:** Test deploy to a scratch resource group
- Creates real resources and incurs cost
- Still doesn't preview delta against the target environment
- What-if is purpose-built for this use case at zero cost

