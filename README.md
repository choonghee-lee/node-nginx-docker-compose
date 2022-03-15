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
