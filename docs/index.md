# Kubernetes Masterclass — For an Experienced Docker Engineer

## Topics to follow in this tutorial:

- Concepts of Docker containerization / images
- Kubernetes fundamentals and relation with Docker.
- Setting up local environment
- Key kubernetes objects such as Pods, Deployments, ReplicaSets, Services, ConfigMaps, Secrets, Namespaces, Volumes, PV, PVC, StorageClasses, Ingress, Health Checks (Probes), Jobs, CronJobs, DaemonSets, StatefulSets, RBAC, Network Policies, Helm, Monitoring with Prometheus and Grafana, Logging, CI/CD with Kubernetes, Production Best Practices, Managed Kubernetes (EKS / GKE / AKS)
- And finally we will create a production application with React + FastAPI + Postgres + Redis + worker, deployed and evolved throughout the course.

The course builds one real production application incrementally. By the end,
every concept below has been used in anger on that app, not just in a toy example.

Use the top navigation to jump into **Docker Basics** or **Kubernetes Basics** —
each section lists every chapter in order.

## Environment status

| Tool | Status |
|---|---|
| Docker | ✅ installed (29.1.3) |
| kubectl | ⏳ installed in Lesson 3 |
| kind | ⏳ installed in Lesson 3 |
| helm | ⏳ installed in Lesson 26 |

## Curriculum & progress

Every chapter in [Kubernetes Basics](kubernetes_basics/01-why-kubernetes.md) follows the
full masterclass format: objectives, theory, architecture diagrams, a Docker
comparison, internal working, a hands-on lab, YAML walkthrough, troubleshooting,
best practices, interview questions, a mini assignment, and a summary.

1. [Why Kubernetes?](kubernetes_basics/01-why-kubernetes.md)
2. [Kubernetes Architecture](kubernetes_basics/02-kubernetes-architecture.md)
3. [Installing a Local Cluster (Kind)](kubernetes_basics/03-installing-local-cluster-kind.md)
4. [kubectl Deep Dive](kubernetes_basics/04-kubectl-deep-dive.md)
5. [Pods](kubernetes_basics/05-pods.md)
6. [YAML for Kubernetes](kubernetes_basics/06-yaml-for-kubernetes.md)
7. [ReplicaSets](kubernetes_basics/07-replicasets.md)
8. [Deployments](kubernetes_basics/08-deployments.md)
9. [Services](kubernetes_basics/09-services.md)
10. [Labels and Selectors](kubernetes_basics/10-labels-and-selectors.md)
11. [Namespaces](kubernetes_basics/11-namespaces.md)
12. [ConfigMaps](kubernetes_basics/12-configmaps.md)
13. [Secrets](kubernetes_basics/13-secrets.md)
14. [Persistent Volumes](kubernetes_basics/14-persistent-volumes.md)
15. [Storage Classes](kubernetes_basics/15-storage-classes.md)
16. [Ingress](kubernetes_basics/16-ingress.md)
17. [Health Checks (Probes)](kubernetes_basics/17-health-checks-probes.md)
18. [Jobs](kubernetes_basics/18-jobs.md)
19. [CronJobs](kubernetes_basics/19-cronjobs.md)
20. [DaemonSets](kubernetes_basics/20-daemonsets.md)
21. [StatefulSets](kubernetes_basics/21-statefulsets.md)
22. [Resource Requests and Limits](kubernetes_basics/22-resource-requests-and-limits.md) *(moved up from #23)*
23. [Horizontal Pod Autoscaler](kubernetes_basics/23-horizontal-pod-autoscaler.md) *(moved down from #22)*
24. [RBAC](kubernetes_basics/24-rbac.md)
25. [Network Policies](kubernetes_basics/25-network-policies.md)
26. [Helm](kubernetes_basics/26-helm.md)
27. [Monitoring with Prometheus and Grafana](kubernetes_basics/27-monitoring-prometheus-grafana.md)
28. [Logging](kubernetes_basics/28-logging.md)
29. [CI/CD with Kubernetes](kubernetes_basics/29-cicd-with-kubernetes.md)
30. [Production Best Practices](kubernetes_basics/30-production-best-practices.md)
31. [Managed Kubernetes (EKS / GKE / AKS)](kubernetes_basics/31-managed-kubernetes.md)

## Final project (built incrementally)

- React frontend
- FastAPI backend
- PostgreSQL database
- Redis cache
- Background worker
- Prometheus + Grafana
- NGINX Ingress
- Persistent storage, ConfigMaps, Secrets
- Deployments, Services, Autoscaling, Health probes, Rolling updates
- Helm charts
- CI/CD pipeline
- Security best practices

## Where to start

- New to this course? Start with [Docker Basics](docker_basics/00_kubernet_in_simple_words.md).
- Already comfortable with Docker? Jump straight into
  [Kubernetes Basics, Chapter 1](kubernetes_basics/01-why-kubernetes.md).

Source lives on [GitHub](https://github.com/arunp77/Kubernets_for_beginner).
