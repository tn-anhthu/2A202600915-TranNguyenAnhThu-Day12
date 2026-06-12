# Deployment Information

## Public URL
https://day8-deploy-production.up.railway.app

## Platform
Railway (Dockerfile builder)

## App Description
RAG Chatbot — Pháp luật Ma tuý Việt Nam
Containerized from Day 08 repo: https://github.com/tn-anhthu/2A202600915-TranNguyenAnhThu-Day08

## Test Commands

### Health Check
```bash
curl https://day8-deploy-production.up.railway.app/_stcore/health
# Expected: ok
```

### App UI
```bash
open https://day8-deploy-production.up.railway.app
```

## Environment Variables Set
- `FIREWORKS_API_KEY` — Fireworks AI (DeepSeek-v4-pro generation)
- `PAGEINDEX_API_KEY` — PageIndex vectorless fallback
- `PORT` — injected automatically by Railway

## Screenshots
- [Deployment dashboard](screenshots/day8/day8-dashboard.png)
- [Service running](screenshots/day8/day8-running.png)
- [Test results](screenshots/day8/day8-test.png)
