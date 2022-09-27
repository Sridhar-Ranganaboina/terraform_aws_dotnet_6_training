# terraform_aws_dotnet_6_training

HOW TO DEPLOY A .NET CORE API TO AWS USING TERRAFORM AND DOCKER
In this article, we will build a .Net 6 API, containerise it using Docker, configure the infrastructure required to run it in AWS using terraform.

PRE-REQUSITES
In order to follow this tutorial, you will need the following:-

AWS Account with Access Key and Secret Key
Docker installed on your machine
Terraform installed
A Terminal (command prompt, Bash (im using bash), zsh etc)
.Net 6 installed on your machine
A code editor (VS code (im using this), Visual studio, Rider, Notepad++)
The source code for everything mentioned in this tutorial is here

CODE AND BUILD THE API
Open up your favourite terminal, i’m using bash, run the following commands to create a folder WebApi and initialise a .net core template project.

> mkdir WebApi && cd WebApi
> dotnet new webapi -o src --no-https
We used the option of “no-https” because we dont want to deal with TLS certificate right now. It’s quite verbose to set up. We will look into that in another tutorial

Now let’s build and run the api we’ve just set up

> dotnet run --project src --urls=http://*:5000/
Navigate to localhost:5000/WeatherForecast


output from the browser
ADD HEALTHCHECK SUPPORT TO API
AWS Application Load balancer requires the destination service to have a healthcheck endpoint in order for it to understand your service health. Let’s add one.
Detailed steps on how to set up healthchecks in .Net is on aspnet core site. In this tutorial, we will implement the basic health probe.

Advertisements

REPORT THIS AD

In your program.cs , add the following lines

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks(); //add this 

var app = builder.Build();

app.MapHealthChecks("/health"); //add this

app.Run();
Run your application again to ensure that it still compiles and the health check endpoint works.

> dotnet new webapi -o src/SampleAPI --no-https
You should see something like below when you navigate to http://localhost:5000/health


NOW LET’S DOCKERIZE THE API
Add a new Dockerfile into src folder with the following content

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["src.csproj", "."]
RUN dotnet restore "./src.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "src.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "src.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "src.dll"]
Now let’s test our dockerfile by building and running it.

Navigate to the src folder and run the following command. The first command builds an image from docker file and the second runs the image in a container using the detached and auto-delete-after-stopping flags.

docker build --rm -t sampleapi .

docker run -p 5000:80 --rm -d sampleapi
Navigate to localhost:5000/weatherforecast , you should see the weather data output.

AWS INFRA SETUP
It is now time to move to AWS side of things. Below is a diagram illustrating the infrastructure set up and flow.


Infra for .Net API setup in AWS
You will notice some numbers near the arrows. They depict the chronological order with which we will set up our infra

We will push the image that we previously built to AWS container registry (ECR)
We will then set up the components needed to a basic API in AWS Fargate.
2a. set up a network security group that will dictate the type of TCP traffic that will hit the load balancer.
2b. Set up a load balancer that route traffic to the listeners
2c. Set up listeners for our our API and a fixed response to verify that the load balancer is being hit
2d. Set up our ECS service that will use task definitions to create a serverless
2e. Configure the service’s to reference (pull) the image
Make a call to the API in AWS using the load balancer’s url
Let’s start with Step 1. Let’s create an ECR repository in AWS. This is where our image will be pushed and stored. Before we do that, let’s set up AWS CLI credential.

SETUP AWS CLI CREDENTIAL

If you have aws configured on your machine, feel free to skip this.
In your terminal, run the following command. The AWS doc explain in detail how to get your ACCESS and SECRET key.
Basically, you need to go to IAM section, choose your username, then the security credentials. There you will find options to generate the key pair.

export AWS_ACCESS_KEYID=<your access key>

export AWS_SECRET_ACCESS_KEY=<your secret access key>
IF you are using AWS SSO, run the following command

export AWS_SESSION_TOKEN=<your session token>
Test your AWS CLI login

aws sts get-caller-identity
you should see an account number if everything works correctly. Otherwise comment on this blog with your error and i’ll try and help. Make note of that account number.

export ACCOUNT_NUMBER=<number>
CREATE ECR NEW REPOSITORY
aws ecr create-repository --repository-name sample_api
In order to push image to your new ECR, you need to login. Run the following commands to login.

EXPORT AWS_DEFAULT_REGION=<REGION>
I’m using eu-west-2

EXPORT ECR_URL=${ACCOUNT_NUMBER}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
LOGIN TO ECR

In order to push your local image to ECR, you need to login ECR.

aws ecr get-login-password --region <YOUR-AWS-DEFAUT-REGION> | docker login --username AWS --password-stdin ${ECR_URL}
PUSH YOUR IMAGE TO ECR
Navigate to the directory where the docker is located (/src). And run the follwong commands.

docker build --rm --pull -f Dockerfile -t sample_api . 
docker tag sample_api:latest ${ECR_URL}/sample_api:latest
docker push ${ECR_URL}/sample_api:latest
Now to step 2. Set up Terraform scripts

We will use terraform to build the AWS infrastructure that will run our API in the cloud.

Create a folder infra in the project root. Create a file named data.tf in it with the following content:-

data "aws_vpc" "this" {
  default = true
}

data "aws_subnet_ids" "this" {
  vpc_id = data.aws_vpc.this.id
}

data "aws_region" "current" {}

data "aws_caller_identity" "current" {}
This defines our VPC and subnets. We are using the default ones here. Every AWS account comes with default VPC and subnet.

Next let’s create the security group that will control access to our Application Load balancer (ALB). sg.tf

resource "aws_security_group" "this" {
  vpc_id = data.aws_vpc.this.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    self      = true
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
Now let’s create a Application load balancer that will use our securoty group to listen to traffic from the internet and route that to the appropiate listener (service). alb.tf

resource "aws_lb" "this" {
  name            = "my-alb"
  security_groups = [aws_security_group.this.id]
  subnets         = data.aws_subnet_ids.this.ids
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.this.arn

  port     = 80
  protocol = "HTTP"

  default_action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "This ALB is live now."
      status_code  = "200"
    }
  }
}

resource "aws_lb_target_group" "this" {
  name_prefix = "my-alb"
  port        = 80
  protocol    = "HTTP"
  target_type = "ip"
  vpc_id      = data.aws_vpc.this.id
}

resource "aws_lb_listener_rule" "this" {
  listener_arn = aws_lb_listener.http.arn

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.this.arn
  }

  condition {
    path_pattern {
      values = ["/WeatherForecast*"]
    }
  }
}
Notice we have two listeners, a fixed-response one that we will use to test our load balancer and a target group type that encompasses our API service. We will route any request that starts with /WeatherForecast to it.

The next thing is to create our ECS cluster. This is a logical grouping/container of our ECS service/tasks. Every ECS service belongs to a cluster.

cluster.tf

resource "aws_ecs_cluster" "this" {
  name = "my-cluster"
}
Now, let’s create our ECS task definition. This is the blueprint that will be used to launch a container of our image on demand.

task-def.tf

resource "aws_ecs_task_definition" "this" {
  family                   = "sample_api"
  memory                   = 512
  cpu                      = 256
  requires_compatibilities = ["FARGATE"]
  task_role_arn            = aws_iam_role.task.arn
  execution_role_arn       = aws_iam_role.task.arn
  network_mode             = "awsvpc"
  container_definitions = jsonencode(
    [{
      "name" : "my-api"
      "image" : "${data.aws_caller_identity.current.account_id}.dkr.ecr.${data.aws_region.current.name}.amazonaws.com/sample_api",
      "portMappings" : [
        { containerPort = 80 }
      ],
    }]
  )
}
Notice that we reference a role in the task-def.tf above (line 6 and line 7). This role contains permission that our containers will have. We give our container the role of a task with a policy that allows it to create a log group in CloudWatch

task_role.tf

locals {
  taskRole_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
  ]
}

resource "aws_iam_role" "task" {
  path = "/"

  assume_role_policy = jsonencode(
    {
      "Version" : "2012-10-17",
      "Statement" : [
        {
          "Action" : "sts:AssumeRole",
          "Principal" : {
            "Service" : "ecs-tasks.amazonaws.com"
          },
          "Effect" : "Allow",
          "Sid" : "AssumeRoleECS"
        }
      ]
    }
  )
}

resource "aws_iam_role_policy_attachment" "task" {
  count      = length(local.taskRole_arns)
  role       = aws_iam_role.task.name
  policy_arn = element(local.taskRole_arns, count.index)
}

resource "aws_iam_role_policy" "task" {
  role = aws_iam_role.task.id
  policy = jsonencode(
    {
      "Version" : "2012-10-17",
      "Statement" : [
        {
          "Effect" : "Allow",
          "Action" : [
            "logs:CreateLogGroup"
          ],
          "Resource" : "*"
        }
      ]
  })
}
The last file we will create is output.tf. This is where we will print out the DNS name of our load balancer. We will use that to access our API in AWS.

output.tf

output "url" {
  value = aws_lb.this.dns_name
}
That’s it. Now to deploy our infrastructure

TERRAFORM DEPLOY
run the following commands to deploy our infra.

At the project root

terraform -chdir=./infra init
This command initialises terraform state using the files in the infra folder. You should see an output like this.


PLAN OUR INFRASTRUCTURE IN AWS
Terraform allow us to preview the infrastructure plan that will be deployed to AWS. Run the following command to see the plan and verify our terraform files doesn’t have any syntax error.

terraform -chdir=./infra plan

APPLY TERRAFORM
Time to set up our infrastructure in AWS. Run the following command to apply our changes. This process will take some time so feel free to grab a cuppa.

terraform -chdir=./infra apply -auto-approve
TEST API
The result of applying the changes should output a url. Open your browser and go to {url}/WeatherForecast.

You should see the weather forecast data as we saw them previously.

Login to your AWS console and have a look at what we’ve created. Familiarise yourself with the components (ECS Cluster, ALB, Cloudwatch). Once you are satisfied, clear the resources

CLEAN UP OUR RESOURCES
Finally, make sure you remove all resources. You dont want to accumulate costs on your AWs because of a tutorial.

Run the following commands:

terraform -chdir=./infra destroy -auto-approve
aws ecr delete-repository --repository-name sample_api
CONCLUSION
In this article, we learn about how to dockerise a .net 6 API and publish the image to AWS ECR. We use terraform script to setup entire ECS Service in Fargate launch type and publish our API for anyone to consume.

If you have any issues with any of the steps or instructions in the tutorial, feel free to comment here or ping me on twitter.

I’ve also publish the source code to github so you can pull and cross reference.

