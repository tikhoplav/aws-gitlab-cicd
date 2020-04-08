# Create Docker Swarm worker EC2 IAM Role

At this page we will create role that will be used by EC2 instance. This role identifies capabilities of our instance to use other AWS resources. For example, we will grant our **Docker Swarm worker** instance ability to pull Docker images from **ECR** as well as send logs to the **AWS CloudWatch**.

- Go to [roles management console](https://console.aws.amazon.com/iam/home#/roles);

- `Create Role`:

  - Select type of trusted entity - select `AWS service`;

  - Choose a use case - select `EC2 - Allows EC2 instances to call AWS services on your behalf`;

  ![EC2 role creation](https://user-images.githubusercontent.com/62797411/78597675-a2ad2c00-7856-11ea-9b48-54579d6a854c.png)

	- `Next: Permissions`;

	  - Attach permissions policies:

	  	- `AmazonEC2ContainerRegistryReadOnly`;

	  	- `AmazonAPIGatewayPushToCloudWatchLogs`;

	- `Next: Tags`;

	  - *(Optional)* Here you can add key-value pairs to identify role in the list. I suggest to set tag `Name` with value `swarm-worker`;

	- `Next: Review`;

	  - Role name - Clear names related to use cases highly suggested, for example `swarm-worker`;

	  - Role description *(Optional)* - `Allows EC2 instance to pull Docker images from ECR and send logs to CloudWatch`;

	- `Create role`.

<br>

Later, if you would like to grant more permissions to Docker Swarm worker instance, for example, to use **S3**, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/swarm-worker). New permissions would automatically applied to all running instances using this role.