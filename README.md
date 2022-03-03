## Node Nginx Docker Compose

Theses are references to up docker-compose for Node.js application and Nginx web server.

### Install Docker

1. [Install Docker](https://docs.docker.com/engine/install/ubuntu/)
2. [Install Docker Compose](https://docs.docker.com/compose/install/#install-compose)

### Modify files

- `Dockerfile` for each
- `nginx.conf`
- `docker-compose.yml`

### Create a Docker network

```bash
docker network create \[YOUR_NETWORK_NAME\]
```

### Handle docker containers

```bash
docker-compose up -d
docker-compose down

docker ps
docker ps -a
docker rm \[COTAINER_ID\]

docker image list
docker image rm \[IMAGE_ID\]
```
