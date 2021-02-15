
# AWS Auto Scaling Spot Fleet Cluster — Quickstart with CloudFormation

Running applications on separate machines quickly becomes an inefficient use of compute and therefore money. In this article, I demonstrate how to create a service which runs on Amazon’s Elastic Container Service with an auto scaling SpotFleet using CloudFormation.

![Photo by [Hello I'm Nik 🎞 ](https://unsplash.com/@helloimnik?utm_source=medium&utm_medium=referral)on U[nsplash](https://unsplash.com?utm_source=medium&utm_medium=referral)](https://cdn-images-1.medium.com/max/5184/0*qBk48TLqhA4e-ZO0)*

Photo by [Hello I'm Nik 🎞 ](https://unsplash.com/@helloimnik?utm_source=medium&utm_medium=referral)on U[nsplash](https://unsplash.com?utm_source=medium&utm_medium=referral)*

Before beginning this article, I want to define some AWS lingo. Otherwise, this may all just go over your head:

* [EC2](https://docs.aws.amazon.com/ec2/index.html) — Elastic Compute Service

* [ASG](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) — Auto Scaling Group

* [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancer-tutorials.html) — Application Load Balancer

* [SpotFleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-fleet.html) — a collection of different machine types of spot instances

* [ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) — Elastic container service — Orchestrates containers

* [ECR](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/private-auth-container-instances.html) — Elastic Container Registry — Stores Docker images

Cloud Computing enables the spinning up and tearing down of servers as compute or memory required to run an application scales. EC2s on AWS can be grouped together in an auto scaling group (ASG) which can scale on metrics such as CPU load and memory.

### Architecture

For this example, I am going to create a Dockerised Python web server and deploy it to an ECS cluster which auto scales the number of containers whilst the ASG of machines it is running on scales too. An ALB is used to create an API which load balances the containers running the service. The machines forming the ASG consist of a single on-demand instance and a SpotFleet comprising two Spot instances of different types of machines.

![](https://cdn-images-1.medium.com/max/2000/1*gzezyNoLlVE2XFvh7YucqQ.png)

### The service

The service is an asynchronous Python web server running on port 5000 with CORS enabled. Note that the healthcheck endpoint is required for ECS to keep track of the service.



### Deployment

To deploy this system, I am using the AWS CLI with CloudFormation. It is as simple as:

    `aws cloudformation create-stack --stack-name service --template-body file://template.yml --capabilities CAPABILITY_NAMED_IAM`

The deployment is split into five templates:

* `vpc.yml`

* `load_balancer.yml` 

* `cluster.yml`

* `machines.yml`

* `service.yml`

## Let’s Build! 🔩

### VPC

I am building this service inside a VPC described in a previous [article](https://medium.com/@t3chflicks/virtual-private-cloud-on-aws-quickstart-with-cloudformation-4583109b2433), but it’s pretty standard. There are three public and three private (hybrid) subnets.

### Load Balancer

The service requires a public facing load balancer which distributes HTTP requests to the machines running the web server.



### Cluster

The cluster orchestrates containers running on the machines. If you are unfamiliar with Docker, check out this [article](https://medium.com/@t3chflicks/home-devops-pipeline-a-junior-engineers-tale-1-4-336ed07a6ec0). Dockerising the Python web server can be done in few lines:



The following template configures an ECS cluster and ECR to store the Docker image of the Python web server.



### Machines

Scaling up and down with demand requires more than one machine in the autoscaling group of EC2s. Launch Templates allow the configuration of the machines:

* SSM for keyless SSH access and patching

* User Data script joins the machine to the ECS cluster.



The scaling configuration sets the group of EC2s to scale up when total CPU usage exceeds 70%.



Spot instances are far cheaper than on demand. The recommended way of taking advantage of spot instances is by creating a SpotFleet — a collection of different types of instances which increase the likelihood of maintaining desired capacity.



### Service

We configure the load balancer to listen on port 80 for HTTP requests and send them to a Target Group — a reference we can use when defining the service to access traffic.



ECS runs the Task Definition as a persistent service using the web server image in ECR.



Configuring an auto-scaling policy on the containers works in much the same way as the EC2 machines. This is because they have defined memory and CPU so can be scaled based on those metrics, too:



## Usage

After a successful deployment, it is possible to access the DNS name of the ALB in the EC2 section of the AWS console, which should look something like:

    loadb-LoadB-R7RVQD09YC9O-1401336014.eu-west-1.elb.amazonaws.com

Now it is possible to make a request to this URL and get a response:

    $ curl loadb-LoadB-R7RVQD09YC9O-1401336014.eu-west-1.elb.amazonaws.com'
    <h1>HELLO WORLD!</h1>

### Use a Domain Name

AWS provides an ugly Load Balancer address such as:

    loadb-LoadB-R7RVQD09YC9O-1401336014.eu-west-1.elb.amazonaws.com

But it’s quite simple to use a custom domain using AWS. Firstly, transfer your DNS management to [Route 53](https://aws.amazon.com/route53/) and then create a new record set aliased to the load balancer.

### Thanks For Reading

I hope you have enjoyed this article. If you like the style, check out [T3chFlicks.org](https://t3chflicks.org/Projects/aws-quickstart-series) for more tech focused educational content ([YouTube](https://www.youtube.com/channel/UC0eSD-tdiJMI5GQTkMmZ-6w), [Instagram](https://www.instagram.com/t3chflicks/), [Facebook](https://www.facebook.com/t3chflicks), [Twitter](https://twitter.com/t3chflicks)).



Resources:

* [https://aws.amazon.com/blogs/containers/deep-dive-on-amazon-ecs-cluster-auto-scaling/](https://aws.amazon.com/blogs/containers/deep-dive-on-amazon-ecs-cluster-auto-scaling/)

* [https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-autoscaling-autoscalinggroup-mixedinstancespolicy.html](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-autoscaling-autoscalinggroup-mixedinstancespolicy.html)

* [https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-purchase-options.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-purchase-options.html)