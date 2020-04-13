<br>

# How to organize multiple GitLab Runners on AWS EC2 instance in the Auto Sacling Group

Greetings, reader. In this tutorial we will configure **AWS** **EC2** instance to host a **GitLab** Runner. We will grant this instance an ability to create, push and pull Docker images to the **ECR**. Configure it to automatically to register Runner in the **GitLab** project or group at startup and unregister on shut down. Also, we will create an **Auto Scale Group** of such **EC2** instances to be able to meet project requirements in the Runners automatically.

In this guide only *aws free tier* resources are used in order to give opportunity to reproduce it to anyone, also this configuration should not be used as the final production infrastruction for any commercial products. This servers only as a basic example.

###Requirements

In order to reproduce steps of this tutorial we will need followings:

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) *(Some commands of v1 and v2 may differs)*;

- [Docker Engine](https://docs.docker.com/engine/install/) *(ver 19.03.08, Community Edition)*;

- GitLab project;

<br>

Before we begin, I suggest you to complete [preparation section of this tutorial](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-on-aws-ec2.md#1-preparations), as the named resorces there, such as: administrator IAM, SSH kay pair and ECR repository with image; are required in this tutorial as well.

<br><br>

## 1. GitLab Runner AWS Network configuration

In this section we will create all necessary aws networking resources, such as:

- [VPC]();

- [Internet Gateway]();

- [Security Group]();

- [Subnet]();

- [Route Table]().

For the sake of clarity we will create name all resources as `gitlab-runner` in this example. This configuration can be used to host **Gitlab Runners** for any **GitLab** project or, group *(as well as the future AMI)*. At the stage of **Auto Scaling Group** creation you should start to diffiriantiate you resources by the projects, as at this step unique project settings will be set. Untill then all of the represented steps are universal.

<br>

### 1.1 VPC creation

This VPC will be used to provide EC2 inter-instances networking all over the cluster. In order to create a new VPC we need to run following command, and save the `VpcId` to the variable:

```
$ aws ec2 create-vpc --cidr-block 120.0.0.0/24

{
    "Vpc": {
        "CidrBlock": "120.0.0.0/24",
        "DhcpOptionsId": "dopt-130fda79",
        "State": "pending",
        "VpcId": "vpc-0dcb3329154b5a1d3",
        "OwnerId": "678005261235",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-0586a4ee2daa29934",
                "CidrBlock": "120.0.0.0/24",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false,
        "Tags": []
    }
}

$ VPC=vpc-0dcb3329154b5a1d3

$ aws ec2 create-tags --resources $VPC --tags Key=Name,Value=gitlab-runner
```

<br>

### 1.2 Internet Gateway creation

Internet gateway serves as the single entrypoint of all outgoing traffic for the VPC. In order to get response from the swarm docker container we need to create one. Later this gateway will be used in the Route Table specification, after the Subnet will be created.

```
$ aws ec2 create-internet-gateway

{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-0b45a098315108531",
        "Tags": []
    }
}

$ IGW=igw-0b45a098315108531

$ aws ec2 create-tags --resources $IGW --tags Key=Name,Value=gitlab-runner
```

<br>

Now VPC needs to be attached to the new Internet Gateway.

```
$ aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW \
  --vpc-id $VPC
```

<br>

### 1.3 Security group creation

Security Group acts like a set of rules, applied to all incoming and outcomming traffic.

```
$ aws ec2 create-security-group \
  --description "Only SSH allowed" \
  --group-name gitlab-runner \
  --vpc-id $VPC

{
    "GroupId": "sg-040358d0339e39db8"
}

$ SG=sg-040358d0339e39db8

$ aws ec2 create-tags --resources $SG --tags Key=Name,Value=gitlab-runner
```

<br>

Next step is to set up inboud traffic rules. As **GitLab Runner** host instance does not requires any incomming connection, only SSH access will be allowed:

```
$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 22
```

<br>

### 1.4 Subnet creation

```
$ aws ec2 create-subnet --cidr-block 120.0.0.0/24 --vpc-id $VPC

{
    "Subnet": {
        "AvailabilityZone": "eu-central-1c",
        "AvailabilityZoneId": "euc1-az1",
        "AvailableIpAddressCount": 251,
        "CidrBlock": "120.0.0.0/24",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "pending",
        "SubnetId": "subnet-07a3dbf45ca04d161",
        "VpcId": "vpc-0dcb3329154b5a1d3",
        "OwnerId": "678005261235",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:eu-central-1:678005261235:subnet/subnet-07a3dbf45ca04d161"
    }
}

$ aws ec2 create-tags --resources subnet-07a3dbf45ca04d161 --tags Key=Name,Value=gitlab-runner
```

<br>

### 1.5 Route Table configuration

After we have created a subnet, new route table should be automatically created. The easiest way to find it, is through the VPC we have just created:

```
$ RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC --query "RouteTables[*].RouteTableId" --output text)

$ aws ec2 create-tags --resources $RT --tags Key=Name,Value=gitlab-runner
```

<br>

Finally, we need to add next rule to make sure that all outcomming traffic from Docker Swarm nodes will reach WWW:

```
$ aws ec2 create-route \
  --route-table-id $RT \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW

{
    "Return": true
}
```

<br><br>

## 2. GitLab Runner EC2 AMI creation

In this section we will create a prototype of the EC2 instance, that is capable to host **GitLab Runner**. After that we will create **Amazon Machine Image** in order to automate **GitLab Runner** hosting EC2 launch. In the next section this **AMI** will be used to create an **Auto Scale Group** to automate the whole process. Following steps will be covered in this section:

- [GitLab Runner IAM EC2 role creation]();

- [EC2 instance launch]();

- [GitLab Runner hosting EC2 instance configuration]();

- [GitLab Runner AMI creation]();

<br>

### 2.1 GitLab Runner IAM EC2 role creation

In this step we will create role that will be used by EC2 instance. This role identifies capabilities of our instance to use other AWS resources. For example, we will grant our GitLab Runner instance ability to push and pull Docker images to ECR. To achieve that, we need to create a special **IAM role** that can be attached to our future EC2 instances. Let's create a file `ec2-role-trust-policy.json` with the following content:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This file should be specified in a IAM role creation process since it specifies that this role could be attached to EC2 instance. Let's create new EC2 instance role:

```
$ aws iam create-role \
  --role-name gitlab-runner \
  --assume-role-policy-document file://ec2-role-trust-policy.json \
  --description "GitLab Runner EC2 IAM role. \
  Allows repository creation, images pulling and pushing to ECR"

{
    "Role": {
        "Path": "/",
        "RoleName": "gitlab-runner",
        "RoleId": "AROAZ3XCDIOZSJLGE2KH6",
        "Arn": "arn:aws:iam::678005261235:role/gitlab-runner",
        "CreateDate": "2020-04-13T10:15:48+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "ec2.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

Next, let's attach required policies to the new `gitlab-runner` role:

```
$ aws iam attach-role-policy \
  --role-name gitlab-runner \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess
```

> The purpose of giving a GitLab runner full access to ECR is that give you an ability to delete repositories through CI/CD pipeline. For example, you could create a special job, that will unregister you microservice, clean up and set free all resources.
>
> If you feel that this permissions is more than you can allow runner to have, you could use `AmazonEC2ContainerRegistryPowerUser` policy instead.

> Later, if you would like to grant more permissions to GitLab Runner instance, for example, to use **S3** to cache builds etc, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/gitlab-runner). New permissions would automatically applied to all running instances using this role.

<br>

In order to pass this policies to the runtime of the EC2 instance we also have to create instance profile, and attach it to the instace role:

```
$ aws iam create-instance-profile --instance-profile-name gitlab-runner

{
    "InstanceProfile": {
        "Path": "/",
        "InstanceProfileName": "gitlab-runner",
        "InstanceProfileId": "AIPAZ3XCDIOZ2RJZFZ5Z5",
        "Arn": "arn:aws:iam::678005261235:instance-profile/gitlab-runner",
        "CreateDate": "2020-04-13T10:19:32+00:00",
        "Roles": []
    }
}

$ aws iam add-role-to-instance-profile --instance-profile-name gitlab-runner --role-name gitlab-runner
```

<br>

### 2.2 EC2 Instance launch

Now we will launch an EC2 instance that serves us as the **GitLab Runner** host prototype.

<details>
<summary>Env variables required</summary>

To be able to launch this instance we need to specify id's of the [security group]() and [subnet](), that has been created earlier:

```
$ SN=$(aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=gitlab-runner \
  --query "Subnets[*].SubnetId" \
  --output text)

$ SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=gitlab-runner \
  --query "SecurityGroups[*].GroupId" \
  --output text)
```
</details>

Let's launch new EC2 instance, by running the following command *(Make sure to save `InstanceId` value to the variable for later usage, also we will save the public ip address of the new instance)*:

```
$ aws ec2 run-instances \
  --image-id ami-076431be05aaf8080 \
  --instance-type t2.micro \
  --key-name ec2 \
  --security-group-ids $SG \
  --subnet-id "$SN" \
  --iam-instance-profile Name=gitlab-runner \
  --associate-public-ip-address
```

> SSH key pair is required in this step. If you haven't created one yet, please use [this](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-on-aws-ec2.md#12-ssh-key-pair-creation) as the reference to create one.

<details>
<summary>Response</summary>

```
{
    "Groups": [],
    "Instances": [
        {
            "AmiLaunchIndex": 0,
            "ImageId": "ami-076431be05aaf8080",
            "InstanceId": "i-0f21548cd9bc52e04",
            "InstanceType": "t2.micro",
            "KeyName": "ec2",
            "LaunchTime": "2020-04-13T11:06:35+00:00",
            "Monitoring": {
                "State": "disabled"
            },
            "Placement": {
                "AvailabilityZone": "eu-central-1c",
                "GroupName": "",
                "Tenancy": "default"
            },
            "PrivateDnsName": "ip-120-0-0-9.eu-central-1.compute.internal",
            "PrivateIpAddress": "120.0.0.9",
            "ProductCodes": [],
            "PublicDnsName": "",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "StateTransitionReason": "",
            "SubnetId": "subnet-07a3dbf45ca04d161",
            "VpcId": "vpc-0dcb3329154b5a1d3",
            "Architecture": "x86_64",
            "BlockDeviceMappings": [],
            "ClientToken": "",
            "EbsOptimized": false,
            "Hypervisor": "xen",
            "IamInstanceProfile": {
                "Arn": "arn:aws:iam::678005261235:instance-profile/gitlab-runner",
                "Id": "AIPAZ3XCDIOZ2RJZFZ5Z5"
            },
            "NetworkInterfaces": [
                {
                    "Attachment": {
                        "AttachTime": "2020-04-13T11:06:35+00:00",
                        "AttachmentId": "eni-attach-055cfc744af991b3c",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attaching"
                    },
                    "Description": "",
                    "Groups": [
                        {
                            "GroupName": "gitlab-runner",
                            "GroupId": "sg-040358d0339e39db8"
                        }
                    ],
                    "Ipv6Addresses": [],
                    "MacAddress": "0a:40:a4:73:28:14",
                    "NetworkInterfaceId": "eni-02f7d17177fcb8801",
                    "OwnerId": "678005261235",
                    "PrivateIpAddress": "120.0.0.9",
                    "PrivateIpAddresses": [
                        {
                            "Primary": true,
                            "PrivateIpAddress": "120.0.0.9"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Status": "in-use",
                    "SubnetId": "subnet-07a3dbf45ca04d161",
                    "VpcId": "vpc-0dcb3329154b5a1d3",
                    "InterfaceType": "interface"
                }
            ],
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SecurityGroups": [
                {
                    "GroupName": "gitlab-runner",
                    "GroupId": "sg-040358d0339e39db8"
                }
            ],
            "SourceDestCheck": true,
            "StateReason": {
                "Code": "pending",
                "Message": "pending"
            },
            "VirtualizationType": "hvm",
            "CpuOptions": {
                "CoreCount": 1,
                "ThreadsPerCore": 1
            },
            "CapacityReservationSpecification": {
                "CapacityReservationPreference": "open"
            },
            "MetadataOptions": {
                "State": "pending",
                "HttpTokens": "optional",
                "HttpPutResponseHopLimit": 1,
                "HttpEndpoint": "enabled"
            }
        }
    ],
    "OwnerId": "678005261235",
    "ReservationId": "r-0c4cc6495adf3810f"
}
```
</details>

```
$ EC2=i-0f21548cd9bc52e04

$ aws ec2 create-tags --resources $EC2 --tags Key=Name,Value=gitlab-runner

$ IP=$(aws ec2 describe-instances --instance-ids $EC2 --query "Reservations[*].Instances[*].PublicIpAddress" --output text) \
  && echo $IP
```

<br>

### 2.3 GitLab Runner hosting EC2 instance configuration

This section is a set of settings and configurations that we need to do to prepare an EC2 instance for hosting the **GitLab Runner**. All of the following command needs to be executed in EC2 isntance so we need to connect it with the next command:

```
$ ssh -i ~/.ssh/ec2.pem ec2-user@$IP
```

Make sure this instance is operational before AMI creation. If the instance closes or reboots, you will most likely have to repeat the entire section from the beginning.

#### 2.3.1 Install Docker
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

<br>

#### 2.3.2 Esablish ECR connection

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
$ aws ecr describe-repositories --region $REGION --query "repositories[*].{uri:repositoryUri,name:repositoryName}"

[
    {
        "uri": "678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx", 
        "name": "nginx"
    }
]
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

> Any docker image can be used in this part, only thins that matters is the fact of pulling this image from ECR repository. This action will create a cached credentials in our EC2 instance.

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
<br>

#### 2.3.3 Install GitLab Runner

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

#### 2.3.4 Create Termination Sequence

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

#### 2.3.5 Clean up

I suggest to clean up an instance, since free tier of ESB volume usage is limited, or if you not using free tier or broke the limit of monthly usage, every single megabyte is charged.

```
$ sudo userdel -r gitlab-runner \
  && sudo docker rmi $(sudo docker images -q) \
  && sudo yum clean all \
  && sudo rm -rf /var/cache/yum
```

<br>

### 2.4 GitLab Runner AMI creation

Now our prototype is ready and we can create an AMI:

```
$ aws ec2 create-image \
  --instance-id $EC2 \
  --name "GitLab Runner" \
  --description "Amazon Linux 2 based. \
  Contains Docker (v19.03.6), Amazon ECR Docker Credentials Helper, GitLab Runner (v12.9.0) \
  Auto unregister on termination. Shell mode executor is supposed."

{
    "ImageId": "ami-004a5a4927e20b106"
}

$ aws ec2 create-tags --resources ami-004a5a4927e20b106 --tags Key=Name,Value=gitlab-runner
```

<br><br>

## 3 GitLab Runner AWS Auto Scaling Group

This is the last seqction of this tutorial in which we will create and configure the **AWS Auto Scaling Group** of **GitLab Runner** hosting **EC2** instances. 

The main reson for creating an autoscaling group is that our host EC2 is configured to use `shell` executor. This gives us performance advantage as well as more simple way to utilize docker in CI/CD pipeline *(compared to **DIND** approach)*. On the other hand, we can not confidure runner to do parallel jobs with `shell` executor, so only one job can be processed with our entire EC2 instance at a time. So, it is suggested to run new hosting instances on demand of you project.

Next steps will be covered in this section:

- [GitLab Runner Launch configuration creation]();

- [GitLab Runner Auto Scaling group creation]();

- [GitLab Runner hosted on AWS EC2 testing]();

<details>
<summary>Env var required</summary>

```
$ SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=gitlab-runner \
  --query "SecurityGroups[*].GroupId" \
  --output text) \
  && echo $SG

$ SN=$(aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=gitlab-runner \
  --query "Subnets[*].SubnetId" \
  --output text) \
  && echo $SN

$ AMI=$(aws ec2 describe-images \
  --filters Name=tag:Name,Values=gitlab-runner \
  --query "Images[*].ImageId" \
  --output text) \
  && echo $AMI
```
</details>

<br>

### 3.1 GitLab Runner Launch configuration creation

Befor we create our Launch configuration we need to specify `url` of our **GilLab** host and `registration-token`. Those can be found in the Settings > CI/CD > Runners tab of your GitLab project or Group. Also, make sure that shared runners option is disabled:

![gitlab project runner register example](https://user-images.githubusercontent.com/62797411/79118457-abda5380-7d96-11ea-8448-4866cac58820.png)

In order to let our instances register runner automatically on launch, we will use user-data file. Let's create a file called `gitlab-runner_user-data.txt` with the following content *(make sure to paste correct `url` and `registration-token`, that have been outputed in a previous step)*.

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
sudo gitlab-runner register -n --url https://gitlab.com/ --registration-token gGH9TSGyDggCygACRqVf --executor shell --description "EC2 runner"
--//
```

<br>

Now we are ready to create a launch configuration *(here we will add project name to the name of the configuration, for the sake of clearence it will be `example`)*:

```
$ aws autoscaling create-launch-configuration \
  --launch-configuration-name example-gitlab-runner \
  --image-id $AMI \
  --key-name ec2 \
  --security-groups "$SG" \
  --instance-type t2.micro \
  --iam-instance-profile gitlab-runner \
  --associate-public-ip-address \
  --user-data file://gitlab-runner_user-data.txt
```

<br>

### 3.2 GitLab Runner Auto Scaling Group creation

Next command will create and launch autoscaling group *(I suggest to specify url of the project in the tags, so make sure to pass right url)*:

```
$ aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name example-gitlab-runner \
  --launch-configuration-name example-gitlab-runner \
  --min-size 1 \
  --max-size 30 \
  --desired-capacity 1 \
  --health-check-grace-period 30 \
  --vpc-zone-identifier "$SN" \
  --tags "Key=Name,Value=example-gitlab-runner,PropagateAtLaunch=true" \
  "Key=project,Value=example,PropagateAtLaunch=true" \
  "Key=url,Value=http://gitlab.com/projects/path,PropagateAtLaunch=true"
```

After a little amount of time you will find that new runner have been registered to your project in the Settings > CI/CD > Runners tab:

![registration of the gitlab runner with aws auto scaling group](https://user-images.githubusercontent.com/62797411/79119107-38d1dc80-7d98-11ea-81c0-30b3427b0a73.png)

<br>

To make our group fully autonomous, we need to add feedback to our auto-scaling group. The following command creates a scaling policy and attaches it to our auto-scaling group. What he does is that if the CPU utilization of the EC2 instances drops below 40%, the group shrinks. And if average usage rises above 40%, the auto-scaling group releases new instances.

Let's create new file called `scaling-policy.json` with the next content:
```
{
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  },
  "TargetValue": 40,
  "DisableScaleIn": false
}
```

Now we are ready to create and apply policy:

```
$ aws autoscaling put-scaling-policy \
  --auto-scaling-group-name example-gitlab-runner \
  --policy-name example-on-cpu \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://scaling-policy.json
  
{
    "PolicyARN": "arn:aws:autoscaling:eu-central-1:678005261235:scalingPolicy:643afd1d-c3a9-43e6-b3bc-ab423aa9dbd6:autoScalingGroupName/example-gitlab-runner:policyName/example-on-cpu",
    "Alarms": [
        {
            "AlarmName": "TargetTracking-example-gitlab-runner-AlarmHigh-f5defed6-1b37-4e29-82e2-c9e60844ca21",
            "AlarmARN": "arn:aws:cloudwatch:eu-central-1:678005261235:alarm:TargetTracking-example-gitlab-runner-AlarmHigh-f5defed6-1b37-4e29-82e2-c9e60844ca21"
        },
        {
            "AlarmName": "TargetTracking-example-gitlab-runner-AlarmLow-b6fb4eff-90a2-4838-a74c-a1113d4aadc1",
            "AlarmARN": "arn:aws:cloudwatch:eu-central-1:678005261235:alarm:TargetTracking-example-gitlab-runner-AlarmLow-b6fb4eff-90a2-4838-a74c-a1113d4aadc1"
        }
    ]
}
```

<br>

### 3.3 GitLab Runner hosted on AWS EC2 testing

Before we finish let's test our new **GitLab Runners** by adding a `.gitlab-ci.yml` file to the root of our project with the following content:

```
variables:
  # make sure you have pasted right region name
  REGION: eu-central-1

test-runner:
  script:
    # This is how we test that aws cli is operational
    - IMAGE=$(aws ecr describe-repositories --region $REGION --query "repositories[*].repositoryUri" --output text) && echo $IMAGE
    - REPO_NAME=$(aws ecr describe-repositories --region $REGION --query "repositories[*].repositoryName" --output text) && echo $REPO_NAME
    - TAG=$(aws ecr list-images --region $REGION --repository-name $REPO_NAME --query "imageIds[0].imageTag" --output text) && echo $TAG

    # This is how we test that credential helper is operational
    # Also this shows us the availability of docker daemon usage
    - docker pull $IMAGE:$TAG

    # Clean up
    - docker rmi $(docker images -q)
```

Commit this changes and you will find a new jon in the CI/CD > Jobs of your project:

![test job with gitlab runner on ec2 instance](https://user-images.githubusercontent.com/62797411/79119974-56a04100-7d9a-11ea-9367-191158e8d03f.png)

<br>

---

Congratulations! You have hosted **GitLab Runner** on **AWS EC2** instances as well as created and configured an **Auto Scale Group**

<br><br><br>
