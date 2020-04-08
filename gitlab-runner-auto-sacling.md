# Create Auto Scaling Group using GitLab Runner AMI

On this page we will create and configure AWS Auto Scaling Group with our GitLab Runner. The purpose of this is to launch new runner when your pipeline requires more runners. Since we are running `shell` executor the only way to scale our runners is to multiply EC2 instances hosting runners. That's the main purpose to use low memory EC2 types, as one `shell` runner could do only one job at a time.

**Requirements**:

- [GitLab Runner AMI](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-ami.md);

- [gitlab-runner IAM EC2 role](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-iam-ec2-role.md);

- [Admin privileged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `AWS Management Console access`;

- GitLab project.

<br><br>

## Create GitLab Runner Launch Configuration

Before we start, please download [gitlab-runner_user-data.txt file](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/files/gitlab-runner_user-data.txt) and correct it's content by inserting your GitLab project host URL and registration token *(Those can be found same way as described [here](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#register-gitlab-runner))*. Final result of the registration command should look like this:

```
sudo gitlab-runner register -n --url https://gitlab.com/ --registration-token gGH9TSGyDggCygACRqVf --executor shell --description "EC2 runner"
```

Everything else, beside registration command, should be left untouched. Save changes to file, we will need it in the next step.

<br>

- Go to [Launch Configuration management console](https://console.aws.amazon.com/ec2/autoscaling/home?#LaunchConfigurations:);

- `Create Launch Configuration`:

	- Choose an Amazon Machine Image - select `GitLab Runner`, inside `My AMIs` tab;

	![Select gitlab runner AMI](https://user-images.githubusercontent.com/62797411/78579253-543d6480-7839-11ea-859e-ef79a58142e3.png)

	- Choose an Instance Type - select `t2.micro` *(make sure that free tier is available)*;

	- `Next: Configure Details`:

		- Name - Use name related to the GitLab project. For example, if you project called `example`, set Launch Group name as `example gitlab-runner`;

		- IAM role - Select `gitlab-runner`;

		- Advanced Details:

		  - User-Data - `As file`, select the `gitlab-runner_user-data.txt` file;

		  - IP Address Type - `Do not assign a public IP address to any instances.`

	- `Next: Add Storage`;

	- `Next: Configure Security Group` *(If you have already [launched EC2 instance using AMI](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-ami.md#instantiate-gitlab-runner-using-ami), then you could select `gitlab-runner` security group)*:

		- Select `Create a new security group`;

		- Security group name - `gitlab-runner`;

		- Description - `Disallow any incoming connections`;

		- Remove any rule from the list;

	- `Review`;

	- `Create Launch Configuration`:

	- Select `Proceed without a key pair`.

![GitLab Runner launch configuration created](https://user-images.githubusercontent.com/62797411/78756192-10567680-7983-11ea-873e-946fc194f58c.png)

Since Launch Configuration is successfully created, choose `Create an Auto Scaling group using this launch configuration` option. THat will bring us directly to hte next step.

<br><br>

## Create GitLab Runner Auto Scaling Group

- `Configure Auto Scaling group details`:

	- Group name - Use name related to the GitLab project. For example, if you project called `example`, set Launch Group name as `example gitlab-runner`;

	- Group size - `Start with 3 instances`;

	- Network - Select your default VPC;

	- Subnet - Select all available subnets in the list;

	![configure gitlab runner auto scaling group details](https://user-images.githubusercontent.com/62797411/78756974-60820880-7984-11ea-9135-0645e7537003.png)

- `Next: Configure scaling policies`:

	- Choose `Use scaling policies to adjust the capacity of this group`:

		- Scale between `3` and `10` instances. These will be the minimum and maximum size of your group.

		- Scale Group Size:

			- Name - `example gitlab-runner`;

			- Metric type - `Average CPU Utilization`;

			- Target value - `40` *(That means, that new runner will be launched if all of existant instances will be loaded more than 40%)*;

	![Configure GitLab Runner scaling policies](https://user-images.githubusercontent.com/62797411/78757391-164d5700-7985-11ea-8a47-a41fd6bedf51.png)

- `Next: Configure Notifications`;

- `Next: Configure Tags`:

	- `Name` - `gitlab-runner`;

	- `Project` - paste url of your GitLab project or group *(That will help you separate runners for different projects later)*;

- `Review`;

- `Create Auto Scaling group`;

![Create GitLab Runner Auto Scaling Group](https://user-images.githubusercontent.com/62797411/78757821-c622c480-7985-11ea-854f-fcd208168f48.png)

<br>

Once the Auto Scale group initialized, you will see three new runners registered in your projects Settings > CI/CD > Runners:

![GitLab Runner Auto Scale group complete](https://user-images.githubusercontent.com/62797411/78760817-5a8f2600-798a-11ea-8626-38671c3fa67f.png)

--- 

Congratulations! You have created GitLab Runner Auto Scaling group. May be you will find following links interesting: 

- [Configure GitLab CI/CD](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-cicd.md);

- [Configure Swarm Cluster with EC2 instances](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster.md);

- [Back to the main page](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/README.md).