# Kubernetes Cost Prediction Action

Predict the cost of Kubernetes manifests (specs) in CI! Make cost decisions before
merging changes.

This is a [GitHub Action](https://docs.github.com/en/actions), powered by [Kubecost](https://docs.kubecost.com/install-and-configure/install), to make cost predictions for K8s
workloads before they are applied to your cluster. It _does not_ require you to
have Kubecost installed, but will have highly-accurate cost and usage
information for your environment if you do.

In action:

![](./media/actioncomment.png)

## Usage

Add this Action as a step in one of your Actions workflows and point it at a single
YAML file or a directory containing at least one YAML file. Non-YAML files will be
ignored. The YAML files will be interpreted as Kubernetes manifests and a cost
prediction will be run on supported types of [Kubernetes objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/).

> If you aren't familiar with GitHub Actions, check out GitHub's [quickstart](https://docs.github.com/en/actions/quickstart)
> documentation.

### Simple

Below is an excerpt from a workflow written with this Action. This is the
easiest way to add Kubernetes cost prediction to your CI. If you want
a premade workflow file to riff on, check out the "Advanced" example
below.

``` yaml
- name: Run prediction
  id: prediction
  uses: kubecost/cost-prediction-action@v0.1.1
  with:
    # Set this to the path containing your YAML specs. It can be a single
    # YAML file or a directory. The Action will recursively search if this
    # is a directory and process all .yaml/.yml files it finds.
    path: ./repo

# Find existing PR comment, then create or update it with prediction results.
- name: Find existing PR comment
  uses: kubecost/github-actions/find-comment@main
  id: find-comment
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    issue-number: ${{ github.event.pull_request.number }}
    comment-author: 'github-actions[bot]'
    body-includes: '<!-- kubecost-prediction-results -->'
- name: Create or update PR comment with prediction results
  uses: kubecost/github-actions/create-or-update-comment@main
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    issue-number: ${{ github.event.pull_request.number }}
    comment-id: ${{ steps.find-comment.outputs.comment-id }}
    edit-mode: replace
    body: |
      <!-- kubecost-prediction-results -->

      ## Kubecost's total cost prediction for K8s YAML Manifests in this PR

      \```
      ${{ steps.prediction.outputs.PREDICTION_TABLE }}
      \```
```

### Advanced (full workflow)

This is a full Actions workflow file, with commented-out sections and
explanations highlighting advanced features and some complex use-cases.
You can copy-paste this into a file in your `.github/workflows` folder
and start tuning it to use as a live Action on your repo.

``` yaml
name: Predict K8s spec cost
on: [pull_request]

jobs:
  predict-cost:
    runs-on: ubuntu-latest
    steps:
      # Check out the current repo to ./repo
      - uses: actions/checkout@v2
        with:
          path: ./repo
          
      # If using the API support, you need to make sure the Action runner has
      # network access to your instance of Kubecost. This is infra dependent;
      # the following example works with GKE (make sure to set up the necessary
      # secrets).
      # https://docs.github.com/en/actions/guides/deploying-to-google-kubernetes-engine
      # - name: Setup gcloud
      #   uses: google-github-actions/setup-gcloud@v0.2.0
      #   with:
      #     service_account_key: ${{ secrets.GCP_SA_KEY_B64 }}
      #     project_id: ${{ secrets.GKE_PROJECT_ID }}
      # 
      # Get GKE credentials so kubectl has access to the cluster
      # - name: Get GKE credentials
      #   uses: google-github-actions/get-gke-credentials@v0.2.1
      #   with:
      #     cluster_name: ${{ secrets.GKE_CLUSTER }}
      #     location: ${{ secrets.GKE_ZONE }}
      #     credentials: ${{ secrets.GCP_SA_KEY_B64 }}
      #     project_id: ${{ secrets.GKE_PROJECT_ID }}
      # 
      # - name: Forward the kubecost service
      #   run: |
      #     kubectl port-forward --namespace kubecost service/kubecost-cost-analyzer 9090 &
      #     sleep 5
      
      # If you use Helm, you should template the chart and then run the Predict
      # Action targeting the result. Here's an example of how to do that.
      # 
      # - name: Install helm
      #   run: |
      #     curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
      # 
      # - name: Helm template
      #   run: |
      #     helm template RELEASENAME ./repo >> ./templated.yaml

      - name: Run prediction
        id: prediction
        uses: kubecost/cost-prediction-action@v0.1.1
        with:
          log_level: "info"
          # Set this to the path containing your YAML specs. It can be a single
          # YAML file or a directory. The Action will recursively search if this
          # is a directory and process all .yaml/.yml files it finds.
          # 
          # If you use Helm, you probably want to run "helm template", output
          # to a path like ./templated.yaml, and set "path: ./templated.yaml".
          path: ./repo
          # Set this to either:
          # - localhost:9090/model if port forwarding OR
          # - The URL of your Kubecost instance if the runner has direct network
          #   access, e.g. "https://kubecost.example.com:9090/model"
          #
          # If unset, the Action will use Kubecost's default pricing to make a
          # prediction and it will be unable to make
          #
          # kubecost_api_path: "http://localhost:9090/model"

      # Find existing PR comment, then create or update it with prediction results.
      - name: Find existing PR comment
        uses: kubecost/github-actions/find-comment@main
        id: find-comment
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: 'github-actions[bot]'
          body-includes: '<!-- kubecost-prediction-results -->'
      - name: Create or update PR comment with prediction results
        uses: kubecost/github-actions/create-or-update-comment@main
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          issue-number: ${{ github.event.pull_request.number }}
          comment-id: ${{ steps.find-comment.outputs.comment-id }}
          edit-mode: replace
          body: |
            <!-- kubecost-prediction-results -->

            ## Kubecost's total cost prediction for K8s YAML Manifests in this PR

            \```
            ${{ steps.prediction.outputs.PREDICTION_TABLE }}
            \```

      # Alternatively, you can just output the prediction in the Action log.
      # - name: output raw yaml prediction
      #   run: |
      #     echo "${{ steps.prediction.outputs.PREDICTION_TABLE }}"
```

### Budget Validation Feature

The enhanced action now supports validating predicted costs against Kubecost budgets. This allows you to automatically block deployments that would exceed your team's budget allocation.

#### Budget Check Example

```yaml
- name: Run prediction with budget check
  id: prediction
  uses: ./action-enhanced
  with:
    path: ./manifests
    kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
    budget_check_enabled: 'true'
    budget_id: 'team-alpha-budget'
    fail_on_budget_exceeded: 'true'

- name: Display budget status
  run: |
    echo "Budget Status: ${{ steps.prediction.outputs.BUDGET_STATUS }}"
    echo "Budget Remaining: \$${{ steps.prediction.outputs.BUDGET_REMAINING }}"
    echo "Budget Utilization: ${{ steps.prediction.outputs.BUDGET_UTILIZATION_PERCENTAGE }}%"
```

#### Budget Check with Delta Mode

Combine budget validation with delta mode to check only the incremental cost impact:

```yaml
- name: Run prediction with budget check and delta mode
  uses: ./action-enhanced
  with:
    path: ./manifests
    kubecost_api_path: ${{ secrets.KUBECOST_API_PATH }}
    enable_delta_mode: 'true'
    base_ref: 'main'
    budget_check_enabled: 'true'
    budget_id: 'team-alpha-budget'
    fail_on_budget_exceeded: 'true'
```

### Inputs/Outputs

#### Action inputs

| Name | Description | Required? | Default |
|------|-------------|-----------|---------|
| `path` | The path of a file or directory that contains K8s YAML manifests to predict the cost impact of | Yes | |
| `kubecost_api_path` | URL of your Kubecost API. If provided, cost predictions will be a diff based on cost data tracked by your Kubecost instance. If not provied, cost predictions will be a total cost based on Kubecost's default pricing. | No | |
| `log_level` | The log level to run the Action with. Set to `debug` for more granularity or `warn` or `error` for less granularity. | No | `info` |
| `enable_delta_mode` | Enable delta/diff mode to compare costs between base and current branch | No | `false` |
| `base_ref` | Base branch/ref to compare against (e.g., main, master). Required if enable_delta_mode is true | No | `main` |
| `max_cost_increase` | Maximum allowed monthly cost increase in dollars (e.g., "100.00"). Action fails if exceeded | No | |
| `max_cost_percentage` | Maximum allowed cost increase percentage (e.g., "20" for 20%). Action fails if exceeded | No | |
| `fail_on_threshold` | Whether to fail the workflow when cost thresholds are exceeded | No | `true` |
| `budget_check_enabled` | Enable budget validation against Kubecost budgets | No | `false` |
| `budget_id` | ID of the budget in Kubecost to check against (required if budget_check_enabled is true) | No | |
| `fail_on_budget_exceeded` | Whether to fail the workflow when predicted cost would exceed budget | No | `true` |

#### Action outputs

| Name | Description |
|------|-------------|
| `PREDICTION_TABLE` | An ASCII-formatted table of the cost prediction. Best rendered in monospace. |
| `PREDICTION_JSON` | JSON formatted prediction data |
| `TOTAL_MONTHLY_COST` | Total monthly cost prediction |
| `COST_CHANGE` | Cost change amount (delta mode only) |
| `COST_CHANGE_PERCENTAGE` | Cost change percentage (delta mode only) |
| `THRESHOLD_EXCEEDED` | Whether cost thresholds were exceeded |
| `BASE_COST` | Base branch cost (delta mode only) |
| `BUDGET_STATUS` | Status of budget check (WITHIN_BUDGET, EXCEEDS_BUDGET, NOT_CHECKED, ERROR) |
| `BUDGET_REMAINING` | Remaining budget amount in dollars |
| `BUDGET_TOTAL` | Total budget allocation in dollars |
| `BUDGET_CURRENT_SPEND` | Current spend against the budget |
| `BUDGET_UTILIZATION_PERCENTAGE` | Current budget utilization percentage |

## Limitations

The Action currently only supports predicting `.yml`/`.yaml` specs. If you have
specs in other formats, you will have to put them into YAML before running
prediction logic. E.g. for Helm, use `helm template`. More support planned,
please open an issue describing your use case if it is not yet supported.

The Action supports a limited set of Kubernetes object types. We are working
to expand the set of supported types.

The Action does not yet support prediction on only changed files.

The Action does not provide predictions for objects/specs without container
resource requests.

## Development

Source code for the container is mostly closed. Kubecost engineers, visit
`cmd/costpredictionaction` in KCM for more information about development, testing, and releasing.
