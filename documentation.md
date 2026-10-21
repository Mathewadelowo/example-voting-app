
## 1. Project Overview

This project deploys the Example Voting App to Amazon EKS using Terraform, Docker, Kubernetes, Helm, and GitHub Actions. It demonstrates infrastructure as code, container security, automated build-and-deploy workflows, and recovery from a deliberately bad release.

## 2. Application Architecture

The application contains five components:

- **Vote:** Web interface for submitting votes.
- **Redis:** Temporarily queues votes.
- **Worker:** Reads votes from Redis and writes them to PostgreSQL.
- **PostgreSQL (`db`):** Stores vote data.
- **Result:** Web interface displaying vote totals.

```text
User ──> Vote Service ──> Redis ──> Worker ──> PostgreSQL
  └──────────────────────────────────────────> Result Service
```

The Vote and Result services are exposed through AWS load balancers. Redis and PostgreSQL use internal Kubernetes services.

## 3. Container Security

The service Dockerfiles include the following security measures, as documented in the repository:

- Base images pinned to SHA-256 digests where specified.
- Non-root execution for Vote, Result, the Result test container, and Worker.
- Python dependencies pinned in `vote/requirements.txt`.
- Overrides for selected transitive Node.js dependencies in `result/package.json`.
- Multi-stage build for the Worker to keep SDK tooling out of its final runtime image.
- Trivy scanning in the GitHub Actions pipeline.

Key files include `vote/Dockerfile`, `result/Dockerfile`, `result/tests/Dockerfile`, `worker/Dockerfile`, and the dependency files. Verify the current repository and latest scan results before claiming all security checks pass.


Architecture Flow Diagram:

```text

[ Voters / Public Browser ]
            │
            ▼ (AWS Load Balancer)
     ┌─────────────┐
     │  Vote (UI)  │ ──(Submits Vote)──> [ Redis Queue ]
     └─────────────┘                            │
                                                ▼
                                         ┌─────────────┐
                                         │ Worker Pod  │ (Processes & saves)
                                         └─────────────┘
                                                │
                                                ▼
[ Viewers / Public Browser ]              ┌─────────────┐
            │                             │ PostgreSQL  │
            ▼ (AWS Load Balancer)         │  Database   │
     ┌─────────────┐                      └─────────────┘
     │ Result (UI) │ ───(Reads Tallies)─────────┘
     └─────────────┘



## 4. Infrastructure with Terraform

Terraform configuration is located in `terraform-eks/` and provisions the EKS infrastructure in `us-east-1`.

Main configuration files:

| File | Purpose |
|---|---|
| `providers.tf` | Terraform and AWS provider configuration |
| `variables.tf` | Configurable region, cluster name, instance type, and Kubernetes version |
| `local.tf` | Derived local values and subnet calculations |
| `iam.tf` | IAM roles and policies for EKS and worker nodes |
| `eks.tf` | Network and EKS resources |
| `outputs.tf` | Useful infrastructure outputs |

Typical commands:

```bash
terraform-eks init
terraform-eks validate
terraform-eks plan
terraform-eks apply
```

Configure `kubectl` using the actual cluster name:

```bash
aws eks --region us-east-1 update-kubeconfig --name <YOUR_CLUSTER_NAME>
kubectl get nodes
```

## 5. GitHub Actions CI/CD

The workflows are under `.github/workflows/`:

- `call-docker-build-vote.yaml`
- `call-docker-build-result.yaml`
- `call-docker-build-worker.yaml`
- `deploy-to-eks.yaml` (reusable deployment workflow)

The configured pipeline performs service-specific tests where defined, Helm linting, Trivy source scanning, image builds, image scanning, Docker Hub publishing on pushes to `main`, and deployment to EKS. Images use the Git commit SHA as the tag for traceability.

The reusable workflow selects the image repository and Kubernetes Deployment for the requested service, runs `helm upgrade --install`, and checks rollout status. The Vote Deployment is named `votes` (plural).

Required credentials and configuration are stored as GitHub Actions secrets, including Docker Hub credentials and AWS/EKS settings. Never commit secret values to the repository. Only report workflow stages as successful when the corresponding run confirms they passed.

## 6. Helm Deployment

The chart is located at `helm/voting-app/` and uses one `values.yaml` for the application services. The Helm release is `voting-app` in the `voting-app` namespace.

Inspect the release and cluster with:

```bash
helm history voting-app -n voting-app
helm get values voting-app -n voting-app
kubectl get deployments,pods,services -n voting-app
```

The Vote and Result Kubernetes Services use `type: LoadBalancer`; Redis and PostgreSQL are internal services. AWS load-balancer hostnames can change, so retrieve current endpoints with:

```bash
kubectl get services -n voting-app
```

Test the Vote page by submitting a vote, then confirm the Result page reflects the updated total.

## 7. Bad Release and Helm Rollback

To demonstrate rollback, an intentionally invalid Vote image tag was deployed:

```bash
helm upgrade voting-app ./helm/voting-app \\
  --namespace voting-app \\
  --reuse-values \\
  --set-string vote.image.tag=bad-tag-999999 \\
  --wait \\
  --timeout 2m
```

The Vote Deployment was confirmed to reference:

$ kubectl get deployment votes -n voting-app \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
devsmaus/voting-app-vote:bad-tag-999999

```text
devsmaus/voting-app-vote:bad-tag-999999
```

Helm history later showed failed upgrades at revisions 17 and 18. The recorded Helm error mentioned a client rate-limiter/context-deadline timeout. The bad image tag was separately confirmed on the Deployment.

Revision 16 was the last revision marked `deployed` in the history captured during the test. After rollback, the Vote Deployment was observed using the following valid SHA-tagged image instead of the bad tag:

````bash
 helm rollback voting-app 16 \
  --namespace voting-app \
  --wait \
  --timeout 5m
Rollback was a success! Happy Helming!
````


```text
devsmaus/voting-app-vote:854efe36e8dac321aff402f561cd6db4b4423f9d
```

Verify the final state with:

```bash
helm history voting-app -n voting-app

kubectl get deployment votes -n voting-app \\
  -o jsonpath='{.spec.template.spec.containers[*].image}'
echo

kubectl rollout status deployment/votes -n voting-app
kubectl get deployments,pods,services -n voting-app
```

If another rollback is needed, inspect the current Helm history and select the correct known-good revision before running:

```bash
helm rollback voting-app <KNOWN_GOOD_REVISION> \\
  --namespace voting-app \\
  --wait \\
  --timeout 5m
```

A rollback creates a new Helm revision; it does not erase the failed revisions. Consider the recovery complete only when the rollout succeeds, all expected replicas are ready, pods are healthy, and the Vote and Result pages work. Run failure tests only in a test environment because they can interrupt the Vote service.


**Recommended demo evidence:** a successful CI/CD run, Terraform output, `kubectl get nodes`, final deployment/pod/service status, Helm history showing the failed release and rollback, Trivy results, and screenshots of the working Vote and Result pages.

## 9. Conclusion

This project demonstrates deployment of a multi-service application to AWS EKS using Terraform, Docker, Kubernetes, Helm, and GitHub Actions. The deliberately bad image release and rollback exercise demonstrate release recovery. Final completion should be supported by current pipeline, infrastructure, security, and post-rollback application evidence.
"""
