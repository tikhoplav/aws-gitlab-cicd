# Create GitLab Runner IAM Role

At this page we will create role that will be used by EC2 instance. This role identifies capabilities of our instance to use other AWS resources. For example, we will grant our GitLab Runner instance ability to push and pull Docker images to ECR.

- Go to [roles management console](https://console.aws.amazon.com/iam/home#/roles);

- `Create Role`:

  - Select type of trusted entity - select `AWS service`;

  - Choose a use case - select `EC2 - Allows EC2 instances to call AWS services on your behalf`;

  ![EC2 role creation](https://user-images.githubusercontent.com/62797411/78597675-a2ad2c00-7856-11ea-9b48-54579d6a854c.png)

- `Next: Permissions`;

  - Attach permissions policies - paste `EC2Container` to the search bar and select *(checkbox)* `AmazonEC2ContainerRegistryFullAccess`;

- `Next: Tags`;

  - *(Optional)* Here you can add key-value pairs to identify role in the list. I suggest to set tag `Name` with value `gitlab-runner`;

- `Next: Review`;

  - Role name - Clear names related to use cases highly suggested, for example `gitlab-runner`;

  - Role description *(Optional)* - `Allows EC2 instance to pull and push Docker images to ECR`;

- `Create role`.

> The purpose of giving a GitLab runner full access to ECR is that give you an ability to delete repositories through CI/CD pipeline. For example, you could create a special job, that will unregister you microservice, clean up and set free all resources.
>
> If you feel that this permissions is more than you can allow runner to have, you could use `AmazonEC2ContainerRegistryPowerUser` policy instead.

<br>

Later, if you would like to grant more permissions to GitLab Runner instance, for example, to use **S3** to cache builds etc, you could add more policies to this [role](https://console.aws.amazon.com/iam/home#/roles/gitlab-runner). New permissions would automatically applied to all running instances using this role.