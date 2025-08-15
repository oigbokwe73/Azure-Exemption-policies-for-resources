# Azure-Exemption-policies-for-resources

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
