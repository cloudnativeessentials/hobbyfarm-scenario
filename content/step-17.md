+++
title = "Kubernetes Basics"
weight = 17
+++

# Kubernetes Basics

With our Kubernetes cluster ready, let's deploy basic Kubernetes resources

## Namespace
A namespace isolates a group of resources within a Kubernetes cluster.
Create a new namespace `my-namespace`
```ctr:kubernetes
kubectl create namespace my-namespace
```

Expected output:
```
namespace/my-namespace created
```

## Pod
A Pod is the smallest deployable unit in Kubernetes. A Pod can be one or more containers that work together. 

Let's use an imperative command to deploy an nginx webserver:
```ctr:kubernetes
kubectl run nginx --image=nginx:1.25.3 --port=80 -n my-namespace
```

Expected output:
```shell
pod/nginx created
```

Find the Pod's virtual IP and test access to the nginx container:
```ctr:kubernetes
kubectl get pods -o wide -n my-namespace
```

Expected output:
```shell
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE              NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          83s   <VIP>       <hostname>   <none>           <none>
```

Curl the nginx container on the Virtual IP, the command below is not click-to-run since you have to enter the Pod's VIP
```
curl -sk http://<VIP> | head -n 4
```

Expected output:
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Delete the Pod as we will deploy multiple replicas of the same Pod in the next step.

```ctr:kubernetes
kubectl delete pod nginx -n my-namespace
```

Expected output:
```shell
pod "nginx" deleted
```

In the next several steps we'll look at how to create resources to scale an application and expose it outside the Kubernetes cluster.

## Deployment
A Deployment is a Kubernetes resource that provides declartive updates at a controlled rate for Pods.
This time you will use a Kubernetes manifest to deploy a Deployment.

Create a Kubernetes manifest for a Deployment in the `my-namespace` namespace for two replicas of Pod that runs the nginx webserver.
This time give the `app=nginx` labels to the Deployment and Pods of the Deployment so we can group and identify the Pods in a later step.

```ctr:kubernetes
mkdir -p ~/manifests
cat <<EOF > ~/manifests/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: my-namespace
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
EOF
```

Use `kubectl apply` to apply the configuration from the deployment.yaml manifest
```ctr:kubernetes
kubectl apply -f ~/manifests/deployment.yaml
```

Expected output:
```shell
deployment.apps/nginx created
```

Check the status of the Deployment
```ctr:kubernetes
kubectl get deployment -n my-namespace
```

Expected output:
```shell
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           30s
```

Check the Pods of the deployment
```ctr:kubernetes
kubectl get pods -n my-namespace -o wide
```

With 2 replicas of the nginx webserver running, let's expose the Pods of the Deployment to outside of the cluster

## Services

A Service abstracts a group of Pods over a network using a label selector identify the Pods to expose.
There are multiple types of Kubernetes Services, we will use the NodePort service to expose the Service on each Node's IP of a cluster.
By default, a NodePort exposes the Service in the port range of 30000 to 32767

Expose the Pods of the nginx Deployment to be reachable with your browser.

Create a manifest for a Service that uses the Pod's label in the `my-namespace` namespace.
```ctr:kubernetes
cat <<EOF > ~/manifests/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: my-namespace
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      nodePort: 30000
EOF
```

Apply the service.yaml manifest to create the Service
```ctr:kubernetes
kubectl apply -f ~/manifests/service.yaml
```

Expected output:
```shell
service/nginx-service created
```

Check if the Service was created
```ctr:kubernetes
kubectl get service -n my-namespace
```

Use your browser to test access to the nginx webserver
<a href="http://kubernetes.${vminfo:kubernetes:public_ip}.sslip.io:30000" target="_blank">http://kubernetes.${vminfo:kubernetes:public_ip}.sslip.io:30000</a>

You should see Welcome to nginx~

At the end of this step, you've successfully deployed 2 replicas of an application container with a Pod and Deployment and exposed the application with a Service type NodePort.
