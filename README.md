```markdown
# Setting Up Karpenter on AWS EKS

This guide walks you through the process of installing and configuring Karpenter on an AWS EKS cluster. Follow the steps below to set up the required tools, create a cluster, and deploy Karpenter.

## Prerequisites

Before proceeding, ensure you have the following tools installed:

1. **kubectl**: The Kubernetes command-line tool.
2. **eksctl**: The CLI for AWS EKS (version >= v0.202.0).
3. **helm**: The package manager for Kubernetes.

Install these tools using your preferred package manager or follow the official documentation for each.

## Step 1: Set Environment Variables

Set the following environment variables to configure Karpenter and the Kubernetes version:

```bash
export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="1.4.0"
export K8S_VERSION="1.32"
export AWS_PARTITION="aws"
export CLUSTER_NAME="${USER}-karpenter-demo"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT="$(mktemp)"
export ALIAS_VERSION="$(aws ssm get-parameter --name "/aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2023/x86_64/standard/recommended/image_id" --query Parameter.Value | xargs aws ec2 describe-images --query 'Images[0].Name' --image-ids | sed -r 's/^.*(v[0-9]+).*$/\1/')"
```

Verify the variables by echoing them:

```bash
echo "${KARPENTER_NAMESPACE}" "${KARPENTER_VERSION}" "${K8S_VERSION}" "${CLUSTER_NAME}" "${AWS_DEFAULT_REGION}" "${AWS_ACCOUNT_ID}" "${TEMPOUT}" "${ARM_AMI_ID}" "${AMD_AMI_ID}" "${GPU_AMI_ID}"
```

## Step 2: Create an EKS Cluster

Deploy the necessary CloudFormation stack and create the EKS cluster.

1. Download the CloudFormation template and deploy it:

```bash
cp karpenter-prerequisite.yaml "${TEMPOUT}"

aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

2. Apply the `eksctl` configuration to create the cluster:

```bash
envsubst < eksctl.yaml | kubectl apply -f -
```

3. Retrieve the cluster endpoint and Karpenter IAM role ARN:

```bash
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name "${CLUSTER_NAME}" --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
echo "${CLUSTER_ENDPOINT} ${KARPENTER_IAM_ROLE_ARN}"
```

4. Create a service-linked role for Spot instances (if it doesn't already exist):

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

## Step 3: Install Karpenter

Install Karpenter using Helm:

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "${KARPENTER_NAMESPACE}" \
  --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

## Step 4: Create NodePool and EC2NodeClass

Apply the NodePool and EC2NodeClass configurations:

```bash
envsubst < nodepool.yaml | kubectl apply -f -
envsubst < ec2_nodeclass.yaml | kubectl apply -f -
```


