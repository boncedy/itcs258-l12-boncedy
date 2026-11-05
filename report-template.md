# Lab 12 report — One local Compose experiment

# รายงาน Lab 12 — ทดลอง Compose ในเครื่องด้วยแอปเดียว

Student ID / รหัสนักศึกษา: 6788185

Name / ชื่อ: Daranpob Sianghen

OS and Docker context / ระบบปฏิบัติการและ Docker context: Windows + Docker Desktop Linux (`desktop-linux`)

Repository URL / URL repository: https://github.com/boncedy/itcs258-l12-boncedy

I ran the required gacha workshop from
`gacha-workshop/gacha-workshop`. The Express, FastAPI, and NestJS folders in the
repository are optional notes API examples, not the gacha app.

## Start here: capture the evidence / เริ่มตรงนี้: เก็บหลักฐาน

The gacha app is working. Its correct address is **http://localhost:8085**.
Port 8082 is for the optional Express notes example. The recharge, exchange,
and one draw for student ID `6788185` have already been completed and saved.
**Do not click Recharge, Exchange, or Draw again.** Use the saved history and
SQL below to show those actions.

For each numbered screenshot, press **Windows + Shift + S**, select the useful
part of the screen, and save it with the suggested name. Hide any password before
capturing. Put the images in an `evidence` folder beside this report, then insert
them into the matching section of the final document. If editing this Markdown
file, an image can be inserted with
`![Screenshot description](evidence/01-docker-config.png)`.

### Screenshot 1 — Setup and Compose / ตั้งค่าและ Compose

Use the terminal whose current folder is `gacha-workshop\gacha-workshop`:

```powershell
docker version
docker compose version
docker compose config --quiet
docker compose config --services
docker compose ps
docker compose logs --tail 10 app
```

Capture Docker/Compose versions, the service names, and both healthy services.
The app should show port `8085->3000`; the database should show `5432/tcp`
without a host port. Save as `01-compose-and-health.png`.

### Screenshot 2 — Health and saved history / Health และประวัติ

Open **http://localhost:8085/test**. In the Student ID field use `6788185`.
Click **GET · Health + DB** and capture the URL, HTTP 200 status, and connected
response. Then click **GET · Saved history** and capture the draw and three
transactions. Save as `02-api-health.png` and `03-api-history.png`.

You may also show Health in PowerShell:

```powershell
Invoke-RestMethod http://localhost:8085/health
```

The result should show `ok`, `connected`, and `gacha`. **Do not use port 8082.**

### Screenshot 3 — SQL / ตรวจข้อมูลด้วย SQL

Back in the same workshop terminal, run:

```powershell
docker compose exec -T db psql -U gacha -d gacha -c "\conninfo"
docker compose exec -T db psql -U gacha -d gacha -c "\dt"
docker compose exec -T db psql -U gacha -d gacha -c "SELECT id, student_id, result_name, rarity, created_at FROM draws WHERE student_id='6788185' ORDER BY id;"
docker compose exec -T db psql -U gacha -d gacha -c "SELECT COUNT(*) AS draw_count FROM draws WHERE student_id='6788185';"
docker compose exec -T db psql -U gacha -d gacha -c "SELECT student_id, coins, gems FROM wallets WHERE student_id='6788185-sql';"
```

Capture the table list, the `Wind Fox / COMMON` row, count `1`, and the separate
SQL practice wallet. Save as `04-sql-proof.png`.

### Screenshot 4 — Persistence / ทดสอบข้อมูลหลัง restart และสร้าง container ใหม่

These are safe to repeat: they only restart/recreate containers and read data.
They do **not** recharge, exchange, draw, delete the volume, or remove your row.
Run the commands in order and capture the output at each stage:

```powershell
$before = (docker compose ps -q db).Trim()
docker inspect --format '{{json .Mounts}}' $before
```

Save the first DB container ID and the volume name (`gacha-6788185_pgdata`).
Then restart:

```powershell
docker compose restart
docker compose up -d --wait
docker compose ps -q db
Invoke-RestMethod http://localhost:8085/api/accounts/6788185/history
docker compose exec -T db psql -U gacha -d gacha -c "SELECT id, student_id, result_name, rarity, created_at FROM draws WHERE student_id='6788185' ORDER BY id;"
```

The restart normally keeps the same DB container ID. Save as
`05-after-restart.png`. Next, replace the containers but keep their volume:

```powershell
docker compose down
docker compose up -d --wait
$after = (docker compose ps -q db).Trim()
$after
docker inspect --format '{{json .Mounts}}' $after
Invoke-RestMethod http://localhost:8085/api/accounts/6788185/history
docker compose exec -T db psql -U gacha -d gacha -c "SELECT id, student_id, result_name, rarity, created_at FROM draws WHERE student_id='6788185' ORDER BY id;"
```

Capture the new DB container ID, same volume name, and unchanged original draw.
The draw should still be ID `1`, `Wind Fox`, `COMMON`, with the same timestamp.
Save as `06-after-down-up.png`. Never add `-v` to `docker compose down`.

### Finish the report / ทำรายงานให้เสร็จ

Add these screenshots under sections 01–04, read the explanations already in
those sections, and fill in the LA demonstration summary after you show the
experiment to the LA. Export one readable PDF named
`6788185_lab12.PDF` and upload it to MyCourse. Keep the Compose stack and volume
until you have finished capturing evidence.

## 01 · Folder, environment and readiness / โฟลเดอร์ environment และความพร้อม

The workshop folder contains `compose.yml` for the app, database, and volume;
`compose.override.yml` adds the local app port; `Dockerfile` builds the Node app;
`server.js`, `app.js`, and `domain.js` contain the app code; `public/test.html`
provides the API test page; `.env.example` is a safe starter configuration; and
`db/init.sql` creates the initial database tables. I copied `.env.example` to
`.env` and set `STUDENT_ID=6788185`. I kept the local password out of this report.

Docker and Compose versions:

```text
Docker Client: 29.8.2
Docker Server: 29.8.2 (Docker Desktop 4.95.0)
Docker context: desktop-linux
Docker Compose: v5.6.0
```

I ran `docker compose config --quiet`; it passed. `docker compose config --services`
listed `db` and `app`. These commands check the resolved Compose configuration;
they do not start containers. Compose uses `.env` to substitute values in the
YAML, such as the project name and database settings. A value is passed into a
container only when it is listed under that service's `environment:` section.
The password is a disposable local demo value and is not included here.

Evidence and explanation / หลักฐานและคำอธิบาย:

> Add screenshots from your own screen showing the folder, Docker versions,
> sanitized `.env` (hide the password), and `config --quiet` /
> `config --services` output.

## 02 · Compose keys and startup / Key และการเริ่มระบบ

The Compose file defines two services: `app` and `db`. `build: .` builds the app
image using the current workshop folder as its build context. `image` selects
the PostgreSQL image. The override maps the laptop's `127.0.0.1:8085` to app
port `3000`. `environment` passes the app's port and PostgreSQL connection
settings into the containers. `depends_on` waits for the database health check
before starting the app. The app health check calls `/health`, which checks the
database, and the database health check uses `pg_isready`. `volumes` stores the
database files and mounts the initial SQL file.

I ran these commands from `gacha-workshop/gacha-workshop`:

```powershell
docker compose config --quiet
docker compose config --services
docker compose up -d --build --wait
docker compose ps
docker compose logs --tail 30 app db
Invoke-RestMethod http://localhost:8085/health
```

Both `app` and `db` started healthy. The health response was:

```json
{"status":"ok","database":"connected","app":"gacha"}
```

I also checked it directly in PowerShell:

```powershell
Invoke-RestMethod http://localhost:8085/health
```

PowerShell displayed `status: ok`, `database: connected`, and `app: gacha`.

The app log said `Gacha API ready; open /test.` The connection path is:

```text
Laptop browser → http://localhost:8085 → app container port 3000
App container → db:5432 → PostgreSQL data in pgdata
```

The database has no published host port. The app uses the Compose service name
`db` to reach PostgreSQL on the internal network.

Evidence and explanation / หลักฐานและคำอธิบาย:

> Add your own screenshots of the build/start commands, healthy `docker compose
> ps`, useful logs, health response, and the connection diagram above.

## 03 · API and basic SQL / API และ SQL พื้นฐาน

I used student ID `6788185`. The wallet started at 0 coins and 0 gems. I checked
Health and Catalog, then made one recharge, one exchange, and one draw:

1. Recharged 1000 virtual coins. The wallet had 1000 coins and 0 gems.
2. Exchanged 1000 coins for 10 gems. The wallet had 0 coins and 10 gems.
3. Drew once. The wallet had 0 coins and 0 gems.
4. Read the saved history.

The PowerShell POST requests returned successful JSON responses. The API's
successful write status is HTTP 201. The saved transactions were:

| Transaction ID | Type | Coin change | Gem change |
| --- | --- | ---: | ---: |
| 1 | recharge | +1000 | 0 |
| 2 | exchange | -1000 | +10 |
| 3 | draw | 0 | -10 |

The actual draw was:

| Draw ID | Result | Rarity | Created at (UTC) |
| --- | --- | --- | --- |
| 1 | Wind Fox | COMMON | 2026-11-05T13:37:28.394Z |

The API history returned this same draw and three transactions: recharge, exchange,
and draw. The SQL result matched:

```text
id | student_id | result_name | rarity | created_at
1  | 6788185    | Wind Fox    | COMMON | 2026-11-05 13:37:28.394212+00
draw_count: 1
```

The later read-only checks returned HTTP 200 for both `/health` and the account
history route. History contained the same draw and all three transactions; I used
GET only during the persistence checks.

I connected to PostgreSQL from the database container:

```powershell
docker compose exec -T db psql -U gacha -d gacha -c '\conninfo'
docker compose exec -T db psql -U gacha -d gacha -c '\dt'
docker compose exec -T db psql -U gacha -d gacha -c "SELECT id, student_id, result_name, rarity, created_at FROM draws WHERE student_id='6788185' ORDER BY id;"
docker compose exec -T db psql -U gacha -d gacha -c "SELECT COUNT(*) AS draw_count FROM draws WHERE student_id='6788185';"
```

`\conninfo` confirmed database `gacha`, user `gacha`, and port `5432`. `\dt`
listed `draws`, `transactions`, and `wallets`. For the separate INSERT practice,
I used the different ID `6788185-sql`, leaving my proof wallet and draw alone:

```sql
INSERT INTO wallets(student_id) VALUES ('6788185-sql')
ON CONFLICT DO NOTHING;
SELECT student_id, coins, gems FROM wallets WHERE student_id='6788185-sql';
```

The practice wallet was created with 0 coins and 0 gems.

The chance of an SSR/legendary result is 2% for each independent draw. There are
five possible SSR results, each with a 0.4% chance per draw. A small number of
draws does not guarantee an SSR. COMMON is a valid result, so I recorded the
result I actually got rather than drawing again to seek a different one.

Evidence and explanation / หลักฐานและคำอธิบาย:

> The commands and API/SQL results above are from the actual run. Add readable
> screenshots from your own screen showing the PowerShell POST request/response
> bodies, the GET URL/status/JSON (or the equivalent `/test` page), and SQL
> output. Keep the password hidden.

## 04 · Persistence and bind mounts / ข้อมูลถาวรและ bind mount

I recorded the same draw before and after each lifecycle step. I did not recharge,
exchange, draw, or insert another proof row during these checks.

| Stage / ช่วง | DB container ID | Named volume | Original draw | Result + rarity + created_at |
| --- | --- | --- | --- | --- |
| Initial / เริ่มต้น | `f5275445ecfb7b5004f74a30d802c3475773d66ebabe66eaee1f805764bd0606` | `gacha-6788185_pgdata` | 1 | Wind Fox, COMMON, 2026-11-05T13:37:28.394Z |
| Restart / หลัง restart | `f5275445ecfb7b5004f74a30d802c3475773d66ebabe66eaee1f805764bd0606` | `gacha-6788185_pgdata` | 1 | Wind Fox, COMMON, 2026-11-05T13:37:28.394Z |
| Down/up / หลังสร้างใหม่ | `c5d02639f8110f0481a4746073831c2ec8351a55580fc049db7e59feae287e4d` | `gacha-6788185_pgdata` | 1 | Wind Fox, COMMON, 2026-11-05T13:37:28.394Z |

After restart, the DB container ID stayed the same. After `docker compose down`
and `docker compose up -d --wait`, the DB container ID changed, but the named
volume and API/SQL row stayed the same. PostgreSQL displayed the timestamp in
UTC with microsecond precision; it is the same instant as the API timestamp.

The two mounts are:

```text
pgdata:/var/lib/postgresql
./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
```

The first is a Docker-managed named volume for PostgreSQL data. The second is a
read-only bind mount from the host file
`gacha-workshop/gacha-workshop/db/init.sql`. `Test-Path .\db\init.sql -PathType
Leaf` returned `True`. `docker inspect` showed the bind source ending in
`gacha-workshop/gacha-workshop/db/init.sql`, `Type=bind`, and `RW=false`; it also
showed the named volume `gacha-6788185_pgdata` mounted at `/var/lib/postgresql`.

One real setup issue was using the `express` folder as the build context for the
gacha Dockerfile. That folder does not contain gacha files such as `app.js` or
`public/`, so the Docker build failed with `"/public": not found`. The fix was to
run Compose from the extracted `gacha-workshop/gacha-workshop` folder, where the
Dockerfile and its source files actually live.

I also changed `POSTGRES_USER` and `POSTGRES_DB` to `gacha-ball` after the named
volume had already been initialized. The app then failed because that existing
database still had the original `gacha` role and database; changing `.env` does
not recreate database roles inside an initialized volume. I restored the workshop
template settings (`gacha`), kept student ID `6788185`, and ran
`docker compose up -d --force-recreate --wait`. This recreated the containers
without deleting the volume, and both services became healthy again. The
original draw was still present. I also used the wrong host port once:
`localhost:8082` is the optional Express notes example; the gacha workshop uses
`localhost:8085`.

More generally, a missing or mistyped bind source can resolve to the wrong
file/folder, and mounting a host folder over `/app` can hide application files
and installed dependencies.

PostgreSQL's image entrypoint runs `db/init.sql` only when the data directory is
empty. Separately, this workshop's `server.js` reads the SQL copied into the app
image and executes it at every app startup. `CREATE TABLE IF NOT EXISTS` creates
missing tables; it does not add columns to a table that already exists. Existing
schema changes need deliberate migrations. Changing app-copied SQL requires
rebuilding the app image, and its statements can still run against existing data.

`docker compose down` removes the containers but keeps the named volume.
`docker compose down -v` also deletes that volume. A volume helps data survive
container replacement, but it is not a backup.

Evidence and explanation / หลักฐานและคำอธิบาย:

> Add screenshots from your own screen showing the restart and down/up commands,
> both DB container IDs, mount inspection, and the unchanged History/SQL row.

## In-room LA check / ตรวจ LA ในห้อง

Demonstration summary / สรุปสิ่งที่แสดง:

> Complete this after showing the working experiment to the LA. Record what you
> demonstrated and any correction the LA asked you to make.

## Optional practice / ฝึกเพิ่มเติมทางเลือก

The repository also has Express, FastAPI, and NestJS notes API examples. They
expose `POST /notes`, `GET /notes`, and `GET /health` on ports 8082, 8083, and
8084. I did not run these optional examples for this report; the required gacha
experiment is complete.

## Final check / ตรวจสอบก่อนส่ง

- [x] Required technical sections 01–04 contain results from the gacha run.
- [ ] Add readable screenshots from my own screen.
- [x] The same original draw survived restart and down/up; I did not replace it.
- [x] Repository URL and sanitized configuration are included.
- [ ] Show the experiment to the LA and record the demonstration summary.
- [ ] Export one PDF named `6788185_lab12.PDF`, check readability, and upload it
      to MyCourse.
