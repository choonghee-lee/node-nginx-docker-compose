# Node Nginx Docker ECS

## 0. Prerequisite

I did this on a free-tier instance of AWS EC2 (ubuntu 20)

### Install Docker

```
curl -fsSL https://get.docker.com/ | sudo sh
sudo usermod -aG docker $USER
docker
```

## 1. Node + Nginx without Docker-Compose

### Docker: Node

#### Dockerfile

```Dockerfile
FROM node:16.14.0

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

#### Build & Run

Before you build, please check app configuration.

You need `--name` option to link to a nginx container later.

```
docker build -t [IMAGE_NAME] .
docker image list
docker run -p [CONTAINER_PORT]:[HOST_PORT] -d --name [COTAINER_NAME] [IMAGE_NAME]
docker ps
```

### Docker: Nginx

#### nginx.conf

Check the port number. It's the number in Dockerfile of Node project.

```
events {
  # Do not remove this block...
}

http {
  server {
    location / {
      proxy_set_header Host              $host;
      proxy_set_header X-Real-IP         $remote_addr;
      proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_pass                         http://[NODE_COTAINER_NAME_HERE]:3000;
    }
  }
}
```

#### Dockerfile

```
FROM nginx:1.20.2

RUN rm /etc/nginx/nginx.conf

COPY nginx.conf /etc/nginx/nginx.conf
```

#### Build & Run

```
docker build -t [IMAGE_NAME] .
docker run -p [CONTAINER_PORT]:[HOST_PORT] -d --link [NODE_CONTAINER_NAME] --name [COTAINER_NAME] [IMAGE_NAME]
```

## 2. Node + Nginx with Docker-Compose

### Install Docker-Compose

```
sudo apt install docker-compose
```

### Create a docker network

```
docker network create [NETWORK_NAME]
```

### docker-compose.yml

```yaml
version: "3"
services:
  mynode:
    build:
      context: [NODE_APP_PATH]
    container_name: mynode
    hostname: mynode
    ports:
      - "3000:3000"
    networks:
      - [NETWORK_NAME]

  mynginx:
    build:
      context: [NGINX_PATH]
    container_name: mynginx
    hostname: mynginx
    ports:
      - "80:80"
    depends_on:
      - mynode
    networks:
      - [NETWORK_NAME]

networks:
  [NETWORK_NAME]:
    external: true
```

### Run&Stop Docker-Compose

```
docker-compose up -d --build
docker-compose down
```

## 3. Upload Image to AWS ECR

ECR (Elastic Container Repository) is the repository service under ECS (Elastic Container Service).

### Install awscli

```
sudo apt install awscli
```

### Login to ECR

Prepare your AWS user credential

- access key
- access secret key

```
aws configure

aws ecr get-login --no-include-email --region [YOUR_AWS_REGION]

#################################################
# COPY AND PASTE the result of previous command #
#################################################
docker login -u AWS -p abcdefg...

```

### Create Repository

```
aws ecr create-repository --repository-name [REPO_NAME] --region [YOUR_AWS_REGIN]
```

### Tag ECR URI to Docker Images

Tag ECS URI to an Docker image. Notice that the image IDs are same.

```
docker tag [IMAGE_NAME_OR_ID] [ECR_URI]
```

### Push Docker Image to ECR

```
docker push [ECR_URI]
```

## 4. Run Node with ECS

### 1. Create Cluster
```
aws ecs create-cluster --cluster-name [CLUSTER_NAME]
```

### 2. Create ECS Task Execution Role

#### ecs-tasks-trust-policy.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Create Role

The file scheme must be included.
```
aws iam create-role --role-name ecsTaskExecutionRole \
--assume-role-policy-document file://[PATH]/ecs-tasks-trust-policy.json
```

#### Attach Role to AWS Account

```
aws iam attach-role-policy \
      --role-name ecsTaskExecutionRole \
      --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

### 3. Generate Task Definition

#### Generate Template File

The file name doesn't need to be `task.json`. you can choose the name you want.
```
aws ecs register-task-definition --generate-cli-skeleton > task.json
```

#### Configure Task Definition

Make a json file to define parameters for a task definition. For more parameter information, check out [this AWS document](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html).

```json
{
    "family": "nodeapp",
    "executionRoleArn": "ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "containerDefinitions": [
        {
            "name": "[CONTAINER_NAME]",
            "image": "[ECR_IMAGE_URI]",
            "portMappings": [
                {
                    "containerPort": [CONTAINER_PORT],
                    "hostPort": [HOST_PORT], 
                    "protocol": "tcp"
                }
            ],
            "essential": true
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "256",
    "memory": "512",
}
```

#### Register Task Definition

```
aws ecs register-task-definition --cli-input-json file://$HOME/[PATH]/task.json
```

### 4. Create ECS Cluster & Service

#### Create Cluster

```
aws ecs create-cluster --cluster-name [CLUSTER_NAME]
```

#### Create Service

You can find the public ip, in task of the service on AWS ECS console.

```
aws ecs create-service --cluster [CLUSTER_NAME] --service-name [SERVICE_NAME] --task-definition [TASK_DEFINITION_NAME:REVISION] --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[[SUBNET_ID], [SUBNET_ID], ...],securityGroups=[[SECURITY_GROUP_ID], [SECURITY_GROUP_ID], ...],assignPublicIp=ENABLED}"
```

#### Delete Service & Cluster

```
aws ecs delete-service --cluster [CLUSTER_NAME] --service [SERVICE_NAME] --force
aws ecs delete-cluster --cluster [CLUSTER_NAME]
```
