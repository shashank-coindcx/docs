# Using AWS Secrets Manager with External Secrets Operator in LocalStack (Kind)

## Overview
This guide explains how to set up **AWS Secrets Manager** locally using **LocalStack** and integrate it with **External Secrets Operator (ESO)** running in a **Kind** Kubernetes cluster.  
It also includes common issues encountered during setup and their fixes.

---

## Prerequisites
- **Docker** installed and running
- **Kind** (Kubernetes in Docker)
- **Helm** package manager
- **AWS CLI** installed
- Basic understanding of Kubernetes manifests (CRDs)

---

## Architecture Diagram
```mermaid
flowchart LR
    subgraph Local Environment
        AWSCLI[AWS CLI] -->|create secret| LocalStack[(LocalStack\nSecrets Manager)]
    end

    subgraph Kubernetes (Kind)
        ESO[External Secrets Operator] -->|fetch secret| LocalStack
        ESO -->|create secret| K8sSecret[(Kubernetes Secret)]
        K8sSecret -->|used by apps| AppPod[Application Pod]
    end
```

---

## 1. Install and Configure LocalStack
1. Follow the [LocalStack official documentation](https://docs.localstack.cloud/) to install.
2. Configure AWS CLI credentials for LocalStack:
    - Using AWS CLI:
      ```bash
      aws configure --profile localstack
      ```
    - **OR** by editing config files:
        - `~/.aws/config`
        - `~/.aws/credentials`
3. Create an AWS profile (e.g., `localstack`) to avoid overriding real AWS credentials.

---

## 2. Create a Secrets Manager in LocalStack
Using AWS CLI with the LocalStack profile:
```bash
aws --endpoint-url=http://localhost:4566     --profile localstack     secretsmanager create-secret     --name my-test-secret     --secret-string '{"username":"admin","password":"pass123"}'
```

---

## 3. Install External Secrets Operator (ESO) on Kind
Install via Helm:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets
```

---

## 4. Fixing `ErrImagePull` in Kind (Local Image Loading)
When ESO pods run inside Kind, they may fail to pull images because Kind nodes are Docker containers.  
To load the image manually:

```bash
docker save -o external-secrets.tar oci.external-secrets.io/external-secrets/external-secrets:v0.19.0
docker cp external-secrets.tar eso-operator-control-plane:/external-secrets.tar
docker exec -it eso-operator-control-plane ctr -n k8s.io images import /external-secrets.tar
```

---

## 5. Set AWS Region for ESO
Manually edit the `external-secrets` Deployment:
```bash
kubectl edit deployment external-secrets
```
Add:
```yaml
env:
  - name: AWS_REGION
    value: us-east-1
```

---

## 6. Create Kubernetes Resources
### 6.1 Create AWS Credentials Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
type: Opaque
stringData:
  access-key: test
  secret-access-key: test
```

### 6.2 Create a SecretStore
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretstore
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
```
> **Alternative:** Credentials can be set directly in the `SecretStore` manifest.

### 6.3 Create an ExternalSecret
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-external-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretstore
    kind: SecretStore
  target:
    name: my-k8s-secret
  data:
    - secretKey: username
      remoteRef:
        key: my-test-secret
        property: username
    - secretKey: password
      remoteRef:
        key: my-test-secret
        property: password
```

---

## Common Issues & Fixes
| Issue | Cause | Fix |
|-------|-------|-----|
| `ErrImagePull` for ESO | Kind nodes are Docker containers and cannot pull image directly | Load the image manually using `docker save` and `ctr import` |
| `MissingRegion: could not find region configuration` | ESO Deployment missing AWS region | Add `AWS_REGION` env variable in Deployment |
| Secrets not syncing | Incorrect profile/credentials in LocalStack | Check profile in `~/.aws/credentials` and SecretStore manifest |

---

## Notes / Learnings
- **Profiles** prevent overwriting your real AWS credentials.
- LocalStack’s AWS CLI calls require `--endpoint-url`.
- ESO running in Kind often needs manual image loading when offline or using private registries.
- The AWS region must be explicitly set for ESO to work.
- Credentials can be stored either as a Kubernetes Secret or directly in the SecretStore.

---

## References
- [LocalStack Docs](https://docs.localstack.cloud/)
- [External Secrets Operator Docs](https://external-secrets.io/)
- [Kind Documentation](https://kind.sigs.k8s.io/)