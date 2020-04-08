# Configure Network for Docker Swarm cluster on EC2 instances

In the page we will go through the process of **Amazon Web Service** network creation and configuration. We will create special **Virtual Private Network** (**VPC**) to allow docker containers talk to each other while beinig part of **Docker Swarm** and receiving incomming connections on public ports.

**Requirements**:

- [Admin privileged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `AWS Management Console access`;

<br><br>

## Create VPC

This VPC will be used to provide connection of EC2 instances all over the cluster.

- Go to [VPC management console](https://console.aws.amazon.com/vpc/home?#vpcs:sort=VpcId);

- `Create VPC`:

	- Name tag - pick the name of the VPC related to the cluster, for example if it’s a production cluster, then name it `production`;

	- IPv4 CIDR block - Set any IP addresses range, for example 10.0.0.0/24 *(This could be later modified in VPC `Actions > Edit CIDRs` dialog)*;

	- IPv6 CIDR block - No IPv6 CIDR Block;

	- Tenancy - Default;

- `Create`.

![docker swarm cluster on ec2 vpc example](https://user-images.githubusercontent.com/62797411/78764750-b01a0180-798f-11ea-9b68-91647c752010.png)

<br><br>

## Create Internet Gateway

Internet gateway serves as the single entrypoint of all outgoing traffic of VPC. In order to get response from the Swarm container we need to create one. Later this gateway will be used in the Route Table specification, after the Subnet will be created.

- Go to [Internet Gateway management console](https://console.aws.amazon.com/vpc/home?#igws:sort=internetGatewayId);

- `Create internet gateway`:

	- Name tag - pick the name of the Internet gateway related to cluster, for example if it’s a production cluster, then name it `production`;

<br>

Now VPC needs to be attached to the new Internet Gateway. To do so find `production` internet gateway in [console](https://console.aws.amazon.com/vpc/home?#igws:search=production;sort=internetGatewayId), select it, `Actions > Attach to VPC`:

- VPC - Select VPC with corresponding name `production`;

![Swarm cluster on ec2 VPC attachment to internet gateway](https://user-images.githubusercontent.com/62797411/78778145-78b55000-79a3-11ea-81b4-9f1916baff11.png)

<br>

Make sure that [internet gateway](https://console.aws.amazon.com/vpc/home?#igws:search=production;sort=internetGatewayId) state changed to `attached`:

![Swarm cluster on ec2 internet gateway example](https://user-images.githubusercontent.com/62797411/78778297-bb772800-79a3-11ea-9d35-8e7b66c7264d.png)

<br><br>

## Create Security Group

Security Group acts like a set of rules, applied to all incoming and outcomming traffic. It’s mandatory to create a Security group for swarm worker instances with opened 80 port. But you could create a shared security group for workers as well as managers nodes, as in example below.

- Go to [Security Group management console](https://console.aws.amazon.com/vpc/home?#SecurityGroups:sort=groupId);

- `Create new security group`:

	- Security group name - pick the name of the security group related to cluster, for example if it’s a production cluster, then name it `production`;

	- Description - `Allow SSH, http, https requests, and openes all swarm needed ports`;

	- VPC - select VPC with name tag `production`;

![swarm cluster on ec2 security group example](https://user-images.githubusercontent.com/62797411/78774614-91226c00-799d-11ea-9c85-2fe86e743408.png)

<br>

Next step is to set up inboud traffic rules. Select `production` security group in [console](https://console.aws.amazon.com/vpc/home?#SecurityGroups:search=production;sort=groupId). In the opened data window, select `Inbound Rules` and add following *(Each row `source` should be set to `custom 0.0.0.0/0`)*:

1. TCP 22 - SSH access;

1. TCP 80 - Public HTTP port (for Nginx / Apache service);

1. TCP 443 - Public HTTPS port (for Nginx / Apache service);

1. TCP 2377 - Docker Swarm cluster management communication;

1. TCP 7946 - Docker Swarm inter node communication;

1. UDP 7946 - Docker Swarm inter node communication;

1. UDP 4789 - Docker Swarm Overlay network traffic;

![Swarm cluster on ec2 security group inbound rules](https://user-images.githubusercontent.com/62797411/78775732-620cfa00-799f-11ea-8df2-363d7281f10e.png)

> You could find Docker Swarm related ports in the [oficial documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts).

<br>

Also we need to make sure that outgoing traffic is allowed. In the same info window choose `Outbound Rules` tab:

![Swarm cluster on ec2 security group outbound rules](https://user-images.githubusercontent.com/62797411/78776626-d5fbd200-79a0-11ea-928a-a0be3f278dbc.png)

<br><br>

## Create Subnet

The subnet is required to launch EC2 instances inside VPC. In order to make sure that containers in swarm are able to connect each other, let's create a single subnet for our VPC.

- Go to [Subnet management console](https://console.aws.amazon.com/vpc/home?#subnets:sort=SubnetId);

- `Create subnet`:

	- Name tag - Name - pick the name of the subnet related to cluster, for example if it’s a production cluster, then name it `production`;

	- VPC - choose the VPC with name tag `production`;

	- Availability zone - No preference;

	- IPv4 CIDR block* - fill the whole range of VPC CIDR range.

![Swarm cluster on ec2 subnet creation](https://user-images.githubusercontent.com/62797411/78777072-91bd0180-79a1-11ea-976a-46b580ea7b6a.png)

<br><br>

## Configure Route Table

After we have created a subnet, new route table should be automatically created. The easiest way to find it, is through the subnet we have just created. You can find it in the [subnet console](https://console.aws.amazon.com/vpc/home?#subnets:search=production;sort=SubnetId) by the `name`. Select that subnet, and you will find link to related route table in the info window:

![Swarm cluster on ec2 subnet route table link](https://user-images.githubusercontent.com/62797411/78778800-89b29100-79a4-11ea-9177-555d34423b03.png)

<br>

Select the corresponding route table and `Actions > Edit routes`. Here we need to add route with destination - `0.0.0.0/0` and the target of [internet gateway](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-network.md#create-internet-gateway), we have created earlier.

![Swarm cluster on ec2 internet gateway attachment](https://user-images.githubusercontent.com/62797411/78779075-ffb6f800-79a4-11ea-9398-f31c2c98c308.png)

---