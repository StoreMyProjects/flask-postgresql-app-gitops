# Flask PostgreSQL App GitOps

A GitOps-based deployment infrastructure for a Flask application with PostgreSQL database using ArgoCD, Kustomize, and Argo Rollouts. DNS records are automatically managed by external-dns.

## Repository Structure

```
flask-postgresql-app-gitops/
├── bootstrap-app/                 # Bootstrap entry point
│   └── root.yaml                 # Root application (syncs from app-of-apps)
├── app-of-apps/                  # App of apps pattern
│   └── parent-app.yaml           # Parent application (manages child apps)
├── base/                         # Kustomize base configurations
│   ├── app/                      # Flask application manifests
│   │   ├── rollout.yaml          # Argo Rollout with canary strategy
│   │   ├── service.yaml          # Stable and canary services
│   │   ├── ingress.yaml          # ALB ingress with ACM certificate
│   │   ├── configmap.yaml        # Application configuration
│   │   ├── secrets.yaml          # Database credentials
│   │   ├── hpa.yaml              # Horizontal Pod Autoscaler
│   │   ├── pdb.yaml              # Pod Disruption Budget
│   │   ├── role.yaml             # RBAC role
│   │   ├── rolebinding.yaml      # Role binding
│   │   ├── serviceaccount.yaml   # Service account
│   │   └── kustomization.yaml    # Kustomize file for app
│   ├── db/                       # PostgreSQL database manifests
│   │   ├── statefulset.yaml      # PostgreSQL StatefulSet
│   │   ├── service.yaml          # Database service
│   │   ├── configmap.yaml        # Database initialization scripts
│   │   ├── secrets.yaml          # Database credentials
│   │   ├── storageclass.yaml     # Storage class for persistence
│   │   └── kustomization.yaml    # Kustomize file for database
│   ├── rollouts/                 # Argo Rollouts analysis templates
│   │   ├── analysis-template.yaml # Health check analysis
│   │   └── kustomization.yaml    # Kustomize file for rollouts
│   └── kustomization.yaml        # Root kustomization file
└── overlays/                     # Environment-specific configurations
    ├── dev/                      # Development environment
    │   ├── kustomization.yaml
    │   ├── dev-ns.yaml           # Development namespace
    │   ├── app/                  # Dev-specific app overrides
    │   ├── db/                   # Dev-specific db overrides
    │   └── rollouts/             # Dev-specific rollout overrides
    └── prod/                     # Production environment
        ├── kustomization.yaml
        ├── app/                  # Prod-specific app overrides
        ├── db/                   # Prod-specific db overrides
        └── rollouts/             # Prod-specific rollout overrides
```

## Prerequisites

- AWS EKS cluster
- ArgoCD installed (via Terraform scripts from separate repo)
- external-dns installed on EKS cluster
- kubectl configured
- Custom domain registered
- ACM certificate created in AWS
- Route 53 hosted zone configured

## Quick Start

### 1. Deploy Everything

Deploy the entire infrastructure with a single command:

```bash
kubectl apply -f bootstrap-app/root.yaml
```

This deploys:
- Root application (from bootstrap-app)
- Parent application (from app-of-apps)
- PostgreSQL database with persistent storage
- Flask application with canary rollouts
- RBAC configuration
- Secrets and ConfigMaps
- Ingress with ACM certificate
- HPA, PDB, and Analysis templates
- DNS records (automatically created by external-dns)

### 2. Verify Deployment

Check root application status:

```bash
kubectl get applications -n argocd
kubectl describe application root-app -n argocd
```

Check parent application:

```bash
kubectl describe application flask-app -n argocd
```

Check application pods:

```bash
kubectl get pods -n dev
kubectl get pods -n default
```

Check services:

```bash
kubectl get svc -n dev
kubectl get svc -n default
```

Check ingress:

```bash
kubectl get ingress
kubectl describe ingress two-tier-app-ingress
```

### 3. Verify DNS Records

external-dns automatically creates DNS records in Route 53 based on ingress definitions.

Verify DNS record was created:

```bash
# Check Route 53
aws route53 list-resource-record-sets --hosted-zone-id <zone-id>

# Test DNS resolution
nslookup your-custom-domain.com
dig your-custom-domain.com

# Verify from within cluster
kubectl run -it --rm debug --image=alpine -- nslookup your-custom-domain.com
```

Wait for DNS propagation (can take a few minutes).

### 4. Access Application

Once DNS is created and propagated:

```bash
curl https://your-custom-domain.com
```

Verify SSL certificate:

```bash
openssl s_client -connect your-custom-domain.com:443
```

## Configuration

### Update ACM Certificate

Edit `base/app/ingress.yaml`:

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<region>:<account-id>:certificate/<cert-id>
```

### Update Custom Domain

Edit `base/app/ingress.yaml`:

```yaml
spec:
  rules:
    - host: your-custom-domain.com
```

external-dns will automatically update Route 53 records when the ingress host changes.

### Modify Replica Count

Edit `base/app/rollout.yaml`:

```yaml
spec:
  replicas: 2  # Change as needed
```

**Note:** Do NOT override replicas in overlay kustomization files.

### Update Database Credentials

Edit `base/db/secrets.yaml`:

```yaml
stringData:
  POSTGRES_PASSWORD: <new-password>
  POSTGRES_USER: <new-username>
```

### Configure Environment Variables

Edit `base/app/configmap.yaml` for application environment variables.

### Adjust Resource Limits

Edit `base/app/rollout.yaml`:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "300m"
    memory: "256Mi"
```

## Environment Customization

### Development Environment

Overlays in `overlays/dev/` override base configurations for development.

### Production Environment

Overlays in `overlays/prod/` override base configurations for production.

To switch environments, update `app-of-apps/parent-app.yaml`:

```yaml
source:
  path: overlays/prod  # Change from dev to prod
```

## Monitoring & Verification

### Check ArgoCD Sync Status

```bash
argocd app get root-app -n argocd
argocd app get flask-app -n argocd
```

### Check Rollout Status

```bash
kubectl get rollouts -n dev
kubectl describe rollout two-tier-app -n dev
```

### View Pod Events

```bash
kubectl get events -n dev --sort-by='.lastTimestamp'
```

### Check Database Logs

```bash
kubectl logs -n dev -l app=postgresql
```

### Check Application Logs

```bash
kubectl logs -n dev -l app=two-tier-app
```

### Monitor Canary Deployment

```bash
kubectl get analysisrun -n dev
kubectl describe analysisrun <name> -n dev
```

### Check external-dns Logs

```bash
kubectl logs -n kube-system -l app=external-dns -f
```

### Check DNS Records in Route 53

```bash
aws route53 list-resource-record-sets --hosted-zone-id <zone-id> --query "ResourceRecordSets[?Name=='your-custom-domain.com.']"
```

## Troubleshooting

### DNS Record Not Created

Check external-dns logs:

```bash
kubectl logs -n kube-system -l app=external-dns
```

Ensure:
- external-dns is running
- Route 53 hosted zone is configured
- Ingress has proper annotations

### DNS Not Resolving

```bash
nslookup your-custom-domain.com
dig your-custom-domain.com

# Check from within cluster
kubectl run -it --rm debug --image=alpine -- nslookup your-custom-domain.com
```

May take up to 5-10 minutes for DNS propagation.

### Ingress Not Showing IP Address

```bash
kubectl describe ingress two-tier-app-ingress
kubectl get ingress two-tier-app-ingress -w
```

Check ALB controller logs if ingress is not getting an address.

### Certificate Not Valid

Verify ACM certificate ARN in `base/app/ingress.yaml` is correct.

### Application Pod Fails to Start

```bash
kubectl describe pod <pod-name> -n dev
kubectl logs <pod-name> -n dev
```

### Database Connection Issues

```bash
kubectl exec -it <postgres-pod> -n dev -- psql -U postgres
```

### Rollout Stuck in Canary

```bash
kubectl describe rollout two-tier-app -n dev
kubectl get analysisrun -n dev
```

Check analysis template logs for health check failures.

### ArgoCD Sync Failures

```bash
argocd app sync root-app -n argocd
argocd app logs root-app -n argocd
```

## Updates & Rollbacks

### Update Application Image

Edit `base/app/rollout.yaml`:

```yaml
containers:
  - name: two-tier-app
    image: <new-image-uri>
```

Commit changes. ArgoCD will automatically sync and perform canary rollout.

### Rollback to Previous Version

```bash
kubectl undo rollout two-tier-app -n dev
```

### Manual Promotion

If canary analysis fails:

```bash
kubectl patch rollout two-tier-app -n dev --type merge -p '{"status":{"currentStepIndex":10}}'
```

## Terraform Infrastructure

The ArgoCD server is installed using Terraform scripts from a separate repository. 

Prerequisites:
- Terraform configured with AWS credentials
- EKS cluster created
- Terraform state storage configured

The Terraform deployment handles:
- ArgoCD installation
- Repository configuration
- RBAC setup
- Initial application registration
- external-dns installation and configuration

## DNS Management

DNS records are managed automatically by external-dns:

- **What it does**: Watches Kubernetes ingress resources and automatically creates/updates DNS records in Route 53
- **Automatic**: No manual Route 53 updates needed
- **Synced**: Deleting ingress will automatically remove DNS records
- **Configuration**: external-dns is installed and configured via Terraform

## Support & Documentation

For additional information:
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [Kustomize Documentation](https://kustomize.io/)
- [AWS ALB Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [external-dns Documentation](https://github.com/kubernetes-sigs/external-dns)

## License

MIT
