## **Advance CI/CD Infrastructure(Github Action) Documentation**

Comprehensive guide for the reusable workflow, and composite action powering the Rideshare platform CI/CD pipeline.

## **Architecture Overview**

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions                           │
│                                                             │
│  service-repo/.github/workflows/ci-cd.yml                  │
│         │                                                   │
│         │  uses: (reusable workflow)                        │
│         ▼                                                   │
│  devops-platform-ci/.github/workflows/ci.yml               │
│         │                                                   │
│         │  uses: (composite action)                         │
│         ▼                                                   │
│  devops-platform-ci/.github/actions/docker-build-push/     │
└─────────────────────────────┬───────────────────────────────┘
                              │  runs-on: self-hosted (EKS)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      EKS Cluster                            │
│                                                             │
│  arc-systems namespace                                      │
│  ├── arc-gha-rs-controller (ARC operator)                   │
│  ├── kadiri-rider-runner-listener                           │
│  ├── kadiri-driver-runner-listener                          │
│  └── kadiri-trip-runner-listener                            │
│                                                             │
│  kadiri-runner namespace                                    │
│  └── runner pods (ephemeral, spawned per job)               │
│      ├── container: runner (GitHub Actions runner)          │
│      └── container: dind   (Docker daemon)                  │
└─────────────────────────────┬───────────────────────────────┘
                              │  push image
                              ▼
                      ┌──────────────┐
                      │   AWS ECR    │
                      │  (per svc)   │
                      └──────────────┘

```

## **Repository Structure**

```
devops-platform-ci/
└── .github/
    ├── workflows/
    │   └── ci.yml                            # Reusable workflow
    └── actions/
        └── docker-build-push/
            └── action.yml                    # Composite action
```
## **Service Repositories (e.g., rider-service)**

```
rider-revamp/
└── .github/
    └── workflows/
        └── ci-cd.yml                         # Caller workflow (thin wrapper)
```

## **Reusable Workflow**

What It Does
The reusable workflow is the single source of truth for all CI/CD logic across every service. Instead of duplicating pipeline code in each repository, every service calls this one workflow.

```
app1/.github/workflows/ci-cd.yml  ─┐
app2/.github/workflows/ci-cd.yml  ─┼──► devops-platform-ci/.github/workflows/ci.yml
app3/.github/workflows/ci-cd.yml  ─┘         │
                                              ├── Job 1: lint-and-test
                                              └── Job 2: build-and-push
                                                          │
                                                          └── composite action
```
## **Pipeline stages:**

* Lint — ESLint checks code quality
* Test — Jest runs unit tests
* Build — Docker image built with commit SHA tag
* Push — Image pushed to AWS ECR

Jobs 3 and 4 only run if jobs 1 and 2 pass (needs: lint-and-test).

## ** Inputs and Secrets*

Inputs:

| Input | Required | Description |
| ----------- | ----------- | ---------- |
| service-name | Yes| ECR repository name (e.g., kadiri/rider-service) |
| eks-deployment-name | No| Kubernetes deployment name for future CD steps|

Secrets:

| Secret | Required | Description |
| ----------- | ----------- | ---------- |
| AWS_ACCESS_KEY_ID | Yes| AWS IAM access key |
| AWS_SECRET_ACCESS_KEY | Yes | AWS IAM secret key |
| AWS_REGION | Yes | AWS region (e.g., eu-north-1) |
| EKS_CLUSTER_NAME | No | EKS cluster name for future CD steps |

## **How to Call It**

```
# your-service/.github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  pipeline:
    uses: khadree/devops-platform-ci/.github/workflows/ci.yml@main
    with:
      service-name: kadiri/rider-service
      eks-deployment-name: rider-service
    secrets: inherit
```
secrets: inherit automatically passes all repository secrets to the reusable workflow — no need to list them individually.

Caller workflow is intentionally thin. All logic lives in the reusable workflow. The caller only provides service-specific configuration.

## **Composite Action**

Location: devops-platform-ci/.github/actions/docker-build-push/action.yml
What It Does
The composite action encapsulates all Docker build and ECR push logic into a single reusable step. It is called by the reusable workflow and can also be used independently in any job.

Step 1: Configure AWS credentials
Step 2: Login to Amazon ECR
Step 3: docker build -t <ecr-registry>/<service>:<commit-sha> .
Step 4: docker push <ecr-registry>/<service>:<commit-sha>
Step 5: Output the full image URI

Images are tagged with the full Git commit SHA for exact traceability:

123456789.dkr.ecr.eu-north-1.amazonaws.com/kadiri/rider-service:abc1234def5678

## **How to Use It**

How to Use It
Inside the reusable workflow (standard usage):

```
- name: Build and push Docker image
  id: docker
  uses: khadree/devops-platform-ci/.github/actions/docker-build-push@main
  with:
    service-name: ${{ inputs.service-name }}
    aws-region: ${{ secrets.AWS_REGION }}
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

# Access the output in subsequent steps
- name: Print image URI
  run: echo "Pushed ${{ steps.docker.outputs.image-uri }}"
```
Directly in any workflow:

```
jobs:
  build:
    runs-on: kadiri-rider-runner
    steps:
      - uses: actions/checkout@v4
      - uses: khadree/devops-platform-ci/.github/actions/docker-build-push@main
        with:
          service-name: my-service
          aws-region: eu-north-1
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```
## **Minimum IAM permissions required:**

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    }
  ]
}
```

**Author**: Kadiri George 
**Version**: 1.0.0  
**Last Updated**: May 2026