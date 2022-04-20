# DevOps Task

## Create Kubernetes cluster with Minikube

```
minikube start --nodes 2 -p multinode-demo
```
this will create a two node cluster

## Create manifest files
Create a kubernetes deployment, svc, hpa, pdb service account in kubernetes cluster

## Enable kubernetes metrics-Server for HPA

```
minikube addons enable metrics-server -p multinode-demo
```

## Kubernetes CA Certificate

Kubernetes does not have a concept of users, instead it relies on certificates and would only 
trust certificates signed by its own CA. </br>

Cluster's CA cert minikube exist in the local host machine` find it under `~/.minikube/.`

Copy the certificates from the ~/.minikube to my project working dir:

```
 cp ~/.minikube/ca.crt ca.crt
 cp ~/.minikube/ca.key ca.key
```

# Kubernetes RBAC



## User Certificates

First thing we need to do is create a certificate signed by our Kubernetes CA. </br>
We have the CA, let's make a certificate. </br>

Easy way to create a cert is use `openssl` and the easiest way to get `openssl` is to simply run a container:

```
docker run -it -v ${PWD}:/work -w /work -v ${HOME}:/root/ --net host alpine sh

apk add openssl
```

Let's create a certificate for Bob Smith:


```
#start with a private key
openssl genrsa -out bob.key 2048

```

Now we have a key, we need a certificate signing request (CSR). </br>
We also need to specify the groups that Bob belongs to. </br>
Let's pretend Bob is part of the `Shopping` team and will be developing 
applications for the `Shopping` 

```
openssl req -new -key bob.key -out bob.csr -subj "/CN=Bob Smith/O=Shopping"
```

Use the CA to generate our certificate by signing our CSR. </br>
We may set an expiry on our certificate as well

```
openssl x509 -req -in bob.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out bob.crt -days 1
```

## Building a kube config

Let's install `kubectl` in our container to make things easier:

```
apk add curl nano
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

We'll be trying to avoid messing with our current kubernetes config. </br>
So lets tell `kubectl` to look at a new config that does not yet exists 

```
export KUBECONFIG=~/.kube/new-config
```

Create a cluster entry which points to the cluster and contains the details of the CA certificate:

```
kubectl config set-cluster dev-cluster --server=https://127.0.0.1:52807 \
--certificate-authority=ca.crt \
--embed-certs=true

#see changes 
nano ~/.kube/new-config
```


kubectl config set-credentials bob --client-certificate=bob.crt  --client-key=bob.key --embed-certs=true

kubectl config set-context dev --cluster=dev-cluster --namespace=shopping --user=bob 

kubectl config use-context dev

kubectl get pods
Error from server (Forbidden): pods is forbidden: User "Bob Smith" cannot list resource "pods" in API group "" in the namespace "shopping"


## Give Bob Smith Access

```
cd kubernetes/rbac
kubectl create ns shopping

kubectl -n shopping apply -f .\role.yaml
kubectl -n shopping apply -f .\rolebinding.yaml
```

## Test Access as Bob

kubectl get pods
No resources found in shopping namespace.

# Kubernetes Service Accounts

So we've covered users, but what about applications or services running in our cluster ? </br>
Most business apps will not need to connect to the kubernetes API unless you are building something that integrates with your cluster, like a CI/CD tool, an autoscaler or a custom webhook. </br>

Generally applications will use a service account to connect. </br>
You can read more about [Kubernetes Service Accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/).

Let's deploy a service account 

```
kubectl -n shopping apply -f serviceaccount.yaml

```
Now we can deploy a pod that uses the service account 
```
kubectl -n shopping apply -f pod.yaml
```
Now we can test the access from within that pod by trying to list pods:

```
kubectl -n shopping exec -it shopping-api -- bash

# Point to the internal API server hostname
APISERVER=https://kubernetes.default.svc

# Path to ServiceAccount token
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount

# Read this Pod's namespace
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)

# Read the ServiceAccount bearer token
TOKEN=$(cat ${SERVICEACCOUNT}/token)

# Reference the internal certificate authority (CA)
CACERT=${SERVICEACCOUNT}/ca.crt

# List pods through the API
curl --cacert ${CACERT} --header "Authorization: Bearer $TOKEN" -s ${APISERVER}/api/v1/namespaces/shopping/pods/ 

# we should see an error not having access
```

Now we can allow this pod to list pods in the shopping namespace
```
kubectl -n shopping apply -f serviceaccount-role.yaml
kubectl -n shopping apply -f serviceaccount-rolebinding.yaml
```

If we try run `curl` command again we can see now we are able to get a json 
response with pod information