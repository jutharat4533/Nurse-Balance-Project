# NurseBalance

แอปพลิเคชันจัดการตารางเวรและรายได้สำหรับพยาบาล — วางแผนเวรทำงาน, คำนวณรายได้สุทธิ (เงินเดือน + ค่าเวร + รายได้พิเศษ − รายการหัก), และค้นหา/สมัครงานพยาบาล

## Tech Stack

| ส่วน | เทคโนโลยี |
|---|---|
| Frontend | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS, shadcn/ui, Zustand, NextAuth (Auth.js) v5 |
| Backend | NestJS 11, TypeScript |
| Database | PostgreSQL |
| ORM | Prisma |

## โครงสร้างโปรเจกต์

```
NurseBalance/
├── api/    # NestJS backend — REST API + Prisma + PostgreSQL
└── web/    # Next.js frontend — หน้าเว็บที่ผู้ใช้เห็น
```

โปรเจกต์นี้เป็น 2 แอปพลิเคชันแยกกัน สื่อสารกันผ่าน HTTP (`web` เรียก `api` ด้วย fetch แนบ JWT access token)

## ฟีเจอร์หลัก

- **ระบบสมาชิก**: สมัคร/เข้าสู่ระบบด้วย JWT, มี 2 role คือ `USER` และ `ADMIN`, หน้าเว็บทุกหน้า (ยกเว้น login/register) ต้อง login ก่อนถึงเข้าได้ (ป้องกันด้วย Next.js proxy/middleware)
- **ตารางเวร**: ปฏิทินรายเดือน, เพิ่ม/แก้ไข/ลบเวร, จัดการโรงพยาบาล/สถานที่ทำงานพร้อมเรทค่าเวรแต่ละกะ
- **แดชบอร์ดรายได้**: คำนวณรายได้สุทธิรายเดือนอัตโนมัติจากเงินเดือนพื้นฐาน + ค่าเวร + รายได้พิเศษ − รายการหัก
- **ตลาดงาน**: ค้นหา/กรองประกาศงาน, สมัครงาน (กันสมัครซ้ำ), ผู้ใช้ role `ADMIN` ลงประกาศ/แก้ไข/ลบงานได้
- **โปรไฟล์**: ดู/แก้ไขข้อมูลส่วนตัว, สรุปสถิติการทำงาน
- **แจ้งเตือนเวร**: แสดงจำนวนและรายละเอียดเวรของวันถัดไปในแอปผ่านปุ่มกระดิ่ง (ยังไม่ใช่ push notification ตอนปิดเว็บ)

## เริ่มต้นใช้งาน (Local Development)

ต้องมี Node.js, pnpm, และ PostgreSQL ที่รันอยู่แล้ว

### 1. ตั้งค่า Backend (`api/`)

```bash
cd api
cp .env.example .env   # แล้วกรอกค่าจริง เช่น DATABASE_URL, JWT_SECRET, ACCESS_TOKEN_SECRET
pnpm install
pnpm prisma migrate dev     # ใช้ migration สำหรับ development
pnpm start:dev           # รันที่ http://localhost:10000 (ตาม PORT ใน .env)
```

### 2. ตั้งค่า Frontend (`web/`)

```bash
cd web
cp .env.example .env   # ค่า default ชี้ไปที่ api บน localhost:10000 อยู่แล้ว
pnpm install
pnpm dev -- -p 3001     # รันที่ http://localhost:3001 (ต้องตรงกับ AUTH_URL ใน .env)
```

เปิดเบราว์เซอร์ไปที่ `http://localhost:3001` — จะเด้งไปหน้า `/login` อัตโนมัติถ้ายังไม่ได้ login

### หมายเหตุ

- ผู้ใช้ที่สมัครใหม่จะได้ role `USER` เสมอ (ไม่มีหน้าเว็บให้สมัครเป็น `ADMIN` เอง) หากต้องการทดสอบสิทธิ์ `ADMIN` (ลงประกาศงาน/แก้ไข/ลบ) ต้องเข้าไปเปลี่ยนค่า `role` ของ user นั้นในฐานข้อมูลโดยตรง
- สำหรับ development ที่ต้องการ sync schema อย่างรวดเร็ว ใช้ `pnpm prisma db push` ได้ แต่ production ต้องใช้ `pnpm prisma migrate deploy`
- หากฐานข้อมูลเดิมถูกสร้างด้วย `db push` แล้ว ให้ตรวจสอบ schema ก่อน และใช้ `pnpm prisma migrate resolve --applied 20261008000000_init` เพื่อ baseline เฉพาะเมื่อ schema ตรงกับ migration นี้

## Production Deployment

ก่อน deploy ให้ตั้งค่า environment จริงและห้ามใช้ secret จากไฟล์ตัวอย่าง:

### Backend (`api/`)

```bash
pnpm install --frozen-lockfile
pnpm prisma migrate deploy
pnpm build
pnpm start:prod
```

ต้องกำหนดอย่างน้อย `PORT`, `CORS_ORIGINS`, `DATABASE_URL`, `ACCESS_TOKEN_SECRET`, `ACCESS_TOKEN_EXPIRES_IN` และค่า Cloudinary ให้ครบ โดย `CORS_ORIGINS` ต้องเป็น origin ของ frontend แบบระบุชัดเจน คั่นด้วย comma และห้ามใช้ `*`

### Frontend (`web/`)

```bash
pnpm install --frozen-lockfile
pnpm build
pnpm start
```

ต้องกำหนด `API_URL`, `NEXT_PUBLIC_API_URL`, `AUTH_URL` และสร้าง `AUTH_SECRET` ใหม่สำหรับ environment นั้น เช่น `openssl rand -base64 32`

การ deploy อัตโนมัติยังไม่ได้ผูกกับ provider ใดใน repository นี้ การ push code จะไม่ deploy เองจนกว่าจะตั้งค่า hosting provider หรือ CI/CD เพิ่ม
