# Swarm

```bash
docker swarm init
docker node ls
# we can have only one leader and mutiple managers
# docker service is like replaces docker run but for swarm
docker service create alpine ping 8.8.8.8
docker service ls
docker container ls # swarm adds info to the container
# in this case, 3 nodes are providing the alpine service
# so if any of the containers (nodes) fails it will send the task to another node
docker service update tj7 --replicas 3
docker service ps tj7
docker container rm -f fc1 # if we shutdown a container, the service will create another one
docker service rm tj7
```

## Multipass

```bash
# windows
multipass launch --name node1 focal
curl https://get.docker.com/ > install_docker.sh && chmod +x install_docker.sh && ./install_docker.sh
sudo addgroup --system docker
sudo adduser $USER docker
newgrp docker

# node1
docker swarm init
# node2 and node3
docker swarm join --token SWMTKN-1-4h7ct15o6yad1j530xjg4kfa2d8gww4xcnwq85subnu613tegj-233qc8ogd324opyplgehnzv4q 172.29.51.46:2377

# node1 is the only that has this access
docker node ls
docker node update --role manager node2
# scale the service to multiple nodes
docker service create --replicas 3 alpine ping 8.8.8.8
docker service ls
docker service ps l07
```

## Balance Loads

```bash
docker service create --name search --replicas 3 -p 9200:9200 elasticsearch:2 # node1
# elasitcsearch is an image that returns the container working on it
docker service ps search
curl 172.29.51.46:9200 # use the public ip of the machine
# I don't know why it's not working with localhost:9200
# this ip is return when docker swarm init
# run this command multiple times to see all nodes
```

## Stack

```bash
docker stack deploy -c file.yml appName
docker stack services appName
docker stack ps appName
```

## Secrets

### Services

```bash
docker secret create psql_user psql_user.txt
echo "myPassword" | docker secret create psql_pass -
docker secret ls
# containers are the only that can access the secrets
# map secrets to the container
docker service create --name psql --secret psql_user --secret psql_pass -e POSTGRES_PASSWORD_FILE=/run/secrets/psql_pass -e POSTGRES_USER_FILE=/run/secrets/psql_user postgres
docker exec -it psql.1.hmetbczuw2yg33gv42ilygfn5 bash
cat /run/secrets/psql_user
# we can update to remove the secrets
# but we reploy the container
# secrets are a part of the immutable design of the containers
docker service update --secret-rm
```

### Stacks

```yml
services:
  psql:
    image: postgres
    secrets:
      - psql_user
      - psql_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/psql_password
      POSTGRES_USER_FILE: /run/secrets/psql_user

secrets:
  psql_user:
    file: ./psql_user.txt
  psql_password:
    file: ./psql_password.txt
```

```bash
docker stack deploy -c docker-compose.yml mydb
docker stack rm mydb # removes also the secrets
```

## Updates

```bash
# everytime we need to update the image
docker service update --image myapp:1.2.1 <servicename (web)>
# add or remove environment variables
docker service update --env-add NODE_ENV=production --publish-rm 8080
# change number of replicas
docker service scale web=8 api=6
# change ports
docker service update --publish-rm 8080 --publish-add 9090:80 <servicename (web)>
```

## Private Docker Registry

```bash
# other nodes can access our registry since it's a service
docker service create --name registry --publish 5000:5000 registry
# http://172.28.196.187:5000/v2/_catalog
docker pull hello-world
docker tag hello-world 127.0.0.1:5000/hello-world
docker image ls
docker push 127.0.0.1:5000/hello-world
# http://172.28.196.187:5000/v2/_catalog
docker image remove 127.0.0.1:5000/hello-world
docker image pull 127.0.0.1:5000/hello-world
```
