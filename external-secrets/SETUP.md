# External Secrets Operator Setup Guide

## Prerequisites

1. - [x] AWS IAM user with permissions to access Parameter Store
2. Parameters created in AWS Systems Manager Parameter Store

## AWS Parameter Store Setup

### Required IAM Policy

Create an IAM user with the following policy attached:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": [
        "arn:aws:ssm:eu-central-1:*:parameter/immoly/*",
        "arn:aws:ssm:eu-central-1:*:parameter/joy-alemazung/*",
        "arn:aws:ssm:eu-central-1:*:parameter/schluesselmomente/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:DescribeParameters"
      ],
      "Resource": "*"
    }
  ]
}
```

### Parameter Creation

Create parameters in AWS Parameter Store with the following structure:

```bash
# Immoly database credentials
aws ssm put-parameter \
  --name "/immoly/db/password" \
  --value "your-postgres-password" \
  --type "SecureString"

aws ssm put-parameter \
  --name "/immoly/db/user" \
  --value "postgres" \
  --type "SecureString"

# Joy-Alemazung Strapi credentials
aws ssm put-parameter \
  --name "/joy-alemazung/strapi/api-token" \
  --value "your-strapi-api-token" \
  --type "SecureString"

aws ssm put-parameter \
  --name "/joy-alemazung/strapi/page-limit" \
  --value "6" \
  --type "String"

aws ssm put-parameter \
  --name "/joy-alemazung/strapi/api-url" \
  --value "https://api.alemazung.immo.jakob-lingel.dev" \
  --type "String"
```

## Kubernetes Setup

### Step 1: Deploy the External Secrets Operator

- [x] step 1 done

```bash
kubectl apply -f external-secrets-operator-app.yaml
```

Wait for the operator to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=external-secrets -n external-secrets --timeout=300s
```

### Step 2: Create AWS Credentials Secret

**IMPORTANT**: Never commit real credentials to git!

1. Create a temporary file with your AWS credentials:

- [x] step 2 done

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: awssm-secret
  namespace: external-secrets
type: Opaque
stringData:
  access-key: "YOUR_AWS_ACCESS_KEY_ID"
  secret-access-key: "YOUR_AWS_SECRET_ACCESS_KEY"
EOF
```

### Step 3: Deploy External Secrets Configuration

```bash
kubectl apply -f external-secrets-config-app.yaml
```

This will deploy:
- ClusterSecretStore for AWS Parameter Store
- ExternalSecret resources for each namespace

### Step 4: Verify Secret Synchronization

Check if the secrets are created:

```bash
# Check Immoly database secret
kubectl get secret immoly-db-secret -n immoly
kubectl describe externalsecret immoly-db-external-secret -n immoly

# Check Joy-Alemazung Strapi secret
kubectl get secret alemazung-strapi-secret -n joy-alemazung
kubectl describe externalsecret alemazung-strapi-external-secret -n joy-alemazung
```

## Migration from Base64 Secrets

1. Create parameters in AWS Parameter Store with current values
2. Deploy External Secrets configuration
3. Verify secrets are synced correctly
4. Remove old secret YAML files from git
5. Update `.gitignore` to exclude `aws-credentials-secret.yaml`

## Troubleshooting

### Check External Secrets Operator logs

```bash
kubectl logs -n external-secrets deployment/external-secrets -f
```

### Check ExternalSecret status

```bash
kubectl describe externalsecret -n <namespace> <external-secret-name>
```

### Common Issues

1. **Authentication Failed**: Verify AWS credentials in `awssm-secret`
2. **Parameter Not Found**: Check parameter exists in AWS with correct path
3. **No Permissions**: Verify IAM policy includes the parameter ARN
4. **Secret Not Updating**: Check `refreshInterval` and operator logs
