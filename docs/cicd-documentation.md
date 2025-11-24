# CI/CD Pipeline

## How It Works

Push code to `dev` branch → GitHub Actions runs automatically → Deploys to production

```mermaid
flowchart LR
    A[Push to dev] --> B[Tests Run]
    B --> C[Build]
    C --> D[Deploy]
    
    style D fill:#6f6
```

---

## Backend (Django)

**File:** `.github/workflows/deploy.yml`

### Steps
1. Run tests with PostgreSQL
2. Build Docker image
3. Push to Docker Hub
4. SSH to EC2 and deploy

### Commands
```bash
# Test
python manage.py test

# Deploy (automatic on push)
docker run -d --name accipere-backend -p 8000:8000 username/image:tag
```

---

## Frontend (React)

**File:** `.github/workflows/deploy-s3.yml`

### Steps
1. Build React app
2. Upload to S3
3. Clear CloudFront cache

### Commands
```bash
# Build
npm run build

# Deploy (automatic on push)
aws s3 sync build/ s3://bucket-name --delete
```

---

## Security Scan

**File:** `.github/workflows/trivy-scan.yml`

Scans for vulnerabilities on `main` branch. Results in GitHub Security tab.

---

## GitHub Secrets

**Backend:**
- `DJANGO_SECRET_KEY`
- `DOCKER_USERNAME`, `DOCKER_PASSWORD`
- `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`

**Frontend:**
- `VITE_API_BASE_URL`
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- `S3_BUCKET`, `CLOUDFRONT_DISTRIBUTION_ID`

Add in: Settings → Secrets → Actions

---

## Common Issues

| Problem | Fix |
|---------|-----|
| Tests fail | Check logs in Actions tab |
| Docker build fails | Check `requirements.txt` |
| EC2 deploy fails | Verify SSH key |
| S3 upload fails | Check AWS credentials |

---

## Rollback

```bash
# Backend - SSH to EC2
docker stop accipere-backend
docker run -d --name accipere-backend -p 8000:8000 username/image:old-tag

# Frontend - Revert commit
git revert HEAD
git push origin dev
```

---

## Logs

```bash
# GitHub Actions
Go to Actions tab → Click workflow

# Backend logs
docker logs accipere-backend

# Security scans
Go to Security tab
```

---

## Cost

| Service | Monthly |
|---------|---------|
| EC2 | ₹600-800 |
| RDS | ₹1200-1600 |
| S3+CloudFront | ₹80-400 |
| **Total** | **₹2000-3000** |

---

## Summary

- Push to `dev` = auto deploy
- Tests must pass first
- Takes 5-10 minutes
- Check Actions tab for status
- Rollback = revert commit or redeploy old image

That's it! 🚀
