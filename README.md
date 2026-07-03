# Kubernetes Masterclass — For an Experienced Docker Engineer

A university-style, hands-on Kubernetes course. No Docker fundamentals are
re-taught — every new concept is anchored to something you already know from
Docker, Compose, and Linux.

## Rules of engagement

- One lesson at a time. We do not move to lesson *N+1* until you've confirmed
  you understand lesson *N* and answered its review questions.
- Every command and every YAML field is explained before it's used — nothing
  is ever "just paste this."
- The course builds one real production application incrementally. By the
  end, every concept below has been used in anger on that app, not just in a
  toy example.

## Environment status

| Tool | Status |
|---|---|
| Docker | ✅ installed (29.1.3) |
| kubectl | ⏳ installed in Lesson 3 |
| kind | ⏳ installed in Lesson 3 |
| helm | ⏳ installed in Lesson 26 |

## Folder layout

```
kubernetes-masterclass/
├── README.md              ← you are here (course index + progress tracker)
├── lessons/                ← one markdown file per lesson (theory + labs)
├── labs/                   ← working YAML/config files you'll apply with kubectl
├── cheatsheets/             ← quick-reference sheets, built up as we go
└── final-project/           ← the production app: React + FastAPI + Postgres + Redis + worker,
                                deployed and evolved throughout the course
```

## Curriculum & progress

I reordered two topics from your original list (noted below) — Resource
Requests/Limits now comes *before* the Horizontal Pod Autoscaler, since HPA's
CPU/memory-based scaling math is literally computed as a percentage of the
resource *requests* you set. Teaching HPA first would mean referencing a
field you haven't learned yet.

- [x] 1. Why Kubernetes?
- [ ] 2. Kubernetes Architecture
- [ ] 3. Installing a Local Cluster (Kind)
- [ ] 4. kubectl Deep Dive
- [ ] 5. Pods
- [ ] 6. YAML for Kubernetes
- [ ] 7. ReplicaSets
- [ ] 8. Deployments
- [ ] 9. Services
- [ ] 10. Labels and Selectors
- [ ] 11. Namespaces
- [ ] 12. ConfigMaps
- [ ] 13. Secrets
- [ ] 14. Persistent Volumes
- [ ] 15. Storage Classes
- [ ] 16. Ingress
- [ ] 17. Health Checks (Probes)
- [ ] 18. Jobs
- [ ] 19. CronJobs
- [ ] 20. DaemonSets
- [ ] 21. StatefulSets
- [ ] 22. Resource Requests and Limits *(moved up from #23)*
- [ ] 23. Horizontal Pod Autoscaler *(moved down from #22)*
- [ ] 24. RBAC
- [ ] 25. Network Policies
- [ ] 26. Helm
- [ ] 27. Monitoring with Prometheus and Grafana
- [ ] 28. Logging
- [ ] 29. CI/CD with Kubernetes
- [ ] 30. Production Best Practices
- [ ] 31. Managed Kubernetes (EKS / GKE / AKS)

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

## Status
Currently on: **Lesson 1 — Why Kubernetes?** (see `lessons/01-why-kubernetes.md`)
