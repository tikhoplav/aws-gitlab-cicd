# Configure amazon EC2 instance for GitLab Runner

In this guide we will lunch and set up **Amazon Web Service EC2 instance** to host **GitLab Runner**.

Runners created following this instruction are abile to [use docker daemon](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html) as well as to pull and push this images to the ECR during CI/CD pipeline.

Current method can be applied to create specific project runner as well as group runner.

<br>

**Requirements**:

- GitLab project;

- [Admin priveleged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `AWS Management Console access`;

- [SSH key pair](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-ssh-key-pair.md);

- [gitlab-runner IAM EC2 role](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-iam-ec2-role.md);

- Any Docker image stored in ECR *(if you still don't have any, use [this](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/awscli-ecr-push-image.md) to push one)*;

> Free tier resources are used in this instruction. Make sure that you account have been created less than 12 months ago. This applies to EC2 [t2.micro](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&all-free-tier.q=t2.micro&all-free-tier.q_operator=AND) as well as [ECR](https://aws.amazon.com/ecr/pricing/).

<br><br><br>

## Create EC2 instance

At this step we will create a prototype EC2 instance. Later we will use this to create custom AMI in order to minimize time consumption for each new GitLab Runner.

- Go to [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances);

- `Launch instance`:

  - Choose an Amazon Machine Image - select `Amazon Linux 2 AMI`;

![Amazon Linux 2 AMI selection](https://user-images.githubusercontent.com/62797411/78602731-9b3e5080-785f-11ea-9d84-835ff5803312.png)

  - Choose an Instance Type - select `t2.micro`;

- `Next: Configure Instance Details`:

  - Auto-assign Public IP - `Enable` *(or `Use subnet settings (Enable)`)*;

  - IAM role - select `gitlab-runner` *(if you haven't got this role, create one using [this](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-iam-ec2-role.md))*;

  - Leave everything else by default;

- `Next: Add Storage`:

- `Next: Add Tags`:

  - `Name` *(With capital N)*- `proto gitlab-runner`;

![adding tags to ec2 instance](https://user-images.githubusercontent.com/62797411/78602242-c96f6080-785e-11ea-962b-90a0b4b09a6a.png)

- `Next: Configure Security Group`:

  - You could create new or existent default security group. The main thing is that SSH connection *(TCP on poet 22)* should be allowed;

![security group selection for gitlab runner prototype](https://user-images.githubusercontent.com/62797411/78607429-b44aff80-7867-11ea-91a1-d900204de120.png)

- `Next: Review and Launch`;

- `Launch`:

  - Choose your existent key pair or create new one.

<br><br><br>