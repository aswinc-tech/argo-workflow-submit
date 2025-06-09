# 🚀 Auto-Triggering Argo Workflows Using GitHub Actions with Python

Modern DevOps pipelines demand seamless integration between different automation tools. This guide demonstrates how to create a powerful integration between **GitHub Actions** and **Argo Workflows** using Python, enabling you to trigger Kubernetes-native workflows directly from your CI/CD pipeline.

## 📋 What You'll Learn

- Setting up a reusable GitHub Action to trigger Argo Workflows
- Leveraging the Argo Workflows Python SDK for workflow orchestration
- Detailed breakdown of key Argo Workflow APIs:
  - `submit_workflow` - Launching workflows with custom parameters
  - `get_workflow` - Monitoring execution status
  - `workflow_logs` - Real-time log streaming

## 🏗️ Architecture Overview

```
GitHub Repository → GitHub Action → Argo Workflows API → Kubernetes Cluster
     |                   |                |                     |
     └─ Code Push        └─ Trigger       └─ Submit            └─ Execute
                           Workflow         Workflow             Workload
```

## ⚙️ Reusable GitHub Action: `action.yml`

Our reusable action handles authentication, workflow submission, and result reporting:

```yaml
name: "Argo Workflow Submit"
description: "Submits an Argo workflow for the specified environment"
author: "Platform Engineering Team"

# Output parameters returned by this action
outputs:
  workflow_name:
    description: "Name of the Argo workflow for the selected environment"
    value: ${{ steps.submit_workflow.outputs.workflow_name }}
  workflow_url:
    description: "URL for the Argo workflow"
    value: ${{ steps.submit_workflow.outputs.workflow_url }}
  app_build_version:
    description: "App build version for deployment"
    value: ${{ steps.submit_workflow.outputs.app_build_version }}
  infra_build_version:
    description: "Infra build version for deployment"
    value: ${{ steps.submit_workflow.outputs.infra_build_version }}
  test_framework:
    description: "Test framework used for deployment"
    value: ${{ steps.submit_workflow.outputs.test_framework }}
  test_version:
    description: "Test version used for deployment"
    value: ${{ steps.submit_workflow.outputs.test_version }}

# Input parameters required by this action
inputs:
  repo_name:
    description: "Full repository name in format 'organization/repository-name'"
    required: true
  auto_deploy_settings:
    description: "Auto deploy configuration settings"
    required: true
  environment:
    description: "Environment to deploy to (e.g., us-dev01, us-int01)"
    required: true
  teams_webhook:
    description: "Microsoft Teams Webhook URL for notifications"
    required: false
  argo_token:
    description: "Token for Argo workflow authentication"
    required: true

# Action execution definition
runs:
  using: "composite"
  steps:
    - name: "Check Python version"
      id: check-python-version
      shell: bash
      run: |
        python --version

    - name: "Install dependencies"
      shell: bash
      run: |
        python -m pip install --upgrade pip
        pip install -r ${GITHUB_ACTION_PATH}/requirements.txt

    - name: "Submit Argo workflow for ${{ inputs.environment }}"
      id: submit_workflow
      shell: bash
      run: |
        python ${GITHUB_ACTION_PATH}/argoworkflows.py
        # Set outputs for GitHub Actions
        if [[ -n "$workflow_name" ]]; then
          echo "workflow_name=$workflow_name" >> $GITHUB_OUTPUT
        fi
        if [[ -n "$workflow_url" ]]; then
          echo "workflow_url=$workflow_url" >> $GITHUB_OUTPUT
        fi
        if [[ -n "$app_build_version" ]]; then
          echo "app_build_version=$app_build_version" >> $GITHUB_OUTPUT
        fi
        if [[ -n "$infra_build_version" ]]; then
          echo "infra_build_version=$infra_build_version" >> $GITHUB_OUTPUT
        fi
        if [[ -n "$test_framework" ]]; then
          echo "test_framework=$test_framework" >> $GITHUB_OUTPUT
        fi
        if [[ -n "$test_version" ]]; then
          echo "test_version=$test_version" >> $GITHUB_OUTPUT
        fi
      env:
        argo_token: ${{ inputs.argo_token }}
        repo_name: ${{ inputs.repo_name }}
        auto_deploy_settings: ${{ inputs.auto_deploy_settings }}
        environment: ${{ inputs.environment }}
```

## 🔍 Deep Dive: Argo Workflows API Integration

### 1️⃣ Authentication & Client Setup

The Python script establishes a secure connection to the Argo server:

```python
# Set up the API client with authentication
def configure_api_client(argo_token):
    configuration = Configuration(host=ARGO_HOST)
    configuration.api_key_prefix['BearerToken'] = 'Bearer'
    configuration.api_key['BearerToken'] = argo_token
    configuration.verify_ssl = False
    
    # Create API client with proper headers
    api_client = ApiClient(configuration)
    api_client.default_headers['Accept'] = 'application/json'
    api_instance = WorkflowServiceApi(api_client)
    
    return configuration, api_client, api_instance
```

### 2️⃣ Workflow Submission

The `submit_workflow` function uses the Argo SDK to submit a workflow based on a template:

```python
# Create the workflow submission request
submit_request = IoArgoprojWorkflowV1alpha1WorkflowSubmitRequest(
    resource_kind="WorkflowTemplate",
    resource_name=workflow_template_name,
    submit_options=submit_opts,
    workflow_template_ref=template_ref
)

# Submit the workflow to Argo server
api_response = api_instance.submit_workflow(namespace=namespace, body=submit_request)
```

The function then handles the submitted workflow and returns useful metadata:

```python
def submit_workflow(api_instance, submit_request, namespace):
    try:
        # Call the submit_workflow API endpoint
        api_response = api_instance.submit_workflow(
            namespace=namespace, 
            body=submit_request,
            _check_return_type=False
        )
        
        # Extract workflow metadata from response
        workflow_name = api_response.metadata['name']
        namespace = api_response.metadata['namespace']
        
        # Generate workflow URL for easy access
        workflow_url = f"{ARGO_HOST}/workflows/{namespace}/{workflow_name}"
        
        return workflow_name, namespace, api_response
        
    except argo_workflows.ApiException as e:
        logger.error(f"× Workflow submission failed: {e}")
        return None, None, None
```

### 3️⃣ Status Monitoring

The `get_workflow_status` function polls the workflow's state:

```python
def get_workflow_status(api_instance, namespace, workflow_name):
    try:
        # Call the get_workflow API endpoint
        workflow_status = api_instance.get_workflow(
            namespace=namespace,
            name=workflow_name,
            _check_return_type=False
        )
        return workflow_status
    except Exception as e:
        logger.error(f"Error getting workflow status: {e}")
        return None
```

### 4️⃣ Real-Time Log Streaming

The workflow_logs API provides visibility into execution:

```python
def retrieve_and_display_logs(api_instance, namespace, workflow_name, seen_logs, tail_lines="200"):
    try:
        # Call the workflow_logs API endpoint
        log_response = api_instance.workflow_logs(
            namespace=namespace,
            name=workflow_name,
            log_options_container='main',
            log_options_tail_lines=tail_lines,
            log_options_timestamps=True,
            _preload_content=False
        )
        
        # Process streamed log data
        log_data = log_response.data.decode('utf-8')
        log_lines = log_data.splitlines()
        
        # Display new logs only (avoid duplicates)
        for line in log_lines:
            if line and line not in seen_logs:
                seen_logs.add(line)
                logger.info(f"📄 {line}")
        
        return seen_logs
        
    except Exception as e:
        logger.error(f"Error retrieving logs: {e}")
        return seen_logs
```

### 5️⃣ Parameter Extraction

Retrieving dynamic outputs from workflows:

```python
def get_workflow_parameters(workflow_status, parameter_names=None):
    parameters = {}
    
    # Extract parameters from workflow status
    if hasattr(workflow_status, 'status') and 'outputs' in workflow_status.status:
        status_params = workflow_status.status.get('outputs', {}).get('parameters', [])
        for param in status_params:
            if param.get('name') and (parameter_names is None or param.get('name') in parameter_names):
                parameters[param.get('name')] = param.get('value')
    
    return parameters
```

## 🧪 Integration Example

Here's how it works in practice:

1. A code change is pushed to your repository
2. GitHub Actions workflow is triggered
3. The Argo Workflow action is called with environment parameters
4. Python script authenticates with Argo server
5. Workflow is submitted with dynamic parameters
6. Script monitors workflow execution in real-time
7. Upon completion, build versions and artifacts are extracted
8. Results are passed back to GitHub Actions as outputs

## 🛠️ Best Practices

- **Secure Token Management**: Store your Argo token in GitHub Secrets
- **Error Handling**: The Python script includes robust error handling and retries
- **Logging**: Comprehensive logging provides visibility into the process
- **Parameter Passing**: Dynamic parameters allow flexible workflow configuration

## 🏁 Conclusion

This integration demonstrates the power of combining GitHub Actions with Argo Workflows. By using Python and the Argo Workflows API, you can create a seamless deployment pipeline that leverages the best of both platforms:

- GitHub Actions for CI triggers and workflow orchestration
- Argo Workflows for Kubernetes-native execution and complex dependency management

This approach is particularly valuable for organizations running microservices or complex distributed applications on Kubernetes.
