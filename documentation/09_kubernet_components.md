# **main components of Kubernetes** 

## 🧠 1. **Control Plane Components**

> These are the brains of the Kubernetes cluster. They manage the cluster, decide what runs where, and handle overall orchestration.

| Component                    | Description                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------- |
| **kube-apiserver**           | The **front door** to your cluster. Accepts API requests (from `kubectl`, other tools).        |
| **etcd**                     | The **key-value store** holding all cluster data/state (like cluster "memory").                |
| **kube-scheduler**           | Decides **which node** a new Pod should run on, based on resources and policies.               |
| **kube-controller-manager**  | Runs background controllers (like keeping replicas running, restarting failed pods).           |
| **cloud-controller-manager** | Handles cloud-specific stuff (e.g., provisioning load balancers or volumes in AWS, GCP, etc.). |

---

## 🖥️ 2. **Node Components**

> These run on every worker node (the servers that actually run your apps).

| Component             | Description                                                                     |
| --------------------- | ------------------------------------------------------------------------------- |
| **kubelet**           | Talks to the control plane, **makes sure containers are running** as requested. |
| **kube-proxy**        | Manages **network routing** on the node (e.g., Service traffic to Pods).        |
| **Container runtime** | Software that runs containers (e.g., **containerd**, **Docker**, **CRI-O**).    |

---

## ⚙️ 3. **Add-ons / Optional Components**

> These extend Kubernetes functionality, often used in production setups.

| Add-on                 | Purpose                                                                          |
| ---------------------- | -------------------------------------------------------------------------------- |
| **DNS (CoreDNS)**      | Enables service discovery via DNS (e.g., `my-service.default.svc.cluster.local`) |
| **Dashboard**          | Web-based UI to manage your cluster                                              |
| **Ingress Controller** | Manages **HTTP routing** and custom domains for services                         |
| **Metrics Server**     | Collects resource usage data (used in auto-scaling)                              |
| **Helm**               | Package manager to install apps easily in Kubernetes                             |

---

## 🔄 Summary Diagram (Text-Based)

```
                [ Control Plane ]
+---------------------------------------------------------+
|  kube-apiserver   etcd   scheduler   controllers        |
+---------------------------------------------------------+

                [ Worker Nodes (N) ]
+-----------------------------------------+
|  kubelet   |  kube-proxy  |  containerd |
+-----------------------------------------+
         Pod 1   Pod 2   Pod 3   ...
```

---

## 🧠 TL;DR Summary

| Layer         | Key Components                           | Role                                                    |
| ------------- | ---------------------------------------- | ------------------------------------------------------- |
| Control Plane | API Server, etcd, Scheduler, Controllers | Makes decisions, stores cluster state                   |
| Node          | kubelet, kube-proxy, container runtime   | Executes containers, handles networking                 |
| Add-ons       | CoreDNS, Ingress, Metrics, Helm, etc.    | Optional features for observability, access, management |

---

Would you like a **visual diagram**, or should we go into **how these components interact during app deployment**?
