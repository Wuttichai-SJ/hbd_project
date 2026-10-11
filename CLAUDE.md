# CLAUDE.md — Birthday App 🎂

เว็บอวยพรวันเกิด: ผู้ใช้ลงทะเบียน → หน้า main แสดงคนที่เกิดวันนี้ → กดอวยพรได้ → เจ้าของวันเกิดสร้างโพล (หัวข้อ, คำอธิษฐาน, รูป, คอมเมนต์) ที่หายไปเมื่อขึ้นวันใหม่ตามเวลาไทย

โปรเจกต์นี้เป็น **โปรเจกต์ฝึกฝน + portfolio** เจ้าของต้องการฝึก API, Database, CI/CD, Deploy, Monitoring ด้วยตัวเอง ส่วน Frontend ให้ AI ทำทั้งหมด

---

## โหมดการทำงาน (สำคัญที่สุด)

| ส่วน | โฟลเดอร์ | บทบาทของ Claude |
|---|---|---|
| Frontend / UI | `src/app/(auth)`, `src/app/(main)`, `src/components`, `src/hooks`, `src/styles` | **Builder** — เขียนให้ครบ พร้อมใช้งาน |
| API | `src/app/api`, `src/server/services`, `src/server/validators` | **Mentor** |
| Database | `src/server/db`, `src/server/repositories`, `drizzle/`, `supabase/` | **Mentor** |
| CI/CD & Deploy | `.github/workflows`, `vercel.json` | **Mentor** |
| Monitoring | `src/server/lib/logger.ts`, `sentry.*`, `instrumentation.ts` | **Mentor** |

**Mentor mode** หมายถึง:
1. อธิบายแนวคิดและสิ่งที่ต้องทำก่อน (สั้นๆ เป็นขั้นตอน)
2. ให้ hint / pseudo-code / โครง function signature ก่อนเขียนโค้ดเต็ม
3. เขียนโค้ดเต็มเมื่อเจ้าของพิมพ์ว่า "เฉลย", "เขียนให้เลย" หรือ "write it"
4. เมื่อเจ้าของเขียนเสร็จ ให้ review: bug, security, performance, naming แล้วบอกเหตุผล
5. ทุกครั้งที่มีการตัดสินใจเชิงสถาปัตยกรรม เสนอให้บันทึกเป็น ADR ใน `docs/adr/`

ถ้าไม่แน่ใจว่าไฟล์ไหนอยู่โหมดไหน ให้ถามก่อน — ดูโครงสร้างโฟลเดอร์และ flow ทั้งหมดใน `docs/structure.md`

---

## Tech Stack

- **Framework:** Next.js (App Router) + TypeScript (strict)
- **UI:** Tailwind CSS, shadcn/ui, Framer Motion
- **Database:** PostgreSQL บน Supabase
- **ORM / Migration:** Drizzle ORM + drizzle-kit
- **Validation:** Zod (ใช้ schema เดียวกันทั้ง API และ form)
- **Auth / Storage / Realtime:** Supabase Auth, Supabase Storage (รูปโพล), Supabase Realtime (คอมเมนต์)
- **Test:** Vitest (unit/integration), Playwright (e2e)
- **CI/CD:** GitHub Actions → Vercel
- **Monitoring:** Sentry (error + performance), PostHog (product analytics), Better Stack (uptime + log)
- **Package manager:** pnpm

---

## คำสั่งที่ใช้บ่อย

```bash
pnpm dev              # รัน dev server
pnpm lint             # ESLint
pnpm typecheck        # tsc --noEmit
pnpm test             # Vitest
pnpm test:e2e         # Playwright
pnpm db:generate      # สร้าง migration จาก schema
pnpm db:migrate       # apply migration
pnpm db:seed          # ใส่ข้อมูลตัวอย่าง
pnpm db:studio        # เปิด Drizzle Studio
```

ก่อนบอกว่างานเสร็จ ต้องรัน `pnpm lint && pnpm typecheck && pnpm test` ผ่านเสมอ

---

## สถาปัตยกรรม Backend (Layered)

```
Route Handler (src/app/api/**/route.ts)
   │  รับ request, เรียก auth, แปลง response เท่านั้น
   ▼
Validator (src/server/validators)  — Zod parse input
   ▼
Service (src/server/services)      — business logic, กฎของแอป
   ▼
Repository (src/server/repositories) — query DB ผ่าน Drizzle เท่านั้น
   ▼
PostgreSQL (Supabase)
```

กฎ:
- Route handler **ห้าม** query DB ตรง ต้องผ่าน service → repository
- Repository **ห้าม** มี business logic
- Service **ห้าม** รู้จัก `Request`/`Response` ของ HTTP
- Error ทั้งหมด throw เป็น `AppError` จาก `src/server/lib/errors.ts` แล้วให้ `handleRoute()` แปลงเป็น JSON

### รูปแบบ Response มาตรฐาน

```json
// สำเร็จ
{ "data": { ... } }
// ผิดพลาด
{ "error": { "code": "POLL_EXPIRED", "message": "โพลนี้หมดอายุแล้ว" } }
```

---

## กฎเรื่องเวลา (Business rule หลัก)

- "วันนี้" ของแอปคือวันตาม **Asia/Bangkok (UTC+7)** เสมอ ห้ามใช้ `new Date()` เทียบวันตรงๆ
- ใช้ helper ใน `src/server/lib/time.ts` เท่านั้น (`bangkokToday()`, `nextBangkokMidnight()`)
- โพลมี `expires_at` = เที่ยงคืนถัดไปตามเวลาไทย (เก็บเป็น UTC ใน DB)
- ทุก query ที่อ่านโพลต้องกรอง `expires_at > now()` — cron ลบทิ้งเป็นแค่การเก็บกวาด ไม่ใช่กลไกหลัก (Vercel Hobby cron อาจรันคลาดเคลื่อนได้ภายในชั่วโมง)
- คนเกิด 29 ก.พ. แสดงในวันที่ 28 ก.พ. ของปีที่ไม่ใช่ปีอธิกสุรทิน
- 1 คนสร้างโพลได้ 1 อันต่อวันเกิด

---

## Database Conventions

- ชื่อตาราง: snake_case พหูพจน์ (`users`, `polls`, `poll_comments`, `wishes`)
- Primary key: `uuid` default `gen_random_uuid()`
- ทุกตารางมี `created_at timestamptz default now()`
- เปลี่ยน schema ผ่าน Drizzle + migration เท่านั้น ห้ามแก้ใน Supabase dashboard
- เปิด RLS ทุกตาราง, policy อยู่ใน `supabase/policies/`
- ใส่ index ให้คอลัมน์ที่ใช้ filter บ่อย (`birth_month, birth_day`, `polls.expires_at`)

---

## Security

- ห้าม commit `.env*` (ยกเว้น `.env.example`)
- `SUPABASE_SERVICE_ROLE_KEY` ใช้ฝั่ง server เท่านั้น ห้ามขึ้นต้นด้วย `NEXT_PUBLIC_`
- endpoint ใน `/api/cron/*` ต้องเช็ก header `Authorization: Bearer ${CRON_SECRET}`
- อัปโหลดรูป: จำกัดชนิด (jpeg/png/webp) และขนาด ≤ 5MB
- Rate limit การอวยพรและคอมเมนต์

---

## Git & Workflow

- Branch: `feat/…`, `fix/…`, `chore/…`, `docs/…`
- Commit: Conventional Commits (`feat(api): add wishes endpoint`)
- ทุก PR ต้องผ่าน CI และได้ Vercel preview ก่อน merge เข้า `main`
- merge เข้า `main` = deploy production อัตโนมัติ

---

## Skills ที่มีในโปรเจกต์

| Skill | ใช้เมื่อ |
|---|---|
| `frontend-builder` | สร้าง/แก้หน้าจอ, component, แอนิเมชัน |
| `api-endpoint` | เพิ่มหรือแก้ API endpoint |
| `db-change` | เปลี่ยน schema, migration, RLS, seed |
| `cicd-deploy` | GitHub Actions, Vercel, environment, release |
| `observability` | Sentry, logging, health check, alert, ตรวจปัญหา production |
| `code-review` | review โค้ดที่เจ้าของเขียนเอง |

---

## ห้ามทำ

- ห้ามเพิ่ม dependency ใหม่โดยไม่บอกเหตุผลก่อน
- ห้ามปิด test ที่ fail เพื่อให้ CI ผ่าน
- ห้ามใช้ `any` ใน TypeScript
- ห้าม log ข้อมูลส่วนตัว (email, วันเกิดเต็ม, token)
