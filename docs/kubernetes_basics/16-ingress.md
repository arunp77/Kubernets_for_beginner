# Chapter 16 — Ingress

## What it is

An **Ingress** is an object describing HTTP(S) routing rules — host/path
based — for traffic entering the cluster from outside, routed to internal
Services (Chapter 9). It is **not itself a running proxy** — it's a
declarative rule set that an **Ingress Controller** (a separate piece of
software you install, e.g. ingress-nginx) reads and acts on.

## Why it exists

Chapter 9 established `LoadBalancer`-type Services each provision a real
cloud load balancer. That's fine for one Service, but consider a real app
with 10 microservices, each needing external HTTP access on standard
ports 80/443 with its own hostname/path — 10 `LoadBalancer` Services means
10 real cloud load balancers, each billed separately, each needing its own
TLS cert management. Ingress solves this the same way it's solved
everywhere in this course: add one more layer of indirection. One (or a
few) Ingress Controller(s), each backed by a single real load balancer,
read routing rules and fan traffic out internally to the right Service
based on hostname/path — dramatically cheaper and more manageable than a
load balancer per service.

## When to use it

Any time you need to expose multiple HTTP(S) services externally under
one IP/domain, with host- or path-based routing, and centralized TLS
termination — which describes the large majority of real production web
traffic. Use a raw `LoadBalancer` Service instead only for non-HTTP
protocols Ingress can't route (raw TCP/UDP), or single-service edge cases.

## Internal architecture

- The Ingress **object** is just YAML describing rules — it does nothing
  on its own. You must separately install an **Ingress Controller**
  (most commonly ingress-nginx, or cloud-specific ones like AWS Load
  Balancer Controller) — this is a real Deployment/DaemonSet running
  actual proxy software (often literally an nginx configuration
  generator) that watches Ingress objects via the API server and
  reconfigures itself to match, exactly the controller pattern from
  Chapter 1/2, applied to HTTP routing.
- Typically, the Ingress Controller itself is exposed via **one**
  `LoadBalancer`-type Service (or NodePort in local/on-prem setups) — so
  you pay for one real external load balancer, and Ingress objects
  multiplex arbitrarily many hostnames/paths behind it internally, for
  free.
- **TLS termination** commonly happens *at* the Ingress Controller — it
  decrypts HTTPS from the client, then talks plain HTTP internally to your
  Services (though re-encryption to the backend is also possible/common
  for stricter security postures) — centralizing certificate management
  in one place instead of every individual service needing its own.
- Routing is evaluated as: **host match** (e.g., `api.example.com`) then
  **path match** (e.g., `/v1`) → forward to a named Service + port.

## YAML Definition

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: whoami.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami-svc
                port:
                  number: 80
    - host: api.local
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
```
- `apiVersion: networking.k8s.io/v1` — Ingress lives in the networking API
  group (Chapter 6's grouping convention).
- `spec.ingressClassName` — which installed Ingress Controller should
  handle this object; a cluster can run multiple controllers
  side-by-side (e.g., internal vs. external-facing), each claiming a
  different `IngressClass`.
- `spec.rules[].host` — the hostname this rule applies to; requests for
  other hostnames simply don't match this rule.
- `http.paths[].path` / `pathType` — `Prefix` matches the path and
  everything under it; `Exact` requires a literal match; `ImplementationSpecific`
  defers matching semantics to the controller.
- `backend.service.name`/`port` — the internal Service (Chapter 9,
  `ClusterIP` type is normal here — the Ingress Controller reaches it
  internally, so it never needs external exposure itself) traffic is
  forwarded to.
- `metadata.annotations` — Ingress's escape hatch for controller-specific
  behavior not covered by the generic spec (here, an nginx-specific
  rewrite rule) — a direct consequence of Ingress being a lowest-common-
  denominator spec across many different controller implementations.

## Hands-on Example

**1. Install ingress-nginx** on your Kind cluster (Kind needs a specific
setup to make host ports reachable — this is the standard documented
pattern, given as a one-time setup step):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```
This applies a full Deployment + Service + RBAC bundle — the Ingress
Controller is a real workload running inside your cluster, using
everything you've learned so far (Deployments, Services, RBAC roles
you'll formally meet in Chapter 24).

**2. Set up your Deployment + Service** (reuse Chapter 9's whoami setup),
then apply the Ingress:
```bash
kubectl apply -f whoami-deploy.yaml -f whoami-svc.yaml -f whoami-ingress.yaml
kubectl get ingress
```

**3. Test host-based routing** — since `whoami.local` isn't real DNS,
resolve it manually per-request instead of editing `/etc/hosts`:
```bash
curl --resolve whoami.local:80:127.0.0.1 http://whoami.local/
```
(On Kind, ingress-nginx is typically exposed on the host's port 80/443
directly via the cluster's special `extraPortMappings` config — if
`curl` doesn't reach it, check `kubectl get svc -n ingress-nginx` for the
actual exposed port and adjust.)

**4. Prove one controller multiplexes many hosts** — add a second
Deployment/Service/host rule (`api.local` from the YAML above) and
confirm both routes work independently through the *same* Ingress
Controller/load balancer:
```bash
curl --resolve api.local:80:127.0.0.1 http://api.local/v1
```

**5. Inspect what the controller actually generated internally** — proving
it's not magic, just an nginx config reconciler:
```bash
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf | grep -A5 "whoami.local"
```

Cleanup:
```bash
kubectl delete -f whoami-ingress.yaml
```

## Debugging Techniques

- **`404` from the Ingress Controller itself (not your app)** — the
  request reached the controller but matched no rule; check `host` header
  and `path`/`pathType` exactly, and confirm `ingressClassName` matches an
  actually-installed controller.
- **Ingress object exists, `kubectl get ingress` shows no `ADDRESS`** —
  the Ingress Controller itself may not be running/healthy, or (on cloud)
  its `LoadBalancer` Service hasn't finished provisioning yet.
- **Works for one host but not a second added later** — Ingress
  Controllers often reload configuration asynchronously; check controller
  logs (`kubectl logs -n ingress-nginx deploy/ingress-nginx-controller`)
  for reload errors, and confirm the backend Service/Endpoints actually
  exist (Chapter 9's empty-Endpoints debugging applies identically here).
- **TLS certificate errors** — commonly a missing/misnamed `tls.secretName`
  Secret (Chapter 13, `kubernetes.io/tls` type) referenced in
  `spec.tls[]`, not shown above for simplicity but standard in real
  configs.

## Best Practices

- Run one (or a small, deliberate number of) Ingress Controller(s) per
  cluster rather than a `LoadBalancer` Service per microservice — this is
  the entire cost/manageability argument for using Ingress at all.
- Terminate TLS at the Ingress layer and manage certificates centrally
  (commonly automated via **cert-manager**, which itself watches Ingress
  objects and provisions/renews certs automatically — a real-world
  extension of the exact controller pattern this whole course is built
  on).
- Use `pathType: Exact` deliberately where ambiguity would be dangerous
  (e.g., don't let `/admin` accidentally prefix-match `/administrator`);
  default to `Prefix` only when that's genuinely the intended behavior.
- Keep provider-specific `annotations` documented in your team's runbooks
  — they're the least portable part of any Ingress manifest, since
  they're specific to whichever controller you've installed.

## Interview Questions

1. **What's the difference between a Service and an Ingress?**
   A Service load-balances traffic to Pods, typically for internal or
   simple external (NodePort/LoadBalancer) access; Ingress adds an HTTP
   -aware routing layer (host/path rules, TLS termination) in front of
   potentially many Services, multiplexed behind one entry point.
2. **Does creating an Ingress object alone route any traffic?**
   No — an Ingress object is inert without an installed Ingress
   Controller actually watching and acting on it; you must deploy one
   (ingress-nginx, cloud-specific controllers, etc.) separately.
3. **Why is Ingress cheaper/more manageable than one `LoadBalancer`
   Service per microservice?** One Ingress Controller, backed by one real
   external load balancer, can route arbitrarily many hostnames/paths to
   different internal Services — avoiding a separate billed cloud load
   balancer (and separate cert management) per service.
4. **Where does TLS termination typically happen with Ingress?**
   At the Ingress Controller itself, which decrypts client HTTPS and
   (usually) talks plain HTTP internally to backend Services — centralizing
   certificate management in one place.
5. **What are `ingressClassName` and Ingress annotations for?**
   `ingressClassName` selects which installed controller handles a given
   Ingress (a cluster can run several); annotations are a
   controller-specific escape hatch for behavior not covered by the
   generic Ingress spec, since the spec is a lowest-common-denominator
   across many different controller implementations.

## Mini Assignment

Deploy two separate applications (reuse `traefik/whoami` twice under
different Deployment/Service names) and configure one Ingress object
routing `app-a.local` to the first and `app-b.local` to the second, both
through the same installed ingress-nginx controller. Confirm both routes
work with `curl --resolve`, then intentionally break the routing (typo a
Service name in the Ingress backend) and use controller logs plus `kubectl
describe ingress` to find and fix it — no solution given here.

## Lesson Summary

- An Ingress is a declarative HTTP(S) routing rule set; it does nothing
  without a separately-installed Ingress Controller actually watching and
  implementing it — the same controller/reconciliation pattern as
  everything else in this course, applied to HTTP routing.
- Ingress lets one real external load balancer serve arbitrarily many
  internal Services via host/path rules, instead of one `LoadBalancer`
  Service (and one cloud load balancer bill) per service.
- TLS termination is typically centralized at the Ingress Controller.
- `ingressClassName` and provider-specific annotations bridge the generic
  Ingress spec to whichever concrete controller implementation you run.

---

### Before Chapter 17 (Health Checks) — tell me:

1. Why doesn't creating an Ingress object by itself do anything?
2. In your own words, what specifically makes Ingress cheaper than one
   `LoadBalancer` Service per microservice?
3. What was actually broken in the Mini Assignment's deliberate typo, and
   how did you find it — controller logs, `describe ingress`, or both?
