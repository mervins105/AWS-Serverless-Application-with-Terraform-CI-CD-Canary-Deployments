# AWS Serverless Application with CI/CD, Canary Deployment & Automatic Rollback

A multi-service web application running on a fully serverless architecture with **AWS Lambda (container images)** and **Amazon API Gateway**. All infrastructure is provisioned with **modular Terraform**, and deployments run through **CodePipeline, CodeBuild and CodeDeploy** with **canary releases and automatic rollback**.

## Highlights

- **Serverless runtime**: each service runs as a Lambda container image behind its own HTTP API, with Express and React/Nginx servers running unchanged through the [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter).
- **Custom domains**: Route 53 alias records, ACM certificates, and API Gateway custom domains, including one service served from the apex domain.
- **Infrastructure as code**: reusable Terraform modules driven by a single `services` map per environment, with separate dev and prod environments and remote state in S3.
- **Safe deployments**: every release publishes an immutable Lambda version; CodeDeploy shifts the `live` alias gradually (canary) and rolls back automatically if a CloudWatch alarm fires.
- **Low latency**: Provisioned Concurrency on the `live` alias keeps warm instances ready and removes cold starts.

## Architecture

| Layer | Services |
|---|---|
| Compute | AWS Lambda (container images, versions, alias, Provisioned Concurrency) |
| API and networking | Amazon API Gateway (HTTP API), Route 53, AWS Certificate Manager |
| Storage | Amazon S3 (images, Terraform state), Amazon ECR (container images) |
| CI/CD | AWS CodePipeline, CodeBuild, CodeDeploy, CodeConnections (GitHub) |
| Monitoring | Amazon CloudWatch (logs and error alarms) |
| Security | AWS IAM (least-privilege roles for Lambda, CodeBuild, CodePipeline) |
| IaC | Terraform |
| Database | MongoDB Atlas |

### Request flow

1. A user opens a custom domain (for example `api-property.example.com`).
2. Route 53 resolves it to the API Gateway custom domain, secured with an ACM certificate.
3. API Gateway forwards every request through a single catch-all route (`ANY /{proxy+}`) to the `live` alias of the service's Lambda function.
4. The Lambda Web Adapter hands the request to the Express (or Nginx) server inside the container, which does its own routing.
5. The service reads and writes MongoDB Atlas and S3.

Each backend gets **its own API Gateway and domain** because the services share the same `/api/...` path style, so one gateway could not tell them apart without listing every route. HTTP APIs are billed per request, so separate gateways cost the same as one.

### Deployment flow

1. A push to GitHub triggers CodePipeline.
2. **CodeBuild** builds the Docker image, tags it with a timestamp, pushes it to ECR, updates the function code, and publishes a new Lambda version.
3. The build generates an `appspec.yaml` containing the current and target versions.
4. **CodeDeploy** shifts the `live` alias from the old version to the new one using a canary configuration (for example 10% of traffic for 5 minutes, then 100%).
5. A **CloudWatch alarm** on the function's `Errors` metric is attached to the deployment group. If it fires during the canary window, CodeDeploy reverts the alias to the previous version automatically.

## Repository structure

```text
.
├── Environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   ├── provider.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   └── prod/                 # same layout as dev
└── modules/
    ├── lambda/               # function, version, alias, Provisioned Concurrency
    ├── api-gateway/          # HTTP API, catch-all route, stage, Lambda permission
    ├── route53/              # ACM certificate and DNS validation
    └── custom-domain/        # API Gateway domain, mapping, alias record
```

Each service repository also contains:

```text
├── Dockerfile
└── buildspec.yml             # build, push, publish version, generate appspec.yaml
```

## Getting started

### Prerequisites

- AWS account with a Route 53 hosted zone for your domain
- Terraform 1.6 or later
- AWS CLI configured with credentials that can create the resources above
- Container images already pushed to ECR (one repository per service)

### 1. Create the remote state bucket (once)

```bash
aws s3api create-bucket \
  --bucket <your-unique-state-bucket> \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

aws s3api put-bucket-versioning \
  --bucket <your-unique-state-bucket> \
  --versioning-configuration Status=Enabled
```

### 2. Configure an environment

Edit `Environments/dev/terraform.tfvars` and define every service in one map:

```hcl
aws_region     = "ap-south-1"
root_domain    = "example.com"
hosted_zone_id = "<route53-hosted-zone-id>"

services = {
  property-service = {
    image_uri               = "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/property:dev"
    lambda_role_arn         = "arn:aws:iam::<account-id>:role/<lambda-execution-role>"
    subdomain               = "dev-api-property.example.com"
    memory_size             = 512
    timeout                 = 60
    ephemeral_storage       = 512
    provisioned_concurrency = 1
    environment_variables = {
      PORT = "8080"
    }
  }
  # add one block per service
}
```

`tfvars` files only accept literal values, so type the domain out instead of referencing `var.root_domain`.

### 3. Deploy

```bash
cd Environments/dev
terraform init
terraform plan
terraform apply
```

Add a new service by adding one more block to `services`. No `.tf` changes are needed.

### 4. Set up the pipeline

1. Create the CloudWatch alarm on the function's `Errors` metric.
2. Create a CodeDeploy application (compute platform: Lambda) and a deployment group with a canary configuration, attaching the alarm for rollback.
3. Create a CodeBuild project with privileged mode enabled and these environment variables: `AWS_ACCOUNT_ID`, `AWS_DEFAULT_REGION`, `REGISTRY_NAME`, `SERVICE_NAME`, `FUNCTION_NAME`, `ALIAS_NAME`.
4. Create a CodePipeline with Source (GitHub through CodeConnections), Build (CodeBuild) and Deploy (CodeDeploy) stages.

## Design notes

- **Reserved Lambda variables**: `AWS_REGION` and the AWS credentials cannot be set manually. The function's execution role provides credentials, so the S3 client is created without an explicit `credentials` block.
- **Lambda is not persistent**: it has no rolling update like ECS. Safe rollouts come from immutable versions plus a weighted alias, which is what the canary deployment uses.
- **Real-time features**: WebSocket servers such as Socket.IO need a persistent process, so they do not fit on Lambda and must run elsewhere.
- **Terraform drift**: changing a function's image outside Terraform (for example through the pipeline) creates drift. Decide whether Terraform or the pipeline owns the image tag, and run `terraform plan` before every `apply`.
- **Provisioned Concurrency** bills continuously per function, regardless of traffic.

## Cost considerations

- Lambda, API Gateway and CloudWatch are billed per use.
- Provisioned Concurrency is a fixed hourly cost per configured instance.
- ECR and S3 storage costs are small but grow with image history; add lifecycle rules to expire old images.

