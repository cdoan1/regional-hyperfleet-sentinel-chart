# regional-hyperfleet-sentinel-chart

Helm chart that deploys an ArgoCD Application for [hyperfleet-sentinel](https://github.com/openshift-hyperfleet/hyperfleet-sentinel) with AWS Pod Identity support for AWS MQ (RabbitMQ) integration.

## Overview

This chart creates:
- ArgoCD Application resource that deploys hyperfleet-sentinel (v0.1.1)
- SecretProviderClass for AWS Secrets Manager integration
- Configuration for AWS Pod Identity to access AWS MQ

The hyperfleet-sentinel application uses [hyperfleet-broker](https://github.com/openshift-hyperfleet/hyperfleet-broker) for message broker abstraction.

## Prerequisites

1. **EKS Cluster** with AWS Pod Identity configured
2. **ArgoCD** installed in the cluster
3. **AWS Secrets Store CSI Driver** installed
4. **AWS MQ RabbitMQ** broker instance
5. **IAM Role** with permissions to access AWS Secrets Manager and AWS MQ
6. **hyperfleet-system namespace** must be created before deploying this chart (managed by another chart)

## AWS Setup

### 1. Create AWS Secrets Manager Secret

Store your RabbitMQ connection string in AWS Secrets Manager:

```bash
aws secretsmanager create-secret \
  --name hyperfleet-sentinel/broker-credentials \
  --secret-string '{"url":"amqps://username:password@b-xxx.mq.us-east-1.amazonaws.com:5671/"}'
```

The URL format for AWS MQ is:
```
amqps://USERNAME:PASSWORD@BROKER_ENDPOINT:5671/
```

### 2. Create IAM Role for Pod Identity

Create an IAM role with the following trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

Attach this policy to allow secret access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT:secret:hyperfleet-sentinel/broker-credentials*"
    }
  ]
}
```

### 3. Update values.yaml

Update the IAM role ARN in `values.yaml`:

```yaml
hyperfleet-sentinel-chart:
  values:
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"
```

Update the AWS region if needed:

```yaml
secretProvider:
  aws:
    region: us-east-1  # Change to your region
```

## Installation

Install the chart with Helm:

```bash
helm install regional-hyperfleet-sentinel .
```

Or customize values:

```bash
helm install regional-hyperfleet-sentinel . \
  --set hyperfleet-sentinel-chart.values.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::123456789:role/hyperfleet-role" \
  --set secretProvider.aws.region=us-west-2
```

## Configuration

### Broker Configuration

The broker configuration is passed to hyperfleet-sentinel and uses the [hyperfleet-broker](https://github.com/openshift-hyperfleet/hyperfleet-broker) library:

```yaml
hyperfleet-sentinel-chart:
  values:
    broker:
      type: rabbitmq
      topic: "hyperfleet-sentinel"
      rabbitmq:
        url: ""  # Injected via environment variable
        exchangeType: topic
```

The `BROKER_RABBITMQ_URL` environment variable is automatically populated from the Kubernetes secret created by SecretProviderClass.

### Key Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `hyperfleet-sentinel-chart.applicationName` | Name of the ArgoCD Application | `hyperfleet-sentinel` |
| `hyperfleet-sentinel-chart.namespace` | Namespace for ArgoCD Application resource | `argocd` |
| `hyperfleet-sentinel-chart.targetNamespace` | Namespace where hyperfleet-sentinel will be deployed | `hyperfleet-system` |
| `hyperfleet-sentinel-chart.source.targetRevision` | hyperfleet-sentinel chart version | `v0.1.1` |
| `secretProvider.enabled` | Enable SecretProviderClass creation | `true` |
| `secretProvider.aws.region` | AWS region for Secrets Manager | `us-east-1` |
| `hyperfleet-sentinel-chart.values.serviceAccount.annotations.eks.amazonaws.com/role-arn` | IAM role ARN for Pod Identity | `""` |

## Verification

After installation, verify the ArgoCD Application:

```bash
kubectl get application -n argocd hyperfleet-sentinel
```

Check the SecretProviderClass:

```bash
kubectl get secretproviderclass -n hyperfleet-system
```

Verify the secret was created:

```bash
kubectl get secret -n hyperfleet-system hyperfleet-sentinel-broker-credentials
```

## Troubleshooting

### Secret not populated

Check the CSI driver logs:
```bash
kubectl logs -n kube-system -l app=secrets-store-csi-driver
```

### Pod Identity issues

Verify the service account annotation:
```bash
kubectl get sa -n hyperfleet-system hyperfleet-sentinel -o yaml
```

### Broker connection issues

Check the hyperfleet-sentinel logs:
```bash
kubectl logs -n hyperfleet-system -l app.kubernetes.io/name=hyperfleet-sentinel
```
