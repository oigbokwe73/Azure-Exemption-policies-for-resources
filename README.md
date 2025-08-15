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
