# DevOps Deployment Procedure

**Document Type:** Deployment Runbook
**Author:** George Amankwaa Sarpong
**Last Updated:** June 2026

---

## Purpose
Standardised procedure for deploying containerised microservices to Kubernetes using CI/CD pipelines ensuring consistent, safe, and well-documented deployments.

---

## Deployment Flow
---

## Pre-Deployment Checklist
- [ ] Code review approved by minimum 2 reviewers
- [ ] All CI/CD pipeline stages passing
- [ ] Security scan shows no CRITICAL vulnerabilities
- [ ] Staging deployment validated
- [ ] Change request approved
- [ ] Rollback procedure confirmed
- [ ] Monitoring dashboards open

---

## Deployment Steps

### Step 1 — Trigger Pipeline
1. Merge approved pull request to main branch
2. GitHub Actions pipeline triggers automatically
3. Monitor pipeline progress in GitHub Actions tab

### Step 2 — Monitor Build Stage
1. Docker image built successfully
2. Unit tests all passing
3. Security scan completed — no critical vulnerabilities
4. Image pushed to ECR with correct tag

### Step 3 — Staging Deployment
1. Deployment to staging namespace automatic
2. kubectl rollout status shows successful
3. Run integration test suite
4. Validate application on staging URL

### Step 4 — Production Approval
1. Review staging test results
2. Approve production deployment in GitHub
3. Monitor production deployment rollout
4. Verify kubectl rollout status complete

### Step 5 — Post-Deployment Validation
1. Check all pods running and healthy
2. Verify no CrashLoopBackOff pods
3. Test critical application functions
4. Check Grafana dashboards for anomalies
5. Verify Prometheus alerts not firing
6. Monitor error rate for 15 minutes

---

## Rollback Procedure

### Immediate Rollback
```bash
# Rollback to previous deployment
kubectl rollout undo deployment/erp-app -n production

# Check rollback status
kubectl rollout status deployment/erp-app -n production

# Verify pods running previous version
kubectl get pods -n production -l app=erp-app
```

### Rollback Triggers
- Application unavailable after deployment
- Error rate above 5%
- Response time above 3 seconds
- Critical alerts firing in Prometheus
- CrashLoopBackOff detected

---

## DORA Metrics Targets

| Metric | Target |
|---|---|
| Deployment Frequency | Multiple per day |
| Lead Time for Changes | Below 1 hour |
| Change Failure Rate | Below 5% |
| Mean Time to Recovery | Below 1 hour |
