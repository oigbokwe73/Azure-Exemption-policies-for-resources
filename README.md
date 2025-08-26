
Here’s a **super-simple PowerShell + Azure CLI** script that hits the **ARM management REST “Tags at scope”** endpoint to **MERGE** (or **REPLACE**) tags on **any scope** (resource, resource group, or subscription).

> Save as `Update-Tags-Simple.ps1`

```powershell
# ---- Simple: update tags via ARM REST (MERGE by default) ----
# Optional: Managed Identity login
# az login --identity           # or: az login --identity --username <client-id-guid>

# SCOPE can be a full resourceId, an RG scope, or a subscription scope:
#   Resource:      /subscriptions/<sub>/resourceGroups/<rg>/providers/<RP>/<type>/<name>
#   ResourceGroup: /subscriptions/<sub>/resourceGroups/<rg>
#   Subscription:  /subscriptions/<sub>
$SCOPE = "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<name>"

# Define your tags (or read from a file: $tags = Get-Content .\tags.json -Raw | ConvertFrom-Json)
$tags = @{
  env        = "prod"
  owner      = "obinna"
  costcenter = "1234"
}

# Choose MERGE (preserve others) or REPLACE (overwrite all)
$operation = "Merge"   # or "Replace"

# Build request
$uri  = "https://management.azure.com$SCOPE/providers/Microsoft.Resources/tags/default?api-version=2021-04-01"
$body = @{
  properties = @{
    operation = $operation
    tags      = $tags
  }
} | ConvertTo-Json -Depth 5

# PATCH tags
az rest --method patch --uri $uri --headers "Content-Type=application/json" --body $body

# Verify (shows tags object)
az rest --method get --uri $uri --query properties.tags -o json
```

### Quick usage

* **MERGE** tags (recommended): just run the script after setting `$SCOPE`, `$tags`, `$operation="Merge"`.
* **REPLACE** all tags: set `$operation="Replace"` (this overwrites existing tags).

That’s it—clean and tiny.

Here are the **Azure CLI** ways to take **JSON tags** and update a resource’s tags.

### Option A — Replace all tags with JSON (no merge)

From a file:

```bash
# tags.json => {"env":"prod","owner":"obinna","costcenter":"1234"}
az resource update --ids "<RESOURCE_ID>" --set tags=@tags.json
# or
az resource update -g "<RG>" -n "<NAME>" -r "Microsoft.Storage/storageAccounts" --set tags=@tags.json
```

Inline JSON:

```bash
az resource update --ids "<RESOURCE_ID>" --set tags='{"env":"prod","owner":"obinna"}'
```

---

### Option B — Merge JSON into existing tags (incremental)

**Bash + jq**:

```bash
# Merge keys from tags.json without removing other existing tags
az resource tag --ids "<RESOURCE_ID>" --is-incremental --tags $(jq -r 'to_entries|.[]|"\(.key)=\(.value|tostring)"' tags.json)
```

**PowerShell (no jq)**:

```powershell
$tags = Get-Content .\tags.json -Raw | ConvertFrom-Json
$kv = @(); $tags.psobject.Properties | ForEach-Object { $kv += "$($_.Name)=$($_.Value)" }
az resource tag --ids "<RESOURCE_ID>" --is-incremental --tags $kv
```

---

### (Optional) Managed Identity login first

```bash
az login --identity                     # system-assigned
# az login --identity --username <client-id-guid>   # user-assigned
```

Tip: verify after:

```bash
az resource show --ids "<RESOURCE_ID>" --query tags -o json
```

Here’s a tiny PowerShell helper that converts a **JSON tags object** (e.g. from Azure) into a **Hashtable**.

```powershell
function Convert-JsonTagsToHashtable {
  [CmdletBinding(DefaultParameterSetName='FromString')]
  param(
    [Parameter(Mandatory=$true, ParameterSetName='FromString', ValueFromPipeline=$true)]
    [string]$Json,

    [Parameter(Mandatory=$true, ParameterSetName='FromFile')]
    [string]$Path
  )

  if ($PSCmdlet.ParameterSetName -eq 'FromFile') {
    $obj = Get-Content -Raw -Path $Path | ConvertFrom-Json
  } else {
    $obj = $Json | ConvertFrom-Json
  }

  if ($null -eq $obj) { return @{} }

  $ht = @{}
  foreach ($p in $obj.PSObject.Properties) {
    $val = $p.Value
    if ($null -eq $val)       { $ht[$p.Name] = "" }
    elseif ($val -is [string]){ $ht[$p.Name] = $val }
    else                      { $ht[$p.Name] = ($val | ConvertTo-Json -Compress) }  # coerce non-strings
  }
  return $ht
}
```

### Examples

```powershell
# From a JSON string
$tags = Convert-JsonTagsToHashtable -Json '{"env":"prod","owner":"obinna","costcenter":"1234"}'
$tags

# From a file
$tags = Convert-JsonTagsToHashtable -Path .\tags.json
$tags

# Pipeline
Get-Content .\tags.json -Raw | Convert-JsonTagsToHashtable
```

### Super-simple one-liner (file → Hashtable)

```powershell
$ht = @{}; (Get-Content .\tags.json -Raw | ConvertFrom-Json).PSObject.Properties |
  % { $ht[$_.Name] = [string]$_.Value }; $ht
```

# Azure-Exemption-policies-for-resources

If you have **only** the resource name and subscription ID, you’ll need to also know the resource **type** (and ideally the provider) to get the full Azure Resource ID, because resource names are not globally unique — multiple resources in different resource groups can share the same name.

Here’s the most direct Azure CLI way:

```bash
# Variables
RESOURCE_NAME="<RESOURCE_NAME>"
SUBSCRIPTION_ID="<SUBSCRIPTION_ID>"
RESOURCE_TYPE="<PROVIDER_NAMESPACE>/<RESOURCE_TYPE>"   # e.g. Microsoft.Storage/storageAccounts

# Get the resource ID
az resource list \
  --subscription $SUBSCRIPTION_ID \
  --name $RESOURCE_NAME \
  --resource-type $RESOURCE_TYPE \
  --query "[0].id" \
  -o tsv
```

**Example for a storage account:**

```bash
RESOURCE_NAME="mystorageacct"
SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
RESOURCE_TYPE="Microsoft.Storage/storageAccounts"

az resource list \
  --subscription $SUBSCRIPTION_ID \
  --name $RESOURCE_NAME \
  --resource-type $RESOURCE_TYPE \
  --query "[0].id" \
  -o tsv
```

**If you don’t know the type**, you can search across all resource types in the subscription and then filter by name:

```bash
az resource list \
  --subscription $SUBSCRIPTION_ID \
  --query "[?name=='$RESOURCE_NAME'].id" \
  -o tsv
```

⚠ **Note:** If the same name exists in multiple resource groups, the above will return multiple IDs — you’ll have to choose the correct one.

---

If you want, I can make you a **one-liner** that takes just the resource name and subscription, auto-finds the ID, and passes it directly into your **policy exemption creation** command. That would make your workflow faster.


In Azure CLI, you can create a **policy exemption** using `az policy exemption create`, and set an expiration date with the `--expires-on` parameter.

Here’s the full example to create an exemption for a specific resource:

```bash
# Variables
EXEMPTION_NAME="myExemption"
POLICY_ASSIGNMENT_NAME="myPolicyAssignment"
RESOURCE_ID="/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP_NAME>/providers/<RESOURCE_PROVIDER>/<RESOURCE_TYPE>/<RESOURCE_NAME>"
EXEMPTION_CATEGORY="Waiver"   # Can be "Waiver" or "Mitigated"
EXPIRATION_DATE="2025-12-31T23:59:59Z" # UTC time in ISO 8601 format

# Create policy exemption
az policy exemption create \
  --name $EXEMPTION_NAME \
  --policy-assignment $POLICY_ASSIGNMENT_NAME \
  --scope $RESOURCE_ID \
  --exemption-category $EXEMPTION_CATEGORY \
  --expires-on $EXPIRATION_DATE \
  --description "Temporary exemption for resource until end of 2025"
```

**Key Notes:**

* `--scope` must be the full Azure resource ID of the resource you want to exempt.
* `--policy-assignment` is the **name** (or full ID) of the policy assignment the exemption applies to.
* `--exemption-category` can be:

  * `Waiver` – You’re permanently exempting the resource from evaluation.
  * `Mitigated` – You’re exempting it because a compensating control exists.
* `--expires-on` must be in **UTC ISO 8601 format** (`YYYY-MM-DDTHH:MM:SSZ`).
