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

The purpose of giving a GitLab runner full access to ECR is that give you an ability to delete repositories through CI/CD pipeline. For example, you could create a special job, that will unregister you microservice, clean up and set free all resources. If you feel that this permissions is more than you can allow runner to have, you could use `AmazonEC2ContainerRegistryPowerUser` policy instead.

Later, if you would like to grant more permissions to GitLab Runner instance, for example, to use **S3** to cache builds etc, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/gitlab-runner). New permissions would automatically applied to all running instances using this role.

## Create EC2 instance

At this step we will create a prototype EC2 instance. Later we will use this to create custom AMI in order to minimize time consumption for each new GitLab Runner.

- Go to [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances);
- `Launch instance`:
  - Choose an Amazon Machine Image - select `Amazon Linux 2 AMI`;
  - Choose an Instance Type - select `t2.micro` *(make sure that free tier is available)*;
- `Next: Configure Instance Details`:
  - Auto-assign Public IP - `Enable` *(or `Use subnet settings (Enable)`)*;
  - IAM role - select `gitlab-runner`;
  - Leave everything else by default;
- `Next: Add Storage`:
- `Next: Add Tags`:
  - *(Optional)* I suggest to add `Name` tag with value - `proto gitlab-runner`;
- `Next: Configure Security Group`:
  - Select `Create a new security group`;
  - Security group name - `proto gitlab-runner`;
  - Description - `Allows SSH access to gitlab-runner prototype instance`;
  - Make sure that rule for port `22` and source `0.0.0.0/0` exists;
- `Next: Review and Launch`;
- `Launch`:
  - Here you will be asked to use existent or to create a new key pair. I suggest you to create special key pair for GitLab Runner prototype, but you could use one key pair for all of your future instances, or even use existent. It is ok, since access to working GitLab Runners will be restricted anyway.

After this you will see new ec2 instance in you instances console. For later steps we will have to establish SSH connection with prototype using `.pem` key file. For simplicity, let's say that the key is called `ec2.pem`. Make sure that `ec2.pem` file have permissions lower or equal to 600, otherwise SSH connection will fail.

## Prepare EC2 instance

After your new instance receive status `running`, you should be able to connect it via SSH. But before we need to figure out it's public ip address. This can be done by clicking our `proto gitlab-runner`, and search graph called `IPv4 Public IP` in the details.

All of the following commands in this section should be executed at runner instance. To do so, next command can be used to establish SSH connection:
```
ssh -i <path to .pem file>/ec2.pem ec2-user@<proto gitlab-runner public ip>
```

### Install Docker

### Install Amazon-ECR-Credential-helper

### Create permanent Docker ECR login

### Install GitLab Runner

### Establish automatic unregister script on instance termination

## Create GitLab Runner AMI

## Lunch GitLab Runner EC2 instance

## Test Runner
