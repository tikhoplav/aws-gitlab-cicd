# Creation administrator AMI with AWS management console

On this page we will create AMI account associated to our AWS account with administrator permissions. It is highly suggested to use AMI instead of the root account. Also, AMI is mandatory if you want to manage AWS resources using `aws cli`.

- Go to [Groups IAM console](https://console.aws.amazon.com/iam/home?#/groups);

- `Create New Group`:

  - Group Name - `admins`;

  - Attach Policy:

    - Find and attach `AdministratorAccess` policy;

    ![aws administrator iam policies](https://user-images.githubusercontent.com/62797411/78603434-cffed780-7860-11ea-80a1-b83545289287.png)

<br>

After that you shoud create actual user:

- Go to [Users IAM console](https://console.aws.amazon.com/iam/home?#/users);

- `Add user`:

  - User name - `admin`;

  - Access type:

    - Check `Programmatic access`;

    - Check `AWS Management Console access`;

    - Set Console Password;

  ![aws admin user creation](https://user-images.githubusercontent.com/62797411/78603707-426fb780-7861-11ea-9db9-487b5f3179b7.png)

  - Set Permissions:

    - Add User to Group - Select `admins` group;

  ![choose admins group during admin user iam creation](https://user-images.githubusercontent.com/62797411/78603972-b316d400-7861-11ea-94c0-07a0faf8c488.png)

  - `Add Tags`;

  - `Review User`;

  - `Add User`:

    - At this stage i suggest to download `.csv` file.

  ![complete admin user creation](https://user-images.githubusercontent.com/62797411/78604580-b2cb0880-7862-11ea-900e-691428bcfa04.png)

<br>

> This user shall not be used in CI/CD pipeline, since it’s credentials is not suppose to be published.