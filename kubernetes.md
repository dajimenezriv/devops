# Kubernetes

## Imperative

### Basics

Kubernetes only creates pods, every node has the kubelet, who tells the node to run a container.
With the pod, we tell the info of how to run a container.

```bash
kubectl run my-ngnix --image nginx
kubectl get pods
kubectl get all

# deploy = deployment = deployments
kubectl create deployment my-apache --image httpd
# we create a deployment, then it creates a replicaset and finally a pod
# the replicaset is for rolling updates (one for each version)
# this is to have the old version running while the new is replacing
# the pod is pod/deployment-replicaset-pod
kubectl scale deploy/my-apache --replicas 2
# changes the deploy/my-apache record
# then the replicaset updates its replicas
kubectl logs deploy/my-apache --follow --tail 1
# to show all logs corresponding to that namespace (-l = label)
kubectl get namespaces
kubectl get all --all-namespaces
kubectl logs -l key=value
kubectl describe pod/my-apache-6f45bc5bd9-rm5dd
# if we remove a pod, the deployment will ensure to deploy a new one
kubectl get pods -w
pod/my-apache-6f45bc5bd9-rm5dd
kubectl delete deploy my-apache
```

### Expose IP

#### Inside Cluster (ClusterIp)

Hierarchy: ClusterIp -> NodePort -> LoadBalancer

```bash
kubectl create deploy httpenv --image=bretfisher/httpenv
kubectl scale deploy/httpenv --replicas=5
# we are exposing a port that it is only accessible from inside the cluster (only for pods)
kubectl expose deploy/httpenv --port 8888
kubectl get service
# -it says to perform a command when the pod is created
# after -- we tell the commands that we are going to run
kubectl run tmp-shell --rm -it --image bretfisher/netshoot -- bash
# bash-5.1# curl httpenv:8888
```

#### Outside Cluster (NodePort and Load Balancer)

Check also Ingress.

```bash
kubectl create deploy httpenv --image=bretfisher/httpenv
kubectl scale deploy/httpenv --replicas=5
# NodePort also creates a ClusterIp
kubectl expose deploy/httpenv --port 8888 --name=httpenv-np --type NodePort
kubectl get service
curl localhost:31083

kubectl expose deploy/httpenv --port 8888 --name=httpenv-lb --type LoadBalancer
curl localhost:8888
```

### Generators

```bash
# create the yaml file
kubectl create deploy sample --image nginx --dry-run -o yaml
# we don't need to nested objects and restartPolicy: Never
kubectl create job sample --image nginx --dry-run -o yaml
# to see the yaml of a port we need to create the deploy
kubectl create deploy my-nginx --image=nginx
kubectl expose deploy/my-nginx --port 80 --dry-run -o yaml
```

## Declarative

See pod.yml and deployment.yml. To create multiple types: split with "---".

```bash
kubectl api-resources
kubectl api-versions
kubectl explain deployments --recursive # all possible keys in yaml
kubectl explain deployments.spec # all possible keys in spec
kubectl explain deployments.spec.template.spec # to access just one key
```

### Dry-run and Selectors (Labels and Annotations)

```bash
kubectl apply -f app.yml --dry-run # shows the objects that are going to be changed (client side)
kubectl apply -f app.yml --server-dry-run # same as above, but takes into account the current server
kubectl diff -f app.yml # difference between previous yml applied and new
```

## More

### Operators

Extends kubernetes capabilities to manage more complex, stateful workloads.

### Helm

To help deploy and create objects in kubernetes.

### Kubernetes Dashboard

A GUI to use kubernetes. Be careful and set a random IP and a proxy in front of the dashboard.
