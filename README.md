This repository contains some tests about the new configuration of `opentelemetry-collector-contrib` agent to send metrics to Amazon Managed Prometheus after the deprecation notice of the `awsprometheusremotewrite` exporter.

# 1. Context

Since version 0.48 of `opentelemetry-collector-contrib`, there is a deprecation notice about the use of the `awsprometheusremotewrite` exporter. This exporter is almost identical to the `prometheusremotewrite` exporter with the exception that it also handles AWS Sigv4 signature of the requests.

`opentelemetry-collector-contrib` now offers an authentication extension named `sigv4authextension` that can be configured to sign requests of the standard `prometheusremotewrite` exporter with AWS Sigv4. It is thus not needed anymore to maintain the dedicated exporter.

# 2. Test

Tests described in this repository are done using a local `k3d` cluster.

Create a test cluster

```bash
k3d cluster create testotelaws
```

Create an Amazon Managed Prometheus (AMP) workspace for the test.

> Note : you need to configure your aws cli before running those commands

```bash
aws amp create-workspace
arn: arn:aws:aps:eu-west-1:{your_aws_account}:workspace/{your_amp_workspace_id}
status:
  statusCode: CREATING
tags: {}
workspaceId: {your_amp_workspace_id}

# Check status of the workspace - if ACTIVE then it's ready for use
aws amp describe-workspace --workspace-id {your_amp_workspace_id}
workspace:
  arn: arn:aws:aps:eu-west-1:{your_aws_account}:workspace/{your_amp_workspace_id}
  createdAt: '2022-06-01T09:09:51.472000+02:00'
  prometheusEndpoint: https://aps-workspaces.eu-west-1.amazonaws.com/workspaces/{your_amp_workspace_id}/
  status:
    statusCode: ACTIVE
  tags: {}
  workspaceId: {your_amp_workspace_id}
```

Now that we have an active AMP workspace, we need to create an IAM user that will be allowed to write metrics to the workspace.

```bash
# Create an IAM user
aws iam create-user --user-name test-otelcol-write

# Create an access key for that user
# Note both AccessKeyId and SecretAccessKey and secure them
aws iam create-access-key --user-name test-otelcol-write
AccessKey:
  AccessKeyId: AKIAY2OQCEXAMPLE
  CreateDate: '2022-06-01T07:28:54+00:00'
  SecretAccessKey: s0EwfYh9IDXVEXAMPLE
  Status: Active
  UserName: test-otelcol-write

# Find the ARN of the AmazonPrometheusRemoteWriteAccess policy
export POLICYARN=$(aws iam list-policies --query 'Policies[?PolicyName==`AmazonPrometheusRemoteWriteAccess`].{ARN:Arn}' --output text)
echo $POLICYARN
arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess

# Attach the policy to the user
aws iam attach-user-policy --user-name test-otelcol-write --policy-arn $POLICYARN

# check that the policy is correctly attached
aws iam list-attached-user-policies --user-name test-otelcol-write
AttachedPolicies:
- PolicyArn: arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess
  PolicyName: AmazonPrometheusRemoteWriteAccess
```

> *Caution : this is a very fast way of testing stuff. Do not use those instructions for a production setup !*

Create a namespace for the deployment of `opentelemetry-collector`

```bash
kubectl create namespace opentelemetry-collector
```

Create a secret that contains AWS credentials created before.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: amp-credentials
stringData:
  aws_access_key_id: 'AKIAY2OQCEXAMPLE'
  aws_secret_access_key: 's0EfwYh9IDXVEXAMPLE'
```

Deploy `opentelemetry-collector-contrib` in the cluster. Use the `Helm` chart.

```bash
# Add the opentelemetry helm chart repository
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

# Install the Helm chart with the customized values found in test-toelcol-aws-values.yml
helm upgrade --install --namespace opentelemetry-collector \
--values test-otelcol-aws-values.yml opentelemetry-collector \
open-telemetry/opentelemetry-collector
```

Deploy a sample application that produces Prometheus metrics

```bash
kubectl apply -f promtest-deployment.yml --namespace opentelemetry-collector
```

Run Grafana in Docker to check if metrics are correctly ingested

```bash
docker container run -d -p 3000:3000 \
-e "GF_AUTH_SIGV4_AUTH_ENABLED=true" \
grafana/grafana-oss:8.5.4
```

Configure the datasource and check that metrics are ingested and available for queries ;)

# 3. Cleanup resources

Delete the Grafana container

```bash
# Find the id of the grafana container
docker container ps

# Stop the grafana container
docker container stop {container id}

# Remove the grafana container
docker container rm {container id}
```

Delete the k3d cluster

```bash
k3d cluster delete testotelaws
```

Delete the AMP workspace

```bash
aws amp delete-workspace --workspace-id {your_amp_workspace_id}

# Check that the amp workspace is deleted
aws amp list-workspaces
workspaces: []
``` 

Delete the AWS IAM credentials

```bash
# First detach the policy
aws iam detach-user-policy --user-name test-otelcol-write --policy-arn $POLICYARN

# Then delete the access key
aws iam delete-access-key --user-name test-otelcol-write --access-key-id AKIAY2OQCEXAMPLE

# Delete the user ... delete fails if other 2 steps are not executed first
aws iam delete-user --user-name test-otelcol-write
```
