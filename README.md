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
cp .env.example .env   # แล้วกรอก DATABASE_URL และค่า ACCESS_TOKEN_* ให้ครบ
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

## Production Deployment (Free)

สถาปัตยกรรมที่ใช้สำหรับการใช้งานฟรี:

```text
Vercel Hobby (web) → Render Free Web Service (api) → Neon Free PostgreSQL
```

ไม่ควรสร้าง Render Postgres สำหรับแผนฟรีระยะยาว เพราะฐานข้อมูลฟรีของ Render มีอายุจำกัด ให้ใช้ Neon เป็นฐานข้อมูลแทน

### Backend บน Render

Repository: `https://github.com/jutharat4533/project.nurse.balance.api`

ตั้งค่า Service เป็น **Web Service / Free**:

```text
Build Command: pnpm install --frozen-lockfile && pnpm prisma migrate deploy && pnpm build
Start Command: pnpm start:prod
```

คำสั่ง `pnpm build` จะ generate Prisma Client ให้อัตโนมัติ และ `start:prod` จะเริ่มจาก `dist/src/main.js`

Environment Variables ที่ต้องกำหนดใน Render:

```env
PORT=10000
DATABASE_URL=<Neon connection string>
CORS_ORIGINS=https://<production-web-domain>.vercel.app
ACCESS_TOKEN_SECRET=<สุ่มอย่างน้อย 32 ตัวอักษร>
ACCESS_TOKEN_EXPIRES_IN=86400
CLOUDINARY_CLOUD_NAME=<Cloudinary Cloud name>
CLOUDINARY_API_KEY=<Cloudinary API Key>
CLOUDINARY_API_SECRET=<Cloudinary API Secret>
```

`DATABASE_URL` ต้องคัดลอกจาก Neon เมนู **Connect** โดยตรง ห้ามใช้ค่า placeholder เช่น `hostname:5432`

### Frontend บน Vercel

Repository: `https://github.com/jutharat4533/first-web`

ตั้งค่าเป็น **Hobby** และใช้:

```text
Install Command: pnpm install --frozen-lockfile
Build Command: pnpm build
```

Environment Variables ที่ต้องกำหนดใน Vercel:

```env
API_URL=https://<render-api-domain>.onrender.com
NEXT_PUBLIC_API_URL=https://<render-api-domain>.onrender.com
AUTH_URL=https://<production-web-domain>.vercel.app
AUTH_SECRET=<สุ่มค่าใหม่อย่างน้อย 32 ตัวอักษร>
```

สร้าง secret ได้ด้วย:

```bash
openssl rand -hex 32
```

ใช้ URL จากเมนู Vercel **Domains** ซึ่งเป็น Production Domain เช่น `https://first-web-sable.vercel.app` ห้ามใช้ URL แบบมีรหัสยาว เช่น `first-xxxx-health-project3.vercel.app` เป็น URL หลัก เพราะเป็น URL เฉพาะของ Deployment

หลังเปลี่ยน `AUTH_URL` หรือ `CORS_ORIGINS` ต้อง Deploy ใหม่ทั้ง Vercel และ Render

### ข้อจำกัดของแผนฟรี

- Render Free อาจพัก API เมื่อไม่มีการใช้งาน ทำให้ request แรกหลังพักช้า จากนั้น request ถัดไปจะเร็วขึ้น
- Neon Free มีโควตาพื้นที่และ compute จำกัด
- Vercel Hobby เหมาะกับโปรเจกต์ส่วนตัวและมีโควตาการใช้งาน
- การแจ้งเตือนเวรปัจจุบันเป็นการแสดงในแอป ยังไม่ใช่ Push Notification ตอนปิดเว็บ
