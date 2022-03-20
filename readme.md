# Kubernetes

## Important Terms
**Cluster** : The nodes, and control plane all live within the cluster
**Control Plane** : Is the controller of the cluster. The control plane will deal with scheduling applications, maintaining applications' desired state, scaling applications, and rolling out new updates
**Node** : A node is a VM or physical machine in a cluster that serves as a worker machine. The node should have tools for interacting with a container Docker, or containerd. The nodes communicate with the Control Plane via [Kubernetes Api ](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)

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

<img src="https://user-images.githubusercontent.com/6717936/159146295-a0a41082-67f0-43a2-80cd-2eb7b341e84f.svg" width="50%" style="background-color:white;"></img> 
