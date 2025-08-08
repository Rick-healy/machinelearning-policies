# Azure ML Deployment Control Policy

## Overview

This Azure Policy controls which machine learning models can be deployed across different Azure ML endpoint types (online, batch, and serverless endpoints). The policy implements a three-tier validation system with deny lists, allow lists, and publisher controls.

## Policy Logic

The policy evaluates deployments in the following priority order:

1. **Deny List (Highest Priority)**: Blocks any model explicitly on the denied models list
2. **Allow List (Medium Priority)**: Permits any model explicitly on the allowed models list
3. **Publisher Validation (Lowest Priority)**: For models not on allow/deny lists:
   - Marketplace models: Validates against allowed publishers list
   - Microsoft-managed models: Blocked unless explicitly allowed

## Files Structure

```
├── azure-ml-deployment-policy-rules.json           # Policy rule definitions
├── azure-ml-deployment-policy-parameters.json      # Policy parameter schema
├── azure-ml-deployment-assignment-parameters.json  # Assignment parameter values
└── README.md                                       # This documentation
```

## Policy Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `allowedModelIds` | Array | List of approved model IDs that can be deployed |
| `deniedModelIds` | Array | List of model IDs explicitly blocked from deployment (highest priority) |
| `allowedPublishers` | Array | List of approved model publishers for marketplace models |
| `effect` | String | Policy effect: `Audit`, `Deny`, or `Disabled` |

## Model ID Format

Model IDs should follow the Azure ML registry format:
```
azureml://registries/{registry-name}/models/{model-name}
```

Examples:
- `azureml://registries/azureml/models/Phi-4`
- `azureml://registries/azureml-meta/models/Llama-3.2-11B-Vision-Instruct`

## Installation

### Prerequisites

- Azure CLI installed and authenticated
- Appropriate permissions to create policy definitions and assignments
- Target subscription or management group identified

### Step 1: Create Policy Definition

```bash
az policy definition create \
  --name "DeploymentControlPolicy" \
  --display-name "ML Deployment Control Policy" \
  --description "Controls which ML models can be deployed across different endpoint types" \
  --rules azure-ml-deployment-policy-rules.json \
  --params azure-ml-deployment-policy-parameters.json \
  --mode "All"
```

### Step 2: Configure Assignment Parameters

Edit `azure-ml-deployment-assignment-parameters.json` with your organization's values:

```json
{
  "allowedPublishers": {
    "value": ["Meta", "Microsoft", "OpenAI"]
  },
  "allowedModelIds": {
    "value": [
      "azureml://registries/azureml/models/Phi-4",
      "azureml://registries/azureml-meta/models/Llama-Guard-3-11B-Vision"
    ]
  },
  "deniedModelIds": {
    "value": [
      "azureml://registries/azureml/models/BlockedModel"
    ]
  },
  "effect": {
    "value": "Audit"
  }
}
```

### Step 3: Create Policy Assignment

#### Subscription Level
```bash
az policy assignment create \
  --name "MLDeploymentControl" \
  --display-name "ML Deployment Control Assignment" \
  --policy "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/policyDefinitions/DeploymentControlPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params azure-ml-deployment-assignment-parameters.json
```

#### Resource Group Level
```bash
az policy assignment create \
  --name "MLDeploymentControl" \
  --display-name "ML Deployment Control Assignment" \
  --policy "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/policyDefinitions/DeploymentControlPolicy" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}" \
  --params azure-ml-deployment-assignment-parameters.json
```

#### Management Group Level
```bash
az policy assignment create \
  --name "MLDeploymentControl" \
  --display-name "ML Deployment Control Assignment" \
  --policy "/providers/Microsoft.Management/managementGroups/{management-group-id}/providers/Microsoft.Authorization/policyDefinitions/DeploymentControlPolicy" \
  --scope "/providers/Microsoft.Management/managementGroups/{management-group-id}" \
  --params azure-ml-deployment-assignment-parameters.json
```

## Policy Scenarios

### Example 1: Microsoft-Managed Model (Compliant)
- **Model**: `azureml://registries/azureml/models/Phi-4`
- **Publisher**: `null` (Microsoft-managed)
- **Result**: ✅ Allowed (on allow list)

### Example 2: Marketplace Model (Compliant)
- **Model**: `azureml://registries/azureml-meta/models/Llama-3.2-11B`
- **Publisher**: `Meta`
- **Result**: ✅ Allowed (publisher on allow list)

### Example 3: Denied Model (Non-Compliant)
- **Model**: `azureml://registries/azureml/models/BlockedModel`
- **Publisher**: Any
- **Result**: ❌ Denied (on deny list - highest priority)

### Example 4: Unauthorized Publisher (Non-Compliant)
- **Model**: `azureml://registries/some-registry/models/SomeModel`
- **Publisher**: `UnknownVendor`
- **Result**: ❌ Denied (publisher not on allow list)

### Example 5: Microsoft Model Not Explicitly Allowed (Non-Compliant)
- **Model**: `azureml://registries/azureml/models/NewModel`
- **Publisher**: `null` (Microsoft-managed)
- **Result**: ❌ Denied (not on allow list, Microsoft models require explicit approval)

## Monitoring and Compliance

### Check Policy Assignment Status
```bash
az policy assignment show \
  --name "MLDeploymentControl" \
  --scope "/subscriptions/{subscription-id}"
```

### View Compliance State (Alternative method due to API issues)
```bash
az rest --method GET \
  --url "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/policyAssignments/MLDeploymentControl?api-version=2024-04-01"
```

### Query Non-Compliant Resources
```bash
az graph query -q "
PolicyResources 
| where type == 'microsoft.policyinsights/policystates' 
  and properties.policyAssignmentName == 'MLDeploymentControl' 
  and properties.complianceState == 'NonCompliant'
| project 
  resourceId = properties.resourceId, 
  complianceState = properties.complianceState,
  reason = properties.complianceReasonCode
"
```

## Updating the Policy

### Update Policy Definition
```bash
az policy definition update \
  --name "DeploymentControlPolicy" \
  --rules azure-ml-deployment-policy-rules.json \
  --params azure-ml-deployment-policy-parameters.json
```

### Update Policy Assignment Parameters
```bash
az policy assignment update \
  --name "MLDeploymentControl" \
  --scope "/subscriptions/{subscription-id}" \
  --params azure-ml-deployment-assignment-parameters.json
```

## Troubleshooting

### Common Issues

1. **API Version Errors**: Some Azure CLI versions may show API version errors when querying policy states. Use the REST API approach shown above as a workaround.

2. **Policy Evaluation Delays**: Azure Policy evaluation can take 10-30 minutes for new assignments, and up to 24 hours for full compliance reporting.

3. **Field Path Validation**: Ensure field paths in the policy rules match exactly with the Azure resource schema.

### Validation Steps

1. **Verify Policy Definition**:
   ```bash
   az policy definition show --name "DeploymentControlPolicy"
   ```

2. **Verify Policy Assignment**:
   ```bash
   az policy assignment list --query "[?name=='MLDeploymentControl']"
   ```

3. **Test with Audit Mode**: Always test with `"effect": "Audit"` before switching to `"effect": "Deny"`

## Security Considerations

- **Principle of Least Privilege**: Start with a restrictive allow list and add models as needed
- **Regular Reviews**: Periodically review allowed models and publishers
- **Monitoring**: Set up alerts for policy violations in production environments
- **Testing**: Always test policy changes in non-production environments first

## Support

For issues related to:
- **Policy Logic**: Review the policy rules in `azure-ml-deployment-policy-rules.json`
- **Azure Policy Service**: Check Azure Service Health and Azure Policy documentation
- **Model Registration**: Consult Azure ML documentation for proper model ID formats
- Access to the target subscription and resource group

### Step 1: Create Policy Definition

```bash
az policy definition create \
  --name "aml-restrict-deployments-test" \
  --display-name "Custom Policy: Azure ML Deployment Control with AssetId and ModelId" \
  --description "Restricts Azure ML model deployments based on publisher, asset IDs, and model IDs" \
  --rules @deployment_control_policy_rules.json \
  --params @deployment_control_policy_parameters.json \
  --mode All
```

### Step 2: Configure Policy Parameters

Edit `policy_assignment_parameters.json` to specify your allowed models:

```json
{
  "allowedPublishers": {
    "value": ["Microsoft", "OpenAI", "Meta"]
  },
  "allowedModelIds": {
    "value": [
      "azureml://registries/azureml-meta/models/Llama-Guard-3-11B-Vision",
      "azureml://registries/azureml/models/Phi-4"
    ]
  },
  "effect": {
    "value": "Audit"
  }
}
```

### Step 3: Assign Policy to Resource Group

```bash
az policy assignment create \
  --name "aml-deployment-control" \
  --policy "aml-restrict-deployments-test" \
  --scope "/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP_NAME}" \
  --params @policy_assignment_parameters.json
```

Replace:
- `{SUBSCRIPTION_ID}` with your Azure subscription ID
- `{RESOURCE_GROUP_NAME}` with your target resource group name

## 🛠️ Configuration Examples

### Example 1: Allow Only Microsoft Models
```json
{
  "allowedPublishers": {"value": ["Microsoft"]},
  "allowedModelIds": {"value": [
    "azureml://registries/azureml/models/Phi-4",
    "azureml://registries/azureml/models/gpt-4o"
  ]},
  "effect": {"value": "Deny"}
}
```

### Example 2: Allow Specific Models Only
```json
{
  "allowedPublishers": {"value": ["Meta"]},
  "allowedModelIds": {"value": [
    "azureml://registries/azureml-meta/models/Llama-Guard-3-11B-Vision",
    "azureml://registries/azureml-meta/models/Llama-3.2-11B-Vision-Instruct"
  ]},
  "effect": {"value": "Audit"}
}
```

### Example 3: Mixed Publishers (Recommended)
```json
{
  "allowedPublishers": {"value": ["Microsoft", "Meta"]},
  "allowedModelIds": {"value": [
    "azureml://registries/azureml/models/Phi-4",
    "azureml://registries/azureml-meta/models/Llama-Guard-3-11B-Vision"
  ]},
  "effect": {"value": "Audit"}
}
```

## 📊 Monitoring and Compliance

### Audit Mode vs Deny Mode
- **Audit Mode** (`"effect": "Audit"`): Logs violations but allows deployments
- **Deny Mode** (`"effect": "Deny"`): Actively blocks non-compliant deployments

### Checking Compliance
View audit events in Azure Activity Log:
```bash
az monitor activity-log list \
  --resource-group {RESOURCE_GROUP_NAME} \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query "[?contains(operationName.value, 'policy')]"
```

## 🔄 Updating the Policy

### Update Parameters Only
1. Edit `policy_assignment_parameters.json`
2. Delete and recreate the assignment:
```bash
az policy assignment delete --name "aml-deployment-control" --scope "/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP_NAME}"
az policy assignment create --name "aml-deployment-control" --policy "aml-restrict-deployments-test" --scope "/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP_NAME}" --params @policy_assignment_parameters.json
```

### Update Policy Logic
1. Edit `deployment_control_policy_rules.json`
2. Delete policy definition, recreate, and reassign:
```bash
az policy assignment delete --name "aml-deployment-control" --scope "/subscriptions/{SUBSCRIPTION_ID}/resourceGroups/{RESOURCE_GROUP_NAME}"
az policy definition delete --name "aml-restrict-deployments-test"
# Then repeat Steps 1-3 from installation
```

## ⚠️ Important Notes

### Policy Limitations
- Custom policies can only access specific Azure Resource Manager field aliases
- Some model metadata may not be available at the control plane level
- Publisher information is primarily available for serverless endpoints

### Best Practices
1. **Start with Audit mode** to understand impact before enforcing
2. **Test thoroughly** with different model types and publishers
3. **Use mixed approach** combining publisher and specific model controls
4. **Monitor compliance** regularly through activity logs
5. **Document exceptions** when allowing specific models outside trusted publishers

### Troubleshooting
- **Policy not working**: Wait 15-30 minutes for propagation
- **Unexpected blocks**: Check exact model ID format in audit logs
- **Missing audit events**: Verify deployment actually occurred and check activity log filters

## 📋 Getting Model Information

### Find Available Models
```bash
# List models from Azure ML catalog
az ml model list --registry-name azureml-meta --max-results 10

# Find specific models by name
az ml model list --registry-name azureml-meta --query "[?contains(name, 'gpt')]"
```

### Get Model Asset IDs
The model IDs follow these patterns (without version):
- **Catalog models**: `azureml://registries/azureml-meta/models/{model-name}`
- **Azure models**: `azureml://registries/azureml/models/{model-name}`
- **Workspace models**: `azureml://locations/{region}/workspaces/{workspace}/models/{model-name}`

**Important**: Use the model ID format without version information, as this matches the actual values evaluated by Azure Policy.
