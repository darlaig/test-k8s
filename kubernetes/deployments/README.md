# DevOps Task

## Create Kubernetes cluster with Minikube

```
minikube start --nodes 2 -p multinode-demo
```
this will create a two node cluster

Created Kubernetes Manifest test-k8s/kubernetes

```
## deploy with kubectl apply -f deployment/
minikube start --nodes 2 -p multinode-demo
```

## Enable kubernetes metrics-Server for HPA

```
minikube addons enable metrics-server -p multinode-demo
```
## Create manifest files
Create a kubernetes deployment, svc, hpa, pdb service account in kubernetes cluster

## Creating RBAC for kubernetes resources access auth.

Kubernetes does not have a concept of users, instead it relies on certificates and would only 
trust certificates signed by its own CA. </br>

Cluster's CA cert in minikube exist in the local host machine` find it under `~/.minikube/.`

I copied the certificates from the ~/.minikube to my project working dir:

```
 cp ~/.minikube/ca.crt ca.crt
 cp ~/.minikube/ca.key ca.key
```

The goal is to allow developers(users[John]) access to resources on namespace level

## Create User Certificates

Create a certificate signed by the K8's CA using `openssl` tool . </br>

```
#start with a private key
openssl genrsa -out john.key 2048

```

Ceate a certificate signing request (CSR). </br>
Using John as part of the `Developers` team </br>

Used the CA cert and key to generate the certificate by signing the CSR with a possible expiry. </br>

```
openssl x509 -req -in john.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out john.crt -days 1
```

## Setup a new kube config
Test this out using a new kube config file in a container.</br>

```
docker run -it -v ${PWD}:/work -w /work -v ${HOME}:/root/ --net host alpine sh
```

Installed `kubectl` in the container

```
apk add curl vim
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

Point `kubectl` to look at the new-config file which i created to test the RBAC.



```
export KUBECONFIG=~/.kube/new-config
```

Create a cluster entry which points to the cluster and contains the details of the CA certificate:

```
kubectl config set-cluster multinode-demo --server=https://192.168.59.105:8443 \
--certificate-authority=ca.crt \
--embed-certs=true

kubectl config set-credentials john --client-certificate=john.crt --client-key=john.key --embed-certs=true

kubectl config set-context dev --cluster=multinode-demo --namespace=developers --user=john 

#This write the confi to the new file
cat ~/.kube/new-config
```

Change config context to dev
kubectl config use-context dev

## Testing developer John's Access
```
kubectl get node
Error from server (Forbidden): nodes is forbidden: User "John Doe" cannot list resource "nodes" in API group "" at the cluster scope

#testing john's access to pods in the nginx namespace
k -n nginx get pod
Error from server (Forbidden): pods is forbidden: User "John Doe" cannot list resource "pods" in API group "" in the namespace "nginx"
```

## Testing developer John's Access to pods in developer namesapces with no resources in the namespace after applying RBAC definition

```
k get pod
No resources found in developers namespace.
