---
title: "Running MLflow in Azure Container Apps: Lessons from our setup"
date: 2025-09-30T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - azure
  - mlflow
  - container app
---

We have been doing more and more work with prompting, and we wanted a way to keep track of which prompts were used, how many tokens they consumed, and what the resulting output was. Just as important, the data needed to be available to everyone on the team so we could all learn from the results. To support this, we decided to set up an MLflow server.

MLflow is an open-source platform for managing the machine learning lifecycle. While it is often used for tracking experiments and storing models, we mainly used it for tracking our prompt runs and related metadata. At first, one team member ran MLflow locally in a Docker container, which worked fine as long as only one person needed it. But as the project grew this was no longer sustainable. We wanted a setup that was reliable, shareable, and preserved our experiment history. Our main goals were to put MLflow online and connect it to a SQL database for persistent storage. Our first thought was simple: take the same Docker image and run it inside an Azure Container App. Easy, right? Unfortunately, we hit some difficulties along the way.

In this blog, we'll walk through the initial `docker run` command we used locally, all the way to the final version that worked in Azure. Along the way, we explained the hurdles we encountered and how we solved them.  

## Running locally

The command to run MLflow locally was:

```bash
docker run -it -p 5000:5000 ghcr.io/mlflow/mlflow mlflow server --host 0.0.0.0
```

The most important part of the command was the `--host 0.0.0.0`. By default, the MLflow server only accepted connections from the local machine. It needed the `--host` argument to accept connections from other machines as well. The `-p 5000:5000` mapped port 5000 from the container to localhost.

The command `mlflow server` was required as a startup command for the image to actually start the server. Without it, nothing would start.

Running MLflow like this meant all data was stored locally in the container's filesystem. This was a problem, because as soon as the container was deleted, all stored data was lost.

## Artifact and backend store

As stated before, all data was stored in a simple folder structure. With only one or two people using MLflow, this could work as changes were appended to existing files. However, there were no concepts like file locks to prevent multiple people from writing simultaneously. The more people used MLflow, the higher the chance of corrupting these files.

To solve this MLflow has the option to configure external stores where it can send its information so it's safely stored outside the container.

MLflow had two types of stores to preserve information about runs:

- Backend store: stores metadata about runs (IDs, start and end times, parameters, metrics, tags, logs, etc.)
- Artifact store: stores artifacts for each run (model weights, images, logs, etc.)

## Setting up database store

MLflow supports a wide range of databases for the backend store. As we were already using Azure SQL Database, we decided to use that for MLflow as well.

To tell MLflow to use a database we added the following argument to the Docker run command:

```bash
--backend-store-uri "mssql+pyodbc://{dbUser}:{dbPassword}@{dbServer}.database.windows.net/{dbName}?driver=ODBC+Driver+18+for+SQL+Server&Encrypt=yes&TrustServerCertificate=no&Connection+Timeout=30"
```

When we added this argument, MLflow failed to start because the base image did not have all the required dependencies for MSSQL. To solve this, we created our own `Dockerfile` and installed the missing dependencies.

```Dockerfile
FROM ghcr.io/mlflow/mlflow:latest

USER root

RUN apt-get update && apt-get install -y \
    curl gnupg2

RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/debian/11/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
    apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql18

RUN pip install --no-cache-dir pyodbc

USER 1000
```

This ensured both `msodbcsql18` and `pyodbc` were installed. With these, MLflow started successfully, but as explained earlier, we still needed an artifact store or else information about individual runs would be missing.

## Setting up the artifacts store

MLflow supports multiple artifact storage types, such as Amazon S3, Azure Blob Storage, and Google Cloud Storage. Since we were most familiar with Azure, we used Azure Blob Storage.

To point MLflow to Azure Blob Storage, we used the following argument:

```bash
--artifacts-destination wasbs://{blobContainerName}@{storageAccountName}.blob.core.windows.net/
```

For authentication, there were two options:

- Connection string: By passing in an environment variable to the container MLflow would automatically use it to connect to the correct Storage Account (`-e AZURE_STORAGE_CONNECTION_STRING="DefaultEndpointsProtocol=https;AccountName={storageAccountName};AccountKey={storageAccountKey};EndpointSuffix=core.windows.net"`)
- Role-Based Access Control (RBAC): our preferred option, since we have been moving away from using keys and connection strings as much as possible in Azure

To use RBAC, we removed the connection string and added two Python packages in our Dockerfile:

- `azure-storage-blob`: required for both RBAC and connection strings
- `azure-identity`: required for RBAC

```Dockerfile
RUN pip install --no-cache-dir pyodbc azure-storage-blob azure-identity
```

## Bringing it to Azure Container Apps

Up to this point, everything we did was still running with a local Docker command. The full command looked like this (`evdbmlflow` was our locally built image):

```bash
docker run -p 5000:5000 evdbmlflow:latest mlflow server --host 0.0.0.0 --backend-store-uri "mssql+pyodbc://{dbUser}:{dbPassword}@{dbServer}.database.windows.net/{dbName}?driver=ODBC+Driver+18+for+SQL+Server&Encrypt=yes&TrustServerCertificate=no&Connection+Timeout=30" --artifacts-destination wasbs://{blobContainerName}@{storageAccountName}.blob.core.windows.net/
```

To get everything running in Azure Container Apps we did the following steps:

**1. Build and push the image**:
  We created our Dockerfile and pushed the image to an Azure Container Registry (ACR)
**2. Provisioned supporting resources**:
  We deployed the required Azure services: Azure SQL Database, Azure Blob Storage, Log Analytics, Container App Environment, Container App, Role Assignments(RBAC). These made sure we had the backend to write our metadata and artifacts to and permissions were set so the container had access to everything it needs.
**3. Deploy the Container App**: Finally we've configured the Container App to use our custom image and added the startup command and arguments we also used for our local container so it could connect to Blob Storage and SQL.

Instead of clicking through the portal, we've provisioned everything using Bicep with a few CLI commands. The full setup is available in this GitHub repository: https://github.com/evdbogaard/blog-mlflow-azure.

It contains:

- `acr.bicep`: Used to create the container registry
- `Dockerfile`: The full `Dockerfile` used
- `dockercommands.sh`: cli commands used to login to the registry, build the image, and push the image
- `infra.bicep`: Provisioning of all the supporting resources and the container app itself.

You now have an MLflow server up and running inside an Azure Container App. In the Bicep templates we kept the resource configurations simple, but in production you would want to tighten security and optimize costs.

## MLflow tracking server behavior in Azure Container Apps

While we had MLflow running in the cloud, we noticed that in some cases artifacts still were not logged. After investigation, we discovered that the bug came from our client-side Python code.

```python
import os
import mlflow

from dotenv import load_dotenv
from langchain_openai import AzureChatOpenAI

load_dotenv()

mlflow.set_experiment(f"My Test Experiment")
mlflow.langchain.autolog()

llm = AzureChatOpenAI(
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),
    azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
    deployment_name=os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME"),
    api_version="2024-12-01-preview",
)

response = llm.invoke("What is MLflow?")
print(response)
```

When we ran the code locally, both traces and artifacts appeared in MLflow. Running it in a local container also worked. But once we moved the container to Azure Container Apps, artifacts stopped working.

It turned out that Azure Container Apps injected a number of environment variables. One of them was `OTEL_EXPORTER_OTLP_ENDPOINT`. When this was set, MLflow switched from its default behavior and attempted to send traces to an OpenTelemetry Collector. We could not find a way to prevent this variable from being injected.

Our workaround was to unset it explicitly before importing MLflow:

```python
os.environ.pop("OTEL_EXPORTER_OTLP_ENDPOINT", None)
```

With this change, behavior was consistent between local and cloud runs. When we opnened the MLflow UI, we could see the traces showing up correctly, including detailed breakdowns of the prompts sent to the LLM, the responses, token usage, and more.

![MLflow - trace](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fqovy440m62fmdbeslyf9.png)

![MLflow - trace details](https://media2.dev.to/dynamic/image/width=800%2Cheight=%2Cfit=scale-down%2Cgravity=auto%2Cformat=auto/https%3A%2F%2Fdev-to-uploads.s3.amazonaws.com%2Fuploads%2Farticles%2Fv5z85nsewzyblrlq3pam.png)

## Resources

If you found the content of this blog useful, here are some resources for further reading:

- https://learn.microsoft.com/en-us/azure/machine-learning/concept-mlflow?view=azureml-api-2
- https://mlflow.org/docs/latest/ml/tracking/server/
- https://mlflow.org/docs/latest/ml/tracking/artifact-stores/
- https://mlflow.org/docs/latest/ml/tracking/backend-stores/
- https://www.mlflow.org/docs/2.17.2/llms/tracing/index.html#using-opentelemetry-collector-for-exporting-traces
