# Configure Amazon EC2 instance to run Docker Swarm node

In this guide we will create cluster of the **Amazon EC2** instances under control of **Docker Swarm**, that can be used to publish microservice based systems. All necessary orchestration functions, like horizontal scaling, private networking, DNS resolvation and even load balancing, Docker Swarm gives out of the box. Automatic node scaling could be achived with AWS Auto Sacling groups so that as the result we will have ready to use universal flexible autosclable cluster.

Cluster management suppose to be organized through one or multiple swarm managers via SSH so that deployment, update and even rollback processes could be fully automated, for example with CI/CD piplines. The code precessing by CI/CD pipelines is suppose to be build, tested and deployed as the docker images, that are stored in the **Amazon Elastic Container Registry** (**ECR**).

<br>

**Requirements:**

- [Admin privileged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `AWS Management Console access`;

- [SSH key pair](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-ssh-key-pair.md);

- Any Docker image stored in ECR *(if you still don't have any, use [this](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/awscli-ecr-push-image.md) to push one)*;

- [Docker Swarm worker node IAM role](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-iam-ec2-role.md);

- [AWS Network configured to Docker Swarm cluster](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-network.md);

> Free tier resources are used in this instruction. Make sure that you account have been created less than 12 months ago. This applies to EC2 [t2.micro](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&all-free-tier.q=t2.micro&all-free-tier.q_operator=AND) as well as [ECR](https://aws.amazon.com/ecr/pricing/).