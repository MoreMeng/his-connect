# HIS Connect API

> ตัวกลางสำหรับส่งข้อมูล nRefer, ISOnline และ PHER Plus พร้อมงาน Cron, การเข้ารหัส token และสคริปต์ Docker ที่อัปเดตล่าสุด

## 🌟 คุณสมบัติหลัก

- **nRefer / ISOnline / PHER Plus** API gateway พร้อมจัดการ token และการส่งข้อมูลแบบ cron
- **Optimised Cron Manager** (`nodecron.optimized.ts`) ควบคุมกำหนดการ multi-instance ผ่าน PM2
- **Secure Refer Token** (`middleware/moph-refer.ts`) ตรวจสอบวันหมดอายุและเข้ารหัส `apikey` ด้วย SHA-1 ก่อนส่ง
- **Multi-platform Docker images** (linux amd64/arm64, Windows) พร้อมสคริปต์ build อัตโนมัติ
- โครงสร้างโค้ด **TypeScript** แยก `src/` → `app/` หลัง build

---

## 📦 โครงสร้างโปรเจกต์

```
src/
	app.ts              # จุดเริ่มต้นของ Fastify server
	nodecron.optimized.ts  # จัดการ cron schedule หลัก
	middleware/
		moph-refer.ts     # จัดการ token สำหรับ nRefer + hash apikey
	routes/
		...               # REST routes และงาน cron ย่อย
app/                  # โค้ด JS ที่ build แล้ว (สร้างด้วย `npm run build`)
config/               # ตัวอย่างค่า environment (ดู config.example)
create_docker_image.sh # สคริปต์ build/push container
Dockerfile            # Production image (multi-stage, non-root, Node 20-alpine)
```

---

## ✅ ความต้องการเบื้องต้น

- Node.js **20** ขึ้นไป และ npm
- ระบบฐานข้อมูล (MySQL/MariaDB, MSSQL, PostgreSQL, Oracle) ตามการตั้งค่าใน `config`
- PM2 (ติดตั้งอัตโนมัติใน Docker image / ติดตั้งเองถ้ารันบนเซิร์ฟเวอร์)
- เครื่องมือ build Docker (ถ้าต้องการสร้าง image เอง)

---

## 🚀 วิธีติดตั้งแบบ Node.js

1. **โคลนโค้ด และติดตั้งแพ็กเกจ**
	 ```bash
	 git clone https://github.com/superpck/his-connect.git
	 cd his-connect
	 npm install
	 ```

2. **ตั้งค่า environment**
	 - คัดลอก `config.example` ไปยัง `config` แล้วปรับค่าเชื่อมต่อฐานข้อมูล, API key, hospcode ฯลฯ
	 - ตรวจสอบไฟล์ `.env` หรือค่าที่ใช้ใน `process.env` เช่น `NREFER_APIKEY`, `NREFER_SECRETKEY`, `REQUEST_KEY`

3. **คอมไพล์ TypeScript → JavaScript**
	 ```bash
	 npm run build
	 ```
	 โค้ดจะถูกสร้างไว้ที่โฟลเดอร์ `app/`

4. **รันในโหมดพัฒนา**
	 ```bash
	 npm start
	 ```
	 หรือใช้ `npm run watch` สำหรับ nodemon (ใช้กับ TypeScript โดยตรง)

---

## 🐳 การใช้งาน Docker

### ดึง image ที่เผยแพร่ไว้แล้ว
```bash
docker pull superpck/his-connect:linux-latest   # multi-arch (amd64/arm64)
docker pull superpck/his-connect:latest         # linux/amd64
docker pull superpck/his-connect:windows-latest # ต้องใช้ Windows builder
```

### เรียกใช้งาน container อย่างรวดเร็ว
```bash
docker run -d \
	--name his-connect \
	-p 3004:3004 \
	-v $(pwd)/config:/usr/src/his_connect/config \
	-e HOSPCODE=XXXXXX \
	superpck/his-connect:latest
```

### สคริปต์สร้าง image (`create_docker_image.sh`)

สคริปต์นี้จะ:

1. คอมไพล์ TypeScript และลบ sourcemap
2. ดึง base image (`node:20-alpine`)
3. สร้าง multi-arch image ด้วย Docker Buildx (linux + windows)
4. Push ขึ้น Docker Hub

ปรับแต่งพฤติกรรมได้ผ่าน environment variable:

| ตัวแปร | ค่าเริ่มต้น | ความหมาย |
| --- | --- | --- |
| `ENABLE_SYSTEM_PRUNE` | `false` | ตั้งค่า `true` หากต้องการ `docker system prune` ก่อน build |
| `ENABLE_WINDOWS_BUILD` | `false` | ตั้งค่า `true` เพื่อ build image สำหรับ Windows |
| `ENABLE_SYSTEM_PRUNE_AFTER` | `false` | ตั้งค่า `true` เพื่อลบ cache หลัง build |

> **เคล็ดลับ:** หากพบ error `toomanyrequests` จาก Docker Hub ให้ทำ `docker login` หรือตั้งค่า mirror ใกล้เคียง

---

## ⏱️ Cron & Background Jobs

- `src/nodecron.optimized.ts` ใช้ `node-cron` + PM2 ตรวจสอบว่า instance ไหนควรส่งงาน โดยอิง PID จาก PM2
- สามารถตั้งค่าเวลาได้ผ่าน environment variables เช่น `IS_AUTO_SEND_EVERY_MINUTE`, `NREFER_AUTO_SEND_EVERY_HOUR` ฯลฯ
- งานประจำ (เช่น nRefer IPD, IS Online) จะตรวจสอบสถานะทุก ๆ X นาที และรายงานผลใน console

---

## 🔐 ความปลอดภัย & Token

- `middleware/moph-refer.ts`
	- ตรวจสอบ token เดิมว่ายังไม่หมดอายุก่อนเรียก API ใหม่
	- แฮช `dataArray.apikey` ด้วย SHA-1 (อิงค่า `REQUEST_KEY`)
	- จัดการการหมดอายุ token อัตโนมัติและ log สถานะเพื่อดีบัก

ข้อควรระวังด้านความปลอดภัยเพิ่มเติม:

1. ให้สิทธิ์ database user เฉพาะ `SELECT`
2. วางเซิร์ฟเวอร์ไว้ในโซนที่ปลอดภัย ป้องกันการอ่านค่าคอนฟิก
3. เปลี่ยนรหัสผ่าน ISOnline / Secret key ทุก 3–6 เดือน
4. ปิดบัญชีผู้ใช้เมื่อมีการย้ายงานหรือออกจากหน่วยงาน

---

## 🗄️ ตัวอย่าง VIEW (สำหรับ SSB – รพ.สระบุรี)

สร้าง VIEW เพื่อให้ระบบ API ดึงข้อมูลได้สะดวก:

```
PERSON
OPD_SERVICE
DIAGNOSIS
ADMISSION
```

---

## 🤝 สำหรับทีมพัฒนา (Develop@MOPH)

```
git add .
git commit -m "คำอธิบายสิ่งที่แก้ไข"
git push origin <branch>
# หาก push ไม่ได้ ให้ git pull ก่อน
```

---

## 📚 บันทึกการเปลี่ยนแปลง

ดูรายละเอียดเวอร์ชันล่าสุดใน [`CHANGELOG.md`](./CHANGELOG.md)

---

## 🙏 ขอขอบคุณ

- อ.สถิตย์ เรียนพิศ — https://github.com/siteslave