# Lab 12 runnable Compose examples

All three projects expose the same notes resource and use a separate, project-scoped
PostgreSQL 18 named volume. Choose the Express reference first; FastAPI and NestJS
show the same approach in the other Lab 11 frameworks. The assessed exercise also
requires applying it to your own backend project.

| Folder | Host URL | App container port | Project prefix |
| --- | --- | --- | --- |
| express | http://localhost:8082 | 3000 | lab12-express |
| fastapi | http://localhost:8083 | 8000 | lab12-fastapi |
| nestjs | http://localhost:8084 | 3000 | lab12-nestjs |

Read each project's README before starting. Each Compose file has two services,
app and db, no published database port, an explicit DB readiness check and a named
volume at /var/lib/postgresql. Set a stable student ID in each project's ignored
.env file. For your own project choose another stable project name.

Each project's POST /notes returns the created row; GET /notes returns an array.
Record fields are id, student_id, body and created_at. The health route verifies
an actual SELECT 1 against the database. Error responses intentionally omit
credentials and database connection details.

The Node dependency trees are pinned by package-lock.json. Python direct packages
are pinned in requirements.txt; its transitive dependency tree is not fully locked.
Base tags pin a major/runtime family and follow patch updates rather than an image
digest. These are local teaching examples, not a high-availability deployment.

## Verification

All three projects were built and run on Docker Desktop's Linux engine. Checks
covered actual HTTP POST/GET, input rejection, SQL-looking text stored as data,
a request from inside the app container, independent SQL results, restart,
container removal/recreation with the original row and volume retained, and DB
outage/recovery without replacing the app container. Disposable test containers,
volumes and QA image tags were removed. No online server was changed.

## Optional API test client / เครื่องมือทดสอบ API เสริม

Each downloaded project includes api-testing/ with a bilingual guide, Postman
collection and environment template. The website also offers api-testing.zip.
Use Postman, PowerShell or curl to create one baseline row and compare the same
row after restart and container replacement. This supports existing report
evidence; it adds no new graded task. In a source checkout the shared assets live
at [../api-testing/](../api-testing/API-TESTING.md).

starter ZIP แต่ละชุดมีคู่มือสองภาษาและไฟล์นำเข้า Postman ใน api-testing/ เลือก Postman,
PowerShell หรือ curl ได้ สร้าง baseline ครั้งเดียว แล้ว GET ตรวจแถวเดิมหลัง restart
และสร้างคอนเทนเนอร์ใหม่ เป็นแนวทางเสริมสำหรับหลักฐานเดิม ไม่ใช่งานส่งเพิ่ม

## Run and compare the three examples

The required assignment in `lab12-instructions.md` is a **gacha** experiment using
the `gacha-workshop` project. It is extracted in
`.\gacha-workshop\gacha-workshop`. The `express`, `fastapi`, and `nestjs` folders
are optional framework examples for a different, small `notes` API; they do not
have the gacha wallet, draw, or history endpoints. Do not use their results as
gacha evidence.

Download `gacha-workshop.zip` from the [Lab 12 course page](https://pisit-devop.com/itcs258/Lab12/)
and extract/open its `gacha-workshop` folder. In this checkout it is at
`.\gacha-workshop\gacha-workshop`; the extra outer folder is from the ZIP
extraction. The three folders in this repository are optional framework practice.

Start Docker Desktop with the Linux engine first. In PowerShell, open a separate
terminal for each folder you want to run. The `.env` files are ignored by Git;
copy the example and replace `STUDENT_ID` with your own stable ID before starting:

```powershell
Set-Location .\express
Copy-Item .env.example .env
notepad .env
docker compose config --quiet
docker compose config --services
docker compose build app
docker compose up -d --build --wait
docker compose ps
docker compose logs --tail 30 app db
Invoke-RestMethod http://localhost:8082/health
```

In a separate terminal for FastAPI or NestJS, use `Set-Location .\fastapi` or
`Set-Location .\nestjs` from the repository root, then run the same Compose
commands. Their host URLs are `http://localhost:8083` and `http://localhost:8084`.
All three can run at once because they publish different loopback ports.

Compose reads `.env` for interpolation such as the project name, image tag, host
port, and PostgreSQL settings. Only values listed under a service's `environment`
are passed into that container. Never paste the real password into a report or
share it in a screenshot.

### Run the required gacha experiment

In PowerShell, open the extracted workshop folder and set up its environment. From
the repository root, use this path (adjust it if you extracted the archive
somewhere else). Keep the same student ID and Compose project for the full
persistence test:

```powershell
Set-Location .\gacha-workshop\gacha-workshop
Copy-Item .env.example .env
notepad .env
docker version
docker compose version
docker compose config --quiet
docker compose config --services
docker compose build app
docker compose up -d --build --wait
docker compose ps
docker compose logs --tail 30 app db
```

Open `http://localhost:8085/test`. Enter your student ID, check Health and Catalog,
load your wallet, recharge 1000 virtual coins, exchange 1000 coins for 10 gems,
draw once, then read History. Do not repeat recharge, exchange, or draw actions
just to get a preferred result. You can also use these API routes; use each POST
only once for the experiment:

```text
GET  http://localhost:8085/health
GET  http://localhost:8085/api/accounts/<your-student-id>
POST http://localhost:8085/api/recharge       {"student_id":"<your-student-id>","coins":1000}
POST http://localhost:8085/api/exchange       {"student_id":"<your-student-id>","coins":1000}
POST http://localhost:8085/api/draw           {"student_id":"<your-student-id>"}
GET  http://localhost:8085/api/accounts/<your-student-id>/history
```

Record the actual draw ID, result name, rarity and creation time from History.
Then inspect the same row with the `psql` queries in `lab12-instructions.md`.
After restart and after `docker compose down` followed by `docker compose up -d
--wait`, read History and SQL again without drawing again. The database container
ID should change after down/up, but the original row and named volume should stay
the same. Follow `report-template.md` for the screenshots and evidence table.

### Create a notes row and check the response

Use your own student ID and a unique marker. Change the URL port for the framework
you started:

```powershell
$note = @{
    student_id = '6788185'
    body = 'lab12-6788185-first-note'
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
  -Uri 'http://localhost:8082/notes' `
  -ContentType 'application/json' `
  -Body $note

Invoke-RestMethod -Uri 'http://localhost:8082/notes'
Invoke-RestMethod -Uri 'http://localhost:8082/health'
```

POST should return HTTP 201 and the saved row (`id`, `student_id`, `body`, and
`created_at`). GET `/notes` returns an array containing it. GET `/health` should
return `status: ok`, `database: connected`, and that app's framework name. FastAPI
uses the same request and response shape, except invalid input is rejected with
HTTP 422; Express and NestJS return HTTP 400. For example, an empty body is invalid.

Check the database independently; this should show the same marker and ID:

```powershell
docker compose exec -T db psql -U lab12 -d lab12 -c `
  "SELECT id, student_id, body, created_at FROM notes ORDER BY id;"
```

### Prove that the row survives restart and container replacement

Capture the database container ID and mounts before changing anything:

```powershell
$oldDbId = docker compose ps -q db
$oldDbId
docker inspect --format '{{json .Mounts}}' $oldDbId
```

Restart the services, then GET `/notes` and run the SQL query again. The same row
should still be there:

```powershell
docker compose restart
docker compose up -d --wait
Invoke-RestMethod -Uri 'http://localhost:8082/notes'
```

Now replace the containers without deleting the volume. Do not POST the row again:

```powershell
docker compose down
docker compose up -d --wait
$newDbId = docker compose ps -q db
$newDbId
docker inspect --format '{{json .Mounts}}' $newDbId
Invoke-RestMethod -Uri 'http://localhost:8082/notes'
docker compose exec -T db psql -U lab12 -d lab12 -c `
  "SELECT id, student_id, body, created_at FROM notes ORDER BY id;"
```

The new DB container ID should differ from `$oldDbId`. The named volume name and
the original row should be unchanged. `docker compose down` keeps the named
volume; `docker compose down -v` deletes it. A named volume survives container
replacement, but it is not a backup.

### Understand the mounts and initialization

Run `Test-Path .\db\init.sql -PathType Leaf` to check that the relative bind-mount
source is a file. `docker inspect` shows its resolved host path, mount type, and
read-only flag. If the source path is misspelled or absent, Docker may create a
directory where a file was expected and PostgreSQL will not run the intended SQL.
Bind mounts also depend on host permissions. Avoid mounting a host folder over
`/app`: it can hide the application and installed dependencies in the image.

`pgdata:/var/lib/postgresql` is the named volume holding database files.
`./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro` is a read-only bind
mount of a host SQL file. PostgreSQL's image entrypoint runs that SQL only when
initializing an empty data directory. In these notes examples, that is the only
schema initialization path. `CREATE TABLE IF NOT EXISTS` does not add columns to
an existing table; schema changes need deliberate migrations. Changing a mounted
init file will not update an already initialized database.

The gacha workshop described in the assignment has a separate behavior: its
`server.js` also runs SQL copied into the app image on every app start. Rebuilding
that image is needed to copy edited SQL, and its statements can affect an existing
database, so schema changes there should also use deliberate migrations.

When finished, keep the volume until the report evidence is complete, then run
`docker compose down` in each folder you started. Do not add `-v` unless you
intentionally want to delete that project's database.
