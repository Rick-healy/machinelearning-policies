# ML Model Deployment Registry Trust Policy

## Overview

This Azure Policy controls which registries can be used as sources for **Azure ML model deployments**. It acts as a foundational security control by validating the source registry of models during deployment operations to Azure ML endpoints.

**Scope**: Model deployment operations only - this policy does not control model training, data access, or other ML lifecycle activities.

## Policy Purpose

- **Deployment Security Boundary**: Establishes organizational trust boundaries for model deployment sources
- **Supply Chain Security**: Prevents deployment of models from untrusted or unknown registries
- **Baseline Deployment Control**: Works in conjunction with the Model Deployment Approval Policy for complete deployment governance

## How It Works

The policy evaluates the `modelId` field during serverless endpoint deployment operations and checks if the model comes from an approved registry pattern. If the model registry is not in the approved list, the deployment is blocked or audited based on the policy effect.

**Trigger**: Only activates during Azure ML model deployment operations to endpoints (online, batch, serverless).

## Policy Logic

```
IF operation is a model deployment to serverless endpoint
AND modelId exists
AND modelId does NOT contain any approved registry pattern
THEN apply policy effect (Deny/Audit)
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowedRegistries` | Array | `["azureml://registries/azureml/", "azureml://registries/azureml-meta/"]` | Registry patterns allowed for model deployments |
| `effect` | String | `"Deny"` | Policy enforcement mode for deployment operations |

## Registry Patterns

Registry patterns should include the trailing slash and follow this format:
```
azureml://registries/{registry-name}/
```

### Common Registry Patterns

- `azureml://registries/azureml/` - Microsoft's built-in models
- `azureml://registries/azureml-meta/` - Meta's models  
- `azureml://registries/azureml-openai/` - OpenAI models
- `azureml://registries/{your-org}/` - Custom organizational registries

## Installation

### Step 1: Create Policy Definition

```bash
az policy definition create \
  --name "MLModelDeploymentRegistryTrustPolicy" \
  --display-name "ML Model Deployment Registry Trust Policy" \
  --description "Controls which registries can be used as sources for Azure ML model deployments" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"
```

### Step 2: Configure Parameters

Edit `assignment-parameters.json` with your organization's trusted registries for model deployments:

```json
{
  "allowedRegistries": {
    "value": [
      "azureml://registries/azureml/",
      "azureml://registries/azureml-meta/",
      "azureml://registries/your-org-registry/"
    ]
  },
  "effect": {
    "value": "Deny"
  }
}
```

### Step 3: Assign Policy

```bash
az policy assignment create \
  --name "ml-model-deployment-registry-trust" \
  --display-name "ML Model Deployment Registry Trust Policy" \
  --policy "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/policyDefinitions/MLModelDeploymentRegistryTrustPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

## Policy Effects

### Recommended: "Deny" Mode
- **Purpose**: Hard enforcement of organizational boundaries
- **Impact**: Blocks deployments from untrusted registries immediately
- **Use Case**: Production environments where registry trust is critical

### Alternative: "Audit" Mode  
- **Purpose**: Monitor registry usage without blocking
- **Impact**: Logs violations but allows deployments to proceed
- **Use Case**: Testing phase or environments where visibility is more important than enforcement

## Monitoring

### Check Compliance
```bash
az policy state list \
  --policy-assignment "ml-registry-trust" \
  --filter "ComplianceState eq 'NonCompliant'"
```

### View Blocked Deployments
```bash
az monitor activity-log list \
  --resource-group {resource-group} \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query "[?contains(operationName.value, 'policy') && contains(operationName.value, 'deny')]"
```

## Example Scenarios

### ✅ Compliant: Microsoft Model
- **Model**: `azureml://registries/azureml/models/Phi-4`
- **Registry**: `azureml` 
- **Result**: Allowed (registry in approved list)

### ✅ Compliant: Meta Model  
- **Model**: `azureml://registries/azureml-meta/models/Llama-3.2-11B`
- **Registry**: `azureml-meta`
- **Result**: Allowed (registry in approved list)

### ❌ Non-Compliant: Unknown Registry
- **Model**: `azureml://registries/unknown-vendor/models/SomeModel`
- **Registry**: `unknown-vendor`
- **Result**: Blocked (registry not in approved list)

## Best Practices

1. **Start Restrictive**: Begin with only essential registries and add as needed
2. **Use Deny Mode**: Registry trust should be strictly enforced in production
3. **Regular Review**: Periodically audit approved registries
4. **Document Exceptions**: Maintain clear documentation for any registry additions
5. **Coordinate with Model Policy**: Ensure this works well with your Model Approval Policy

## Troubleshooting

### Common Issues

1. **Policy Not Triggering**
   - Verify the resource type matches serverless endpoints
   - Check that modelId field exists in the deployment

2. **Unexpected Blocks**
   - Verify registry patterns include trailing slashes
   - Check exact registry name formatting

3. **Pattern Matching**
   - Registry patterns use `contains()` matching
   - Ensure patterns are specific enough to avoid false positives

### Validation Commands

```bash
# Verify policy definition
az policy definition show --name "MLRegistryTrustPolicy"

# Check assignment status
az policy assignment list --query "[?name=='ml-registry-trust']"

# Test with specific model ID
az policy state trigger-scan --resource-group {rg-name}
```

## Security Considerations

- **Defense in Depth**: This policy provides the foundational layer of model governance
- **Registry Compromise**: Even trusted registries can be compromised; combine with model approval policies
- **Custom Registries**: Carefully vet any custom or third-party registries before adding to allowed list
- **Monitoring**: Set up alerts for any policy violations to detect potential security issues

## Related Policies

This policy works best when combined with:
- **Model Deployment Approval Policy**: For specific model allow/deny lists (see `../model-deployment-approval-policy/`)
- **Compute Resource Policies**: For controlling where models can be deployed
- **Network Access Policies**: For controlling model endpoint accessibility
