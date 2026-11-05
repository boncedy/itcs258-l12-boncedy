# Lab 12 — Local Docker Compose and Persistent Data

**Required: one gacha app, experiments 01–04.**

**งานบังคับ: แอป gacha เดียว ทำการทดลอง 01–04**

Website: https://pisit-devop.com/itcs258/Lab12/

Dr. Pisit Praiwattana, your mentor in 2026

Run one Express + PostgreSQL stack, explain its Compose keys and commands, inspect real records with SQL, and prove persistence after container replacement. Currency in this example is virtual; no payment or cloud account is needed for Lab 12.

รัน Express กับ PostgreSQL หนึ่ง stack อธิบาย Compose key และคำสั่ง ตรวจแถวข้อมูลจริงด้วย SQL และพิสูจน์ข้อมูลหลังเปลี่ยน container เงินในตัวอย่างเป็นเงินจำลอง ไม่ต้องชำระเงินหรือใช้บัญชี cloud สำหรับ Lab 12

## 01 · Folder, environment and readiness / โฟลเดอร์ environment และความพร้อม

1. Download and extract [gacha-workshop.zip](gacha-workshop.zip). Work inside `gacha-workshop/`.
2. Check `docker version` and `docker compose version`.
3. Identify `compose.yml`, the auto-loaded `compose.override.yml`, Dockerfile, application source, `public/test.html` and `db/init.sql`.
4. Copy `.env.example` to `.env`. Set `STUDENT_ID` to your actual ID; keep this and the project name unchanged throughout the experiment. Use only disposable local credentials.
5. Run `docker compose config --quiet` and `docker compose config --services`. Explain CLI interpolation versus values explicitly passed into containers.

ดาวน์โหลดและแตกไฟล์ ใช้โฟลเดอร์ `gacha-workshop/` ตรวจ Engine กับ Compose ระบุหน้าที่ไฟล์ คัดลอก `.env.example` เป็น `.env` และตั้งรหัสตนเอง ตรวจค่าตั้งก่อนเริ่ม โดยแยก interpolation ของ CLI ออกจาก environment ใน container

```powershell
# Windows PowerShell
Copy-Item .env.example .env
```

```bash
# macOS / Linux
cp .env.example .env
```

## 02 · Compose keys and everyday commands / Key และคำสั่งที่ใช้บ่อย

Read the real YAML and explain `services`, `build`, `image`, `ports`, `environment`, `depends_on`, `healthcheck` and `volumes`. Explain that `env_file`, a custom `networks` declaration, `restart` policy and `command` are useful optional configuration patterns, not extra services to add blindly.

อ่าน YAML จริงและอธิบาย key ของตัวอย่าง ส่วน `env_file`, custom `networks`, นโยบาย `restart` และ `command` เป็นรูปแบบค่าตั้งที่เลือกใช้ตามความจำเป็น ไม่ต้องเพิ่มทุกอย่างโดยไม่เข้าใจ

```text
docker compose build app
docker compose up -d --build --wait
docker compose ps
docker compose logs --tail 30 app db
```

Show both services healthy. Open `http://localhost:8085/health` and `http://localhost:8085/test`. Explain laptop `localhost:8085` → app port 3000, and app → `db:5432`. PostgreSQL has no published host port in the reference stack.

แสดงทั้งสองบริการ healthy เปิด `/health` กับ `/test` อธิบายพอร์ตโฮสต์ 8085 ไป app 3000 และ app ไป `db:5432` โดย DB ไม่ publish พอร์ตโฮสต์

## 03 · API experiment and PostgreSQL basics / ทดลอง API และพื้นฐาน PostgreSQL

Use `/test` and your actual student ID:

1. Check Health and Catalog; load your wallet.
2. Recharge **1000 virtual coins**.
3. Exchange **1000 coins for 10 gems**.
4. Draw **once**, spending 10 gems.
5. Read history and record the real draw ID, `result_name`, `rarity` and `created_at`.

ใช้ `/test` ใส่รหัสตนเอง ตรวจ Health/Catalog โหลด wallet เติม 1000 coins แลกเป็น 10 gems สุ่มหนึ่งครั้งแล้วอ่าน history เก็บค่าของแถวจริง ไม่ต้องทำให้ได้ SSR เพื่อผ่านแลป

Legendary probability is 2% for each independent draw, with five equally possible SSR results within that outcome. This does not guarantee a legendary result after a fixed number of draws. Save the actual result, including COMMON results.

Legendary มีโอกาส 2% ต่อครั้ง หากได้ผลนั้นจะมี SSR ห้าแบบที่มีโอกาสเท่ากัน ไม่ได้รับประกันว่าจะได้ legendary หลังสุ่มจำนวนหนึ่ง ผล COMMON จริงใช้เป็นหลักฐานได้เช่นกัน

Connect to the SQL client already inside the database container:

```text
docker compose exec db psql -U gacha -d gacha
```

Then type these **inside psql**, replacing the sample ID:

```sql
\conninfo
\dt
SELECT id, student_id, result_name, rarity, created_at
FROM draws WHERE student_id = '6688123' ORDER BY id;
SELECT COUNT(*) AS draw_count FROM draws
WHERE student_id = '6688123';
```

The API history and SQL must show the same original row. For the short INSERT practice, use a **separate** ID so your proof account stays untouched:

API history กับ SQL ต้องแสดงแถวเดิมตรงกัน สำหรับฝึก INSERT ใช้ ID แยกเพื่อไม่แก้บัญชีหลักฐาน:

```sql
INSERT INTO wallets (student_id) VALUES ('6688123-sql')
ON CONFLICT (student_id) DO NOTHING;
SELECT student_id, coins, gems FROM wallets
WHERE student_id = '6688123-sql';
\q
```

Postman, PowerShell or curl can perform equivalent requests. Read the [API specification](api-spec.html) and [OpenAPI JSON](openapi.json) for route/body/response details. Do not directly manufacture replacement draws or edit balances for the proof account.

ใช้ Postman, PowerShell หรือ curl ส่งคำขอเทียบเท่ากันได้ อ่านเอกสาร API เพื่อดู route/body/response ห้ามสร้าง draw ทดแทนหรือแก้ยอดของบัญชีหลักฐานโดยตรง

## 04 · Persistence and in-folder mapping / ข้อมูลถาวรและ mapping ในโฟลเดอร์

Capture the original draw and these values before testing:

```text
docker compose ps -q db
docker inspect --format "{{json .Mounts}}" $(docker compose ps -q db)
```

**A. Restart**, then read history and SQL again:

```text
docker compose restart
docker compose up -d --wait
```

**B. Remove/recreate**, then read history and SQL again:

```text
docker compose down
docker compose up -d --wait
docker compose ps -q db
docker inspect --format "{{json .Mounts}}" $(docker compose ps -q db)
```

**Do not draw again or reinsert the original row between checks.** Keep the same workshop folder, project and named volume. A new DB container ID after down/up must read the same original row from the same volume. Compare timestamps as equivalent instants; text formatting can differ.

**ห้ามสุ่มเพิ่มหรือเพิ่มแถวเดิมใหม่ระหว่างตรวจ** คงโฟลเดอร์ project และ named volume เดิม หลัง down/up ต้องเห็น container DB ใหม่อ่านแถวเดิมจาก volume เดิมได้ รูปแบบข้อความเวลาอาจต่างแต่ต้องเป็นขณะเดียวกัน

| Stage / ช่วง | DB container ID | Volume | Original draw ID | Result + rarity + created_at |
| --- | --- | --- | --- | --- |
| Initial / เริ่มต้น | | | | |
| Restart / หลัง restart | | | | |
| Down/up / หลังสร้างใหม่ | | | | |

Inspect and explain:

- `pgdata:/var/lib/postgresql`: named volume for PostgreSQL 18 database files.
- `./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro`: read-only host file bind mount.
- Verify the source is a real file on the Docker host; explain relative paths and file/directory mismatches.
- Explain why a bind mount over `/app` can hide files such as installed dependencies, and why host permissions still matter.
- Explain initialization versus existing-schema migration, and persistence versus backup.

The PostgreSQL image entrypoint reads the mounted SQL only on an empty database directory. Separately, `server.js` executes the SQL copied into the app image on every app startup. The current `CREATE TABLE IF NOT EXISTS` statements create missing tables, not new columns in existing tables. Edited app-copy SQL requires rebuilding the image and can run against existing data; use deliberate migrations for schema changes.

Entrypoint ของ image PostgreSQL อ่าน SQL ที่ mount เมื่อ directory ว่างเท่านั้น ส่วน `server.js` รัน SQL ที่คัดลอกเข้า image แอปทุกครั้งที่เริ่ม `CREATE TABLE IF NOT EXISTS` ปัจจุบันสร้างตารางที่ยังไม่มี ไม่เพิ่ม column ในตารางเดิม การแก้ SQL ฝั่งแอปต้อง rebuild และอาจรันกับข้อมูลเดิมได้ จึงใช้ migration อย่างตั้งใจเมื่อเปลี่ยน schema

ตรวจและอธิบาย named volume กับ bind mount แบบ read-only ตรวจว่าไฟล์ต้นทางมีจริงบน Docker host อธิบาย path สัมพัทธ์ ชนิดไฟล์/โฟลเดอร์ การ mount ทับที่บังไฟล์ใน image สิทธิ์โฮสต์ initialization/migration และเหตุใด persistence จึงไม่ใช่ backup

Ordinary cleanup is **`docker compose down` without `-v`**. Keep your volume until LA checking and report evidence are complete.

การหยุดงานปกติใช้ **`docker compose down` โดยไม่ใส่ `-v`** เก็บ volume จนตรวจ LA และเก็บหลักฐานรายงานเสร็จ

## In-room LA check and submission / ตรวจ LA ในห้องและส่งงาน

Before leaving the room, call the LA to your workstation. Show the working gacha stack, actual API/SQL result, original record after replacement and mount explanation. Be ready to explain each command in your own words.

ก่อนออกจากห้อง เรียก LA มาที่เครื่อง แสดง gacha stack ผล API/SQL จริง แถวเดิมหลังเปลี่ยน container และอธิบาย mount กับแต่ละคำสั่งด้วยคำของตนเอง

Submit **one PDF** named **`student_ID_lab12.PDF`** to MyCourse, replacing `student_ID` with your actual ID. Include sections 01–04, your repository URL, relevant sanitized source/configuration and readable screenshots from **your own screen**. Follow the opening/deadline shown in MyCourse. Use the [report template](report-template.md).

ส่ง **PDF หนึ่งไฟล์** ชื่อ **`student_ID_lab12.PDF`** ใน MyCourse โดยแทนรหัสจริง ใส่หัวข้อ 01–04, URL repository, ซอร์ส/ค่าตั้งที่ไม่มีข้อมูลลับ และภาพที่อ่านได้จาก **หน้าจอตนเอง** ตามกำหนดใน MyCourse

Finish the [web self-quiz](self-quiz.html) after the workshop. **Optional extension (05):** apply the pattern to your own backend or another provided framework example. This extension is additional practice, not another required project.

ทำ [self-quiz บนเว็บ](self-quiz.html) หลัง workshop **ส่วนเสริมทางเลือก (05):** ปรับรูปแบบไปใช้กับแบ็กเอนด์ตนเองหรือ framework ตัวอย่างอื่น เป็นการฝึกเพิ่ม ไม่ใช่โปรเจกต์บังคับอีกชุด
