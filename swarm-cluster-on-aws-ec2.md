# How to organize Docker Swarm cluster on AWS EC2 instances in Auto Scaling Group

Greetings, reader. In this tutorial we will create a **Docker Swarm** cluster of **Amazon Web Service** **EC2** instances as a nodes. We will start from scratch, configuring AWS network, creating special **AMI** and **Auto Scaling Group** to make a fully autonomous cluster that scales it self in order to meet swarm resource requirements.

In this guide only *aws free tier* resources are used in order to give opportunity to reproduce it to anyone, also this configuration should not be used as the final production infrastruction for any commercial products. This servers only as a basic example.

### Requirements

In order to reproduce steps of this tutorial we will need followings:

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) *(Some commands of v1 and v2 may differs)*;

- [Docker Engine](https://docs.docker.com/engine/install/) *(ver 19.03.08, Community Edition)*;

<br><br>

## 1. Preparations

This section contains basic steps that you should perform before any actual work with AWS, such as:

- [Administrator IAM creation]();

- [SSH Key Pair creation]();

- [AWS CLI configuration]();

- [ECR Repository creation]();

<br>

### 1.1 Administrator IAM creation

Creation and managing resources with **awc cli** requires AWS account with programmatic access. In order to achive this, we need to create administrator IAM user using AWS management console.

- Go to [Users IAM console](https://console.aws.amazon.com/iam/home?#/users);

- `Add user`:

  - User name - `admin`;

  - Access type:

    - Check `Programmatic access`;

  - Set Permissions - `Attach existing policies directly`:

  	- Select `AdministratorAccess`;

  - `Tags`;

  - `Review`;

  - `Create user`;

  - Download `.csv` file (filename will be `creadentials.csv`).

<br>

Also, you could follow the [official documentation](https://docs.aws.amazon.com/mediapackage/latest/ug/setting-up-create-iam-user.html) related to this step.

<br>

### 1.2 SSH Key Pair creation

In order to configure future EC2 instances SSH access required. It is good practice to create separated key pair for any group if EC2 instances you launch. But for this time we only need one.

- Go to [Keys Pairs management console](https://console.aws.amazon.com/ec2/v2/home?#KeyPairs);

- `Create key pair`:

  - Name - `ec2`;

  - File format - If you are using Linux or Mac chose `pem`. If windows - `ppk`;

<br>

After key pair is created the download process will begin. Make sure to store it in a safe place, for example `~/.ssh/` folder. In addition, if you are using Linux, you have to apply permission to this file:

```
$ sudo shmod 600 ~/.ssh/ec2.pem
```

<br>

More info about AWS key pairs, you could find in the [official documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html).

<br>

### 1.3 AWS CLI configuration

To gain access to AWS resources, **awc cli** needs to be configured to use administrator IAM user. This can be done by running following command and pass content of the `creadentials.csv` to it:

```
$ aws configure
```

> As well as `access key id` and `secret access key` the region, where reseources is going to be created, should be specified. Resources from different regions can be used, but you have to know that interregion communications charged additionally.

<br>

More information about **aws cli** configuration you could find in the [documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

<br>

### 1.4 ECR Repository creation

In order to test ECR availability of our future EC2 instances, the repository needs to be created and a docker image should be stored there in advance. I suggest to use `nginx:alpine` image, because it can be used to test public request processing with instances out of the box. Let's start with repository creation:

```
$ aws ecr describe-repositories

{
    "repositories": []
}

$ aws ecr create-repository --repository-name nginx

{
    "repository": {
        "repositoryArn": "arn:aws:ecr:eu-central-1:678005261235:repository/nginx",
        "registryId": "678005261235",
        "repositoryName": "nginx",
        "repositoryUri": "678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx",
        "createdAt": "2020-04-09T11:38:29+03:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        }
    }
}
```

We will need to use specify `repositoryUri` and ECR url in a following step, so let's save those as variables:
```
$ ECR=678005261235.dkr.ecr.eu-central-1.amazonaws.com
$ REPOSITORY=678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx
```

<br>

Now, we will register our Docker daemon, in order to push image to ECR repository. This can be achived with the next command:

```
$ aws ecr get-login-password | docker login --username AWS --password-stdin $ECR

WARNING! Your password will be stored unencrypted in /home/psyke/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

<br>

Next we will push the `nginx:alpine` image to the ECR repository:

```
$ docker pull nginx:alpine \
  && docker tag nginx:alpine $REPOSITORY:alpine \
  && docker push $REPOSITORY:alpine

alpine: Pulling from library/nginx
Digest: sha256:abe5ce652eb78d9c793df34453fddde12bb4d93d9fbf2c363d0992726e4d2cad
Status: Image is up to date for nginx:alpine
docker.io/library/nginx:alpine
The push refers to repository [678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx]
6f23cf4d16de: Pushed 
531743b7098c: Pushed 
alpine: digest: sha256:ef2b6cd6fdfc6d0502b77710b27f7928a5e29ab5cfae398824e5dcfbbb7a75e2 size: 739
```

To test that image is pushed successfully, let's do:

```
$ aws ecr describe-images --repository-name nginx

{
    "imageDetails": [
        {
            "registryId": "678005261235",
            "repositoryName": "nginx",
            "imageDigest": "sha256:ef2b6cd6fdfc6d0502b77710b27f7928a5e29ab5cfae398824e5dcfbbb7a75e2",
            "imageTags": [
                "alpine"
            ],
            "imageSizeInBytes": 8492473,
            "imagePushedAt": "2020-04-09T11:47:56+03:00"
        }
    ]
}
```

Also, you could find your new image in [ECR management console > repositories](https://console.aws.amazon.com/ecr/repositories) *(Don't forget to choose the right region)*:

![example of the ECR repository](https://user-images.githubusercontent.com/62797411/78608513-bc0ba380-7869-11ea-929d-694e49ae9225.png)

<br><br>

## 2. Docker Swarm cluster AWS Network configuration

In this section we will create all necessary aws networking resources, such as:

- [VPC]();

- [Internet Gateway]();

- [Security Group]();

- [Subnet]();

- [Route Table]();

For the sake of clarity we will create cluster called `production` as an example. Any resources names and tags will be set related to the `production`, so that you will be able to recognize this resources later.

<br>

### 2.1 VPC creation

This VPC will be used to provide EC2 inter-instances networking all over the cluster. In order to create a new VPC we need to run following command, and save the `VpcId` to the variable:

```
$ aws ec2 create-vpc --cidr-block 10.0.0.0/24

{
    "Vpc": {
        "CidrBlock": "10.0.0.0/24",
        "DhcpOptionsId": "dopt-130fda79",
        "State": "pending",
        "VpcId": "vpc-0ee00f62537074796",
        "OwnerId": "678005261235",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-0d6a946d35a48e406",
                "CidrBlock": "10.0.0.0/24",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false,
        "Tags": []
    }
}

$ VPC=vpc-0ee00f62537074796

$ aws ec2 create-tags --resources $VPC --tags Key=Name,Value=production
```

<br>

### 2.2 Internet Gateway creation

Internet gateway serves as the single entrypoint of all outgoing traffic for the VPC. In order to get response from the swarm docker container we need to create one. Later this gateway will be used in the Route Table specification, after the Subnet will be created.

```
$ aws ec2 create-internet-gateway

{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-0ef64205485d8e81a",
        "Tags": []
    }
}

$ IGW=igw-0ef64205485d8e81a

$ aws ec2 create-tags --resources $IGW --tags Key=Name,Value=production
```

<br>

Now VPC needs to be attached to the new Internet Gateway.

```
$ aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW \
  --vpc-id $VPC
```

<br>

### 2.3 Security group creation

Security Group acts like a set of rules, applied to all incoming and outcomming traffic.

```
$ aws ec2 create-security-group \
  --description "Allows public requestes, SSH and all Docker network communications" \
  --group-name production \
  --vpc-id $VPC

{
    "GroupId": "sg-0177b3a255b9fc450"
}

$ SG=sg-0177b3a255b9fc450

$ aws ec2 create-tags --resources $SG --tags Key=Name,Value=production
```

<br>

Next step is to set up inboud traffic rules. We will allow standart HTTP and HTTPS as well as SSH. Moreover, in order to let Docker Swarm nodes communicate with each other, we will allow specific TCP and UDP ports *(You could find Docker Swarm related ports in the [oficial documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts))*:

```
$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 22

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 80

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 443

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 2377

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol tcp --port 7946

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol udp --port 7946

$ aws ec2 authorize-security-group-ingress --group-id $SG --cidr 0.0.0.0/0 \
  --protocol udp --port 4789
```

<br>

### 2.4 Subnet creation

The subnet is required to launch EC2 instances inside VPC. In order to make sure that containers in the swarm are able to connect each other, let's create a single subnet for our VPC.

```
$ aws ec2 create-subnet --cidr-block 10.0.0.0/24 --vpc-id $VPC

{
    "Subnet": {
        "AvailabilityZone": "eu-central-1a",
        "AvailabilityZoneId": "euc1-az2",
        "AvailableIpAddressCount": 251,
        "CidrBlock": "10.0.0.0/24",
        "DefaultForAz": false,
        "MapPublicIpOnLaunch": false,
        "State": "pending",
        "SubnetId": "subnet-05d781c4237ffdf39",
        "VpcId": "vpc-0ee00f62537074796",
        "OwnerId": "678005261235",
        "AssignIpv6AddressOnCreation": false,
        "Ipv6CidrBlockAssociationSet": [],
        "SubnetArn": "arn:aws:ec2:eu-central-1:678005261235:subnet/subnet-05d781c4237ffdf39"
    }
}

$ aws ec2 create-tags --resources subnet-05d781c4237ffdf39 --tags Key=Name,Value=production
```

<br>

### 2.5 Route Table configuration

After we have created a subnet, new route table should be automatically created. The easiest way to find it, is through the VPC we have just created:

```
$ RT=$(aws ec2 describe-route-tables --filters Name=vpc-id,Values=$VPC --query "RouteTables[*].RouteTableId" --output text)

$ aws ec2 create-tags --resources $RT --tags Key=Name,Value=production
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

## 3. Swarm node EC2 AMI creation

In this section we will create special **AMI** (**Amazon Machine Image**) that serves as a preset of our future Docker Swarm nodes. Later we will use this AMI to configure **Auto Scaling Group**, that will regulate number of EC2 instances automatically.

<br>

### 3.1 Swarm node EC2 role creation

As it was described earlier, our Docker Swarm node needs access to docker images stored in ECR. Moreover, in order to be able to debug services we need to allow nodes to push logs in to **AWS CloudWatch**. To achieve that, we need to create a special **IAM role** that can be attached to our future EC2 instances. Let's create a file `ec2-role-trust-policy.json` with the following content:

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
  --role-name swarm-node \
  --assume-role-policy-document file://ec2-role-trust-policy.json \
  --description "Docker Swarm EC2 IAM role. \
  Allows docker images pulling from ECR, and logging to CloudWatch"

{
    "Role": {
        "Path": "/",
        "RoleName": "swarm-node",
        "RoleId": "AROAZ3XCDIOZ64THBEVA5",
        "Arn": "arn:aws:iam::678005261235:role/swarm-node",
        "CreateDate": "2020-04-09T13:50:29+00:00",
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

Next, let's attach required policies to the new `swarm-node` role:

```
$ aws iam attach-role-policy \
  --role-name swarm-node \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

$ aws iam attach-role-policy \
  --role-name swarm-node \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs
```

<br>

In order to pass this policies to the runtime of the EC2 instance we also have to create instance profile, and attach it to the instace role:

```
$ aws iam create-instance-profile --instance-profile-name swarm-node

{
    "InstanceProfile": {
        "Path": "/",
        "InstanceProfileName": "swarm-node",
        "InstanceProfileId": "AIPAZ3XCDIOZWEB5AXHHG",
        "Arn": "arn:aws:iam::678005261235:instance-profile/swarm-node",
        "CreateDate": "2020-04-09T13:58:03+00:00",
        "Roles": []
    }
}

$ aws iam add-role-to-instance-profile --instance-profile-name swarm-node --role-name swarm-node
```

<br>

### 3.2 Launch EC2 instance

Now we need to build a prototype of our Swarm node. In this step we will launch fresh EC2 instance that will serve two important purposes: 

1. In order to minimize time consumption of the future Swarm nodes launch, we will create AMI based on this instance;

2. We will create Docker Swarm on this instance so it will be our first swarm manager.

<details>
<summary>Env variables required</summary>

To be able to launch this instance we need to specify id's of the [security group](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/2.cluster-network-configuration.md#23-security-group-creation) and [subnet](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/2.cluster-network-configuration.md#24-subnet-creation), that has been created earlier:

```
$ SN=$(aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=production \
  --query "Subnets[*].SubnetId" \
  --output text)

$ SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=production \
  --query "SecurityGroups[*].GroupId" \
  --output text)
```
</details>

<br>

Let's launch new EC2 instance, by running the following command *(Make sure to save `InstanceId` value to the variable for later usage)*:

```
$ aws ec2 run-instances \
  --image-id ami-076431be05aaf8080 \
  --instance-type t2.micro \
  --key-name ec2 \
  --security-group-ids $SG \
  --subnet-id "$SN" \
  --iam-instance-profile Name=swarm-node \
  --associate-public-ip-address
```

<details>
<summary>Response</summary>

```
{
    "Groups": [],
    "Instances": [
        {
            "AmiLaunchIndex": 0,
            "ImageId": "ami-076431be05aaf8080",
            "InstanceId": "i-0514c5025fd2cdb16",
            "InstanceType": "t2.micro",
            "KeyName": "ec2",
            "LaunchTime": "2020-04-09T19:07:27+00:00",
            "Monitoring": {
                "State": "disabled"
            },
            "Placement": {
                "AvailabilityZone": "eu-central-1a",
                "GroupName": "",
                "Tenancy": "default"
            },
            "PrivateDnsName": "ip-10-0-0-198.eu-central-1.compute.internal",
            "PrivateIpAddress": "10.0.0.198",
            "ProductCodes": [],
            "PublicDnsName": "",
            "State": {
                "Code": 0,
                "Name": "pending"
            },
            "StateTransitionReason": "",
            "SubnetId": "subnet-05d781c4237ffdf39",
            "VpcId": "vpc-0ee00f62537074796",
            "Architecture": "x86_64",
            "BlockDeviceMappings": [],
            "ClientToken": "",
            "EbsOptimized": false,
            "Hypervisor": "xen",
            "IamInstanceProfile": {
                "Arn": "arn:aws:iam::678005261235:instance-profile/swarm-node",
                "Id": "AIPAZ3XCDIOZWEB5AXHHG"
            },
            "NetworkInterfaces": [
                {
                    "Attachment": {
                        "AttachTime": "2020-04-09T19:07:27+00:00",
                        "AttachmentId": "eni-attach-069f621968d0a7e5d",
                        "DeleteOnTermination": true,
                        "DeviceIndex": 0,
                        "Status": "attaching"
                    },
                    "Description": "",
                    "Groups": [
                        {
                            "GroupName": "production",
                            "GroupId": "sg-0177b3a255b9fc450"
                        }
                    ],
                    "Ipv6Addresses": [],
                    "MacAddress": "02:2e:45:72:77:ee",
                    "NetworkInterfaceId": "eni-02384bdab7a346b18",
                    "OwnerId": "678005261235",
                    "PrivateIpAddress": "10.0.0.198",
                    "PrivateIpAddresses": [
                        {
                            "Primary": true,
                            "PrivateIpAddress": "10.0.0.198"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Status": "in-use",
                    "SubnetId": "subnet-05d781c4237ffdf39",
                    "VpcId": "vpc-0ee00f62537074796",
                    "InterfaceType": "interface"
                }
            ],
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SecurityGroups": [
                {
                    "GroupName": "production",
                    "GroupId": "sg-0177b3a255b9fc450"
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
    "ReservationId": "r-0f66d9485c4378f3c"
}
```
</details>

```
$ EC2=i-0514c5025fd2cdb16

$ aws ec2 create-tags --resources $EC2 --tags Key=Name,Value=production-manager \
  && aws ec2 create-tags --resources $EC2 --tags Key=cluster,Value=production \
  && aws ec2 create-tags --resources $EC2 --tags Key=role,Value=manager
```

<br>

We also need the public ip of our new EC2 instance, which we can get with the following command *(MIP - manager ip)*:

```
$ MIP=$(aws ec2 describe-instances --instance-ids $EC2 --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
```

<br>

### 3.3 EC2 instance configuration

This section is a set of settings and configurations that we need to do to prepare an EC2 instance for Docker Swarm. All of the following command needs to be executed in EC2 isntance so we need to connect it with the next command:

```
$ ssh -i ~/.ssh/ec2.pem ec2-user@$MIP
```

Make sure this instance is operational before AMI creation. If the instance closes or reboots, you will most likely have to repeat the entire section from the beginning.

<br>

#### 3.3.1 Docker installation

In order to grant access to docker daemon to the CI/CD pipeline, we will need to add `ec2-user` to the `docker` group. Following steps requires to be run as `ec2-user`, so we need to recreate ssh connection after following command sequence:

```
$ sudo yum -y update \
  && sudo amazon-linux-extras install -y docker \
  && sudo systemctl enable docker \
  && sudo service docker start \
  && sudo usermod -a -G docker ec2-user

$ exit
```

<details>
<summary>To test that docker is running:</summary>

```
$ docker info

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

#### 3.3.2 Amazon ECR Docker credential helper installation

Before we can push and pull Docker images to ECR, we have to login our Docker daemon into ECR registry. Normally we aquire token that lasts 12 hours with `awc-cli`. In order to let Swarm node use ECR constantly we need to use [Amazon ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper). Let's install it by running:

```
$ sudo yum install -y amazon-ecr-credential-helper
```

<br>

Next, we need to get ECR url. To do so we can use the following command *(Make sure to specify correct region)*:

```
$ REGION=eu-central-1

$ REPOSITORY=$(aws ecr describe-repositories \
  --region $REGION \
  --repository-names nginx \
  --query "repositories[*].repositoryUri" \
  --output text)
```

<br>

Now we need to aquire the token and store it with help of credential helper. To achive that we could use [this approach](https://github.com/awslabs/amazon-ecr-credential-helper/issues/63#issuecomment-328318116):

```
$ $(aws ecr get-login --no-include-email --region $REGION) \
  && echo -e "{\n\t\"credsStore\": \"ecr-login\"\n}" | sudo tee ~/.docker/config.json \
  && docker pull $REPOSITORY:alpine
```

After loading the image, we can verify that the Docker login is cached by running the following command *(The answer should be somewhat similar)*:

```
$ docker-credential-ecr-login list

{"https://678005261235.dkr.ecr.eu-central-1.amazonaws.com":"AWS"}
```

<br>

#### 3.3.3 EC2 instance termination sequence registration

Later we will register workers to our swarm using this prototype. We have to add some feature to our prototype, in order to unregister this workers automaticaly when instance is going to be terminated. To achive that we can use [this](https://www.golinuxcloud.com/run-script-with-systemd-before-shutdown-linux/) approach:

```
$ echo -e "#! /bin/bash\ndocker swarm leave --force" | sudo tee /etc/init.d/ec2-terminate.sh \
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

<br>

#### 3.3.4 Clean up

I suggest to clean up an instance, since free tier of ESB volume usage is limited, or if you not using free tier or broke the limit of monthly usage, every single megabyte is charged.

```
$ docker rmi $(sudo docker images -q) \
  && sudo yum clean all \
  && sudo rm -rf /var/cache/yum

$ exit
```

<br>

### 3.4 Docker Swarm node EC2 AMI creation

Now our prototype is ready and we can create an AMI:

```
$ aws ec2 create-image \
  --instance-id $EC2 \
  --name "Docker Swarm Node" \
  --description "Amazon Linux 2 AMI based. \
  Docker (19.03.6), Amazon ECR Docker credentials helper. \
  Auto leave swarm on termination"

{
    "ImageId": "ami-0901c7867b6172373"
}

$ aws ec2 create-tags --resources ami-0901c7867b6172373 --tags Key=Name,Value=swarm-node
```

<br><br>

## 4. Docker Swarm EC2 cluster configuration

In this section we will finally launch our cluster as an **Auto Scaling** group with use of AMI created earlier.

<details>
<summary>Env var required</summary>

```
$ SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=production \
  --query "SecurityGroups[*].GroupId" \
  --output text)

$ SN=$(aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=production \
  --query "Subnets[*].SubnetId" \
  --output text)

$ AMI=$(aws ec2 describe-images \
  --filters Name=tag:Name,Values=swarm-node \
  --query "Images[*].ImageId" \
  --output text)

$ MIP=$(aws ec2 describe-instances \
  --filters Name=tag:Name,Values=production-manager \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text)
```
</details>

<br>

### 4.1 Docker Swarm initialization

Let's get our prototypes public ip address and login with ssh:

```
$ ssh -i ~/.ssh/ec2.pem ec2-user@$MIP
```

<br>

Once we logged in, let's initiate Docker Swarm *(Make sure to copy output, or leave the console opened, we will need this in the next step)*:

```
$ docker swarm init

Swarm initialized: current node (syeua0gytrp3y4k2qn1ul9ymd) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-5lqzkrmd1doqbcczeb8xsvz7s70s5nqiv02t3arymwijc9fcfo-6vacsu0jo5qn6mw3dn2zgcimp 10.0.0.115:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

<br>

Also, let's create a new overlay network, that we will use to connect our services:

```
$ docker network create -d overlay --scope swarm production
```

<br>

### 4.2 Swarm node EC2 launch configuration creation

Next step is to create the launch configuration, which serves as a template for future EC2 instances. In order to let our instances join swarm automatically on launch, we will use user-data file. Let's create a file called `swarm-node_user-data.txt` with the following content *(make sure to paste correct `token` and `manager's ip address`, that have been outputed in a previous step)*.

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
docker swarm join --token SWMTKN-1-5lqzkrmd1doqbcczeb8xsvz7s70s5nqiv02t3arymwijc9fcfo-6vacsu0jo5qn6mw3dn2zgcimp 10.0.0.115:2377
--//
```

<br>

Now we are ready to create a launch configuration:

```
$ aws autoscaling create-launch-configuration \
  --launch-configuration-name production \
  --image-id $AMI \
  --key-name ec2 \
  --security-groups "$SG" \
  --instance-type t2.micro \
  --iam-instance-profile swarm-node \
  --associate-public-ip-address \
  --user-data file://swarm-node_user-data.txt
```

<br>

### 4.3 Swarm cluster AWS Auto Scaling Group creation

```
$ aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name production \
  --launch-configuration-name production \
  --min-size 1 \
  --max-size 30 \
  --desired-capacity 3 \
  --health-check-grace-period 30 \
  --vpc-zone-identifier "$SN" \
  --tags "Key=Name,Value=production-worker,PropagateAtLaunch=true" \
  "Key=cluster,Value=production,PropagateAtLaunch=true" \
  "Key=role,Value=worker,PropagateAtLaunch=true"
```

Now, let's list our instances:

```
$ aws ec2 describe-instances \
  --filter Name=tag:cluster,Values=production \
  --query 'Reservations[*].Instances[*].{
  id:InstanceId,
  public_ip:PublicIpAddress,
  private_ip:PrivateIpAddress,
  status:State.Name,
  tags:Tags[*]}'
```

<details>
<summary>Response example:</summary>

```
[
    [
        {
            "id": "i-035b6f78f78637577",
            "public_ip": "3.125.33.222",
            "private_ip": "10.0.0.115",
            "status": "running",
            "tags": [
                {
                    "Key": "role",
                    "Value": "manager"
                },
                {
                    "Key": "cluster",
                    "Value": "production"
                },
                {
                    "Key": "Name",
                    "Value": "production-manager"
                }
            ]
        }
    ],
    [
        {
            "id": "i-00d40bf18e8034dc2",
            "public_ip": "18.185.149.74",
            "private_ip": "10.0.0.252",
            "status": "running",
            "tags": [
                {
                    "Key": "role",
                    "Value": "worker"
                },
                {
                    "Key": "cluster",
                    "Value": "production"
                },
                {
                    "Key": "aws:autoscaling:groupName",
                    "Value": "production"
                },
                {
                    "Key": "Name",
                    "Value": "production-worker"
                }
            ]
        },
        {
            "id": "i-0d84faf6e30aa8e09",
            "public_ip": "3.125.34.82",
            "private_ip": "10.0.0.41",
            "status": "running",
            "tags": [
                {
                    "Key": "Name",
                    "Value": "production-worker"
                },
                {
                    "Key": "aws:autoscaling:groupName",
                    "Value": "production"
                },
                {
                    "Key": "role",
                    "Value": "worker"
                },
                {
                    "Key": "cluster",
                    "Value": "production"
                }
            ]
        },
        {
            "id": "i-0f43f6fdf59060fdc",
            "public_ip": "18.185.99.169",
            "private_ip": "10.0.0.240",
            "status": "running",
            "tags": [
                {
                    "Key": "role",
                    "Value": "worker"
                },
                {
                    "Key": "Name",
                    "Value": "production-worker"
                },
                {
                    "Key": "aws:autoscaling:groupName",
                    "Value": "production"
                },
                {
                    "Key": "cluster",
                    "Value": "production"
                }
            ]
        }
    ]
]
```
</details>

<br>

### 4.5 Docker Swarm cluster on EC2 instances testing

It is time to test that our new cluster is working properly. For this purpose I have prepared two docker images available at DockerHub. Let's start couple of services in our swarm *(Make SSH connection to `production-manager` and run following commands)*. But before let's check our node list:

```
$ docker node ls

ID                            HOSTNAME                                      STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
aev7gfjc00dfwio890c7z4wp5     ip-10-0-0-17.eu-central-1.compute.internal    Ready               Active                                  19.03.6-ce
syeua0gytrp3y4k2qn1ul9ymd *   ip-10-0-0-115.eu-central-1.compute.internal   Ready               Active              Leader              19.03.6-ce
lrx2hf28sgzlyrt82zq7fluvz     ip-10-0-0-118.eu-central-1.compute.internal   Ready               Active                                  19.03.6-ce
qats8cebj2ywo2sfh5yafwtod     ip-10-0-0-236.eu-central-1.compute.internal   Ready               Active                                  19.03.6-ce
```

Now let's start [testing service](https://github.com/tikhoplav/swarm-net-test) *(Make sure to specify right region)*:

```
$ docker service create \
  --name test \
  --network production \
  --replicas 3 \
  --log-driver=awslogs \
  --log-opt awslogs-region=eu-central-1 \
  --log-opt awslogs-group=test \
  --log-opt awslogs-create-group=true \
  tikhoplav/swarm-net-test

8ipcved7eqb6v0y9412wi5aum
overall progress: 3 out of 3 tasks 
1/3: running   
2/3: running   
3/3: running   
verify: Service converged 

$ docker service ls

ID                  NAME                MODE                REPLICAS            IMAGE                             PORTS
8ipcved7eqb6        test                replicated          3/3                 tikhoplav/swarm-net-test:latest   
```

Next service is a slightly [modified nginx](https://github.com/tikhoplav/swarm-nginx-test), that forwards requests to the `test` service *(Three of the containers will start fast, while one of them will take some time. Don't panic, it's perfectly fine, since this contanier is looking for it's upstream at the different ec2 node)*:

```
$ docker service create \
  -p 80:80 \
  --name nginx \
  --network production \
  --mode global \
  --log-driver=awslogs \
  --log-opt awslogs-region=eu-central-1 \
  --log-opt awslogs-group=nginx \
  --log-opt awslogs-create-group=true \
  tikhoplav/swarm-nginx-test

mvr0ii9x6qw8g8ljcavxro484
overall progress: 4 out of 4 tasks 
syeua0gytrp3: running   
r63hz31emoec: running   
oer2ycdbrrf3: running   
owt6df52y0ys: running   
verify: Service converged

$ docker service ls

ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
mq7ttidy9pmu        nginx               global              4/4                 tikhoplav/swarm-nginx-test:latest   *:80->80/tcp
zqum3zllnxnr        test                replicated          3/3                 tikhoplav/swarm-net-test:latest       
```

<br>

We can test our cluster using saved public ip of the manager. Let's make the following request from the local terminal:

```
$ curl $MIP/test/

{
  "timestamp": "2020-04-10 18:49:17.084938608 +0000 UTC m=+34.295498134",
  "container_id": "7fb5bada0a580a9794bb256035597a5ec539f2c1c3813ed500a8b3ce76708049"
}
```

Try to run this multiple times and see how Docker Swarm integrated load balancer gives us new available node each new request.

<br>

### 4.6 Auto Scaling policy attachment

To make our cluster fully autonomous, we need to add feedback to our auto-scaling group. The following command creates a scaling policy and attaches it to our auto-scaling group. What he does is that if the CPU utilization of the EC2 instances drops below 40%, the group shrinks. And if average usage rises above 40%, the auto-scaling group releases new instances. Do not worry, this applies only to the swarm working, as our manager is not controlled by the Auto Scaling group, so you will not lose control over the cluster.

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
  --auto-scaling-group-name production \
  --policy-name production-on-cpu \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration file://scaling-policy.json
  
{
    "PolicyARN": "arn:aws:autoscaling:eu-central-1:678005261235:scalingPolicy:4ae51a5e-9a0f-434e-a079-6778958a480d:autoScalingGroupName/production:policyName/production-on-cpu",
    "Alarms": [
        {
            "AlarmName": "TargetTracking-production-AlarmHigh-7dbffa52-0765-4af6-a9c2-fe14130ea395",
            "AlarmARN": "arn:aws:cloudwatch:eu-central-1:678005261235:alarm:TargetTracking-production-AlarmHigh-7dbffa52-0765-4af6-a9c2-fe14130ea395"
        },
        {
            "AlarmName": "TargetTracking-production-AlarmLow-81ea8866-fddb-4cf3-a0c3-ad8a8c9ce737",
            "AlarmARN": "arn:aws:cloudwatch:eu-central-1:678005261235:alarm:TargetTracking-production-AlarmLow-81ea8866-fddb-4cf3-a0c3-ad8a8c9ce737"
        }
    ]
}
```

<br>

---

Congratulations! You have launched you Docker Swarm cluster on AWS EC2 isntances with fully automated scaling strategy.