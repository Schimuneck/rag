# 🚀 RAG Pipeline Deployment Guide
## PDF Ingestion and RAG Indexing with Docling, LlamaStack, and Milvus

This guide provides step-by-step instructions to deploy a complete RAG (Retrieval-Augmented Generation) pipeline on OpenShift AI using Docling, LlamaStack, and Milvus.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [Testing and Validation](#testing-and-validation)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Next Steps](#next-steps)
- [Resources](#resources)

## Prerequisites

- OpenShift cluster with OpenShift AI (RHOAI) installed
- OpenShift CLI (`oc`) installed and configured
- Access to create projects and deploy resources
- GPU resources available (optional, for acceleration)
- **Python 3.10 or higher** (required for llama-stack package compatibility)
- Amazon S3 bucket with PDF documents (for data ingestion)

## Architecture Overview

The RAG pipeline consists of the following components:

- **LlamaStack Operator**: Manages LlamaStackDistribution resources
- **LlamaStack Distribution**: Provides vector database (Milvus) and embedding services
- **vLLM Inference Service**: Serves the language model for text generation
- **Kubeflow Pipeline**: Orchestrates the PDF processing workflow
- **Docling**: Converts PDF documents to embeddings and stores them in the vector database

## Step-by-Step Deployment

### Step 1: Connect to OpenShift Cluster

```bash
# Login to your OpenShift cluster
oc login --token=<your-token> --server=<your-cluster-url>

# Verify connection
oc whoami
oc project
```

### Step 2: Create New Project

```bash
# Create a new project for the RAG deployment
oc new-project rag-demo

# Verify project creation
oc project rag-demo
```

### Step 3: Install LlamaStack Operator

The LlamaStack operator manages LlamaStackDistribution resources and provides the infrastructure for RAG operations.

```bash
# Set the LlamaStack repository URL
export LLAMASTACK_REPO=https://raw.githubusercontent.com/llamastack/llama-stack-k8s-operator

# Install the LlamaStack operator
oc apply -f $LLAMASTACK_REPO/main/release/operator.yaml
```

**Expected Output:**
```
namespace/llama-stack-k8s-operator-system created
customresourcedefinition.apiextensions.k8s.io/llamastackdistributions.llamastack.io created
serviceaccount/llama-stack-k8s-operator-controller-manager created
role.rbac.authorization.k8s.io/llama-stack-k8s-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/llama-stack-k8s-operator-manager-role configured
clusterrole.rbac.authorization.k8s.io/llama-stack-k8s-operator-metrics-reader configured
clusterrole.rbac.authorization.k8s.io/llama-stack-k8s-operator-proxy-role configured
rolebinding.rbac.authorization.k8s.io/llama-stack-k8s-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/llama-stack-k8s-operator-manager-rolebinding configured
clusterrolebinding.rbac.authorization.k8s.io/llama-stack-k8s-operator-proxy-rolebinding configured
configmap/llama-stack-k8s-operator-distribution-images created
configmap/llama-stack-k8s-operator-manager-config created
service/llama-stack-k8s-operator-controller-manager-metrics-service created
deployment.apps/llama-stack-k8s-operator-controller-manager created
```

### Step 4: Verify LlamaStack Operator Installation

```bash
# Check operator pod status
oc get pods -n llama-stack-k8s-operator-system

# Check operator logs
oc logs -l app.kubernetes.io/name=llama-stack-k8s-operator -n llama-stack-k8s-operator-system -f --tail=20
```

**Expected Log Output:**
```
INFO    setup   starting manager
INFO    controller-runtime.metrics      Starting metrics server
INFO    controller-runtime.metrics      Serving metrics server  {"bindAddress": ":8080", "secure": false}
INFO    starting server {"kind": "health probe", "addr": "[::]:8081"}
INFO    Starting Controller     {"controller": "llamastackdistribution", "controllerGroup": "llamastack.io", "controllerKind": "LlamaStackDistribution"}
INFO    Starting workers        {"controller": "llamastackdistribution", "controllerGroup": "llamastack.io", "controllerKind": "LlamaStackDistribution", "worker count": 1}
```

### Step 5: Deploy RAG Infrastructure

Deploy the LlamaStack distribution with Milvus integration and vLLM inference service.

```bash
# Navigate to the stack directory
cd stack

# Apply the base configuration
oc apply -k base/
```

**Expected Output:**
```
secret/llama-32-3b-instruct created
llamastackdistribution.llamastack.io/lsd-llama-milvus created
route.route.openshift.io/lsd-llama-milvus unchanged
servingruntime.serving.kserve.io/vllm created
inferenceservice.serving.kserve.io/vllm created
```

### Step 6: Verify Infrastructure Deployment

```bash
# Check all pods are running
oc get pods

# Check LlamaStack distribution status
oc get llamastackdistributions

# Check services
oc get services | grep llama
```

**Expected Pod Status:**
```
NAME                                                    READY   STATUS    RESTARTS   AGE
lsd-llama-milvus-d8f78fdb8-xjt5q                        1/1     Running   0          5m
vllm-predictor-7d675cf7c8-tfrtc                         2/2     Running   0          5m
```

### Step 7: Test LlamaStack Service Connectivity

```bash
# Test the LlamaStack API from within the cluster
oc exec -it <notebook-pod-name> -- curl -s http://lsd-llama-milvus-service:8321/v1/models
```

**Expected Output:**
```json
{
  "data": [
    {
      "identifier": "vllm",
      "provider_resource_id": "vllm",
      "provider_id": "vllm-inference",
      "type": "model",
      "metadata": {},
      "model_type": "llm"
    },
    {
      "identifier": "ibm-granite/granite-embedding-125m-english",
      "provider_resource_id": "ibm-granite/granite-embedding-125m-english",
      "provider_id": "sentence-transformers",
      "type": "model",
      "metadata": {"embedding_dimension": 768},
      "model_type": "embedding"
    }
  ]
}
```

### Step 8: Prepare Pipeline Environment

Before deploying the pipeline, you need to set up the Python environment and update the pipeline configuration.

**Set up Python environment and install requirements:**

```bash
# Navigate to the pipeline directory
cd docs/kfp-pipelines/docling_rag/

# Create a Python 3.11 virtual environment (recommended)
python3.11 -m venv venv

# Activate the virtual environment
# On macOS/Linux:
source venv/bin/activate
# On Windows:
# venv\Scripts\activate

# Verify Python version (should be 3.10+)
python --version

# Install required packages
pip install -r requirements.txt

# If you encounter issues with llama-stack, try installing with specific version:
pip install llama-stack==0.2.10
```

**Update the service URL in the pipeline file:**

```bash
# Edit the pipeline file to update the service URL
# Change line 227 in docling_convert_pipeline.py from:
# service_url: str = "http://llama-test-milvus-kserve-service:8321",
# to:
# service_url: str = "http://lsd-llama-milvus-service:8321",
```

**Rebuild the compiled pipeline:**

```bash
# Compile the updated pipeline
python docling_convert_pipeline.py

# Verify the compiled file was created
ls -la docling_convert_pipeline_compiled.yaml
```

**Expected Output:**
```
# The script will compile the pipeline and create:
# docling_convert_pipeline_compiled.yaml
```

### Step 9: Configure S3 Data Connection in OpenShift AI

Configure the Amazon S3 data connection in OpenShift AI Data Science Pipelines to access your PDF files.

**Access OpenShift AI Dashboard:**

1. Open your OpenShift cluster console
2. Navigate to **AI/ML** → **Data Science Pipelines**
3. Click on your project (e.g., `rag-demo`)

**Create S3 Data Connection:**

1. In the Data Science Pipelines dashboard, click on **Data connections** in the left sidebar
2. Click **Create data connection**
3. Fill in the following details:

   **Connection Details:**
   - **Name:** `s3-pdf-connection`
   - **Description:** `S3 connection for PDF documents`
   - **Provider:** `Amazon S3`

   **S3 Configuration:**
   - **Access key ID:** `<access_key_id>`
   - **Secret access key:** `<secret_access_key>`
   - **Endpoint:** `https://s3.amazonaws.com`
   - **Region:** `us-east-1` (or your preferred region)
   - **Bucket:** `<bucket_name>`

4. Click **Create** to save the data connection

### Step 10: Import and Configure Pipeline

Upload the compiled pipeline to OpenShift AI Data Science Pipelines.

**Import Pipeline:**

1. In the Data Science Pipelines dashboard, click on **Pipelines** in the left sidebar
2. Click **Import pipeline**
3. Choose **Upload a file**
4. Select the compiled pipeline file: `docs/kfp-pipelines/docling_rag/docling_convert_pipeline_compiled.yaml`
5. Click **Import**

**Pipeline Configuration:**

After importing, configure the pipeline with the following parameters:

- **base_url:** `s3://<bucket-name>/` (or the specific folder containing your PDFs)
- **pdf_filenames:** Comma-separated list of your PDF files (e.g., `document1.pdf,document2.pdf`)
- **num_workers:** `2` (adjust based on your cluster resources)
- **vector_db_id:** `my_demo_vector_id`
- **service_url:** `http://lsd-llama-milvus-service:8321`
- **embed_model_id:** `ibm-granite/granite-embedding-125m-english`
- **max_tokens:** `512`
- **use_gpu:** `true` (if GPU is available)

### Step 11: Execute Pipeline

Execute the pipeline to process your PDF documents.

**Create Pipeline Run:**

1. In the Pipelines section, find your imported pipeline
2. Click on the pipeline name to open it
3. Click **Create run**
4. Configure the run parameters as specified above
5. Click **Start** to begin the pipeline execution

**Monitor Pipeline Execution:**

```bash
# Check pipeline run status
oc get pods | grep docling-convert-pipeline

# Check pipeline logs
oc logs <pipeline-pod-name> -f

# Monitor in the OpenShift AI dashboard
# Navigate to Pipelines → Your Pipeline → Runs
```

**Expected Pipeline Stages:**
1. **Register Vector DB** - Sets up the Milvus vector database
2. **Import Test PDFs** - Downloads PDFs from S3
3. **Create PDF Splits** - Splits PDFs for parallel processing
4. **Docling Convert** - Converts PDFs to embeddings and stores in vector DB

## Testing and Validation

### Step 12: Create Workbench for Testing

After the pipeline completes successfully, create a workbench to test the RAG functionality.

**Create Workbench:**

1. In OpenShift AI dashboard, navigate to **Workbenches**
2. Click **Create workbench**
3. Configure the workbench:

   **Workbench Details:**
   - **Name:** `rag-test-workbench`
   - **Description:** `Workbench for testing RAG functionality`
   - **Image:** `Minimal Python`
   - **Container size:** `Small`
   - **Accelerator:** `None`
   - **Persistent storage:** `10 GiB` (or as needed)

4. Click **Create workbench**

**Access Workbench:**

1. Wait for the workbench to start (status should show "Running")
2. Click **Launch** to open JupyterLab
3. Create a new Python notebook for testing

**Clone Repository and Open Notebook:**

1. In your JupyterLab workbench, clone the repository:
   ```bash
   git clone https://github.com/opendatahub-io/rag
   ```

2. In JupyterLab, navigate to the `rag/docs/kfp-pipelines/docling_rag/` folder
3. Click on `docling_rag.ipynb` to open the notebook
4. Update **service_url:** `http://lsd-llama-milvus-service:8321` (if necessary)
5. Run the cells to test the RAG functionality with your deployed pipeline

**Notebook Features:**
- Connect to LlamaStack service
- List available models and vector databases
- Test RAG queries against your indexed documents
- Visualize embeddings and search results
- Interactive testing of the complete RAG pipeline

## Troubleshooting

### Common Issues

1. **Python Version Compatibility Issues**
   - Ensure you're using Python 3.10 or higher
   - Create a fresh virtual environment with the correct Python version
   - Reinstall dependencies in the new environment
   - If using Python 3.9 or lower, upgrade to Python 3.11

2. **LlamaStack Package Installation Issues**
   - Try installing with specific version: `pip install llama-stack==0.2.10`
   - Check for conflicting dependencies
   - Ensure all required packages are installed from requirements.txt

3. **LlamaStack Operator CrashLoopBackOff**
   - Check if ConfigMap exists: `oc get configmap -n llama-stack-k8s-operator-system`
   - Reinstall operator if needed

4. **Image Pull Errors**
   - Verify image exists in registry
   - Check network connectivity
   - Ensure proper authentication if using private images

5. **Service Connection Issues**
   - Verify service names are correct
   - Check pod status and logs
   - Ensure proper network policies

6. **S3 Connection Issues**
   - Verify S3 credentials are correct and have proper permissions
   - Check if the bucket exists and is accessible
   - Ensure the endpoint URL is correct for your region
   - Check if PDF files exist in the specified S3 path

7. **Pipeline S3 Download Failures**
   - Verify the `base_url` parameter is correctly formatted (e.g., `s3://bucket/`)
   - Check if PDF filenames in the `pdf_filenames` parameter match files in S3
   - Ensure the pipeline has access to S3 credentials (via data connection)
   - Check pipeline logs for specific S3 error messages

8. **Data Connection Issues in OpenShift AI**
   - Verify the data connection is created in the correct project
   - Check if the data connection is properly configured with all required fields
   - Ensure the data connection is accessible to the pipeline
   - Test the data connection from within a workbench

## Cleanup

To remove all resources:

```bash
# Delete LlamaStack distribution
oc delete llamastackdistribution lsd-llama-milvus

# Delete inference service
oc delete inferenceservice vllm

# Delete serving runtime
oc delete servingruntime vllm

# Delete operator
oc delete namespace llama-stack-k8s-operator-system
oc delete crd llamastackdistributions.llamastack.io

# Delete project (optional)
oc delete project rag-demo
```

## Next Steps

- Customize the pipeline for your specific use case
- Add more PDF documents for indexing
- Implement custom RAG queries
- Scale the infrastructure as needed
- Monitor performance and optimize
- Integrate with additional data sources
- Implement authentication and access controls

## Resources

- [LlamaStack Documentation](https://github.com/llamastack/llama-stack)
- [Kubeflow Pipelines](https://www.kubeflow.org/docs/components/pipelines/)
- [OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed)
- [Milvus Vector Database](https://milvus.io/docs)
- [Docling Documentation](https://github.com/docling-ai/docling)
