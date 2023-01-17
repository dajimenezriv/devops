# Docker

Run this in WSL2.

## Basics

```bash
# to delete an image/container when can put the first 3 characters of the hash
docker image rm cf0 # instead of using the whole hash
docker image ls -a
docker container ls -a # -a show also stopped containers
docker container start/stop/rm [name]/[hash]
docker container logs [name]
docker container top [name] # shows the process list of the container
docker container inspect [name] # shows the config of the container
docker container stats [name] # shows the performance of the container
```

```bash
# delete all cache
docker system prune
# delete all containers
docker container prune
docker rm -vf $(docker ps -aq)
# delete all images
docker image prune
docker rmi -f $(docker images -aq)
```

## Runing bash inside the container

If we run an OS like ubuntu. We need to know that it doesn't run a service, so it will be stopped by default.

```bash
# we have two options, create an interactive container
# or run a process of bash inside an existing and running container

# creates and runs the container
# with -it we can run a command after creating the container (bash)
# once we close the bash session the container stops running
docker container run -it ubuntu bash
ps aux
# root@cfeca623d994:/# ps aux
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root         1  0.1  0.0   4624  3864 pts/0    Ss   14:54   0:00 bash
# root        10  0.0  0.0   7056  1644 pts/0    R+   14:54   0:00 ps aux

# we can see that the only process runinng is bash

docker container exec -it ubuntu bash/sh
apt-get update && apt-get install procps # in case we need ps
ps aux
# root@cfeca623d994:/# ps aux
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root         1  0.0  0.0   4620  3508 pts/0    Ss+  15:00   0:00 bash
# root         9  0.5  0.0   4620  3776 pts/1    Ss   15:00   0:00 bash
# root        17  0.0  0.0   7056  1644 pts/1    R+   15:00   0:00 ps aux

# now, the bash process is runing aside, it's not the main process
```

## Networks

```bash
# ip inside virtual private network
docker container inspect nginx
# with format we can use GO templates
# https://docs.docker.com/config/formatting/
docker container inspect --format '{{ .NetworkSettings.IPAddress }}' nginx
# 172.17.0.2
```

```bash
docker network ls
# we have these three interfaces for our network
# bridge is the virtual private network,
# host attaches we using the host network directly
# none doesn't use a network, only the localhost of the container
docker network inspect bridge/name
# we can see the containers attached to that network bridge
# here for example the virtual private network is 172.17.0.0/16
# "Containers": {
#   "2ed9e11f2df46854d3a96d3aa8a5e9f74b741cdc595c90f77a2b7937671d3fa9": {
#       "Name": "nginx",
#       "EndpointID": "5169bd6a1f74e720daed88f51f574d00938316ed2005ba71fd1644aec995c8bd",
#       "MacAddress": "02:42:ac:11:00:02",
#       "IPv4Address": "172.17.0.2/16",
#       "IPv6Address": ""
#   },
#   "cfeca623d994d0b2c12fcdc140b11e60ff105f182261dd0bde7e785fb67079d4": {
#       "Name": "loving_fermat",
#       "EndpointID": "b1620e9cbe0c781db9ccfbaf7e02445a54de4b50187c4923bd2308ba5ac92844",
#       "MacAddress": "02:42:ac:11:00:03",
#       "IPv4Address": "172.17.0.3/16",
#       "IPv6Address": ""
#   }
# }

# connect and disconnect from a virtual private network.
docker network create my_network # my_network hash f87...
docker network connect f87 2ed
docker container inspect 2ed
# now, the container is connected to two virtual private networks
docker network disconnect f87 2ed
```

For example, a backend and a frontend, we can expose only the port of the frontend with -p.<br>
But they can communicate inside the same virtual private network.

### Domain Name System

Instead of using IPs we use containers name.<br>
The default bridge network doesn't have the DNS system by default.

```bash
docker container run --name ngnix1 -d --network my_network nginx:alpine
docker container run --name ngnix2 -d --network my_network nginx:alpine
docker container exec -it ngnix1 ping ngnix2 # we are using the name to perform the ping command

# if we want them responding to the same dns name, we have to add --net-alias (see assignments)
```

## Tags

```bash
docker tag b35 49748609/node_server
docker tag [local_image] [repository/my_account]/[new_image_name]
docker push 49748609/node_server
```

## Persistent Data -> Volumes

### Named Volumes

```bash
# rename volume
# in the container the volume its called /var/lib/postgresql/data
# in our host it will be psql-data
docker container run -d --name postgres1 -v psql-data:/var/lib/postgresql/data postgres:9.6.1
```

### Bindmount

```bash
# it will make the folder to the volume of the container
# every change we make in our host folder will be taken into account in the container
docker run -p 80:4000 -v $(pwd):/site bretfisher/jekyll-serve
```

## Docker Compose

Instead of building our containers from docker-compose.yml we can import like a Dockerfile.<br>
DNS names are derived from the names of the services.

```yml
services:
  proxy:
    build:
      context: .
      # we have an nginx.Dockerfile file
      dockerfile: nginx.Dockerfile
    ports:
      - '80:80'
```

### Secrets

```bash
# docker compose maps the secrets to the container, so it's not secure
# this only works with file secrets
cd secrets-sample-2
docker compose up -d
# psql is the name of container inside the docker-compose.yml
docker compose exec psql cat /run/secrets/psql_user
```

We have a real example of docker compose and stack in swarm-stack-3.

```bash
cd swarm-stack-3
# local development
docker compose up
# testing
docker compose -f docker-compose.yml -f docker-compose.test.yml up -d
# production
docker compose -f docker-compose.yml -f docker-compose.prod.yml config > output.yml
docker stack deploy -c output.yml <stackname>
```

## Healthchecks

```bash
docker container run --name p2 -d -e POSTGRES_PASSWORD=password --health-cmd="pg_isready -U postgres || exit1" postgres
docker container ls
docker container inspect p2 # go to "Health"
docker service create --name p2 -e POSTGRES_PASSWORD=password --health-cmd="pg_isready -U postgres || exit 1" postgres
```

## Private Docker Registry

```bash
docker container run -d -p 5000:5000 --name registry -v $(pwd)/registry-data:/var/lib/registry registry
ls registry-data/
docker pull hello-world
docker tag hello-world 127.0.0.1:5000/hello-world
docker image ls
docker push 127.0.0.1:5000/hello-world
ls registry-data/ # uses blobs like github
# to access our image, we need to use 127.0.0.1:5000/hello-world
docker image remove 127.0.0.1:5000/hello-world
docker image pull 127.0.0.1:5000/hello-world
```

