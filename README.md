# Kubernetes Masterclass — For an Experienced Docker Engineer

## Topics to follow in this tutorial:

- Concepts of Docker containerization / images
- Kubernetes fundamentals and relation with Docker.
- Setting up local environment
- Key kubernetes objects such as Pods, Deployments, ReplicaSets, Services, ConfigMaps, Secrets, Namespaces, Volumes, PV, PVC, StorageClasses, Ingress, Health Checks (Probes), Jobs, CronJobs, DaemonSets, StatefulSets, RBAC, Network Policies, Helm, Monitoring with Prometheus and Grafana, Logging, CI/CD with Kubernetes, Production Best Practices, Managed Kubernetes (EKS / GKE / AKS)
- And finally we will create a production application with React + FastAPI + Postgres + Redis + worker, deployed and evolved throughout the course.

The course builds one real production application incrementally. By the end,
every concept below has been used in anger on that app, not just in a toy example.

## Environment status

| Tool | Status |
|---|---|
| Docker | ✅ installed (29.1.3) |
| kubectl | ⏳ installed in Lesson 3 |
| kind | ⏳ installed in Lesson 3 |
| helm | ⏳ installed in Lesson 26 |

## Folder layout

```
Kubernets_for_beginner/
├── README.md              ← you are here (GitHub landing page)
├── mkdocs.yml              ← MkDocs site config (published via GitHub Pages)
└── docs/                   ← everything the published site is built from
    ├── index.md            ← site homepage (mirrors this README)
    ├── docker_basics/       ← Docker fundamentals chapters
    └── kubernetes_basics/   ← the full Kubernetes masterclass, chapter per file
```

## Documentation site

This repo is published as a browsable site via [MkDocs](https://www.mkdocs.org/)
+ [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/), deployed
to GitHub Pages. The site has two top-level sections — **Docker Basics** and
**Kubernetes Basics** — each expandable to every chapter. See
[Publishing the site](#publishing-the-site) below for how it's built/deployed.

## Curriculum & progress

Every chapter lives in [docs/kubernetes_basics/](docs/kubernetes_basics/) and follows the
full masterclass format: objectives, theory, architecture diagrams, a Docker
comparison, internal working, a hands-on lab, YAML walkthrough, troubleshooting,
best practices, interview questions, a mini assignment, and a summary.

- [x] 1. [Why Kubernetes?](docs/kubernetes_basics/01-why-kubernetes.md)
- [x] 2. [Kubernetes Architecture](docs/kubernetes_basics/02-kubernetes-architecture.md)
- [x] 3. [Installing a Local Cluster (Kind)](docs/kubernetes_basics/03-installing-local-cluster-kind.md)
- [x] 4. [kubectl Deep Dive](docs/kubernetes_basics/04-kubectl-deep-dive.md)
- [x] 5. [Pods](docs/kubernetes_basics/05-pods.md)
- [x] 6. [YAML for Kubernetes](docs/kubernetes_basics/06-yaml-for-kubernetes.md)
- [x] 7. [ReplicaSets](docs/kubernetes_basics/07-replicasets.md)
- [x] 8. [Deployments](docs/kubernetes_basics/08-deployments.md)
- [x] 9. [Services](docs/kubernetes_basics/09-services.md)
- [x] 10. [Labels and Selectors](docs/kubernetes_basics/10-labels-and-selectors.md)
- [x] 11. [Namespaces](docs/kubernetes_basics/11-namespaces.md)
- [x] 12. [ConfigMaps](docs/kubernetes_basics/12-configmaps.md)
- [x] 13. [Secrets](docs/kubernetes_basics/13-secrets.md)
- [x] 14. [Persistent Volumes](docs/kubernetes_basics/14-persistent-volumes.md)
- [x] 15. [Storage Classes](docs/kubernetes_basics/15-storage-classes.md)
- [x] 16. [Ingress](docs/kubernetes_basics/16-ingress.md)
- [x] 17. [Health Checks (Probes)](docs/kubernetes_basics/17-health-checks-probes.md)
- [x] 18. [Jobs](docs/kubernetes_basics/18-jobs.md)
- [x] 19. [CronJobs](docs/kubernetes_basics/19-cronjobs.md)
- [x] 20. [DaemonSets](docs/kubernetes_basics/20-daemonsets.md)
- [x] 21. [StatefulSets](docs/kubernetes_basics/21-statefulsets.md)
- [x] 22. [Resource Requests and Limits](docs/kubernetes_basics/22-resource-requests-and-limits.md) *(moved up from #23)*
- [x] 23. [Horizontal Pod Autoscaler](docs/kubernetes_basics/23-horizontal-pod-autoscaler.md) *(moved down from #22)*
- [x] 24. [RBAC](docs/kubernetes_basics/24-rbac.md)
- [x] 25. [Network Policies](docs/kubernetes_basics/25-network-policies.md)
- [x] 26. [Helm](docs/kubernetes_basics/26-helm.md)
- [x] 27. [Monitoring with Prometheus and Grafana](docs/kubernetes_basics/27-monitoring-prometheus-grafana.md)
- [x] 28. [Logging](docs/kubernetes_basics/28-logging.md)
- [x] 29. [CI/CD with Kubernetes](docs/kubernetes_basics/29-cicd-with-kubernetes.md)
- [x] 30. [Production Best Practices](docs/kubernetes_basics/30-production-best-practices.md)
- [x] 31. [Managed Kubernetes (EKS / GKE / AKS)](docs/kubernetes_basics/31-managed-kubernetes.md)

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

## Topics to follow

- For Docker basics refer this [docker basics](docs/docker_basics/)
- For Kubernetes basics refer this [kubernetes basics](docs/kubernetes_basics/)

## Publishing the site

The site is built with MkDocs + Material and auto-deployed to GitHub Pages by
[.github/workflows/deploy-docs.yml](.github/workflows/deploy-docs.yml) on every
push to `main`.

**One-time setup** (repo owner only): in GitHub, go to
**Settings → Pages → Build and deployment → Source**, and select
**GitHub Actions**. After that, every push to `main` rebuilds and redeploys the
site automatically.

**Local preview**, using the project's own virtualenv:

```bash
python3 -m venv .venv
.venv/bin/pip install mkdocs-material
.venv/bin/mkdocs serve
```

Then open `http://127.0.0.1:8000`.
