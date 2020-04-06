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

<br><br><br>

## Create GitLab Runner IAM Role

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

<br>

Later, if you would like to grant more permissions to GitLab Runner instance, for example, to use **S3** to cache builds etc, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/gitlab-runner). New permissions would automatically applied to all running instances using this role.

<br><br><br>

## Create EC2 instance

At this step we will create a prototype EC2 instance. Later we will use this to create custom AMI in order to minimize time consumption for each new GitLab Runner.

- Go to [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances);
- `Launch instance`:
  - Choose an Amazon Machine Image - select `Amazon Linux 2 AMI`;
  - Choose an Instance Type - select `t2.micro` *(make sure that free tier is available)*;
- `Next: Configure Instance Details`:
  - Auto-assign Public IP - `Enable` *(or `Use subnet settings (Enable)`)*;
  - IAM role - select `gitlab-runner` [that we created earlier](https://github.com/tikhoplav/AWS-Gitlab-CICD/blob/master/gitlab-runner-on-ec2.md#create-gitlab-runner-iam-role);
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

<br>

After this you will see new ec2 instance in you instances console. For later steps we will have to establish SSH connection with prototype using `.pem` key file. For simplicity, let's say that the key is called `ec2.pem`. Make sure that `ec2.pem` file have permissions lower or equal to 600, otherwise SSH connection will fail.

<br><br><br>

## Prepare EC2 instance

#### Install Docker
```
$ sudo yum -y update \
  && sudo amazon-linux-extras install -y docker \
  && sudo systemctl enable docker \
  && sudo service docker start
```
Now we can test that docker is running by executing:
```
$ sudo docker info
```

<br><br>

#### Esablish ECR connection

Before we can push and pull Docker images to ECR, we have to login our Docker daemon to ECR registry. Normally we aquire token that lasts 12 hours with `awc-cli`. In order to let GitLab Runner use ECR constantly we need to do the following steps:

[Amazon ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) will allow EC2 instance push and pull images to ECR, since it requires special authentification sequence. Let's install it by running:

```
$ sudo yum install -y amazon-ecr-credential-helper
```

<br>

Next we need to specify `$IMAGE` and `$REGION`is the parameters of your ECR. You could find that opening [ECR management console](https://console.aws.amazon.com/ecr/).

> For example if you repository looks like this, you could set your variables as following:
>
> ![ECR repository example](https://user-images.githubusercontent.com/62797411/78579678-f65d4c80-7839-11ea-8096-93fed83b6673.png)
>
> ```
> REGION=eu-central-1
> IMAGE=678005261235.dkr.ecr.eu-central-1.amazonaws.com/alpine:latest
> ```
>
> The actual image we are using here is not important, later we will delete it from instance. Only thing is matter, is that this image have to be stored in the ECR you are plannig to use with CI/CD pipeline.

<br>

Now we need to aquire the token and store it with help of credential helper. To achive that we could use [this approach](https://github.com/awslabs/amazon-ecr-credential-helper/issues/63#issuecomment-328318116):

```
$ sudo $(aws ecr get-login --no-include-email --region $REGION) \
  && echo -e "{\n\t\"credsStore\": \"ecr-login\"\n}" | sudo tee /root/.docker/config.json \
  && sudo docker pull $IMAGE
```

<br>

After image is downloaded we can make sure that Docker login is cached, by running following command. The answer have to be similar:

```
$ sudo docker-credential-ecr-login list
{"https://678005261235.dkr.ecr.eu-central-1.amazonaws.com":"AWS"}
```
<br><br>

#### Install GitLab Runner

According to official documentation, we can install GitLab Runner by doing:

```
$ curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/rpm/gitlab-runner_amd64.rpm \
  && sudo yum install -y git \
  && sudo rpm -i gitlab-runner_amd64.rpm
```

To make sure that GitLab Runner is operational:

```
$ journalctl -u gitlab-runner
```

<br>

Now we need to reconfigure GitLab Runner so that it will run jobs as a `root` user.

```
$ sudo sed -i 's/"--user" "gitlab-runner"/"--user" "root"/' /etc/systemd/system/gitlab-runner.service \
  && sudo sed -i 's/"/home/gitlab-runner"/"/tmp"/' /etc/systemd/system/gitlab-runner.service
  && sudo systemctl daemon-reload \
  && sudo service gitlab-runner restart
```

> For what purpose do we need to run GitLab Runner as root?
>
> I have been testing with different settings of GitLab Runner. With current version of it (12.9.0) there are problems with POST requests to GitLab server so that you need to run `sudo gitlab-runner run` manually or with help of other tools (like `cloud-init`).
>
> Since it is highly suggested to use this kind of EC2 instances for hosting GitLab Runner only (Even all ports of it's security group should be disabled), there are the same security issues as if it will run as not a root.
>
> Plus, this way we could allow our runner environment to use ECR much easier *(Take a look to `sudo` in front of every command)*, without files coping and permission management.

<br><br>

#### Create Termination Sequence

Later we will register runners to our projects using this prototype. We have to add some feature to our prototype, in order to unregister this runners automaticaly when instance is going to be terminated. Otherwise, we will have to detach runners from our projects manually *(What doesn't sounds great)*.

To achive that we can use [this](https://www.golinuxcloud.com/run-script-with-systemd-before-shutdown-linux/) approach:

```
$ echo -e "#! /bin/bash\ngitlab-runner unregister --all-runners" | sudo tee /etc/init.d/ec2-terminate.sh \
  && sudo chmod u+x /etc/init.d/ec2-terminate.sh
```

<br>

Next, we will create a `systemd` service, that will run our script at system shutdown:

```
$ sudo vim /etc/systemd/system/ec2-terminate.service
```

With the following content *(`a` to edit file, `esc :x` to save our file and exit)*:

```
[Unit]
Description=EC2 termination sequence
DefaultDependencies=no
Before=shutdown.target

[Service]
Type=oneshot
ExecStart=/etc/init.d/ec2-terminate.sh
TimeoutStartSec=0

[Install]
WantedBy=shutdown.target
```

<br>

Now let's register our new service:

```
$ sudo systemctl enable ec2-terminate.service
Created symlink from /etc/systemd/system/shutdown.target.wants/ec2-terminate.service to /etc/systemd/system/ec2-terminate.service.
```

<br><br>

#### Clean up

I suggest to clean up an instance before creation of the AMI, since free tier of ESB volume usage is limited, or if you not using free tier or broke the limit of monthly usage, every single megabyte is charged.

```
$ sudo userdel -r gitlab-runner \
  && sudo docker rmi $(sudo docker images -q) \
  && sudo yum clean all \
  && sudo rm -rf /var/cache/yum
```

<br><br><br>

## Create GitLab Runner AMI

Before we register our runner and let it take and execute jobs for our pipeline, I highly suggest to create special GitLab Runner AMI. Using this we will minimize time consumtion that is required to set new runner operational, since we will not have to repeat all the actions given in [preparation section](https://github.com/tikhoplav/AWS-Gitlab-CICD/blob/master/gitlab-runner-on-ec2.md#prepare-ec2-instance).

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
> ```
> $ sudo docker info | grep "Server Version"
> $ sudo gitlab-runner --version | grep "Version"
> ```

<br><br><br>

## Instantiate GitLab Runner using AMI

Since our AMI is ready, we could now run new instance and configure it's startup so that GitLab Runner will register itself to our project. But before that let's configre our project and aquire registration token and url for the runner registration:

- Go to your GitLab project setting > CI/CD > Runners *(If you are planning to make group runner go to your group settings > CI/CD > Runners)*;
- Disable shared runner by clicking `disable shared runners` button;
- Find `URL` to register new runner;
- Find `Registration token` to register new runner.

![GitLab Runner settings](https://user-images.githubusercontent.com/62797411/78564461-21897100-7825-11ea-8926-6e89d3b1f9be.png)

> Why do we disable shared runners?
>
> At first, using shared runners makes your data vulnerable. More info about it you can find [here](https://stackoverflow.com/questions/35527655/what-are-the-security-risks-of-using-gitlab-ci-shared-test-runners) and [here](https://gitlab.com/gitlab-org/gitlab-runner/issues/4430).
>
> At second, shared runner are shared between all users of the GitLab CI/CD pipelines. That means that you will have to wait until shared runner gets free to run your jobs.
>
> Ofcourse, you can choose which runner will get your job by using runner tags and leave shared runners enabled, for example, to save EC2 computing time. But you have to manage your risks on your own.

<br>

Now it is time to lunch new runner using AMI that we have created:

- Go to [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances);

- `Launch instance`:
  - Choose an Amazon Machine Image - select `GitLab Runner`, inside `My AMIs` tab;
  
  ![Select gitlab runner AMI](https://user-images.githubusercontent.com/62797411/78579253-543d6480-7839-11ea-859e-ef79a58142e3.png)
  
  - Choose an Instance Type - select `t2.micro` *(make sure that free tier is available)*;
  
- `Next: Configure Instance Details`:
  - Auto-assign Public IP - `Enable` *(or `Use subnet settings (Enable)`)*;
  - IAM role - select `gitlab-runner` [that we created earlier](https://github.com/tikhoplav/AWS-Gitlab-CICD/blob/master/gitlab-runner-on-ec2.md#create-gitlab-runner-iam-role);
  - Go to Advanced Details section:
    - User-data - set `As text` and paste next text to the field:
    
    ```
    Content-Type: multipart/mixed; boundary="//"
    MIME-Version: 1.0

    --//
    Content-Type: text/cloud-config; charset="us-ascii"
    MIME-Version: 1.0
    Content-Transfer-Encoding: 7bit
    Content-Disposition: attachment; filename="cloud-config.txt"

    #cloud-config
    cloud_final_modules:
    - [scripts-user, always]

    --//
    Content-Type: text/x-shellscript; charset="us-ascii"
    MIME-Version: 1.0
    Content-Transfer-Encoding: 7bit
    Content-Disposition: attachment; filename="userdata.txt"

    #! /bin/bash
    sudo gitlab-runner register -n --url <Your URL> --registration-token <Your Registartion Token> --executor shell --description "EC2 runner"
    --//
    ```
    
    ![User-Data settings](https://user-images.githubusercontent.com/62797411/78579109-1d674e80-7839-11ea-9b3d-6454f8a1944d.png)
    
  - Leave everything else by default;
  
- `Next: Add Storage`:

- `Next: Add Tags`:
  - `Name` - `gitlab-runner`;
  - `Project` - paste url of your GitLab project or group;
  
- `Next: Configure Security Group`:
  - Select `Create a new security group`;
  - Security group name - `gitlab-runner`;
  - Description - `Disallow any incoming connections`;
  - Remove any rulle from the list;
  
- `Next: Review and Launch`;

- `Launch`:
  - Select `Proceed without any key`.

<br>

After new instance is operational, we need to wait a little more to let it initiate runner and register it to our project. You could see that new runner is connect in projects CI/CD settings, runner section:

![Runner is registered successfully](https://user-images.githubusercontent.com/62797411/78578907-d6795900-7838-11ea-9937-dc176d7e7db8.png)

<br><br><br>

## Test runner

Now it is time to test our new runner. To do that let's create a pipeline in our project by adding a new file to the root of our project, called `.gitlab-ci.yml`. But before we neew to aquire a name of the image, exactly the same as have used in preparation stage.

Let's say that our image called: `678005261235.dkr.ecr.eu-central-1.amazonaws.com/alpine:latest`. Then our `.gitlab-ci.yml` will look like this:

```
test:
  script:
    - docker pull 678005261235.dkr.ecr.eu-central-1.amazonaws.com/alpine:latest
    - docker rmi $(docker images -q)
```

Yes, thats all we need. By running docker command, we ensure that GitLab Runner configured proper way. By pulling image from ECR we test our ECR connection.

<br>

If everything is set up correctly, we will find our job in Project > CI/CD > Jobs with status passed:

![Successful job with EC2 runner](https://user-images.githubusercontent.com/62797411/78578762-9e721600-7838-11ea-8420-a74de9e950b3.png)

If you now terminate your `gitlab-runner` EC2 instance, you will see that the runner have been automatically uregistered in your Project > Settings > CI/CD > Runners.

---

Congratulations! You have created and tested your new GitLab Runner AMI, with help of which you can now create as many runners as you need.
