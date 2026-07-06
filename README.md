# ArgoCD Bootstrap

## Prerequisites
- EKS cluster running
- `kubectl` configured to point to the cluster
- `helm` v4 installed
- IAM role for AWS Load Balancer Controller with the following permissions:
  - `elasticloadbalancing:RegisterTargets`
  - `elasticloadbalancing:DeregisterTargets`
  - `elasticloadbalancing:DescribeTargetGroups`
  - `elasticloadbalancing:DescribeTargetHealth`
  - `ec2:DescribeInstances`
  - `ec2:DescribeNetworkInterfaces`
  - `ec2:DescribeVpcs`
  - `ec2:DescribeSubnets`
  - `ec2:DescribeSecurityGroups`
- IAM role for Fluent Bit with the following permissions:
  - `logs:CreateLogGroup`
  - `logs:CreateLogStream`
  - `logs:PutLogEvents`
  - `logs:DescribeLogStreams`
  - `logs:DescribeLogGroups`
  - `logs:PutRetentionPolicy`
- SSH deploy key (or other SSH key with read access) for the git repo(s) ArgoCD will sync from

> **Note:** Karpenter must be installed manually before bootstrapping ArgoCD.

## Bootstrap Steps

### 1. Install Karpenter
```bash
helm upgrade --install karpenter ./karpenter -f karpenter/<env>.values.yaml
```

### 2. Create namespace and repo secret
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
spec:
  finalizers:
    - kubernetes
---
apiVersion: v1
kind: Secret
metadata:
  name: <repo-name>
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:<org>/<repo>.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```


### 3. Create Alertmanager Slack secret
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  slack-webhook-url: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### 4. Create incode-backend database secret
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: incode-backend
---
apiVersion: v1
kind: Secret
metadata:
  name: incode-backend-db
  namespace: incode-backend
stringData:
  DATABASE_URL: postgres://<user>:<password>@<host>:5432/<db>
```

### 5. Install ArgoCD
```bash
helm dependency update ./argocd
helm upgrade --install argocd ./argocd -f argocd/<env>.values.yaml -n argocd
```

### 6. Install ArgoCD apps
```bash
helm install argocd-apps ./argocd-apps -f argocd-apps/<env>.values.yaml -n argocd
```

ArgoCD will now manage all applications from the repo.

> **Note:** The ArgoCD `IngressRoute` (`argocd/templates/ingress.yaml`) is commented out. On a first install, Traefik (and its CRDs) don't exist yet — ArgoCD (wave 1) is deployed before Traefik (wave 3), so the `IngressRoute` would fail to apply. Once Traefik is up, uncomment it and push to git; the self-managed `argocd` Application will pick it up on its next sync. Until then, access ArgoCD via `kubectl port-forward` (see below).

## Application Dependency Tree

> **Note:** Application order of sync is regulated by sync wave (`argocd.argoproj.io/sync-wave` annotation). Lower wave number syncs first.


```
Karpenter (Wave 0)
│   Provisions nodes — must be running before anything else
│
└── ArgoCD (Wave 1)
│   GitOps controller — manages all applications below
│
└── AWS Load Balancer Controller (Wave 2)
│   Installs TargetGroupBinding CRD
│   Registers/deregisters Traefik pod IPs in AWS Target Group
│
└── Traefik (Wave 3)
│   DaemonSet — one pod per node
│   Exposed via TargetGroupBinding → existing Terraform ALB
│   Handles all L7 routing inside the cluster
│   Requires: AWS Load Balancer Controller (TargetGroupBinding CRD)
│
└── Fluent Bit (Wave 4)
│   Ships container logs to CloudWatch
│   Organized by: /eks/vortex-cluster-dev/<namespace>/<container>
│   Independent — no dependency on Traefik or LB Controller
│
└── Your Applications (Wave 5+)
    Web apps, APIs, workers etc.
    Routing handled by Traefik IngressRoute
    Logs automatically collected by Fluent Bit
    Requires: Traefik (routing), Fluent Bit (logs)
```

### Traffic Flow
```
Browser → Cloudflare (HTTPS) → ALB 443 (ACM SSL termination) → Traefik 8000 (HTTP) → Services
```

### Node Scaling Flow
```
Karpenter provisions node → Traefik DaemonSet pod starts → App pods start → LB Controller registers Traefik pod IP in Target Group
```

## Accessing ArgoCD

```bash
kubectl port-forward svc/argocd-server 8080:80 -n argocd
```

Open `http://localhost:8080` and login with:
- **Username:** `admin`
- **Password:**
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

> **Note:** Once ArgoCD is set up on a new environment, generate an API token for the `github-actions` account — this token is used by GitHub Actions to sync deployments:
> ```bash
> argocd login <argocd-host> --grpc-web
> argocd account generate-token --account github-actions --grpc-web
> ```
> Store the output as `ARGOCD_TOKEN` in your GitHub repository secrets.

