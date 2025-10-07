## 🧠 What is the Kubernetes API?

The **Kubernetes API** is a **RESTful HTTP API** that acts as the **gateway to your entire cluster**.

* Everything in Kubernetes — Pods, Deployments, Services, Nodes, etc. — is a **resource** exposed through the API.
* Whether you're using:

  * `kubectl`
  * the Kubernetes Dashboard
  * or automated CI/CD tools

➡️ You're **talking to the Kubernetes API** behind the scenes.

---

## 🧩 Why is it important?

* **Single point of control** for the whole cluster.
* Enables **automation**, **monitoring**, and **external integrations**.
* Used by **controllers, schedulers**, and even **custom tools**.

---

## 🔑 Key Concepts

### 1. **Resources**

Kubernetes exposes everything as a resource.

Common ones include:

| Kind               | Description                       |
| ------------------ | --------------------------------- |
| Pod                | Smallest unit of compute          |
| Deployment         | Manages replicas of Pods          |
| Service            | Exposes Pods on the network       |
| Node               | A machine (worker) in the cluster |
| Namespace          | Logical grouping of resources     |
| ConfigMap / Secret | Store config and sensitive data   |

Each resource can be **created**, **read**, **updated**, or **deleted** (CRUD) via the API.

---

### 2. **API Groups and Versions**

Kubernetes organizes resources into **API groups** and **versions**, such as:

```
apiVersion: v1               # Core group
apiVersion: apps/v1          # Deployments, StatefulSets, etc.
apiVersion: networking.k8s.io/v1  # Ingress, NetworkPolicy
```

This versioning ensures **backward compatibility** and allows Kubernetes to evolve safely.

---

### 3. **API Paths (Endpoints)**

Here’s what a typical API path looks like:

```
/api/v1/pods
/apis/apps/v1/deployments
/apis/networking.k8s.io/v1/ingresses
```

These are HTTP endpoints. You can interact with them directly using tools like:

```bash
kubectl get pods
curl -k https://<api-server>/api/v1/pods
```

---

## 🧰 Example: Create a Pod via API

When you run this:

```bash
kubectl apply -f pod.yaml
```

You’re sending a `POST` request to the Kubernetes API like this:

```
POST /api/v1/namespaces/default/pods
Content-Type: application/json

{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": { "name": "my-pod" },
  ...
}
```

The API server validates it, stores it in **etcd**, and the **kubelet** gets the instruction to run it.

---

## 🔐 API Security (RBAC & Auth)

Access to the API is protected using:

| Feature                   | Purpose                                                           |
| ------------------------- | ----------------------------------------------------------------- |
| **Authentication**        | Verifies *who* is calling the API (e.g., user, service)           |
| **Authorization (RBAC)**  | Controls *what* they can do (e.g., read pods, create deployments) |
| **Admission Controllers** | Validate/modify requests before they are persisted                |

---

## 📌 Summary

| Concept        | Description                                       |
| -------------- | ------------------------------------------------- |
| Kubernetes API | RESTful HTTP API for managing the entire cluster  |
| kubectl        | CLI tool that talks to the API server             |
| Resources      | Everything (Pods, Services, etc.) is a resource   |
| API Paths      | Organized by groups and versions                  |
| Security       | Handled via Auth, RBAC, and Admission Controllers |

---
