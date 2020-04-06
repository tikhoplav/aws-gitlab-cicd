# Pull image to ECR using AWS CLI

On this page we will push a docker image to the ECR. This image will be `nginx:alpine` so that you would be able to test your cluster with it.

**Requirements:**

- [Docker](https://docs.docker.com/install/);

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html);

- [Admin priveleged account](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/aws-admin-iam.md) with `Programmatic access` and `AWS Management Console access`;

<br>

First of all, we need to set our account credentials to AWS CLI. You could see credentials in `.csv` file, or you could create new [here](https://console.aws.amazon.com/iam/home?#/users/admin?section=security_credentials).

![aws creation of new secret access key pair](https://user-images.githubusercontent.com/62797411/78605753-d0996d00-7864-11ea-8164-728cbaf1b597.png)

<br>

Use interactive console script to paste your credentials and register your docker daemon using:

```
$ aws configure

$ $(aws ecr get-login --no-include-email)
```

<br>

Next we need to create new repository by running *(Copy value of the `repositoryUri` field, as it is needed in the next step)*:

```
$ aws ecr create-repository --repository-name nginx
{
    "repository": {
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
}
```

<br>

Finally, let's push image to this repository:

```
$ docker pull nginx:alpine \
  && docker tag nginx:alpine 678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx:alpine
  && docker push 678005261235.dkr.ecr.eu-central-1.amazonaws.com/nginx:alpine
```

![example of the ECR repository](https://user-images.githubusercontent.com/62797411/78608513-bc0ba380-7869-11ea-929d-694e49ae9225.png)