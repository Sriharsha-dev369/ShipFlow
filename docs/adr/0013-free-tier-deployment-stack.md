---
status: accepted
---

# Deployment stack: Vercel + Render + Neon + Upstash + Cloudflare R2 + Sentry

Production runs on Vercel (Next.js), Render (NestJS API + worker as separate services), Neon (Postgres), Upstash (Redis), Cloudflare R2 (object storage), and Sentry (error tracking) — all free-tier. We considered AWS end-to-end (S3, RDS, ECS) but rejected it: AWS S3/RDS free tiers are time-limited (12 months) rather than permanently free, and running a real ECS/RDS setup is a materially bigger operational surface than a student project needs. R2 specifically avoids S3's egress fees. Each of these is swappable independently later (they're accessed through their respective standard protocols/SDKs, not deeply coupled to the app), so this is a real but not especially painful choice to reverse piece by piece if a provider's free tier changes terms.
