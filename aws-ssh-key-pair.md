# Create SSH key pair with AWS console

On this page we will create a key pair to use it in process of SSH connection establishment to our EC2 instances. It is good practice to create separated key pair for any group if EC2 instances you launch. But for the first time you only need one.

- Go to [Keys Pairs management console](https://console.aws.amazon.com/ec2/v2/home?#KeyPairs);

- `Create key pair`:

  - Name - use name that related to your ec2 instance group. For example, if you are launching instances to join production cluster, name it `production`;

  - File format - If you are using Linux or Mac chose `pem`. If windows - `ppk`;

![Key pair creation](https://user-images.githubusercontent.com/62797411/78598652-55ca5500-7858-11ea-99c8-0bb103c22c04.png)

<br>

After key pair will be created the download process will begin. Make sure to store it in a safe place, `~/.ssh/` for example. In addition, if you are using Linux, you have to apply permission to this file:

```
$ sudo shmod 600 ~/.ssh/prduction.pem
```

<br><br><br>

## Establish SSH connection with EC2 instance using .pem key

To login to EC2 instance need to figure out it's public ip address and also we have to make sure that instances security group is available for SSH connection. In [instances management console](https://console.aws.amazon.com/ec2/v2/home?#Instances) choose instance you want to login and find the `IPv4 Public IP` graph:

![Instance public ip](https://user-images.githubusercontent.com/62797411/78601375-59140f80-785d-11ea-9e50-b5ee599aaaba.png)

<br>

Also, make sure that SSH connection *(TCP on port 22)* is allowed in instance sequrity group:

![ec2 instance inbound rules](https://user-images.githubusercontent.com/62797411/78601575-aee8b780-785d-11ea-8d77-189cd4531e28.png)

![ec2 instance inbound rules list](https://user-images.githubusercontent.com/62797411/78601631-c4f67800-785d-11ea-978d-c440b4fa67fc.png)

<br>

If everything is set up, use next command to establish connection:

```
$ ssh -i ~/.ssh/production.pem ec2-user@<your ec2 instance public ip address>
```