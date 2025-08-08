# Azure ML Model Deployment Control Policies

## Overview

This repository contains Azure Policy definitions for controlling **Azure ML model deployments** specifically. These policies govern which models can be deployed to Azure ML endpoints (online, batch, and serverless) and from which sources they can be deployed.

**Scope**: Model deployment operations only - these policies do not control model training, data access, compute resources, or other ML lifecycle activities.

## Architecture

The governance framework consists of two complementary policies that implement defense-in-depth for **ML model deployment operations**:

### 🛡️ Model Deployment Registry Trust Policy
**Purpose**: Controls which registries can be used as sources for model deployments
- **Location**: `./model-deployment-registry-trust-policy/`
- **Function**: Ensures deployed models come only from trusted registries
- **Mode**: Typically **Deny** (hard enforcement)
- **Scope**: Organizational deployment source boundaries

### 📋 Model Deployment Approval Policy  
**Purpose**: Controls which specific models can be deployed
- **Location**: `./model-deployment-approval-policy/`
- **Function**: Manages specific model deployment approvals and blocking
- **Mode**: Typically **Audit** (tracking and governance)
- **Scope**: Individual model deployment control

## Repository Structure

```
├── model-deployment-registry-trust-policy/
│   ├── README.md                    # Registry policy documentation
│   ├── policy-rules.json           # Registry validation logic
│   ├── policy-parameters.json      # Registry policy parameters
│   └── assignment-parameters.json  # Default registry configuration
├── model-deployment-approval-policy/
│   ├── README.md                    # Model policy documentation
│   ├── policy-rules.json           # Model approval/denial logic
│   ├── policy-parameters.json      # Model policy parameters
│   └── assignment-parameters.json  # Default model configuration
└── README.md                       # This overview document
```

## Quick Start

### Prerequisites
- Azure CLI installed and authenticated
- Policy definition creation permissions
- Target subscription/resource group identified

### 1. Deploy Model Deployment Registry Trust Policy

```bash
cd model-deployment-registry-trust-policy

# Create policy definition
az policy definition create \
  --name "MLModelDeploymentRegistryTrustPolicy" \
  --display-name "ML Model Deployment Registry Trust Policy" \
  --description "Controls which registries can be used as sources for Azure ML model deployments" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"

# Assign policy
az policy assignment create \
  --name "ml-model-deployment-registry-trust" \
  --policy "MLModelDeploymentRegistryTrustPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

### 2. Deploy Model Deployment Approval Policy

```bash
cd ../model-deployment-approval-policy

# Create policy definition
az policy definition create \
  --name "MLModelDeploymentApprovalPolicy" \
  --display-name "ML Model Deployment Approval Policy" \
  --description "Controls which specific models can be deployed to Azure ML endpoints" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"

# Assign policy
az policy assignment create \
  --name "ml-model-deployment-approval" \
  --policy "MLModelDeploymentApprovalPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

## Recommended Deployment Strategy

### Phase 1: Foundation (Deployment Registry Control)

1. Deploy **Model Deployment Registry Trust Policy** in **Deny** mode
2. Configure trusted registries (azureml, azureml-meta, etc.)
3. Validate all model deployments come from approved registry sources

### Phase 2: Governance (Deployment Model Control)

1. Deploy **Model Deployment Approval Policy** in **Audit** mode
2. Monitor model deployment patterns for 30 days
3. Identify commonly deployed models for approval

### Phase 3: Enforcement (Optional)

1. Create model deployment allow lists based on audit data
2. Switch Model Deployment Approval Policy to **Deny** mode for strict environments
3. Implement exception processes for new model deployment requests

## Policy Interaction

The policies work together in this sequence for **model deployment operations**:

1. **Model Deployment Registry Trust Policy** evaluates first (if in Deny mode, blocks deployments from untrusted registries)
2. **Model Deployment Approval Policy** evaluates next (tracks/blocks specific model deployments)
3. Both policies generate audit logs for deployment compliance tracking

**Note**: These policies only trigger during model deployment operations to Azure ML endpoints. They do not affect model training, data access, or other ML lifecycle activities.

## Best Practices

### Policy Management

1. **Version Control**: Store policy files in git with proper versioning
2. **Testing**: Test policy changes in non-production environments first
3. **Rollback Plan**: Maintain ability to quickly disable policies if needed
4. **Documentation**: Keep clear records of policy decisions and changes

### Governance Process

1. **Regular Reviews**: Monthly review of allowed registries and models
2. **Exception Handling**: Clear process for emergency model approvals
3. **Stakeholder Communication**: Keep ML teams informed of policy changes
4. **Compliance Reporting**: Regular reports to security and compliance teams

## Support and Troubleshooting

For detailed information about each model deployment policy, see:

- **Model Deployment Registry Trust Policy**: `./model-deployment-registry-trust-policy/README.md`
- **Model Deployment Approval Policy**: `./model-deployment-approval-policy/README.md`

### Common Issues

1. **Policy Not Triggering**: Check resource type and field paths for Azure ML serverless endpoints
2. **Unexpected Blocks**: Verify configuration and wait for propagation (deployment policies only affect endpoint deployments)
3. **Performance Issues**: Policies only evaluate during model deployment operations, not ongoing model usage

### Support Commands

```bash
# Check both model deployment policy assignments
az policy assignment list --query "[?contains(name, 'ml-model-deployment')]"

# View compliance across both model deployment policies
az policy state list --all --query "[?contains(properties.policyAssignmentName, 'ml-model-deployment')]"
```
