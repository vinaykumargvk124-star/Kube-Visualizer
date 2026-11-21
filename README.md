# Kube Visualizer

An interactive, single-file Kubernetes learning tool. Watch requests travel through a cluster and deployments roll out pod by pod, styled like a terminal session.

No build step and no dependencies: open `index.html` in a browser. The only external request is to Google Fonts for JetBrains Mono.

## Views

### `traffic.flow`: one namespace, many clusters, two entry IPs

The app lives in **one namespace, `shop`**, deployed identically to **two clusters** (`cluster-east`, `cluster-west`). Each cluster has several pods per Deployment, spread across its nodes. Live request dots travel through the whole path. Click any box to inspect it, or any log line to replay its hop-by-hop trace, including the headers the pod receives.

- **Every cluster has an IFA IP and a non-IFA IP** for the same app:
  - **IFA** (`shop.example.com`): public DNS → CDN/WAF, which picks a healthy cluster from its origin pool → that cluster's IFA IP (`34.120.8.10` east, `34.120.9.20` west) → `ingress-ifa` pods.
  - **non-IFA** (`shop.corp.internal`): corporate GSLB DNS, which answers with a healthy cluster's non-IFA IP (`10.0.0.50` east, `10.8.0.50` west) → `ingress-nonifa` pods.
  - Both ingress controllers run inside namespace `shop`, and their Ingress objects (`shop-ifa`, `shop-nonifa`) point at the same Services, so both kinds of traffic reach the same pods. Each pod's label counts the IFA and non-IFA requests it served.
- **The namespace is drawn as the main container.** Nodes are a strip underneath to show that the namespace spans all of them. Click the namespace for a `kubectl get deploy,svc,ingress -n shop` view of either cluster.
- **externalTrafficPolicy `Cluster` vs `Local`**: SNAT, extra hops, LB health checks, and where the client IP is lost or kept.
- **Scenarios**: one namespace across clusters, the same app on two IPs, client IP lost to SNAT, an internet client trying the non-IFA name (NXDOMAIN), the WAF blocking SQL injection, **cluster failover**, node failure, a pod crash (502 until readiness fails), and a 30% canary.

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
