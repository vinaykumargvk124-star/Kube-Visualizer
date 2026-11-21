# Kube Visualizer

An interactive, single-file Kubernetes learning tool. Watch requests travel through a cluster and deployments roll out pod by pod, styled like a terminal session.

No build step and no dependencies: open `index.html` in a browser. The only external request is to Google Fonts for JetBrains Mono.

## Views

### `traffic.flow`: follow a request through the cluster

Live request dots travel from the client through DNS, the load balancer, the node, the ingress controller and the Service to a pod. Click any box to inspect it, or any log line to replay its full hop-by-hop trace, including the headers the pod receives.

- **IFA vs non-IFA traffic, one app.** The same cluster, the same `shop` namespace and the same Deployments are reachable on two entry IPs:
  - **IFA IP** `34.120.8.10`: `shop.example.com` → public DNS → CDN/WAF → IFA load balancer → `nginx-ifa` ingress controller.
  - **non-IFA IP** `10.0.0.50`: `shop.corp.internal` → corporate DNS → private load balancer → `nginx-nonifa` ingress controller.
  - Each controller has its own LoadBalancer Service and Ingress object (`shop-ifa`, `shop-nonifa`). Both route to the same Services and pods. Each pod's label counts how many IFA and non-IFA requests it served.
- **externalTrafficPolicy `Cluster` vs `Local`**: SNAT, extra hops, load balancer health checks, and where the client IP is lost or kept (`X-Forwarded-For`, `X-Real-IP`).
- **Ingress routing**: host and path rules, TLS termination, default backend 404s, `limit-rps`, canary weight.
- **Services and EndpointSlices**: readiness, crashed pods still listed as endpoints (502), no endpoints (503).
- **Failure scenarios**: pod crash, node failure, WAF blocking a SQL injection, an internet client trying the non-IFA name (NXDOMAIN), rate limiting.

### `rollouts.yaml`: deployment strategies, pod by pod

Pick a strategy and deploy v2:

| Strategy | What it shows |
|---|---|
| Recreate | Every v1 pod goes first, so there is an outage until v2 is Ready |
| Rolling update | `maxSurge` and `maxUnavailable`; both versions serve at once; only readiness gates progress |
| Blue / green | Full green stack behind `web-preview`, smoke tests, instant switch of `web-active`, `scaleDownDelaySeconds` |
| Canary | `setWeight` 10 → 25 → 50 → 100 with error-rate analysis and automatic abort |
| A/B testing | Header-based routing (`X-Beta`) and a conversion comparison |
| Shadow | Traffic mirroring, where v2 responses are discarded so users are never affected |

You can make v2 **healthy**, **buggy (5xx on 30% of requests)**, or **failing its readiness probe**, then compare how each strategy protects users. Controls include step one tick, pause, `kubectl rollout undo` and reset. There are live charts, a Kubernetes-style event stream, and YAML for each strategy.

## Keyboard shortcuts

| Key | Action |
|---|---|
| `t` / `r` | Switch to traffic.flow / rollouts.yaml |
| `1`–`9` | Run a traffic scenario (traffic.flow) |
| `1`–`6` | Pick a strategy (rollouts.yaml) |
| `space` | Toggle live traffic / pause the rollout |
| `s`, `b`, `p` | Send and trace, burst ×15, flip externalTrafficPolicy |
| `d`, `n`, `u`, `x` | Deploy, step one tick, undo, reset |
| `/` | Filter the log |
| `?` | Help |

## Hosting on GitHub Pages

Settings → Pages → Deploy from a branch → `main` / root. The site is then served at `https://<user>.github.io/Kube-Visualizer/`.

## Note

The cluster is simulated: names, IPs, timings and error rates are illustrative. The routing behaviour follows how ingress-nginx, kube-proxy, Deployments and Argo Rollouts work.
