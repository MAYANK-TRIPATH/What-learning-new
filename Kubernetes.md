# Kubernetes

To run a Kubernetes (k8s) cluster on a cloud provider like AWS, GCP, DigitalOcean, or Vultr, follow these steps. The instructions will cover creating a cluster, configuring your local environment, and deploying an application.

### 1. **AWS (Amazon Web Services)**

#### Create a Kubernetes Cluster with EKS (Elastic Kubernetes Service)
1. **Sign in to AWS Management Console**:
- Navigate to the EKS service.
- Click "Create cluster".
- Configure your cluster name, role, networking, and logging.
- Create the cluster and wait for it to become active.

2. **Install AWS CLI and eksctl**:
- AWS CLI: [Installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- eksctl: [Installation guide](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

3. **Create EKS Cluster using eksctl**:
```sh
eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name my-nodes --node-type t2.micro --nodes 3
```

4. **Configure kubectl**:
```sh
aws eks --region us-west-2 update-kubeconfig --name my-cluster
```

### 2. **GCP (Google Cloud Platform)**

#### Create a Kubernetes Cluster with GKE (Google Kubernetes Engine)
1. **Sign in to Google Cloud Console**:
- Navigate to the GKE service.
- Click "Create cluster".
- Configure your cluster name, location, and node settings.
- Create the cluster.

2. **Install Google Cloud SDK**:
- [Installation guide](https://cloud.google.com/sdk/docs/install)

3. **Initialize the SDK and configure kubectl**:
```sh
gcloud init
gcloud container clusters get-credentials my-cluster --zone us-central1-a --project my-project
```

### 3. **DigitalOcean**

#### Create a Kubernetes Cluster with DigitalOcean Kubernetes (DOKS)
1. **Sign in to DigitalOcean Console**:
- Navigate to the Kubernetes service.
- Click "Create a Kubernetes Cluster".
- Configure your cluster name, region, and node pool.
- Create the cluster.

2. **Install doctl**:
- [Installation guide](https://docs.digitalocean.com/reference/doctl/how-to/install/)

3. **Download kubeconfig**:
```sh
doctl kubernetes cluster kubeconfig save my-cluster
```

### 4. **Vultr**

#### Create a Kubernetes Cluster with Vultr Kubernetes Engine (VKE)
1. **Sign in to Vultr Console**:
- Navigate to the Kubernetes service.
- Click "Deploy Cluster".
- Configure your cluster name, region, and node pool.
- Deploy the cluster.

2. **Download kubeconfig**:
- Navigate to your cluster details.
- Download the kubeconfig file.

### Replace `~/.kube/config` with the Credentials File
Download the credentials file from the respective cloud provider and replace your local kubeconfig:
```sh
mv path-to-downloaded-kubeconfig ~/.kube/config
```

### Create a Deployment Manifest
Create a file named `deployment.yml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-deployment
spec:
replicas: 3
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
image: nginx:latest
ports:
- containerPort: 80
```

### Create the Deployment
Run the following command to apply the deployment manifest:
```sh
kubectl apply -f deployment.yml
```

This sequence will help you create a Kubernetes cluster on any of these cloud providers, configure your local environment, and deploy an NGINX application.

To expose your app over the internet using Kubernetes Services, you can create either a NodePort or a LoadBalancer service. Below are the steps to create both types of services and access your application.

### 1. **NodePort Service**

A NodePort service exposes the application on a specific port on each node in the cluster.

#### Create a NodePort Service (service.yml)
Create a file named `service.yml` with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
name: nginx-service
spec:
selector:
app: nginx
ports:
- protocol: TCP
port: 80
targetPort: 80
nodePort: 30007 # This port can be any valid port within the NodePort range
type: NodePort
```

#### Apply the NodePort Service
Run the following command to apply the service manifest:

```sh
kubectl apply -f service.yml
```

#### Access the Application
Visit any of the nodes on port 30007. If you’re running Kubernetes locally (e.g., with `kind`), you can visit:

```sh
http://localhost:30007/
```

### 2. **LoadBalancer Service**

A LoadBalancer service is designed to work with cloud providers to create an external load balancer that routes traffic to the service.

#### Create a LoadBalancer Service
Modify `service.yml` to use the `LoadBalancer` type:

```yaml
apiVersion: v1
kind: Service
metadata:
name: nginx-service
spec:
selector:
app: nginx
ports:
- protocol: TCP
port: 80
targetPort: 80
type: LoadBalancer
```

#### Apply the LoadBalancer Service
Run the following command to re-apply the service manifest:

```sh
kubectl apply -f service.yml
```

#### Access the Application
1. **Check the LoadBalancer IP**:
After applying the LoadBalancer service, it may take a few minutes for the external IP to be assigned. You can check the status using:

```sh
kubectl get services
```

Look for the `EXTERNAL-IP` field for your `nginx-service`.

2. **Visit the LoadBalancer IP**:
Once the external IP is available, you can visit it in your browser:

```sh
http://<external-ip>
```

### Notes
- **Vultr Specific**: For Vultr, the LoadBalancer service will automatically work with Vultr’s cloud load balancer setup.
- **Local Development with kind**: If you are using `kind` for local Kubernetes, ensure you start the kind cluster with the appropriate configuration to support NodePort. For LoadBalancer, a cloud environment is required.

By following these steps, you can expose your application over the internet using either NodePort or LoadBalancer services in Kubernetes. This allows external traffic to reach your application running in the Kubernetes cluster.

You've outlined several key downsides of using services in a microservices architecture. Let's delve into each one of these issues in detail and discuss potential solutions or considerations:

### 1. **Scaling to Multiple Apps**
When you have multiple applications (frontend, backend, websocket server), each service needs its own load balancer. This can lead to complexities in routing traffic and centralized traffic management.

#### **Challenges:**
- **Traffic Routing:** Centralized traffic management (such as path-based routing from a single URL) is challenging.
- **Load Balancer Limits:** There might be limits on the number of load balancers you can create, depending on your cloud provider.

#### **Potential Solutions:**
- **API Gateway:** Use an API Gateway to centralize routing. An API Gateway can handle routing based on URLs or paths, directing traffic to the appropriate service.
- **Ingress Controllers:** In Kubernetes, an Ingress controller can provide centralized routing. It can route traffic based on URL paths and consolidate multiple services under a single load balancer.

### 2. **Multiple Certificates for Every Route**
Managing SSL/TLS certificates for multiple services can be cumbersome. Each load balancer needs its own certificate, which must be maintained and updated manually.

#### **Challenges:**
- **Certificate Management:** Certificates need to be created, managed, and renewed outside the cluster.
- **Manual Updates:** Certificates need to be manually updated if they expire.

#### **Potential Solutions:**
- **Centralized Certificate Management:** Use tools like Cert-Manager in Kubernetes, which automates the management and renewal of TLS certificates.
- **Wildcard Certificates:** Use wildcard certificates to cover multiple subdomains with a single certificate, reducing the number of certificates you need to manage.

### 3. **No Centralized Logic to Handle Rate Limiting**
Each load balancer can have its own set of rate limits, but there’s no centralized rate limiting across all services.

#### **Challenges:**
- **Inconsistent Rate Limiting:** Different services might have different rate limits, leading to inconsistent traffic management.
- **Complex Configuration:** Managing rate limits across multiple load balancers can be complex.

#### **Potential Solutions:**
- **API Gateway:** An API Gateway can provide centralized rate limiting, ensuring consistent rate limits across all services.
- **Service Mesh:** Implementing a service mesh (like Istio) can provide centralized rate limiting and traffic management features.

### Sample Manifest for Deployments and LoadBalancers
Here’s a sample Kubernetes manifest that deploys two separate services (nginx and apache) and attaches them to two separate LoadBalancer services:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-deployment
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
image: nginx:alpine
ports:
- containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: apache-deployment
spec:
replicas: 2
selector:
matchLabels:
app: apache
template:
metadata:
labels:
app: apache
spec:
containers:
- name: my-apache-site
image: httpd:2.4
ports:
- containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
name: nginx-service
spec:
selector:
app: nginx
ports:
- protocol: TCP
port: 80
targetPort: 80
type: LoadBalancer
---
apiVersion: v1
kind: Service
metadata:
name: apache-service
spec:
selector:
app: apache
ports:
- protocol: TCP
port: 80
targetPort: 80
type: LoadBalancer
```

### Deployment Steps
1. Save the above manifest to a file (e.g., `manifest.yml`).
2. Apply the manifest using `kubectl apply -f manifest.yml`.

This will create two deployments (nginx and apache) and expose them using two separate LoadBalancer services. You will notice two load balancers created for your services.

### Visualization and Management Tools
- **Kubernetes Dashboard:** Provides a UI to manage and visualize your Kubernetes resources, including load balancers.
- **Prometheus and Grafana:** For monitoring and visualizing metrics, including traffic and rate limits.

By using tools like API Gateways, Ingress controllers, and centralized certificate management, you can overcome some of the downsides associated with microservices architectures and improve the scalability, security, and manageability of your services.
Ingress and Ingress Controllers in Kubernetes provide a robust solution for managing external access to services within a cluster, especially for HTTP and HTTPS traffic. Here’s a detailed explanation of how they work and their benefits:

### **Ingress**
An Ingress is an API object that manages external access to services within a Kubernetes cluster. It typically handles HTTP and HTTPS traffic and provides features like load balancing, SSL termination, and name-based virtual hosting.

#### **Features of Ingress:**
- **Load Balancing:** Distributes traffic among multiple service instances.
- **SSL Termination:** Manages SSL certificates and terminates SSL/TLS traffic.
- **Name-Based Virtual Hosting:** Routes traffic to different services based on the hostname or URL path.

### **Ingress Controller**
An Ingress Controller is a specialized load balancer for Kubernetes that implements the Ingress resource. It watches the Kubernetes API server for updates to Ingress resources and configures the load balancing rules accordingly.
![](https://www.notion.so/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F085e8ad8-528e-47d7-8922-a23dc4016453%2F07041d43-0f69-4781-aa76-265f4111a913%2FScreenshot_2024-06-08_at_4.17.39_AM.png?table=block&id=cd9df811-016d-43e4-a03a-7bb8566b0d7f&cache=v2)
#### **Popular Ingress Controllers:**
- **NGINX Ingress Controller:** Widely used and supports a range of features including custom annotations and configurations.
- **Traefik:** Known for its dynamic configuration capabilities and ease of use.
- **HAProxy:** High-performance load balancing with extensive configuration options.
- **Istio:** Service mesh that includes ingress capabilities with advanced traffic management features.

### **Example: Deploying an Ingress Resource**

Here is a sample manifest to deploy an Ingress resource along with two services (nginx and apache):

#### **Service and Deployment Definitions:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-deployment
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
image: nginx:alpine
ports:
- containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: apache-deployment
spec:
replicas: 2
selector:
matchLabels:
app: apache
template:
metadata:
labels:
app: apache
spec:
containers:
- name: my-apache-site
image: httpd:2.4
ports:
- containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
name: nginx-service
spec:
selector:
app: nginx
ports:
- protocol: TCP
port: 80
targetPort: 80
type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
name: apache-service
spec:
selector:
app: apache
ports:
- protocol: TCP
port: 80
targetPort: 80
type: ClusterIP
```

#### **Ingress Definition:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: example-ingress
annotations:
nginx.ingress.kubernetes.io/rewrite-target: /
spec:
rules:
- host: nginx.example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: nginx-service
port:
number: 80
- host: apache.example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: apache-service
port:
number: 80
```

### **Steps to Deploy:**
1. **Deploy the Services and Deployments:**
```bash
kubectl apply -f service-deployment.yml
```
2. **Deploy the Ingress Resource:**
```bash
kubectl apply -f ingress.yml
```

### **Explanation:**
- **Services:** Both nginx and apache services are of type `ClusterIP`, making them accessible only within the cluster.
- **Ingress:** The Ingress resource routes traffic based on the hostname. Requests to `nginx.example.com` are directed to the `nginx-service`, and requests to `apache.example.com` are directed to the `apache-service`.

### **Benefits of Using Ingress:**
- **Centralized Management:** Single point of entry for all HTTP/HTTPS traffic.
- **SSL Termination:** Simplifies certificate management by handling SSL/TLS termination at the Ingress level.
- **Flexible Routing:** Supports path-based and host-based routing to direct traffic to the appropriate services.
- **Cost-Effective:** Reduces the need for multiple LoadBalancer services, which can be costly in cloud environments.

By using Ingress and Ingress Controllers, you can efficiently manage external access to your services, implement advanced routing rules, and enhance the security and scalability of your Kubernetes applications.

Absolutely, let’s delve into more details about Ingress Controllers in Kubernetes.

### **Ingress Controller Overview**
In Kubernetes, an Ingress Controller is responsible for implementing the Ingress resource. Unlike other controllers (like the ReplicaSet or Deployment controllers), Ingress Controllers are not part of the default Kubernetes control plane components. You need to install them manually.

### **Why You Need an Ingress Controller**
While the Ingress resource defines how to route external traffic to services within your cluster, the Ingress Controller is the component that actually implements this functionality. Without an Ingress Controller, the Ingress resource won't have any effect.

### **Popular Kubernetes Ingress Controllers**
Here are some of the most widely used Ingress Controllers:

#### 1. **NGINX Ingress Controller**
- **Overview:** Uses NGINX as a reverse proxy and load balancer.
- **Features:** Supports SSL termination, URL rewrites, request/response transformations, rate limiting, and more.
- **Installation:** Commonly deployed via Helm or Kubernetes manifests.

#### 2. **HAProxy Ingress**
- **Overview:** Uses HAProxy, a high-performance TCP/HTTP load balancer.
- **Features:** Advanced load balancing algorithms, SSL termination, HTTP/2 support, and dynamic scaling.
- **Installation:** Deployed via Helm charts or YAML manifests.

#### 3. **Traefik Ingress Controller**
- **Overview:** Uses Traefik, which is known for its dynamic configuration capabilities.
- **Features:** Supports dynamic service discovery, automatic certificate management with Let’s Encrypt, metrics, and more.
- **Installation:** Deployed via Helm or Kubernetes manifests.

### **Installation and Configuration Example**

Let’s go through an example of installing and configuring the NGINX Ingress Controller using Helm.

#### **Prerequisites:**
- A running Kubernetes cluster
- `kubectl` configured to interact with the cluster
- Helm installed

#### **Steps to Install NGINX Ingress Controller:**

1. **Add the NGINX Ingress Helm repository:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

2. **Install the NGINX Ingress Controller:**
```bash
helm install nginx-ingress ingress-nginx/ingress-nginx \
--set controller.publishService.enabled=true
```

3. **Verify the installation:**
```bash
kubectl get pods -n default -l app.kubernetes.io/name=ingress-nginx
```

#### **Creating an Ingress Resource:**
After installing the Ingress Controller, you can create Ingress resources to manage traffic routing.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: example-ingress
annotations:
nginx.ingress.kubernetes.io/rewrite-target: /
spec:
rules:
- host: nginx.example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: nginx-service
port:
number: 80
- host: apache.example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: apache-service
port:
number: 80
```

1. **Apply the Ingress resource:**
```bash
kubectl apply -f ingress.yml
```

### **Summary**
- **Manual Installation:** Ingress Controllers are not included by default and need to be installed manually.
- **Popular Controllers:** NGINX, HAProxy, and Traefik are among the most commonly used Ingress Controllers.
- **Enhanced Functionality:** They provide critical features like load balancing, SSL termination, and dynamic routing.

### **Conclusion**
Using an Ingress Controller like NGINX, HAProxy, or Traefik greatly enhances the flexibility and manageability of routing traffic within a Kubernetes cluster. By leveraging these controllers, you can centralize your traffic management, improve security with SSL termination, and implement complex routing rules efficiently.

Namespaces in Kubernetes provide a mechanism to partition resources within a cluster, facilitating better management and organization. They are particularly useful in environments with multiple teams or projects, or to separate environments like development, staging, and production. Here’s a detailed guide on working with namespaces in Kubernetes:

### **Working with Namespaces in Kubernetes**

#### **Viewing Pods in the Default Namespace**
```bash
kubectl get pods
```
This command lists all pods in the default namespace.

#### **Creating a New Namespace**
To create a new namespace, use:
```bash
kubectl create namespace backend-team
```

#### **Listing All Namespaces**
To list all namespaces in the cluster, use:
```bash
kubectl get namespaces
```

#### **Viewing Pods in a Specific Namespace**
To view all pods in a specific namespace (e.g., `my-namespace`), use:
```bash
kubectl get pods -n my-namespace
```

### **Creating a Deployment in a Specific Namespace**

#### **Sample Deployment Manifest:**
Here’s a manifest for an NGINX deployment in the `backend-team` namespace:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: nginx-deployment
namespace: backend-team
spec:
replicas: 3
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
image: nginx:latest
ports:
- containerPort: 80
```

Save this manifest to a file named `deployment-ns.yml`.

#### **Applying the Deployment Manifest:**
To apply the manifest and create the deployment in the specified namespace, use:
```bash
kubectl apply -f deployment-ns.yml
```

#### **Getting Deployments in the Namespace:**
To get all deployments in the `backend-team` namespace, use:
```bash
kubectl get deployment -n backend-team
```

#### **Getting Pods in the Namespace:**
To get all pods in the `backend-team` namespace, use:
```bash
kubectl get pods -n backend-team
```

### **Setting the Default Context to a Namespace**

#### **Setting the Namespace in the Current Context:**
To set the default namespace for the current context to `backend-team`, use:
```bash
kubectl config set-context --current --namespace=backend-team
```

#### **Viewing Pods in the Set Namespace:**
Now, when you run:
```bash
kubectl get pods
```
It will show the pods in the `backend-team` namespace.

### **Reverting Back to the Default Namespace**

#### **Resetting the Default Namespace in the Current Context:**
To revert back to using the default namespace:
```bash
kubectl config set-context --current --namespace=default
```

By setting the context's namespace, you avoid having to specify the `-n` flag for every command, streamlining your workflow when working extensively within a particular namespace.

### **Summary**
Namespaces in Kubernetes provide a powerful way to manage and organize resources in a cluster, especially in multi-team or multi-environment scenarios. By effectively using namespaces, you can isolate and manage resources more efficiently, improve security, and streamline operational workflows. The steps and commands provided here cover the basic operations you need to get started with namespaces in Kubernetes.

To install the NGINX Ingress Controller using Helm, you can follow these steps:

### **Install Helm**

First, ensure Helm is installed on your system. You can find detailed instructions on the Helm website. Here’s a brief summary for different operating systems:

#### **For macOS:**
```bash
brew install helm
```

#### **For Linux:**
Download the latest version of Helm from the [Helm GitHub releases page](https://github.com/helm/helm/releases).

```bash
wget https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

#### **For Windows:**
Download the Helm binary from the [Helm GitHub releases page](https://github.com/helm/helm/releases) and add it to your PATH.

### **Add the Ingress-NGINX Chart Repository**

Once Helm is installed, add the ingress-nginx chart repository and update it:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### **Install the NGINX Ingress Controller**

Now, install the NGINX Ingress Controller using Helm. This command will install the controller in the `ingress-nginx` namespace and create the namespace if it doesn't exist:

```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

### **Verify the Installation**

To check if the NGINX Ingress Controller pods are running, use the following command:

```bash
kubectl get pods -n ingress-nginx
```

You should see the NGINX Ingress Controller pods running in the `ingress-nginx` namespace.

### **Check the Services**

The Helm installation will create a LoadBalancer service for the NGINX Ingress Controller. To see this, list all services in all namespaces:

```bash
kubectl get services --all-namespaces
```

You should see a LoadBalancer service that routes traffic to the NGINX Ingress Controller pods.

### **Summary**

By following these steps, you’ve installed the NGINX Ingress Controller using Helm and verified that it is running correctly. This controller will now handle Ingress resources in your cluster, providing load balancing, SSL termination, and name-based virtual hosting.

Here's a quick summary of the commands used:

1. **Install Helm:**
- macOS: `brew install helm`
- Linux:
```bash
wget https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
tar -zxvf helm-v3.9.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```
- Windows: Download and add to PATH from the [Helm GitHub releases page](https://github.com/helm/helm/releases).

2. **Add and Update the Ingress-NGINX Chart Repository:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

3. **Install the NGINX Ingress Controller:**
```bash
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

4. **Verify the Installation:**
```bash
kubectl get pods -n ingress-nginx
```

5. **Check the Services:**
```bash
kubectl get services --all-namespaces
```

This setup ensures that your Kubernetes cluster can handle external traffic through the NGINX Ingress Controller, routing it to the appropriate services based on your Ingress rules.
