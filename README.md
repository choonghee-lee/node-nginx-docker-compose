# Node Nginx Docker ECS

## 0. Prerequisite

I did this on a free-tier instance of AWS EC2

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
sudo apt-get install docker-compose
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
