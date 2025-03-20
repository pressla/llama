# Kubernetes Manifests for LLaMA Stack

This repository contains Kubernetes manifests for deploying a complete LLaMA stack with the following components:

- **Postgres**: Database for storing metadata
- **Pinecone** (using Milvus): Vector database for storing embeddings
- **LlamaIndex**: Data framework for LLM applications
- **LLaMA Pipeline**: Orchestration service for LLM workflows

## Prerequisites

- Kubernetes cluster (Harvester single node)
- kubectl configured to access your cluster
- Longhorn storage class installed

## Deployment

You can deploy the entire stack using kubectl and kustomize:

```bash
kubectl apply -k .
```

Or deploy individual components:

```bash
# Deploy Postgres
kubectl apply -f postgres/

# Deploy Pinecone (Milvus)
kubectl apply -f pinecone/

# Deploy LlamaIndex
kubectl apply -f llamaindex/

# Deploy LLaMA Pipeline
kubectl apply -f llamapipeline/
```

## Accessing Services

All services are exposed through ingress with the following paths:

- **Pinecone UI**: https://harvester/pinecone
- **LlamaIndex API**: https://harvester/llamaindex
- **LLaMA Pipeline UI**: https://harvester/llamapipeline

### SSH Access

SSH access to the containers is available through NodePort services:

- **LlamaIndex SSH**: NodePort 30023 (port 22)
- **LLaMA Pipeline SSH**: NodePort 30022 (port 22)

You can SSH into the containers using:

```bash
# SSH into LlamaIndex
ssh root@192.168.178.58 -p 30023

# SSH into LLaMA Pipeline
ssh root@192.168.178.58 -p 30022
```

Default credentials:
- Username: root
- Password: password

## Configuration

### Secrets

The Postgres credentials are stored in a Kubernetes Secret. The default values are:

- Username: postgres
- Password: password

To change these values, update the `postgres-secret.yaml` file with your base64-encoded credentials:

```bash
echo -n "your-username" | base64
echo -n "your-password" | base64
```

### Storage

All components use Longhorn for persistent storage:

- Postgres: 10Gi
- Pinecone: 10Gi
- LlamaIndex: 5Gi
- LLaMA Pipeline: 5Gi

To change the storage size, update the respective PVC files.

## Verification

Check if all pods are running:

```bash
kubectl get pods -n llama
```

Check if all services are created:

```bash
kubectl get svc -n llama
```

Check if all ingresses are created:

```bash
kubectl get ingress -n llama
```

### Verify PostgreSQL

Check PostgreSQL version:

```bash
kubectl exec postgres-0 -n llama -- psql -U postgres -d llamadb -c "SELECT version();" -t
```

Test PostgreSQL functionality by creating a table and inserting data:

```bash
kubectl exec postgres-0 -n llama -- psql -U postgres -d llamadb -c "CREATE TABLE test (id serial PRIMARY KEY, name VARCHAR(100)); INSERT INTO test (name) VALUES ('test_data'); SELECT * FROM test;"
```

### Verify Pinecone (Qdrant)

Check if Qdrant is running:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- pinecone:6333/healthz
```

List collections:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- pinecone:6333/collections
```

Access Qdrant through ingress:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- harvester/pinecone
```

### Verify LlamaIndex

Check if LlamaIndex API is running:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- llamaindex:8000/health
```

Check LlamaIndex configuration:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- llamaindex:8000/config
```

Access LlamaIndex API through ingress:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- harvester/llamaindex
```

### Verify LlamaPipeline

Check if LlamaPipeline API is running:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- llamapipeline:8000/health
```

Check LlamaPipeline configuration:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- llamapipeline:8000/config
```

Access LlamaPipeline API through ingress:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- harvester/llamapipeline
```

Test LlamaPipeline with a sample document:

```bash
kubectl run -n llama busybox --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- --header="Content-Type: application/json" --post-data='{"documents":[{"text":"This is a test document"}],"query":"test"}' llamapipeline:8000/pipeline
```

### Verify SSH Services

Check if SSH services are created:

```bash
kubectl get services -n llama | grep ssh
```

Verify SSH daemon is running in the containers:

```bash
# Check SSH daemon in LlamaIndex
kubectl exec -t -n llama $(kubectl get pod -n llama -l app=llamaindex -o jsonpath='{.items[0].metadata.name}') -- ps aux | grep sshd

# Check SSH daemon in LlamaPipeline
kubectl exec -t -n llama $(kubectl get pod -n llama -l app=llamapipeline -o jsonpath='{.items[0].metadata.name}') -- ps aux | grep sshd
```

Test SSH connection:

```bash
# Test SSH to LlamaIndex (from another machine)
ssh -o StrictHostKeyChecking=no root@192.168.178.58 -p 30023 "echo SSH connection successful"

# Test SSH to LlamaPipeline (from another machine)
ssh -o StrictHostKeyChecking=no root@192.168.178.58 -p 30022 "echo SSH connection successful"
```
The default credentials are:

Username: root
Password: password

## Troubleshooting

If you encounter issues, check the pod logs:

```bash
kubectl logs -f <pod-name>
```

Check the pod status:

```bash
kubectl describe pod <pod-name>
```

## Cleanup

To remove all resources:

```bash
kubectl delete -k .
