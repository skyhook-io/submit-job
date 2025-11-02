# Submit Job

Submits/triggers Kubernetes CronJobs or Argo Workflows. This action handles the actual execution of jobs, creating Job resources from CronJobs or submitting Argo Workflows from templates.

## Usage

```yaml
- name: Submit job
  uses: skyhook-io/submit-job@v1
  id: submit
  with:
    job_type: kubernetes-cronjob
    resource_name: my-cronjob
    namespace: production

- name: Check created job
  run: |
    echo "Created: ${{ steps.submit.outputs.created_name }}"
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `job_type` | Type of job | ✅ | - |
| `resource_name` | Name of the resource to trigger | ✅ | - |
| `namespace` | Kubernetes namespace | ✅ | - |
| `parameters` | JSON object of parameters (Argo only) | ❌ | `''` |
| `argo_version` | Argo CLI version to install | ❌ | `3.5.3` |

### Valid Job Types
- `kubernetes-cronjob` - Creates a Job from a CronJob
- `argo-workflow` - Submits a Workflow from a WorkflowTemplate
- `argo-cronworkflow` - Submits a Workflow from a CronWorkflow

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `created_name` | Name of the created Job or Workflow | `my-job-manual-1234567890` |
| `created_kind` | Kind of created resource | `job` or `workflow` |

## Examples

### Kubernetes CronJob

```yaml
- name: Trigger CronJob
  uses: skyhook-io/submit-job@v1
  with:
    job_type: kubernetes-cronjob
    resource_name: nightly-backup
    namespace: production
```

### Argo WorkflowTemplate

```yaml
- name: Submit workflow
  uses: skyhook-io/submit-job@v1
  with:
    job_type: argo-workflow
    resource_name: data-processing-template
    namespace: workflows
    parameters: '{"input_file":"s3://bucket/data.csv","output_path":"s3://bucket/results/"}'
```

### Argo CronWorkflow

```yaml
- name: Submit from CronWorkflow
  uses: skyhook-io/submit-job@v1
  with:
    job_type: argo-cronworkflow
    resource_name: scheduled-report
    namespace: workflows
    parameters: '{"report_date":"2024-01-15"}'
```

### Complete Workflow with Resolve

```yaml
name: Execute Job

on:
  workflow_dispatch:
    inputs:
      job_name:
        description: "Logical job name"
        required: true
      namespace:
        description: "Namespace"
        required: true
      job_type:
        description: "Job type"
        required: true
      parameters:
        description: "Parameters (JSON)"
        required: false

jobs:
  execute:
    runs-on: ubuntu-latest
    steps:
      - name: Authenticate to cluster
        uses: skyhook-io/cloud-login@v1
        with:
          provider: gcp
          account: my-project
          location: us-central1
          cluster: production

      - name: Resolve job resource
        id: resolve
        uses: skyhook-io/resolve-job-template@v1
        with:
          job_name: ${{ inputs.job_name }}
          namespace: ${{ inputs.namespace }}
          job_type: ${{ inputs.job_type }}

      - name: Submit job
        id: submit
        uses: skyhook-io/submit-job@v1
        with:
          job_type: ${{ inputs.job_type }}
          resource_name: ${{ steps.resolve.outputs.name }}
          namespace: ${{ inputs.namespace }}
          parameters: ${{ inputs.parameters }}

      - name: Report
        run: |
          echo "Submitted: ${{ steps.submit.outputs.created_name }}"
```

## How It Works

### Kubernetes CronJob
1. Generates a unique job name with timestamp suffix
2. Creates a Job from the CronJob using `kubectl create job --from`
3. Ensures name doesn't exceed 63 characters (K8s limit)
4. Outputs the created Job name

### Argo Workflows
1. Installs Argo CLI (if not already installed)
2. Parses JSON parameters and converts to `-p key=value` flags
3. Submits workflow using `argo submit --from`
4. Captures the created Workflow name from output
5. Outputs the workflow name

## Requirements

- kubectl must be installed and configured
- For Argo workflows: jq must be available (for parameter parsing)
- Cluster authentication must be completed before using this action
- The specified resource (CronJob/WorkflowTemplate/CronWorkflow) must exist in the namespace

## Error Handling

The action will fail if:
- Invalid `job_type` is specified
- Resource doesn't exist in the namespace
- kubectl/argo commands fail
- Parameter JSON is malformed (for Argo workflows)

## Notes

- Job names for CronJobs include a timestamp to ensure uniqueness
- Workflow names for Argo are generated automatically by Argo
- Parameters are only used for Argo workflows (ignored for CronJobs)
- The action automatically installs the Argo CLI when needed
- All commands run with `set -euo pipefail` for safety

## License

MIT
