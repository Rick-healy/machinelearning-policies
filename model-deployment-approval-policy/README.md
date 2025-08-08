# Azure ML Model Deployment Approval Policy

## Overview

This Azure Policy manages explicit approval and blocking of specific ML models for deployment operations. It provides granular control over which individual models can be deployed as Azure ML serverless endpoints, regardless of their registry source.

## Policy Purpose

This policy is designed to govern **model deployment operations only**. It provides:

- **Deployment Authorization**: Manage approval of specific models for serverless endpoint deployment
- **Security Controls**: Explicitly block problematic or deprecated models from being deployed
- **Compliance Tracking**: Audit and track deployment of specific models for governance
- **Change Management**: Control model deployments through explicit approval processes

**Scope Clarification**: This policy only affects model deployment operations. It does not control model training, data access, workspace creation, or other Azure ML lifecycle activities.

## How It Works

The policy maintains two lists for deployment control:
1. **Deny List** (Highest Priority): Models explicitly blocked from deployment
2. **Allow List** (When configured): Models explicitly approved for deployment

If a model appears on the deny list, deployment is blocked. If an allow list is configured and not empty, only models on that list are permitted for deployment.

## Policy Logic

```
IF deployment is a serverless endpoint
AND modelId exists
THEN:
  IF modelId is in deniedModelIds → BLOCK DEPLOYMENT
  ELSE IF allowedModelIds is not empty AND modelId is NOT in allowedModelIds → BLOCK DEPLOYMENT
  ELSE → ALLOW DEPLOYMENT
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowedModelIds` | Array | `[]` | Specific models approved for deployment |
| `deniedModelIds` | Array | `[]` | Models explicitly blocked from deployment (highest priority) |
| `effect` | String | `"Audit"` | Policy enforcement mode for deployment operations |

## Model ID Format

Model IDs must use the exact Azure ML registry format:
```
azureml://registries/{registry-name}/models/{model-name}
```

Examples:
- `azureml://registries/azureml/models/Phi-4`
- `azureml://registries/azureml-meta/models/Llama-3.2-11B-Vision-Instruct`

## Operating Modes

### Mode 1: Deny List Only (Recommended for Most Organizations)
Configure only `deniedModelIds`, leave `allowedModelIds` empty.
- **Behavior**: Block specific problematic models, allow all others
- **Use Case**: Block known problematic models while allowing innovation

```json
{
  "allowedModelIds": {"value": []},
  "deniedModelIds": {"value": [
    "azureml://registries/azureml/models/deprecated-model",
    "azureml://registries/azureml/models/security-issue-model"
  ]}
}
```

### Mode 2: Explicit Allow List (High Security Environments)
Configure specific models in `allowedModelIds`.
- **Behavior**: Only explicitly approved models are allowed
- **Use Case**: Highly regulated environments requiring explicit approval

```json
{
  "allowedModelIds": {"value": [
    "azureml://registries/azureml/models/Phi-4",
    "azureml://registries/azureml-meta/models/Llama-Guard-3-11B-Vision"
  ]},
  "deniedModelIds": {"value": []}
}
```

### Mode 3: Combined (Most Flexible)
Use both allow and deny lists for maximum control.
- **Behavior**: Allow specific models, but still block denied ones
- **Use Case**: Phased rollouts with emergency blocking capability

## Installation

### Step 1: Create Policy Definition

```bash
az policy definition create \
  --name "MLModelDeploymentApprovalPolicy" \
  --display-name "ML Model Deployment Approval Policy" \
  --description "Manages explicit approval and blocking of specific ML models for deployment operations" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"
```

### Step 2: Configure Parameters

Edit `assignment-parameters.json` based on your deployment governance model:

```json
{
  "allowedModelIds": {
    "value": [
      "azureml://registries/azureml/models/Phi-4",
      "azureml://registries/azureml-meta/models/Llama-3.2-11B-Vision-Instruct"
    ]
  },
  "deniedModelIds": {
    "value": [
      "azureml://registries/azureml/models/Phi-4-multimodal-instruct"
    ]
  },
  "effect": {
    "value": "Audit"
  }
}
```

### Step 3: Assign Policy

```bash
az policy assignment create \
  --name "ml-model-deployment-approval" \
  --display-name "ML Model Deployment Approval Policy" \
  --policy "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/policyDefinitions/MLModelDeploymentApprovalPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

## Policy Effects

### Recommended: "Audit" Mode
- **Purpose**: Track model usage and compliance without blocking
- **Impact**: Logs violations but allows deployments to proceed
- **Use Case**: Most environments where visibility and governance tracking is key

### Alternative: "Deny" Mode
- **Purpose**: Strict enforcement of model approval processes
- **Impact**: Blocks deployments of non-approved models immediately
- **Use Case**: High-security or highly regulated environments

## Monitoring

### Track Model Usage
```bash
az policy state list \
  --policy-assignment "ml-model-approval" \
  --filter "ComplianceState eq 'NonCompliant'" \
  --query "[].{Model: properties.resourceId, Reason: properties.complianceReasonCode}"
```

### View Denied Models
```bash
az monitor activity-log list \
  --resource-group {resource-group} \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query "[?contains(operationName.value, 'policy') && properties.statusCode=='Forbidden']"
```

## Example Scenarios

### ✅ Scenario 1: Model on Allow List
- **Model**: `azureml://registries/azureml/models/Phi-4`
- **Allow List**: Contains this model
- **Deny List**: Empty
- **Result**: Allowed

### ❌ Scenario 2: Model on Deny List
- **Model**: `azureml://registries/azureml/models/Phi-4-multimodal-instruct`
- **Allow List**: Contains this model
- **Deny List**: Contains this model
- **Result**: Blocked (deny list takes priority)

### ❌ Scenario 3: Model Not on Allow List
- **Model**: `azureml://registries/azureml/models/NewModel`
- **Allow List**: Configured but doesn't contain this model
- **Deny List**: Empty
- **Result**: Blocked (not explicitly approved)

### ✅ Scenario 4: Deny List Only Mode
- **Model**: `azureml://registries/azureml/models/SomeModel`
- **Allow List**: Empty (not configured)
- **Deny List**: Doesn't contain this model
- **Result**: Allowed (no restrictions when allow list is empty)

## Model Lifecycle Management

### Adding New Models
1. **Evaluation Phase**: Deploy in Audit mode to test model performance
2. **Approval Process**: Internal review and security assessment
3. **Allow List Update**: Add model to `allowedModelIds` parameter
4. **Policy Update**: Update assignment with new parameters

### Blocking Problematic Models
1. **Issue Identification**: Security vulnerability or performance issue discovered
2. **Emergency Block**: Add model to `deniedModelIds` immediately
3. **Impact Assessment**: Review current deployments using the model
4. **Communication**: Notify teams about the blocking and alternatives

### Model Deprecation
1. **Deprecation Notice**: Announce model deprecation timeline
2. **Migration Support**: Provide guidance for moving to newer models
3. **Enforcement**: Add deprecated model to deny list after migration period

## Best Practices

1. **Start with Audit**: Begin with Audit mode to understand current model usage
2. **Gradual Rollout**: Implement allow lists gradually, not all at once
3. **Regular Review**: Periodically review and update model lists
4. **Documentation**: Maintain clear records of why models are approved/denied
5. **Emergency Process**: Have a fast-track process for blocking problematic models
6. **Communication**: Keep development teams informed of model approval status

## Integration with Registry Trust Policy

This policy works best when deployed alongside the Registry Trust Policy:

1. **Registry Policy** (Deny mode): Enforces organizational boundaries for model sources
2. **Model Policy** (Audit mode): Tracks and manages specific model approvals

This combination provides:
- **Hard boundaries** on trusted sources (Registry Policy)
- **Flexible tracking** of specific model usage (Model Policy)
- **Defense in depth** for comprehensive model governance

## Troubleshooting

### Common Issues

1. **Policy Not Evaluating**
   - Verify model ID format matches exactly
   - Check that the deployment is using serverless endpoints
   - Ensure modelId field exists in the deployment

2. **Unexpected Blocks**
   - Check if model is on deny list (highest priority)
   - Verify allow list is configured correctly
   - Confirm exact model ID spelling and format

3. **Case Sensitivity**
   - Model IDs are case-sensitive
   - Ensure exact match including registry name and model name

### Validation Commands

```bash
# Check current policy assignment
az policy assignment show --name "ml-model-approval"

# List all policy states for this assignment
az policy state list --policy-assignment "ml-model-approval"

# Trigger immediate compliance scan
az policy state trigger-scan --resource-group {rg-name}
```

## Advanced Configuration

### Dynamic Allow Lists
For organizations with frequent model deployment changes, consider:
- Using Azure Key Vault for storing model deployment lists
- Implementing automated approval workflows for model deployments
- Integrating with CI/CD pipelines for model deployment governance

### Custom Compliance Reporting
Create custom queries for specific deployment compliance reporting:

```bash
# Models blocked by deny list during deployment attempts
az graph query -q "
PolicyResources
| where type == 'microsoft.policyinsights/policystates'
| where properties.policyAssignmentName == 'ml-model-deployment-approval'
| where properties.complianceReasonCode contains 'denied'
"

# Most commonly attempted non-compliant model deployments
az graph query -q "
PolicyResources
| where type == 'microsoft.policyinsights/policystates'
| where properties.policyAssignmentName == 'ml-model-deployment-approval'
| where properties.complianceState == 'NonCompliant'
| summarize count() by ModelId = tostring(properties.resourceDetails.modelId)
| order by count_ desc
"
```

## Architecture Notes

This policy works in conjunction with the **ML Model Deployment Registry Trust Policy** to provide defense-in-depth for model deployments:

- **Registry Trust Policy** (see `../model-deployment-registry-trust-policy/`): Controls which registries can be used as sources for model deployments
- **Model Approval Policy**: Controls which specific models can be deployed from any allowed registry

Both policies focus exclusively on deployment operations and do not affect other Azure ML activities like training, data processing, or workspace management.
