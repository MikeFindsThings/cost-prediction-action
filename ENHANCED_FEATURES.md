# Enhanced Features: Delta Mode, Cost Thresholds, and Budget Validation

This document describes the enhanced features added to the Kubecost Cost Prediction Action.

## Overview

The enhanced action adds three major features:
1. **Delta/Diff Mode** - Compare costs between branches to see the impact of changes
2. **Cost Threshold Checks** - Automatically fail workflows when cost increases exceed configured limits
3. **Budget Validation** - Validate predicted costs against Kubecost budgets to prevent budget overruns

## Delta/Diff Mode

### What is Delta Mode?

Delta mode compares the cost prediction of your current branch against a base branch (typically `main` or `master`). This shows you exactly how much your changes will increase or decrease costs.

### How to Enable

```yaml
- name: Run prediction with delta mode
  uses: ./action-enhanced
  with:
    path: ./manifests
    enable_delta_mode: "true"
    base_ref: "main"  # Branch to compare against
```

### Outputs

When delta mode is enabled, you get additional outputs:

- `BASE_COST` - The cost from the base branch
- `COST_CHANGE` - The absolute cost difference (e.g., "+15.50" or "-5.25")
- `COST_CHANGE_PERCENTAGE` - The percentage change (e.g., "+12.5" or "-8.3")

### Example Output

```
Current Cost: $125.50
Base Cost (main): $110.00
Cost Change: +$15.50 (+14.09%) - INCREASE
```

## Cost Threshold Checks

### What are Cost Thresholds?

Cost thresholds allow you to automatically fail a workflow if cost increases exceed acceptable limits. This prevents accidentally merging expensive changes.

### Configuration Options

```yaml
- name: Run prediction with thresholds
  uses: ./action-enhanced
  with:
    path: ./manifests
    enable_delta_mode: "true"
    base_ref: "main"
    
    # Threshold settings
    max_cost_increase: "50.00"      # Fail if cost increases by more than $50/month
    max_cost_percentage: "20"       # Fail if cost increases by more than 20%
    fail_on_threshold: "true"       # Set to "false" to only warn
```

### Threshold Behavior

- **Both thresholds are checked** - If either threshold is exceeded, the action will fail (if `fail_on_threshold` is true)
- **Absolute threshold** (`max_cost_increase`) - Checks the dollar amount increase
- **Percentage threshold** (`max_cost_percentage`) - Checks the percentage increase
- **Warning mode** - Set `fail_on_threshold: "false"` to only warn without failing

### Example Scenarios

#### Scenario 1: Cost increase within limits
```
Current: $110.00
Base: $100.00
Change: +$10.00 (+10%)

Thresholds:
- max_cost_increase: $50.00 ✓ PASS
- max_cost_percentage: 20% ✓ PASS

Result: Workflow continues
```

#### Scenario 2: Absolute threshold exceeded
```
Current: $160.00
Base: $100.00
Change: +$60.00 (+60%)

Thresholds:
- max_cost_increase: $50.00 ✗ FAIL
- max_cost_percentage: 20% ✗ FAIL

Result: Workflow fails (if fail_on_threshold is true)
```

#### Scenario 3: Percentage threshold exceeded
```
Current: $130.00
Base: $100.00
Change: +$30.00 (+30%)

Thresholds:
- max_cost_increase: $50.00 ✓ PASS
- max_cost_percentage: 20% ✗ FAIL

Result: Workflow fails (if fail_on_threshold is true)
```

## Complete Input Reference

### Required Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `path` | Path to workload file or directory | (required) |

### Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `kubecost_api_path` | URL of Kubecost API | (none) |
| `log_level` | Log level (debug, info, warn, error) | `info` |
| `enable_delta_mode` | Enable cost comparison with base branch | `false` |
| `base_ref` | Base branch for comparison | `main` |
| `max_cost_increase` | Max allowed cost increase in dollars | (none) |
| `max_cost_percentage` | Max allowed cost increase percentage | (none) |
| `fail_on_threshold` | Fail workflow when threshold exceeded | `true` |

## Complete Output Reference

### Standard Outputs

| Output | Description |
|--------|-------------|
| `PREDICTION_TABLE` | ASCII table of cost prediction |
| `PREDICTION_JSON` | JSON formatted prediction data |
| `TOTAL_MONTHLY_COST` | Total monthly cost |
| `THRESHOLD_EXCEEDED` | Whether thresholds were exceeded (true/false) |

### Delta Mode Outputs

| Output | Description |
|--------|-------------|
| `BASE_COST` | Cost from base branch |
| `COST_CHANGE` | Absolute cost difference |
| `COST_CHANGE_PERCENTAGE` | Percentage cost change |

## Usage Examples

### Example 1: Basic Delta Mode

```yaml
name: Cost Check
on: [pull_request]

jobs:
  cost-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Required for delta mode
      
      - name: Check cost impact
        uses: ./action-enhanced
        with:
          path: ./k8s
          enable_delta_mode: "true"
          base_ref: "main"
```

### Example 2: Strict Cost Governance

```yaml
name: Cost Governance
on: [pull_request]

jobs:
  enforce-cost-limits:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Enforce cost limits
        uses: ./action-enhanced
        with:
          path: ./k8s
          enable_delta_mode: "true"
          base_ref: "main"
          max_cost_increase: "100.00"
          max_cost_percentage: "15"
          fail_on_threshold: "true"
```

### Example 3: Warning Only (No Failure)

```yaml
name: Cost Monitoring
on: [pull_request]

jobs:
  monitor-costs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Monitor cost changes
        uses: ./action-enhanced
        with:
          path: ./k8s
          enable_delta_mode: "true"
          base_ref: "main"
          max_cost_increase: "50.00"
          max_cost_percentage: "10"
          fail_on_threshold: "false"  # Only warn
```

### Example 4: PR Comment with Delta

```yaml
name: Cost Prediction
on: [pull_request]

jobs:
  predict:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Run prediction
        id: prediction
        uses: ./action-enhanced
        with:
          path: ./k8s
          enable_delta_mode: "true"
          base_ref: "main"
      
      - name: Comment on PR
        uses: actions/github-script@v6
        with:
          script: |
            const output = `## Cost Impact Analysis
            
            **Current Cost:** $\`${{ steps.prediction.outputs.TOTAL_MONTHLY_COST }}\`
            **Base Cost:** $\`${{ steps.prediction.outputs.BASE_COST }}\`
            **Change:** \`${{ steps.prediction.outputs.COST_CHANGE }}\` (\`${{ steps.prediction.outputs.COST_CHANGE_PERCENTAGE }}%\`)
            
            <details>
            <summary>Detailed Breakdown</summary>
            
            \`\`\`
            ${{ steps.prediction.outputs.PREDICTION_TABLE }}
            \`\`\`
            </details>`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

## Implementation Notes

### Requirements for Delta Mode

1. **Full Git History** - Use `fetch-depth: 0` in checkout action
2. **Base Branch Access** - The base branch must exist in the repository
3. **Consistent Paths** - The path must exist in both current and base branches

### Cost Parsing

The action parses cost information from the prediction table output. The parsing logic looks for:
- Lines containing "total" (case-insensitive)
- Numeric values in the format `XX.XX`

If your prediction table has a different format, the parsing may need adjustment.

### Threshold Logic

- Thresholds are only checked when `enable_delta_mode` is true
- If no thresholds are configured, no checks are performed
- Both absolute and percentage thresholds are checked independently
- If either threshold is exceeded, `THRESHOLD_EXCEEDED` is set to true

## Troubleshooting

### Delta mode not working

**Problem:** Base branch prediction fails
**Solution:** Ensure `fetch-depth: 0` is set in checkout action

### Thresholds not triggering

**Problem:** Workflow doesn't fail despite exceeding thresholds
**Solution:** Check that `fail_on_threshold` is set to "true" (not true without quotes)

### Cost parsing errors

**Problem:** Costs show as 0.00
**Solution:** Check that the prediction table format matches expected format with "total" line

## Budget Validation

### What is Budget Validation?

Budget validation checks predicted workload costs against Kubecost budgets to ensure deployments stay within allocated budget limits. This prevents cost overruns before changes are merged.

### How to Enable

```yaml
- name: Run prediction with budget check
  uses: ./action-enhanced
  with:
    path: ./manifests
    kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
    budget_check_enabled: "true"
    budget_id: "team-alpha-budget"
    fail_on_budget_exceeded: "true"
```

### Budget API Integration

The action queries the Kubecost Budgets API to retrieve:
- Total budget allocation (`spendLimit`)
- Current spend (`currentSpend`)
- Budget window and interval
- Budget name and ID

API Endpoint: `GET {kubecost_api_path}/budget?id={budget_id}`

### Budget Check Logic

1. **Query Budget** - Retrieves budget information from Kubecost
2. **Get Predicted Cost** - Uses total cost or delta cost (if delta mode enabled)
3. **Calculate Impact** - Adds predicted cost to current spend
4. **Validate** - Checks if new total would exceed budget limit
5. **Report** - Sets outputs and fails workflow if configured

### Budget Outputs

| Output | Description |
|--------|-------------|
| `BUDGET_STATUS` | Status: WITHIN_BUDGET, EXCEEDS_BUDGET, NOT_CHECKED, ERROR |
| `BUDGET_REMAINING` | Remaining budget in dollars |
| `BUDGET_TOTAL` | Total budget allocation |
| `BUDGET_CURRENT_SPEND` | Current spend against budget |
| `BUDGET_UTILIZATION_PERCENTAGE` | Current utilization percentage |

### Budget Check with Delta Mode

When combined with delta mode, budget validation checks only the incremental cost impact:

```yaml
- name: Budget check with delta mode
  uses: ./action-enhanced
  with:
    path: ./manifests
    kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
    enable_delta_mode: "true"
    base_ref: "main"
    budget_check_enabled: "true"
    budget_id: "team-alpha-budget"
```

This is recommended for most use cases as it validates only the cost change, not the total workload cost.

### Example Scenarios

#### Scenario 1: Within Budget
```
Budget Total: $5,000.00
Current Spend: $3,200.00
Remaining: $1,800.00
Predicted Cost: $500.00
New Total: $3,700.00

Result: ✓ WITHIN_BUDGET - Workflow continues
```

#### Scenario 2: Exceeds Budget
```
Budget Total: $5,000.00
Current Spend: $4,500.00
Remaining: $500.00
Predicted Cost: $800.00
New Total: $5,300.00

Result: ✗ EXCEEDS_BUDGET - Workflow fails (if fail_on_budget_exceeded is true)
Error: Budget would be exceeded by $300.00
```

#### Scenario 3: Cost Reduction
```
Budget Total: $5,000.00
Current Spend: $4,500.00
Predicted Cost: -$200.00 (reduction)

Result: ✓ WITHIN_BUDGET - Cost reductions always pass
```

### Warning Mode

Set `fail_on_budget_exceeded: "false"` to only warn without failing:

```yaml
- name: Budget check - warning only
  uses: ./action-enhanced
  with:
    path: ./manifests
    budget_check_enabled: "true"
    budget_id: "team-budget"
    fail_on_budget_exceeded: "false"  # Only warn
```

### Combined Checks

Budget validation works alongside cost threshold checks:

```yaml
- name: Complete cost governance
  uses: ./action-enhanced
  with:
    path: ./manifests
    kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
    enable_delta_mode: "true"
    base_ref: "main"
    
    # Threshold checks
    max_cost_increase: "100.00"
    max_cost_percentage: "20"
    fail_on_threshold: "true"
    
    # Budget checks
    budget_check_enabled: "true"
    budget_id: "team-alpha-budget"
    fail_on_budget_exceeded: "true"
```

Both checks are independent - the workflow fails if either check fails.

### Error Handling

The budget check handles various error scenarios:

| Scenario | Behavior |
|----------|----------|
| Budget not found | Fails with error message |
| API unavailable | Fails with connectivity error |
| Invalid budget ID | Fails with validation error |
| Missing kubecost_api_path | Skips check with warning |
| Budget check disabled | Skips check silently |

### Budget Check Requirements

1. **Kubecost API Access** - `kubecost_api_path` must be configured
2. **Valid Budget ID** - Budget must exist in Kubecost
3. **API Connectivity** - Action runner must have network access to Kubecost
4. **Budget API Enabled** - Kubecost instance must have budgets feature enabled

## Complete Input Reference (Updated)

### Required Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `path` | Path to workload file or directory | (required) |

### Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `kubecost_api_path` | URL of Kubecost API | (none) |
| `log_level` | Log level (debug, info, warn, error) | `info` |
| `enable_delta_mode` | Enable cost comparison with base branch | `false` |
| `base_ref` | Base branch for comparison | `main` |
| `max_cost_increase` | Max allowed cost increase in dollars | (none) |
| `max_cost_percentage` | Max allowed cost increase percentage | (none) |
| `fail_on_threshold` | Fail workflow when threshold exceeded | `true` |
| `budget_check_enabled` | Enable budget validation | `false` |
| `budget_id` | Kubecost budget ID to check against | (none) |
| `fail_on_budget_exceeded` | Fail when budget would be exceeded | `true` |

## Complete Output Reference (Updated)

### Standard Outputs

| Output | Description |
|--------|-------------|
| `PREDICTION_TABLE` | ASCII table of cost prediction |
| `PREDICTION_JSON` | JSON formatted prediction data |
| `TOTAL_MONTHLY_COST` | Total monthly cost |
| `THRESHOLD_EXCEEDED` | Whether thresholds were exceeded (true/false) |

### Delta Mode Outputs

| Output | Description |
|--------|-------------|
| `BASE_COST` | Cost from base branch |
| `COST_CHANGE` | Absolute cost difference |
| `COST_CHANGE_PERCENTAGE` | Percentage cost change |

### Budget Validation Outputs

| Output | Description |
|--------|-------------|
| `BUDGET_STATUS` | WITHIN_BUDGET, EXCEEDS_BUDGET, NOT_CHECKED, ERROR |
| `BUDGET_REMAINING` | Remaining budget in dollars |
| `BUDGET_TOTAL` | Total budget allocation |
| `BUDGET_CURRENT_SPEND` | Current spend against budget |
| `BUDGET_UTILIZATION_PERCENTAGE` | Budget utilization percentage |

## Usage Examples (Updated)

### Example 5: Budget Validation

```yaml
name: Budget Check
on: [pull_request]

jobs:
  validate-budget:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Check against team budget
        id: budget-check
        uses: ./action-enhanced
        with:
          path: ./k8s
          kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
          budget_check_enabled: "true"
          budget_id: "team-alpha-budget"
          fail_on_budget_exceeded: "true"
      
      - name: Report budget status
        run: |
          echo "Budget Status: ${{ steps.budget-check.outputs.BUDGET_STATUS }}"
          echo "Budget Total: \$${{ steps.budget-check.outputs.BUDGET_TOTAL }}"
          echo "Current Spend: \$${{ steps.budget-check.outputs.BUDGET_CURRENT_SPEND }}"
          echo "Remaining: \$${{ steps.budget-check.outputs.BUDGET_REMAINING }}"
          echo "Utilization: ${{ steps.budget-check.outputs.BUDGET_UTILIZATION_PERCENTAGE }}%"
```

### Example 6: Complete Cost Governance

```yaml
name: Complete Cost Governance
on: [pull_request]

jobs:
  enforce-all-limits:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Comprehensive cost validation
        id: cost-check
        uses: ./action-enhanced
        with:
          path: ./k8s
          kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
          
          # Delta mode
          enable_delta_mode: "true"
          base_ref: "main"
          
          # Cost thresholds
          max_cost_increase: "100.00"
          max_cost_percentage: "15"
          fail_on_threshold: "true"
          
          # Budget validation
          budget_check_enabled: "true"
          budget_id: "team-alpha-budget"
          fail_on_budget_exceeded: "true"
      
      - name: Create detailed PR comment
        uses: actions/github-script@v6
        if: always()
        with:
          script: |
            const output = `## 💰 Cost Governance Report
            
            ### Cost Impact
            - **Current Cost:** \`\$${{ steps.cost-check.outputs.TOTAL_MONTHLY_COST }}\`
            - **Base Cost:** \`\$${{ steps.cost-check.outputs.BASE_COST }}\`
            - **Change:** \`${{ steps.cost-check.outputs.COST_CHANGE }}\` (\`${{ steps.cost-check.outputs.COST_CHANGE_PERCENTAGE }}%\`)
            - **Threshold Check:** ${{ steps.cost-check.outputs.THRESHOLD_EXCEEDED == 'true' && '❌ EXCEEDED' || '✅ PASSED' }}
            
            ### Budget Status
            - **Budget:** \`${{ steps.cost-check.outputs.BUDGET_TOTAL }}\`
            - **Current Spend:** \`\$${{ steps.cost-check.outputs.BUDGET_CURRENT_SPEND }}\`
            - **Remaining:** \`\$${{ steps.cost-check.outputs.BUDGET_REMAINING }}\`
            - **Utilization:** \`${{ steps.cost-check.outputs.BUDGET_UTILIZATION_PERCENTAGE }}%\`
            - **Status:** ${{ steps.cost-check.outputs.BUDGET_STATUS == 'WITHIN_BUDGET' && '✅ WITHIN BUDGET' || '❌ EXCEEDS BUDGET' }}
            
            <details>
            <summary>Detailed Breakdown</summary>
            
            \`\`\`
            ${{ steps.cost-check.outputs.PREDICTION_TABLE }}
            \`\`\`
            </details>`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

## Troubleshooting (Updated)

### Budget validation not working

**Problem:** Budget check is skipped
**Solution:** Ensure `budget_check_enabled` is set to "true" and `kubecost_api_path` is configured

### Budget not found error

**Problem:** API returns budget not found
**Solution:** Verify the budget ID exists in Kubecost and is spelled correctly

### API connectivity issues

**Problem:** Cannot reach Kubecost API
**Solution:** Ensure the action runner has network access to Kubecost (may require port forwarding or VPN)

## Future Enhancements

Potential improvements for future versions:
- Support for multiple base branches
- Cost breakdown by namespace/resource type
- Historical cost tracking
- Integration with cost allocation labels
- Support for custom cost parsing rules
- Support for env-label for environment specific pricing logic
- Multi-cluster prediction support
- Budget forecasting and trend analysis
- Support for multiple budget checks in single workflow
- Integration with budget actions/notifications
