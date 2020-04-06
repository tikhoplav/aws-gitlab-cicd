# Create GitLab Runner AMI

On this page we will create GitLab Runner AMI in order to minimize time required to start new runner.

**Requirements:**

- [Configured EC2 instance to host GitLab Runner](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md).

<br>

- Go to [instance management console](https://console.aws.amazon.com/ec2/v2/home?#Instances:sort=desc:launchTime);

- Find one with name `proto gitlab-runner` (that one, that we have created and prepared earlier);

- `Action` > `Image` > `Create Image`:

  - Image name - `GitLab Runner`;

  - Image descriotion - `Amazon Linux 2 based. Contains Docker (v19.03.6), Amazon ECR Docker Credentials Helper, GitLab Runner (v12.9.0)`;

- `Create Image`.

<br>

After, you will be able to see new AMI in [images management console](https://console.aws.amazon.com/ec2/v2/home?#Images:sort=name). Time required to set image `ready` depends on the used hardware space do our instance. With 8 Gb SSD it requires several minutes.

![New AMI example](https://user-images.githubusercontent.com/62797411/78580235-c8c4d300-783a-11ea-9c00-5dc6fc43a6e0.png)

> In the description I have mentioned versions of software that we have installed. This versions may differs with yours. To get versions of Docker and GitLab Runner you can run next command:
>
> ```
> $ sudo docker info | grep "Server Version"
> $ sudo gitlab-runner --version | grep "Version"
> ```

<br><br><br>

## Instantiate GitLab Runner using AMI

Before we start, please download [gitlab-runner_user-data.txt file](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/files/gitlab-runner_user-data.txt) and correct it's content by inserting your GitLab project host URL and registration token *(Those can be found same way as described [here](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#register-gitlab-runner))*. Final result of the registration command should look like this:

```
sudo gitlab-runner register -n --url https://gitlab.com/ --registration-token gGH9TSGyDggCygACRqVf --executor shell --description "EC2 runner"
```

Everything else, beside registration command, should be left untouched. Save changes to file, we will need it in the next step.

<br>

Now it is time to lunch new runner using AMI that we have created:

- Go to [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances);

- `Launch instance`:

  - Choose an Amazon Machine Image - select `GitLab Runner`, inside `My AMIs` tab;
  
  ![Select gitlab runner AMI](https://user-images.githubusercontent.com/62797411/78579253-543d6480-7839-11ea-859e-ef79a58142e3.png)
  
  - Choose an Instance Type - select `t2.micro` *(make sure that free tier is available)*;
  
- `Next: Configure Instance Details`:

  - Auto-assign Public IP - `Enable` *(or `Use subnet settings (Enable)`)*;

  - IAM role - select `gitlab-runner` *(if you haven't got this role, create one using [this](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-iam-ec2-role.md))*;

  - Go to Advanced Details section:

    - User-data - set `As filetext` and select `gitlab-runner_user-data.txt`;
    
  - Leave everything else by default;
  
- `Next: Add Storage`:

- `Next: Add Tags`:

  - `Name` - `gitlab-runner`;

  - `Project` - paste url of your GitLab project or group *(That will help you separate runners for different projects later)*;
  
- `Next: Configure Security Group`:

  - Select `Create a new security group`;

  - Security group name - `gitlab-runner`;

  - Description - `Disallow any incoming connections`;

  - Remove any rule from the list;
  
- `Next: Review and Launch`;

- `Launch`:

  - Select `Proceed without any key`.

---

Congratulations! You have created GitLab Runner AMI. I suggest you to go through the next step:

- [GitLab Runner Auto Scaling Group Creation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-auto-scaling.md) - To launch new runners for your project automatically on heavy CI/CD pipeline load.