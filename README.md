# Deploy Kubernetes Dashboard locally on AWS EC2

[![N|Solid](https://cldup.com/dTxpPi9lDf.thumb.png)](https://nodesource.com/products/nsolid)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Dillinger is a cloud-enabled, mobile-ready, offline-storage, AngularJS powered HTML5 Markdown editor.

  - Type some Markdown on the left
  - See HTML in the right
  - Magic

# Prerequisition
- AWS EC2 Ubuntu 16.04
- Security group allow SSH in 
- All configuration/deployment should be done under root account

# Install Docker
- Install docker
```sh
# apt-get update -y &&  apt-get install -y docker.io
```
- Verify docker has been installed  
```sh
# which docker
/usr/bin/docker
```

# Install Minikube
- Install Minikube, give correct permission and add into PATH
```sh
# curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```
- Check Minikube version
```sh
# root@ip-172-31-3-123:~# minikube version
minikube version: v0.32.0
```

# Install Kubectl
- Install Kubectl
```sh
# curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```
- Add Kubectl auto-completion
```sh
# source <(kubectl completion bash)
```

# Launch master cluster using Minikube
The first time launch will take quite long time. You should be able to see below output. 
```sh
# minikube start --vm-driver=none
```
Output:
```sh
Starting local Kubernetes v1.12.4 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Downloading kubeadm v1.12.4
Downloading kubelet v1.12.4
Finished Downloading kubeadm v1.12.4
Finished Downloading kubelet v1.12.4
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Stopping extra container runtimes...
Starting cluster components...
Verifying kubelet health ...
Verifying apiserver health ...Kubectl is now configured to use the cluster.
===================
WARNING: IT IS RECOMMENDED NOT TO RUN THE NONE DRIVER ON PERSONAL WORKSTATIONS
        The 'none' driver will run an insecure kubernetes apiserver as root that may leave the host vulnerable to CSRF attacks

When using the none driver, the kubectl config and credentials generated will be root owned and will appear in the root home directory.
You will need to move the files to the appropriate location and then set the correct permissions.  An example of this is below:

        sudo mv /root/.kube $HOME/.kube # this will write over any previous configuration
        sudo chown -R $USER $HOME/.kube
        sudo chgrp -R $USER $HOME/.kube

        sudo mv /root/.minikube $HOME/.minikube # this will write over any previous configuration
        sudo chown -R $USER $HOME/.minikube
        sudo chgrp -R $USER $HOME/.minikube

This can also be done automatically by setting the env var CHANGE_MINIKUBE_NONE_USER=true
Loading cached images from config file.


Everything looks great. Please enjoy minikube!
```
If you see error, check below point:
- Make sure you are in root account
- Make sure you installed docker via first step
- Make sure you specify "--vm-drive=none"

# Add Kubernetes dashboard plug-in
- Get the YAML file
```sh
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/alternative/kubernetes-dashboard.yaml
```

# Deploy monitoring and performance analysis on cluster
```sh
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

# Create eks-admin account and bind the role
Kubernetes dashboard by default has limited access. In this step, you'll create a eks-admin account and bind the role, so you can connect the dashboard with admin priviledges
- Create a file called "eks-admin-service-account.yml" with below content
```sh
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```
- Deploy service account and bind with cluster role
```sh
# kubectl apply -f eks-admin-service-account.yaml
```
Output:
```sh
serviceaccount "eks-admin" created
clusterrolebinding "eks-admin" created
```

# Access Kubernetes dashboard
There're 2 ways of access Kubernetes dashboard: via kubectl proxy and via nodeport
If we access it via kubectl proxy, we can only access it locally. In EC2, it's hard to find a web browser that can run locally and see the dashboard. Therefore, we choose Nodeport

**WARNING:**
You should NEVER deploy via Nodeport on product environment
```sh
kubectl -n kube-system edit service kubernetes-dashboard
```
You should see "type:ClusterIP" in the end of output. Change it to "type: Nodeport" and save it
```sh
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Check the port which is used by dashboard
```sh
# kubectl -n kube-system get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   NodePort   10.104.145.106   <none>        80:31710/TCP   17m
```
You can then access using 31710 port. 

**NOTE:**
For security considerations, it is HIGHLY RECOMMENDED to restrict this port to your IP only. Otherwise it will be exposed to whole Internet

# Reference
- AWS Official tutorial
https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html
- Kubernetes official repo
https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above/




