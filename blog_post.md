
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

    from aiohttp import web
    import aiohttp_cors
    import json

    async def healthcheck(_):
        headers = {
            "Cache-Control": "no-cache, no-store, must-revalidate",
            "Pragma": "no-cache",
            "Expires": "0",
        }
        return web.Response(text=json.dumps("Healthy"), headers=headers, status=200)
    
    async def helloworld(_):
        return web.Response(text="<h1>HELLO WORLD</h1>", content_type='text/html', status=200)


    app = web.Application()
    cors = aiohttp_cors.setup(app)
    app.router.add_get("/healthcheck", healthcheck)
    app.router.add_get("/", helloworld)

    cors = aiohttp_cors.setup(app, defaults={
        "*": aiohttp_cors.ResourceOptions(
                allow_credentials=True,
                expose_headers="*",
                allow_headers="*",
            )
    })

    # Configure CORS on all routes.
    for route in list(app.router.routes()):
        cors.add(route)

    if __name__ == "__main__":
        print("Starting service")
        web.run_app(app, host="0.0.0.0", port=(5000))


### Deployment

To deploy this system, I am using the AWS CLI with CloudFormation. It is as simple as:

    aws cloudformation create-stack --stack-name service --template-body file://template.yml --capabilities CAPABILITY_NAMED_IAM

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

    LoadBalancerSecGroup:
        Type: AWS::EC2::SecurityGroup
        Properties:
            GroupDescription: Load balancer only allow http port traffic
            VpcId: !ImportValue VPCID
            SecurityGroupIngress:
            CidrIp: 0.0.0.0/0
            FromPort: 80
            IpProtocol: TCP
            ToPort: 80
    LoadBalancer:
        Type: AWS::ElasticLoadBalancingV2::LoadBalancer
        Properties:
            SecurityGroups:
            - !Ref LoadBalancerSecGroup
            Subnets:
            - !ImportValue PublicSubnetA
            - !ImportValue PublicSubnetB


### Cluster

The cluster orchestrates containers running on the machines. If you are unfamiliar with Docker, check out this [article](https://medium.com/@t3chflicks/home-devops-pipeline-a-junior-engineers-tale-1-4-336ed07a6ec0). Dockerising the Python web server can be done in few lines:

    FROM python:3.7-slim
    COPY requirements.txt /app/requirements.txt
    WORKDIR /app
    RUN pip install -r requirements.txt
    COPY src /app
    EXPOSE 5000
    CMD python app.py

The following template configures an ECS cluster and ECR to store the Docker image of the Python web server.

    ECSCluster:
        Type: 'AWS::ECS::Cluster'
        Properties:
        ClusterName: ECSCluster
        CapacityProviders:
            - FARGATE_SPOT
        DefaultCapacityProviderStrategy:
            - CapacityProvider: FARGATE_SPOT
            Weight: 1
        ClusterSettings:
            - Name: containerInsights
            Value: enabled
    
    ECRRepository: 
        Type: AWS::ECR::Repository

### Machines

Scaling up and down with demand requires more than one machine in the autoscaling group of EC2s. Launch Templates allow the configuration of the machines:

* SSM for keyless SSH access and patching

* User Data script joins the machine to the ECS cluster
>
    
    LaunchTemplate:
        Type: AWS::EC2::LaunchTemplate
        Metadata:
            AWS::CloudFormation::Init:
                config:
                packages:
                    yum:
                    amazon-ssm-agent: []
                commands:
                    00_amazon_ssm_agent_start:
                    command: systemctl start amazon-ssm-agent
                files:
                    /etc/cfn/cfn-hup.conf:
                    content: |
                        [main]
                        stack=${AWS::StackId}
                        region=${AWS::Region}
                        interval=1
                    mode: "000400"
                    owner: root
                    group: root
                    /etc/cfn/hooks.d/cfn-auto-reloader.conf:
                    content: |
                        [cfn-auto-reloader-hook]
                        triggers=post.update
                        path=Resources.LaunchTemplate.Metadata.AWS::CloudFormation::Init
                        action=/opt/aws/bin/cfn-init --verbose --stack=${AWS::StackName} --region=${AWS::Region} --resource=LaunchTemplate
                        runas=root
            services:
                sysvinit:
                cfn-hup:
                    enabled: true
                    ensureRunning: true
                    files:
                    - "/etc/cfn/cfn-hup.conf"
                    - "/etc/cfn/hooks.d/cfn-auto-reloader.conf"
        Properties: 
            LaunchTemplateData: 
                CreditSpecification: 
                CpuCredits: Unlimited
                ImageId: 'ami-09266271a2521d06f' #ecs optimised image for eu-west-1
                InstanceType: t2.micro 
                IamInstanceProfile:
                Arn: !GetAtt InstanceProfile.Arn
                Monitoring: 
                Enabled: true
                SecurityGroupIds: 
                - !Ref EC2SecurityGroup
                UserData:
                Fn::Base64:
                    Fn::Sub: 
                    - |
                        #!/bin/bash -xe
                        yum update -y
                        yum install -y aws-cli aws-cfn-bootstrap
                        echo ECS_CLUSTER=${Cluster} >> /etc/ecs/ecs.config
                        echo ECS_ENABLE_SPOT_INSTANCE_DRAINING=1 >> /etc/ecs/ecs.config
                        # -r needs to be name of the LaunchTemplate resource. In this case: LaunchTemplate
                        /opt/aws/bin/cfn-init -v -s ${AWS::StackName} -r LaunchTemplate --region ${AWS::Region}
                    - { Cluster: !ImportValue ECSCluster }

The scaling configuration sets the group of EC2s to scale up when total CPU usage exceeds 70%.

    AutoScalingPolicy:
        Type: AWS::AutoScaling::ScalingPolicy
        Properties:
        AutoScalingGroupName: !Ref AutoScalingGroup
        PolicyType: TargetTrackingScaling
        TargetTrackingConfiguration:
            PredefinedMetricSpecification:
            PredefinedMetricType: ASGAverageCPUUtilization
            TargetValue: 70

Spot instances are far cheaper than on demand. The recommended way of taking advantage of spot instances is by creating a SpotFleet — a collection of different types of instances which increase the likelihood of maintaining desired capacity.

    AutoScalingGroup: 
        Type: AWS::AutoScaling::AutoScalingGroup
        Properties:
        MinSize: "1"
        MaxSize: "2"
        DesiredCapacity: "2"
        HealthCheckGracePeriod: 300 
        MixedInstancesPolicy:
            InstancesDistribution:
            OnDemandBaseCapacity: 1
            OnDemandPercentageAboveBaseCapacity: 0
            SpotAllocationStrategy: capacity-optimized
            LaunchTemplate:
            LaunchTemplateSpecification: 
                LaunchTemplateId: !Ref LaunchTemplate
                Version: !GetAtt LaunchTemplate.LatestVersionNumber
            Overrides:
                - InstanceType: t3.micro
                - InstanceType: t3a.micro
                - InstanceType: t2.micro
        VPCZoneIdentifier:
            - !ImportValue PrivateSubnetA
            - !ImportValue PrivateSubnetB

### Service

We configure the load balancer to listen on port 80 for HTTP requests and send them to a Target Group — a reference we can use when defining the service to access traffic.

    TargetGroup:
        Type: AWS::ElasticLoadBalancingV2::TargetGroup
        Properties:
        Port: 5000
        Protocol: HTTP
        VpcId: !ImportValue VPCID
        HealthCheckIntervalSeconds: 60
        HealthCheckTimeoutSeconds: 5
        UnhealthyThresholdCount: 5
        HealthCheckPath: /healthcheck
        TargetGroupAttributes:
            - Key: deregistration_delay.timeout_seconds
            Value: 2
            
    LoadBalancerListener:
        Type: AWS::ElasticLoadBalancingV2::Listener
        Properties:
        DefaultActions:
            - TargetGroupArn: !Ref TargetGroup
            Type: forward
        LoadBalancerArn: !ImportValue LoadBalancerArn
        Port: 80
        Protocol: HTTP
        
    ListenerRule:
        Type: AWS::ElasticLoadBalancingV2::ListenerRule
        Properties:
        Actions:
            - TargetGroupArn: !Ref TargetGroup
            Type: forward
        Conditions:
            - Field: path-pattern
            Values:
                - '*'
        ListenerArn: !Ref LoadBalancerListener
        Priority: 1

ECS runs the Task Definition as a persistent service using the web server image in ECR.

    TaskDefinition:
        Type: AWS::ECS::TaskDefinition
        Properties:
        Family: AppTaskDefinition
        TaskRoleArn: !GetAtt TaskRole.Arn
        ExecutionRoleArn: !ImportValue ExecutionRoleArn
        Memory: 0.5Gb
        Cpu: 256
        ContainerDefinitions:
            - Name: ServiceContainer
            PortMappings:
                - ContainerPort: 5000
            Essential: true
            Image: <your_image_tag>
            LogConfiguration:
                LogDriver: awslogs
                Options:
                awslogs-group: !Ref LogGroup
                awslogs-region: !Ref 'AWS::Region'
                awslogs-stream-prefix: ecs
                
    Service:
        Type: AWS::ECS::Service
        DependsOn:
        - ListenerRule
        Properties:
        Cluster: !ImportValue ECSCluster
        LaunchType: EC2
        DesiredCount: 2
        LoadBalancers:
            - ContainerName: ServiceContainer
            ContainerPort: 5000
            TargetGroupArn: !Ref TargetGroup
        DeploymentConfiguration:
            MinimumHealthyPercent: 100
            MaximumPercent: 300
        HealthCheckGracePeriodSeconds: 30
        TaskDefinition: !Ref TaskDefinition

Configuring an auto-scaling policy on the containers works in much the same way as the EC2 machines. This is because they have defined memory and CPU so can be scaled based on those metrics, too:

    AutoScalingTarget:
        Type: AWS::ApplicationAutoScaling::ScalableTarget
        Properties:
        MaxCapacity: 3
        MinCapacity: 2
        ResourceId: !Join ["/", [service, !ImportValue ECSCluster, !GetAtt Service.Name]]
        RoleARN: !ImportValue ECSServiceAutoScalingRoleArn
        ScalableDimension: ecs:service:DesiredCount
        ServiceNamespace: ecs
        
    AutoScalingPolicy:
        Type: AWS::ApplicationAutoScaling::ScalingPolicy
        Properties:
        PolicyName: ServiceAutoScalingPolicy
        PolicyType: TargetTrackingScaling
        ScalingTargetId: !Ref AutoScalingTarget
        TargetTrackingScalingPolicyConfiguration:
            PredefinedMetricSpecification:
            PredefinedMetricType: ECSServiceAverageCPUUtilization
            ScaleInCooldown: 10
            ScaleOutCooldown: 10
            TargetValue: 70

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