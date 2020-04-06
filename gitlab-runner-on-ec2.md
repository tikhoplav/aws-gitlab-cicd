# Configure amazon EC2 instance for GitLab Runner

In this guide we will lunch and set up **Amazon Web Service EC2 instance** to host **GitLab Runner**.

Runners created following this instruction are abile to [use docker daemon](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html) as well as to pull and push this images to the ECR during CI/CD pipeline.

Current method can be applied to create specific project runner as well as group runner.

<br>

**Requirements**:

- GitLab project;

- [Admin privileged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `AWS Management Console access`;

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

## Set up EC2 instance to host GitLab Runner

In this section we will install all necessary software to host GitLab Runner. It consist of multiple steps:

- [Docker Installation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#install-docker);

- [ECR connection establishment](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#establish-ecr-connection);

- [GitLab Runner installation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#install-gitlab-runner);

- [Termination script creation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#create-termination-sequence);

- [Clean up sequence](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-on-ec2.md#clean-up);

<br><br>

### Install Docker
```
$ sudo yum -y update \
  && sudo amazon-linux-extras install -y docker \
  && sudo systemctl enable docker \
  && sudo service docker start
```

<details>
<summary>To test that docker is running:</summary>

```
$ sudo docker info

Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 1
 Server Version: 19.03.6-ce
 Storage Driver: overlay2
  Backing Filesystem: xfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: ff48f57fc83a8c44cf4ad5d672424a98ba37ded6
 runc version: dc9208a3303feef5b3839f4323d9beb36df0a9dd
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.14.171-136.231.amzn2.x86_64
 Operating System: Amazon Linux 2
 OSType: linux
 Architecture: x86_64
 CPUs: 1
 Total Memory: 983.4MiB
 Name: ip-172-31-43-48.eu-central-1.compute.internal
 ID: SC7V:XQZY:PPVK:YIKT:HST2:M4OG:53JH:HLIU:WCWO:SQ6R:3VA4:YAPR
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

```
</details>

<br><br>

### Esablish ECR connection

Before we can push and pull Docker images to ECR, we have to login our Docker daemon to ECR registry. Normally we aquire token that lasts 12 hours with `awc-cli`. In order to let GitLab Runner use ECR constantly we need to do the following steps:

[Amazon ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) will allow EC2 instance push and pull images to ECR, since it requires special authentification sequence. Let's install it by running:

```
$ sudo yum install -y amazon-ecr-credential-helper
```

<br>

Next step is to aquire repository name, but before we need to specify region *(Specify the region, that your EC2 instance is running in)*:

```
$ REGION=eu-central-1
```

Then get list of repositories by running next comman *(Copy `perositoryName` value to clipboard)*: 

```
$ aws ecr describe-repositories --region $REGION

{
    "repositories": [
        {
            "repositoryUri": "678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx", 
            "imageScanningConfiguration": {
                "scanOnPush": false
            }, 
            "registryId": "678005261235", 
            "imageTagMutability": "MUTABLE", 
            "repositoryArn": "arn:aws:ecr:eu-central-1:678005261235:repository/nginx", 
            "repositoryName": "nginx", 
            "createdAt": 1586209507.0
        }
    ]
}
```

In order get available tags of repository run:

```
$ aws ecr list-images --region $REGION --repository-name nginx

{
    "imageIds": [
        {
            "imageTag": "alpine", 
            "imageDigest": "sha256:ef2b6cd6fdfc6d0502b77710b27f7928a5e29ab5cfae398824e5dcfbbb7a75e2"
        }
    ]
}

```

Next we need to combine full image name using `repositoryUri` and `imageTag` aquired:

```
$ IMAGE=678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx:alpine
```

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

### Install GitLab Runner

According to [official documentation](https://docs.gitlab.com/runner/install/linux-manually.html#using-debrpm-package), we can install GitLab Runner by doing:

```
$ curl -LJO https://gitlab-runner-downloads.s3.amazonaws.com/latest/rpm/gitlab-runner_amd64.rpm \
  && sudo yum install -y git \
  && sudo rpm -i gitlab-runner_amd64.rpm
```

<details>
<summary>To make sure that GitLab Runner is operational:</summary>

```
$ journalctl -u gitlab-runner

апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal systemd[1]: Started GitLab Runner.
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal systemd[1]: Starting GitLab Runner...
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Runtime platform                                    arch=amd64 os=linux pid=10026 revision=4c96e5ad version=12.9.0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Starting multi-runner from /etc/gitlab-runner/config.toml...  builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Starting multi-runner from /etc/gitlab-runner/config.toml...  builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Running in system-mode.                           
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Running in system-mode.                           
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]:                                                   
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: 
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Configuration loaded                                builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: Configuration loaded                                builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: listen_address not defined, metrics & debug endpoints disabled  builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: listen_address not defined, metrics & debug endpoints disabled  builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: [session_server].listen_address not defined, session endpoints disabled  builds=0
апр 06 22:21:33 ip-172-31-43-48.eu-central-1.compute.internal gitlab-runner[10026]: [session_server].listen_address not defined, session endpoints disabled  builds=0
```
</details>

<br>

Now we need to reconfigure GitLab Runner so that it will run jobs as a `root` user.

```
$ sudo sed -i 's/"--user" "gitlab-runner"/"--user" "root"/' /etc/systemd/system/gitlab-runner.service \
  && sudo sed -i 's/"\/home\/gitlab-runner"/"\/tmp"/' /etc/systemd/system/gitlab-runner.service \
  && sudo systemctl daemon-reload \
  && sudo service gitlab-runner restart
```

> For what purpose do we need to run GitLab Runner as root?
>
> I have been testing with different settings of GitLab Runner. With current version of it (12.9.0) there are problems with POST requests to GitLab server using non root so that you need to run `sudo gitlab-runner run` manually or with help of other tools (like `cloud-init`).
>
> Since it is highly suggested to use this kind of EC2 instances for hosting GitLab Runner only (Even all ports of it's security group should be disabled), there are the same security issues as if it will run as not a root.
>
> Plus, this way we could allow our runner environment to use ECR much easier *(Take a look to `sudo` in front of every command)*, without files coping and permission management.

<br><br>

### Create Termination Sequence

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

### Clean up

I suggest to clean up an instance, since free tier of ESB volume usage is limited, or if you not using free tier or broke the limit of monthly usage, every single megabyte is charged.

```
$ sudo userdel -r gitlab-runner \
  && sudo docker rmi $(sudo docker images -q) \
  && sudo yum clean all \
  && sudo rm -rf /var/cache/yum
```

<br><br><br>

## Register GitLab Runner

Before we could register new runner GitLab host URL and registration token need to be aquired. You can find them at projects Setting > CI/CD > Runners section:

![gitlab host url and registration token for new runner](https://user-images.githubusercontent.com/62797411/78612325-bdd96500-7871-11ea-8992-73f7e4f902d1.png)

```
$ sudo gitlab-runner register -n --url https://gitlab.com/ --registration-token gGH9TSGyDggCygACRqVf --executor shell --description "EC2 runner"

Runtime platform                                    arch=amd64 os=linux pid=10343 revision=4c96e5ad version=12.9.0
Running in system-mode.                            
                                                   
Registering runner... succeeded                     runner=gGH9TSGy
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

<br>

After a little you will see new runner connected in the same rojects Settings > CI/CD > Runners section:

![connected gitlab runner](https://user-images.githubusercontent.com/62797411/78612507-2fb1ae80-7872-11ea-841c-fc821188f522.png)

---

Congratulations! You have lunched GitLab Runner using EC2 instance. I suggest you to go through the next steps:

- [GitLab Runner AMI Creation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-ami.md) - To minimize time consumption of new runners launch;

- [GitLab Runner Auto Scaling Group Creation](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-auto-scaling.md) - To launch new runners for your project automatically on heavy CI/CD pipeline load.