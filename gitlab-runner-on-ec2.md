# Organize GitLab Runner on AWS EC2 instance

This page is a step-by-step instruction about creation,
connection and hosting a **GitLab Runner** using **Amazon Web Service EC2** instance.

In order to test capabilities of this approach, only a free tier resources are used in this instruction.
To be able to use free tier, make sure that you account have been created less than 12 months ago. This applies to EC2 [t2.micro](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&all-free-tier.q=t2.micro&all-free-tier.q_operator=AND) as well as [ECR](https://aws.amazon.com/ecr/pricing/).

Runners created following this instruction gives ability to [build docker images](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html)
as well as to push this images to the ECR during CI/CD pipeline.

**Prerequisites:**
 - GitLab project;
 - AWS account;
 
 Current method can be applied to create specific project runner as well as group runner.

## Create GitLab Runner IAM

At this step we will create role that will be injected to EC2 instance. This role identifies capabilities of our instance to use other AWS resources. For example, we will grant our GitLab Runner instance ability to push and pull Docker images to ECR.

- Go to [roles management console](https://console.aws.amazon.com/iam/home#/roles);
- `Create Role`:
  - Select type of trusted entity - select `AWS service`;
  - Choose a use case - select `EC2 - Allows EC2 instances to call AWS services on your behalf`;
- `Next: Permissions`;
  - Attach permissions policies - paste `EC2Container` to the search bar and select *(checkbox)* `AmazonEC2ContainerRegistryFullAccess`;
- `Next: Tags`;
  - *(Optional)* Here you can add key-value pairs to identify role in the list. I suggest to set tag `Name` with value `gitlab-runner`;
- `Next: Review`;
  - Role name - Clear names related to use cases highly suggested, for example `gitlab-runner`;
  - Role description *(Optional)* - `Allows EC2 instance to pull and push Docker images to ECR`;
- `Create role`.

Later, if you would like to grant more permissions to GitLab Runner instance, for example, to use **S3** to cache builds etc, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/gitlab-runner). New permissions would automatically applied to all running instances using this role.

## Create EC2 instance

## Prepare EC2 instance

### Install Docker

### Install Amazon-ECR-Credential-helper

### Create permanent Docker ECR login

### Install GitLab Runner

### Establish automatic unregister script on instance termination

## Create GitLab Runner AMI

## Lunch GitLab Runner EC2 instance

## Test Runner
