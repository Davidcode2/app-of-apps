# External Secrets Configuration

This folder contains the configuration for External Secrets Operator to sync secrets from AWS Parameter Store.

## Structure

- `aws-secret-store.yaml` - ClusterSecretStore configuration for AWS Parameter Store
- `aws-credentials-secret.yaml` - AWS credentials secret (needs to be manually created with actual credentials)
- Example external secrets for various namespaces