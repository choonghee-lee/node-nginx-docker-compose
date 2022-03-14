## Node Nginx Docker ECS

### Install Docker

```
curl -fsSL https://get.docker.com/ | sudo sh
sudo usermod -aG docker $USER
```

### Docker: Node Application

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

```
docker build -t [IMAGE_NAME] .
docker image list
docker run -p [CONTAINER_PORT]:[HOST_PORT] [IMAGE_NAME]
```
