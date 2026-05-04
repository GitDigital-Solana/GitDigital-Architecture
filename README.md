# Gitdigital Products — Architecture overview (MVP)
Purpose: AI PaaS + Model Marketplace

## Domains
- Frontend: gitdigital-frontend
- Auth: gitdigital-auth
- API Gateway: gitdigital-api-gateway
- User Service: gitdigital-user-service
- Model Registry: gitdigital-model-registry
- Model Storage: gitdigital-model-storage
- Sandbox: gitdigital-sandbox
- GPU Scheduler: gitdigital-gpu-scheduler
- Job Worker: gitdigital-job-worker
- Billing: gitdigital-billing
- Marketplace: gitdigital-marketplace
- Admin Portal: gitdigital-admin
- Ops / IaC: gitdigital-ops

## Key contracts
(See /openapi in each relevant repo)

## Roadmap (next 12 weeks)
- Sprint 0: repo creation + CI templates
- Sprint 1: auth + frontend skeleton
- Sprint 2: model registry + storage
- Sprint 3: sandbox prototype
- Sprint 4: scheduler + metering
- Sprint 5: billing + first user flow
- Sprint 6: marketplace + admin

## Security
- Image signing, seccomp, network egress deny, Vault, dependency scanning

## Contacts / Owners
- Tech lead: @<github RickCreator87>
- Backend lead: @<github RickCreator87>
- Sandbox engineer: @<github RickCreator87>
