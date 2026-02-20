---
title: "Deploy a Shiny App with Helm (SSPCloud & EDITO)"
output: html_document
---

# Overview

This document explains how to deploy a Dockerized R Shiny application using the `template-shiny-deployment` Helm chart on:

- **SSPCloud**
- **EDITO (Onyxia)**

Only the `values.yaml` file changes. The chart itself should **not** be modified.

---

# Prerequisites

- A Docker image containing your Shiny app  
  (e.g. `ghcr.io/USER/IMAGE:tag`)
- Helm installed
- kubectl configured
- Access to SSPCloud or EDITO terminal

---

# Deployment on SSPCloud

## 1. Edit `values.yaml`

```yaml
shiny:
  image:
    repository: ghcr.io/USER/IMAGE
    tag: latest
    pullPolicy: Always

  ingress:
    enabled: true
    hostname: myapp.lab.sspcloud.fr
```

Make sure the hostname is unique.

---

## 2. Update Helm dependencies

```bash
helm dependency update ../template-shiny-deployment/
```

---

## 3. Deploy

```bash
helm upgrade --install my-shiny ../template-shiny-deployment -f values.yaml
```

---

## 4. Verify

```bash
kubectl get pods
kubectl get ingress
```

If the pod is `1/1 Running`, the app should be accessible at your hostname.

---

## 5. Remove the app

```bash
helm uninstall my-shiny
```

---

# Deployment on EDITO (Onyxia)

⚠️ EDITO requires:

- CPU & memory limits (ResourceQuota enforcement)
- Sometimes an Ingress patch

---

## 1. Edit `values_edito.yaml`

```yaml
shiny:
  image:
    repository: ghcr.io/USER/IMAGE
    tag: latest
    pullPolicy: Always

  ingress:
    enabled: true
    hostname: myapp.lab.dive.edito.eu

  resources:
    requests:
      cpu: 200m
      memory: 1Gi
    limits:
      cpu: "1"
      memory: 2Gi
```

Resource limits are mandatory on EDITO.

---

## 2. Update dependencies

```bash
helm dependency update ../template-shiny-deployment/
```

---

## 3. Deploy

```bash
REL=template-shiny-deployment-1771402299
NS=user-bastiengrassetird

helm upgrade $REL ../template-shiny-deployment -n $NS \
  --set-string shiny.resources.requests.cpu=500m \
  --set-string shiny.resources.requests.memory=2Gi \
  --set-string shiny.resources.limits.cpu=2 \
  --set-string shiny.resources.limits.memory=4Gi
```
---

## 4. Verify

```bash
kubectl get pods
kubectl get ingress
```

---

## 5. IMPORTANT — EDITO Ingress Patch

If your URL returns:

```
HTTP/2 404
```

Run:

```bash
kubectl annotate ingress "${REL}-shiny" kubernetes.io/ingress.class- --overwrite
```

Then test:

```bash
curl -I https://myapp.lab.dive.edito.eu
```

This patch allows EDITO’s default ingress controller to process your Ingress.

---

# Debug (Both Platforms)

### Check logs

```bash
kubectl logs <pod-name>
```

### Test without ingress

```bash
kubectl port-forward <pod-name> 3838:3838
```

Open:

```
http://localhost:3838
```

---

# Summary

| Platform   | Resource limits required | Ingress patch needed |
|------------|--------------------------|----------------------|
| SSPCloud   | No                       | No                   |
| EDITO      | Yes                      | Sometimes            |

---