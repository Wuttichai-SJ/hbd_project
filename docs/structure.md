# โครงสร้างโปรเจกต์ Birthday App

สัญลักษณ์: 🤖 = AI ทำให้ (Frontend) · 🧑‍💻 = เจ้าของฝึกเขียนเอง

```
birthday-app/
├── CLAUDE.md                     # กติกาให้ Claude
├── .claude/skills/               # skill ประจำโปรเจกต์
│   ├── frontend-builder/  api-endpoint/  db-change/
│   └── cicd-deploy/       observability/  code-review/
│
├── .github/workflows/            🧑‍💻 CI/CD
│   ├── ci.yml                    # lint → typecheck → test → build
│   ├── e2e.yml                   # Playwright บน preview URL
│   └── migrate.yml               # migrate DB เมื่อ merge main
│
├── src/
│   ├── app/
│   │   ├── (auth)/               🤖 login, register
│   │   ├── (main)/               🤖 หน้า main, profile/[id], poll/[id], poll/new
│   │   ├── api/                  🧑‍💻 Route handlers
│   │   │   ├── birthdays/today/route.ts
│   │   │   ├── users/[id]/route.ts
│   │   │   ├── wishes/route.ts
│   │   │   ├── polls/route.ts
│   │   │   ├── polls/[id]/route.ts
│   │   │   ├── polls/[id]/comments/route.ts
│   │   │   ├── uploads/sign/route.ts
│   │   │   ├── health/route.ts
│   │   │   └── cron/cleanup/route.ts
│   │   ├── layout.tsx            🤖
│   │   └── global-error.tsx      🤖 (ส่ง error ไป Sentry)
│   │
│   ├── components/               🤖 ui/, birthday/, poll/, wish/
│   ├── hooks/                    🤖 useLiveComments, useBlowCandle
│   ├── styles/                   🤖 globals.css (design tokens)
│   ├── lib/                      🤖 api-client.ts, mocks/
│   │
│   └── server/                   🧑‍💻 Backend ทั้งหมด
│       ├── db/
│       │   ├── client.ts         # Drizzle connection
│       │   ├── schema/           # users, polls, poll_comments, wishes
│       │   └── seed.ts
│       ├── repositories/         # query DB อย่างเดียว
│       ├── services/             # business logic
│       ├── validators/           # Zod schemas (ใช้ร่วมกับ frontend)
│       └── lib/
│           ├── errors.ts         # AppError + handleRoute()
│           ├── time.ts           # bangkokToday(), nextBangkokMidnight()
│           ├── auth.ts           # ดึง user จาก Supabase session
│           ├── rate-limit.ts
│           └── logger.ts         # pino + requestId
│
├── drizzle/migrations/           🧑‍💻 SQL migrations (generate แล้วอ่านทุกไฟล์)
├── supabase/
│   ├── policies/                 🧑‍💻 RLS ต่อตาราง
│   └── storage.sql               🧑‍💻 bucket + policy รูปโพล
│
├── tests/
│   ├── unit/                     🧑‍💻 time.ts, services
│   ├── integration/api/          🧑‍💻 ยิง route handler กับ test DB
│   └── e2e/                      🤖 Playwright flows
│
├── docs/
│   ├── structure.md              # ไฟล์นี้
│   ├── api.md                    🧑‍💻 API contract (เขียนก่อนโค้ด)
│   ├── adr/                      🧑‍💻 บันทึกการตัดสินใจ เช่น 001-postgres-over-nosql.md
│   └── postmortems/              🧑‍💻 เมื่อ production มีปัญหา
│
├── instrumentation.ts            🧑‍💻 Sentry server init
├── sentry.client.config.ts       🧑‍💻
├── drizzle.config.ts             🧑‍💻
├── vercel.json                   🧑‍💻 cron: "0 17 * * *" (เที่ยงคืนไทย)
├── .env.example
├── next.config.ts · tsconfig.json · package.json
└── README.md                     # หน้าโชว์ portfolio: demo, architecture, สิ่งที่ได้เรียน
```

## Flow ของ request

```
ผู้ใช้ (มือถือ)
   │
   ▼
Vercel Edge/CDN ──▶ Next.js page (🤖)
                        │ fetch ผ่าน api-client
                        ▼
                  /api/... route handler ──▶ validator (Zod)
                        │                        │
                        ▼                        ▼
                     service  ◀──────────── ข้อมูลที่ตรวจแล้ว
                        │
                        ▼
                   repository ──▶ Supabase Postgres (RLS)
                        │
                        └─ log (pino → Better Stack) · error (Sentry)

คอมเมนต์ใหม่ ──▶ Postgres ──▶ Supabase Realtime ──▶ หน้าจอทุกคนที่เปิดโพลอยู่
เที่ยงคืนไทย ──▶ Vercel Cron ──▶ /api/cron/cleanup ──▶ ลบโพล + รูปที่หมดอายุ
```

## Flow การทำงาน (Dev → Production)

```
issue (GitHub Projects)
  → branch feat/xxx
  → เขียน docs/api.md → schema/migration → repo → service → route → test
  → ขอ Claude review (skill code-review)
  → เปิด PR ──▶ ci.yml ✅ ──▶ Vercel Preview ──▶ e2e.yml ✅
  → merge main ──▶ migrate.yml ──▶ Vercel Production ──▶ Sentry release
  → ดู Sentry / Better Stack / PostHog
```

## ลำดับการลงมือ (แนะนำ)

1. Setup: Next.js + Supabase project + Drizzle + ci.yml ตัวแรก + deploy หน้าเปล่าขึ้น Vercel
2. Database: schema 4 ตาราง + RLS + seed
3. API: birthdays/today → wishes → polls → comments → uploads
4. Frontend (🤖) ต่อกับ API ทีละหน้า
5. Monitoring: health → logger → Sentry → uptime → PostHog
6. Cron cleanup + e2e + migrate.yml
7. ฟีเจอร์ว้าว + README portfolio
