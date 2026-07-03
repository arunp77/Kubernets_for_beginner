
# Deploy Docker image to Kubernets


---
**NOTE:** 
1. 🧠 Understanding Pods, Deployments, and Services

    🎁 POD: The Smallest Unit
        - A Pod is like a wrapper around your container(s).
        - Each Pod usually runs one container.
        - If the Pod dies, Kubernetes can restart it or replace it.

    👉 Think: One Pod = One running copy of your app

2. 🛠️ DEPLOYMENT: The Manager

    - A Deployment manages replicas of your Pods.
    - You define the desired state (e.g., 3 replicas), and Kubernetes ensures it.
    - It handles rolling updates, scaling, and more.

👉 Think: Deployment is the blueprint and watchdog for your Pods.

3. 🌐 SERVICE: The Network Access Point

    - A Service gives you a stable way to access Pods.
    - It load balances traffic between Pods.
    - It stays constant even if Pods come and go.

| Type             | Use Case                           |
| ---------------- | ---------------------------------- |
| **ClusterIP**    | Internal access only               |
| **NodePort**     | Exposes app on a port on each node |
| **LoadBalancer** | Works with cloud providers         |
| **Ingress**      | For routing by domain/URL path     |

---



Let’s deploy the Docker image (yourusername/my-python-app) to a Kubernetes cluster using:

- Minikube (for local practice), or
- Play with Kubernetes (browser-based, no install), or
- A cloud provider (e.g., Google Kubernetes Engine) — more advanced

## 🛠️ PART 1: Set Up Minikube (Local Kubernetes Cluster)

Minikube is a tool that lets you run a single-node Kubernetes cluster **locally** on your machine.

---

### ✅ Step 1: Install Prerequisites

You need:

1. **Docker** installed ✅ *(You already have this!)*
2. **Kubectl** – the Kubernetes command-line tool
3. **Minikube** – the tool to run Kubernetes locally

---

### 🧰 Step 2: Install `kubectl` and `minikube`

#### 🔹 For Windows/macOS/Linux:

* Install `kubectl` (official guide):
  [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

* Install `minikube` (official guide):
  [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)

Or use direct install commands:

#### On macOS (with Homebrew):

```bash
brew install kubectl
brew install minikube
```

#### On Ubuntu:

```bash
sudo apt update && sudo apt install -y kubectl
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

#### On Windows (with Chocolatey):

```powershell
choco install kubernetes-cli
choco install minikube
```

---

### ▶️ Step 3: Start Minikube

Once installed, run this command to start your local Kubernetes cluster:

```bash
minikube start
```

* This may take a few minutes the first time.
* It will spin up a virtual machine or Docker container as your Kubernetes node.

---

### 🔍 Step 4: Check That It’s Running

```bash
kubectl get nodes
```

You should see something like:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.xx.x
```

You now have a fully working **Kubernetes cluster** on your machine! 🎉

---

## 🚀 PART 2: Deploy Your Docker Image to Kubernetes


Now, let’s deploy your `yourusername/my-python-app` image (from Docker Hub).

---

### 📄 Step 1: Create a Deployment YAML File

Create a file named `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-python-app
  template:
    metadata:
      labels:
        app: my-python-app
    spec:
      containers:
      - name: my-python-app
        image: yourusername/my-python-app:latest
        ports:
        - containerPort: 5000
```

To scale up i.e how many copies (replicas) of your app should be running, change `replicas: 2` to `replicas: 10` for replicating it from 2 to 10 pods. After changing it, run following commnad to apply changes:

```sh
kubectl apply -f deployment.yaml
```

Another way, we can also do is to use following command without changing in yaml file:

```sh
kubectl scale deployment my-python-app --replicas=10
```
and then you check the number of pods used using: `kubectl get pods`.


> Replace `yourusername` with your actual Docker Hub username. 

### 🌐 Step 2: Create a Service to Expose Your App

Create a file called `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-python-app-service
spec:
  type: NodePort
  selector:
    app: my-python-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

This will expose your app on a random high port on your localhost.

---

### 🧱 Step 3: Apply the Configurations

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

### 🚦 Step 4: Verify Everything Is Running

```bash
kubectl get pods
kubectl get services
```

You’ll see:

* 2 pods of your app running
* A service exposing it with a `NodePort` (e.g., port 30000)

---

### 🌍 Step 5: Access Your App

Run:

```bash
minikube service my-python-app-service
```

This will open your browser and take you to:

```
http://<minikube-ip>:<node-port>
```

You’ll see: **"Hello from my container!"** — but this time served by Kubernetes! 🎉

---

## ✅ Recap: You Just...

* Built and pushed your Docker image to Docker Hub
* Installed Minikube and kubectl
* Created a Kubernetes Deployment and Service
* Accessed your app running inside a Kubernetes cluster

---
