# Azure ML Deployment Control Policies

## Overview

This repository contains Azure Policy definitions for comprehensive ML model deployment governance. The policies are designed to work together to provide layered security and compliance controls for Azure Machine Learning deployments.

## Architecture

The governance framework consists of two complementary policies that implement defense-in-depth for ML model deployments:

### 🛡️ Registry Trust Policy
**Purpose**: Foundational security boundary
- **Location**: `./registry-trust-policy/`
- **Function**: Ensures models come only from trusted registries
- **Mode**: Typically **Deny** (hard enforcement)
- **Scope**: Organizational boundaries

### 📋 Model Approval Policy  
**Purpose**: Granular model management
- **Location**: `./model-approval-policy/`
- **Function**: Manages specific model approvals and blocking
- **Mode**: Typically **Audit** (tracking and governance)
- **Scope**: Individual model control

## Repository Structure

```
├── registry-trust-policy/
│   ├── README.md                    # Registry policy documentation
│   ├── policy-rules.json           # Registry validation logic
│   ├── policy-parameters.json      # Registry policy parameters
│   └── assignment-parameters.json  # Default registry configuration
├── model-approval-policy/
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

### 1. Deploy Registry Trust Policy

```bash
cd registry-trust-policy

# Create policy definition
az policy definition create \
  --name "MLRegistryTrustPolicy" \
  --display-name "ML Model Registry Trust Policy" \
  --description "Ensures ML models are deployed only from trusted registries" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"

# Assign policy
az policy assignment create \
  --name "ml-registry-trust" \
  --policy "MLRegistryTrustPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

### 2. Deploy Model Approval Policy

```bash
cd ../model-approval-policy

# Create policy definition
az policy definition create \
  --name "MLModelApprovalPolicy" \
  --display-name "ML Model Approval Policy" \
  --description "Manages explicit approval and blocking of specific ML models" \
  --rules policy-rules.json \
  --params policy-parameters.json \
  --mode "Indexed"

# Assign policy
az policy assignment create \
  --name "ml-model-approval" \
  --policy "MLModelApprovalPolicy" \
  --scope "/subscriptions/{subscription-id}" \
  --params assignment-parameters.json
```

## Recommended Deployment Strategy

### Phase 1: Foundation (Registry Trust)
1. Deploy **Registry Trust Policy** in **Deny** mode
2. Configure trusted registries (azureml, azureml-meta, etc.)
3. Validate all deployments come from approved sources

### Phase 2: Governance (Model Approval)
1. Deploy **Model Approval Policy** in **Audit** mode
2. Monitor model usage patterns for 30 days
3. Identify commonly used models for approval

### Phase 3: Enforcement (Optional)
1. Create model allow lists based on audit data
2. Switch Model Approval Policy to **Deny** mode for strict environments
3. Implement exception processes for new model requests

## Policy Interaction

The policies work together in this sequence:

1. **Registry Trust Policy** evaluates first (if in Deny mode, blocks untrusted registries)
2. **Model Approval Policy** evaluates next (tracks/blocks specific models)
3. Both policies generate audit logs for compliance tracking

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

For detailed information about each policy, see:
- **Registry Trust Policy**: `./registry-trust-policy/README.md`
- **Model Approval Policy**: `./model-approval-policy/README.md`

### Common Issues

1. **Policy Not Triggering**: Check resource type and field paths
2. **Unexpected Blocks**: Verify configuration and wait for propagation
3. **Performance Issues**: Policies only evaluate at deployment time

### Support Commands

```bash
# Check both policy assignments
az policy assignment list --query "[?contains(name, 'ml-')]"

# View compliance across both policies
az policy state list --all --query "[?contains(properties.policyAssignmentName, 'ml-')]"
```
