# Docker Assignments

## Multiple Containers

We want nginx in port 80, httpd in port 8080 and mysql in port 3306.

```bash
# -p redirect ports => host port/container port
# -d/--detach, we run it in background
# nginx:alpine contains ping
docker run -d -p 80:80 --name nginx nginx:alpine # for example, nginx runs in port 80
docker run -d -p 8080:80 --name httpd httpd  # for example, nginx runs in port 80
# we send an environment variable as input with -e
docker run -d -p 3306:3306 --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=yes mysql # for example, nginx runs in port 3306

docker container logs mysql | grep "GENERATED ROOT PASSWORD"
```

http://localhost:80/<br>
http://localhost:8080/

## Jump Between Linux Distributions to Test

Test curl on Ubuntu:14.04 and Centos:7.

```bash
# with --rm we are performing a cleanup after stopping the container.

# centos
docker container run --rm -it centos:7 bash
yum update curl
curl --version

# ubuntu
docker container run --rm -it ubuntu:14.04 bash
apt-get install curl
curl --version
```

## Round Robin DNS

Multiple containers responding to the same DNS name.

```bash
docker container run -d --name elastic1 --net my_network --net-alias search elasticsearch:2 # ports open: 9200 and 9300
docker container run -d --name elastic2 --net my_network --net-alias search elasticsearch:2
docker container run --rm -it --net my_network alpine nslookup search
# if we run this command multiple times we are going to get the response from both previous servers
docker container run --rm --net my_network centos curl -s search:9200
```

## Node Dockerfile

```dockerfile
FROM node:6-alpine
RUN apk add --no-cache tini && mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . /usr/src/app
RUN npm install && npm cache clean --force
EXPOSE 3000
CMD [ "/sbin/tini", "--", "node", "./bin/www" ]
```

```bash
docker build .
docker tag b35 49748609/node_server
docker push 49748609/node_server
docker container run --rm -p 80:3000 49748609/node_server
```

http://localhost:80/

## Update Database Version (Volumes)

We can only update the minor version automatically.
If it's a major version, we need to perform a manual update.

```bash
# the postgres volume is in /var/lib/postgresql/data
# we are renaming the host volume to psql-data
docker container run -d --name postgres1 -v psql-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres:9.6.1
docker container logs postgres1 # see if we have any error
docker container inspect postgres1 # inside "Mounts" section we can see our volume named
docker container stop postgres1 # otherwise the volume will have multiple connections => ERROR
docker container run -d --name postgres2 -v psql-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres:9.6.2
docker container logs postgres2 # see if we have any error
# if its says that ready to accept connections, everything worked well
```

## Bindmount (Volumes)

```bash
# we are going to map our repository with the volume that jekyll system needs
cd udemy-docker-mastery/bindmount-sample-1
docker run -p 80:4000 -v $(pwd):/site bretfisher/jekyll-serve
# now modify the content inside the _posts folder and refresh
```

http://localhost:80/

## Multi-Container Project (Docker Compose)

```bash
cd udemy-docker-mastery/compose-assignment-1
docker compose up
# -v removes the volumes
docker compose down -v
```

http://localhost:8080

In the configuration of postgres, we have to change localhost to postgres in Advanced Options.

## Compose and Dockerfile (Docker Compose)

Building an image from a Dockerfile.

```bash
cd udemy-docker-mastery/compose-assignment-2
docker compose up
# -v removes the volumes
# if we don't remove the volume and we launch it again we will still have bootstrap as theme installed and by default
docker compose down -v
```

http://localhost:8080/admin/appearance<br>
Install the bootstrap theme and set it by default. Now the site has changed (< Back to site).

## Swarm Vote App

```bash
# WE DON'T NEED 6 NODES
multipass launch --name node1 focal
# install docker
# node 1
docker swarm init
# other nodes
docker swarm join ...
# node 1
docker network create --driver overlay frontend
docker network create --driver overlay backend
docker service create --network frontend                   --replicas 2 --name vote   -p 80:80 bretfisher/examplevotingapp_vote
docker service create --network frontend                   --replicas 1 --name redis  redis:3.2
docker service create --network frontend --network backend --replicas 1 --name worker bretfisher/examplevotingapp_worker
docker service create --network backend                    --replicas 1 --name db     -e POSTGRES_HOST_AUTH_METHOD=trust --mount type=volume,source=db-data,target=/var/lib/postgresql/data postgres:9.4
docker service create --network backend                    --replicas 1 --name result -p 5001:80 bretfisher/examplevotingapp_result
```

http://172.29.51.46/
http://172.29.51.46:5001/

## Stack and Secrets

```bash
cd swarm-secrets-assignment-1
echo "password" | docker secret create psql-pw -
docker stack deploy -c docker-compose.yml basic
```
