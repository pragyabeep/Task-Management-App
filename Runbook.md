# TaskFlow Operations Runbook

## 1. How to Deploy

\*\*Automated via CI/CD:\*\* Push to `develop` → triggers staging deploy automatically.
Push to `main` → triggers production deploy after manual approval in GitHub → Environments → production.

\*\*Manual deployment:\*\*
ssh -i taskflow-keypair.pem ubuntu@<EC2\_IP>
cd /home/ubuntu/Task-Management-App
git pull origin main
/home/ubuntu/fetch-secrets.sh prod
docker compose -f docker-compose.prod.yml --env-file .env.prod -p taskflow-prod up -d --build

## 2. How to Rollback

If a deployment fails or the application behaves unexpectedly after a deploy:

ssh -i taskflow-keypair.pem ubuntu@<EC2\_IP>
cd /home/ubuntu/Task-Management-App

# Check what commit was last known-good
cat /tmp/rollback\_commit.txt

# Reset to that commit and redeploy
git checkout $(cat /tmp/rollback\_commit.txt)
/home/ubuntu/fetch-secrets.sh prod
docker compose -f docker-compose.prod.yml --env-file .env.prod -p taskflow-prod up -d --build

## 3. How to Check Logs

# Live logs from the backend
docker logs -f taskflow-backend-prod

# Last 100 lines from each container
docker logs --tail 100 taskflow-backend-prod
docker logs --tail 100 taskflow-frontend-prod
docker logs --tail 100 taskflow-mongo-prod

# Via CloudWatch Console: CloudWatch → Log groups → /taskflow/docker/containers

## 4. How to Restart Services

# Restart a single container without downtime
docker restart taskflow-backend-prod

# Restart entire production stack
docker compose -f docker-compose.prod.yml -p taskflow-prod restart

# Hard restart (stops all containers, then starts fresh — use when env vars changed)
docker compose -f docker-compose.prod.yml --env-file .env.prod -p taskflow-prod down
docker compose -f docker-compose.prod.yml --env-file .env.prod -p taskflow-prod up -d

## 5. How to Verify Application Health

# Check all containers show "running" status
docker compose -p taskflow-prod ps

# Backend health endpoint (should return 200)
curl http://localhost:5000/health

# Frontend (should return 200)
curl -o /dev/null -s -w "HTTP status: %{http\_code}\\n" http://localhost:80

# Database connectivity from the backend container
docker exec taskflow-backend-prod sh -c \\
  "node -e \\"require('mongoose').connect(process.env.MONGODB\_URI).then(()=>console.log('DB OK')).catch(e=>console.error('DB FAIL',e.message))\\""

# CloudWatch: open the TaskFlow-Overview dashboard and verify all metrics are in normal range
```
