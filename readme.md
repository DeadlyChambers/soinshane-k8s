# Kubernetes

## Important Terms
**Cluster** : The nodes, and control plane all live within the cluster
**Control Plane** : Is the controller of the cluster. The control plane will deal with scheduling applications, maintaining applications' desired state, scaling applications, and rolling out new updates
**Node** : A node is a VM or physical machine in a cluster that serves as a worker machine. The node should have tools for interacting with a container Docker, or containerd. The nodes communicate with the Control Plane via [Kubernetes Api ](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

<img src="https://user-images.githubusercontent.com/6717936/159148752-5435b3e9-bf91-49aa-8c9f-55b964165078.svg" width="100%" style="background-color:white;"></img> 

## Minikube
Can be used locally to run a kubernetes cluster locally with a vm, like Kde or VirtualBox [Setting up Minikube](https://computingforgeeks.com/how-to-install-minikube-on-ubuntu-debian-linux/)
### Commands
```
minikube start
kubectl cluster-info
kubectl config view
kubectl get nodes
minikube ssh
minikube stop
minikube delete
minikube addons list
minikube dashboard
```



### Deploying
You will need to create a Deployment configuration. The Control Plane will use this to deploy containers. It will also monitor the instances and if a node goes down. The Deployment Controller on the CP will copy over image running on another node to ensure the application is running.

```
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
kubectl get deployments
# Need to setup proxy to be able to hit the  #Pods via api
kubectl proxy
curl http://localhost:8001/version

# Set a env to access the pod via api 
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')

curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME
```

**Pods** : A pod is created to contain app/applications for Docker Apps. It includes other apps, volumes, Networking (ip  cluster address), and info how to run each container.
They are the atomic unit of Kubernetes. When a Deployment is created, k8s creates a pod with containers inside of it. The containers in a pod share an ip and port space. The pod lives on the node that it was created for the entirety if its lifecycle.
*Note*: A pod should only be used for tightly coupled items. If the they need to share disk space.

A node can have multiple pods.

**Kubelet** : This is the process responsible for communicating between the K8s control plane and the node; it manages the pods and containers. 

### Node, Pod, and Kubectl

<img src="https://user-images.githubusercontent.com/6717936/159146295-a0a41082-67f0-43a2-80cd-2eb7b341e84f.svg" width="100%" style="background-color:white;"></img> 

## Troubleshooting w/Kubectl
You can connect to the container in a pod using a few of the commands below
```
kubectl get
kubectl describe
kubectl logs
kubectl exec
```
If you know how these work specifically with a docker container, you should understand how these can work in Cluster

Pods have a lifecycle, and when a node goes, the pod goes. When a node dies, is is possible for a ReplicaSet to get the state back.

Pod has it's own IP address, the port can be shared with containers in the pod.

**Service** : This is an abstraction in k8s that has an access policy for a collection of pods. A set of pods targeted by a service  is usually determined by a LabelSelector.

The service can be exposed by various types

- **ClusterIp(default)** : exposes the service on an internal ip which makes the service reachable only within the cluster
- **NodePort** : Exposed on the same port of each selected node using NAT. Using \<nodeip>:\<nodeport>
- **LoadBalancer** : Creates an External load balancer  in the current cloud and assigns a fixed external ip to the service
- **ExternalName** : Maps service to contents of `externalName` field by returning a `CNAME` records with its value. No proxy and requires `kube-dns` >= v1.7 or `CoreDNS`>=0.0.8

*more info* - [Using Source IP](https://kubernetes.io/docs/tutorials/services/source-ip/) and [Connecting Apps with Services](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service)

You will likely not use a `selector` if you are manaully mapping a service to an endpoint or if you are strictly using `type:ExternalName`

![module_04_labels](https://user-images.githubusercontent.com/6717936/159146966-23626ec2-93c4-41bc-ba81-30263ddab49b.svg)

Notice how the Selector is on the Service, and the Label is on the Pod. Labels can be added/removed/changed at any point

### Setting up Service
You will want to figure out what pods you are using. A default service is `kubernetes`

```
kubectl get pods
kubectl get services
# Initially they are not linked

# using type NodePort, we map to the port the pod is using
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080

# See what ip:port is exposed
kubectl describe services/kubernetes-bootcamp

# Create an env var
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

# Now using the minikube cli you can hit the api
curl $(minikube ip):$NODE_PORT
```

### Using Labels
K8s will create a default label for you, see what describing the deployment
```
kubectl describe deployment

# Which you can get the pod via lable
kubectl get pods -l app=kubernetes-bootcamp

# Or via the Service
kubectl get services -l app=kubernetes-bootcamp

# Get the name of the pod
export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo Name of the Pod: $POD_NAME

# Update the label on the pod
kubectl label pods $POD_NAME version=v1

# You will see multiple labels on the pod
kubectl describe pods $POD_NAME

# Now we are able to query pods on the label
kubectl get pods -l version=v1
```

## Deleting a Service
You can delete a service, but remember the service is what targets the pods. It doesn't control the pods, that is the Deployment. By deleting a service we are no longer able to access the pods which makes the underlying containers unreachable externally.

```
# Delete it
kubectl delete service -l app=kubernetes-bootcamp

# Validate Service
kubectl get services

# Validate the pod is not accessible
curl $(minikube ip):$NODE_PORT

# Run a command on the pod like you  would a container locally
kubectl exec -ti $POD_NAME -- curl localhost:8080
```
To remove the pod, you need to go after the the Deployment, not the service.

## Scaling a Deployment
You can scale by changing the number of replicas in a Deployment. Using a service, you will be able to communicate to the pods that are scaled. Since a pod has a unique IP you would need balance traffic to the pods. When you have multiple instances running you are able to doing a rolling deploy without any service disruptions.

We will want to start from the Deployment

```
kubectl get deployments

# Show the ReplicaSets
kubectl get rs
```

With the `ReplicaSet` you have `Desired` and `Current` with desired being the number of replicas you wish to have. You can simply updated the `Deployment` to scale

```
kubectl scale deployments/kubernetes-bootcamp --replicas=4

# Check the status
kubectl get deployments

# You can see the newly created pods w/IP
kubectl get pods -o wide

# See the events
kubectl describe deployments/kubernetes-bootcamp

# See all the pods target by the service
kubectl get services -l app=kubernetes-bootcamp

# Then curl the <nodeip>:<nodeport> to see various responses
export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
curl $(minikube ip):$NODE_PORT
```

After you are ready to scale down follow the same steps with a simple
```
kubectl scale deployments/kubernetes-bootcamp --replicas=2
```

## Rolling Updates
If a deployment is exposed public, the service will load balance the pods only to available pods. The combination of these two processes will help to orchestrate your rolling deployments.

To get started gather some info about your cluster
```
kubectl get deployments
kubectl get pods
kubectl describe pods
```

Now with the image you gathered from the describe, you can update the image for the deployment by using
```
# Use set image for the deployment to a new image
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

# Look at the change in pods
kubectl get pods
```

### Validate Service
You can see the service is running by getting the node port using a go-template

```
kubectl describe services/kubernetes-bootcamp

export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT

curl $(minikube ip):$NODE_PORT

# You can also verify the rollout of the deployemnt
kubectl rollout status deployments/kubernetes-bootcamp

# Validate the image is updated directly on the pod
kubectl describe pods
```

### Deploy issues
After you attempt a deployment, there may be some issues, if you look at `kubectl get pods` and see `ImagePullback` you will know the deployment failed. You will need to look at the events called out by `kubectl describe pods` In the case you have issues you should ensure your deployments are not left in a bad state. Which is when you should rollback the deployment

```
kubectl rollout undo deployments/kubernetes-bootcamp
```