# Persu Deployment (DeployKit)

مخزن رسمی استقرار خودکار سرویس‌های **Persu** (شامل Backend جنگو، Frontend نکست، PostgreSQL 17 و Redis 7) بر روی سرور با استفاده از Ansible و Docker.

---

## معماری

- **Backend (Django):** `ghcr.io/ehmoeini/persu-backend:latest` (`api.persu.fpna.ir`)
- **Frontend (Next.js):** `ghcr.io/ehmoeini/persu-frontend:latest` (`persu.fpna.ir`)
- **Database:** `postgres:17-alpine` (ایزوله روی شبکه داخلی با والیوم `postgres_data`)
- **Cache:** `redis:7-alpine` (ایزوله روی شبکه داخلی)

---