<br>

# GitLab CI/CD with Docker Swarm cluster on AWS EC2 instances. 

Greetings, reader. In this series of pages you will find some topics related to organization of the **AWS** **EC2** cluster under control of **Docker Swarm**, **GitLab CI/CD** establishment and some other usefull tips.

<details>
	<summary>Rrerequisites:</summary><br>

  I'm not a DevOps guy. I'm a full stack developer. But it happend that company that I worked with has faced the necessity of CI/CD integration into the workflow. Let me tell a little more about it.

  Despite the fact that company has already had self-hosted **GitLab** server with couple of projects, which has had configured gitlab CI/CD pipelines, and the fact that code from this repositories has already being built, tested and deployed to **AWS**, the urgent task of CI/CD implementation has appeared.

  The reason for that is pretty common, the guy who have implemented it has gone, and it turned out that no one else have any clue about how does it work and how to modify it.

  Everything would be fine, the search for a professional DevOps engineer was already in process, if not for the fact that work of the team of dozen developers couldn't reach production for a month already.

  The thing is that it has been desided that new products that team has developed should have been based on the microservice architecture. Microservices meaned many new separated code repositories, with it's own software dependencies, testing, building and deployment.

  I'm not a DevOps guy. I had worked with docker before, and thats it. But story turned that I was that guy who figured out how can new CI/CD pipeline coupled with the existing AWS resources be implemented. And this article is an essence of that experience squeezed in to step-by-step instruction, that any non-DevOps guy like me could understanc and reproduce.
</details>

<br>

## Articles:

- [How to organize Docker Swarm cluster on AWS EC2 instances in Auto Scaling Group](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/swarm-cluster-on-aws-ec2.md);

- [How to host GitLab Runner with the AWS EC2 instances in Auto Scaling Group](https://github.com/tikhoplav/aws-gitlab-cicd/blob/master/gitlab-runner-in-ec2.md);