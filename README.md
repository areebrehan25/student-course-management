# Student Course Management System

**DevOps Fundamentals — Assignment 03 (Docker Compose, Volumes, Networks & Multi-Container Applications)**

| Field | Value |
|---|---|
| Student Name | Areeb Rehan |
| Roll Number | L1S22BSCS0025 |
| Semester | Spring 2026 |

---

## 1. Overview

This assignment containerises a three-tier database administration environment for a Student
Course Management System using a single `docker-compose.yml` file. Docker Compose owns and
manages **every** resource in the stack — the custom bridge network, the named volume, and all
three services:

| Resource | Name |
|---|---|
| PostgreSQL container | `postgres-std-L1S22BSCS0025` |
| pgAdmin container | `pgadmin-std-L1S22BSCS0025` |
| Adminer container | `adminer-std-L1S22BSCS0025` |
| Docker Network | `std-net-L1S22BSCS0025` (bridge) |
| Docker Volume | `std-volume-L1S22BSCS0025` (local) |

No network or volume was created manually via the CLI. Both are declared in the top-level
`networks:` and `volumes:` sections of `docker-compose.yml`, each with an explicit `name:` field
so the actual Docker resource name matches the required naming convention exactly (Compose would
otherwise prefix resource names with the project/folder name, `student-course-management_...`).

## 2. Project Structure

```
student-course-management/
├── docker-compose.yml      ← main Compose configuration (owns all resources)
├── .env                    ← environment variables (credentials, ports, image tag)
├── README.md                ← this file
└── screenshots/
    ├── task1/ ... task8/    ← one folder per task, per the screenshot checklist
```

## 3. Design Decisions

### 3a. Compose-managed network & volume (Task 2)
The network and volume are declared inline in `docker-compose.yml` rather than pre-created with
`docker network create` / `docker volume create`. This means:
- `docker compose up` creates them automatically on first run.
- `docker compose down` (without `-v`) cleanly removes the network but **preserves** the volume.
- The whole environment is fully self-contained and reproducible from one file — nothing external
  has to exist beforehand.

`external: true` was **not** used, since that pattern only applies when multiple independent
Compose projects need to share one pre-existing resource (not the case here).

### 3b. Healthcheck + `depends_on: condition: service_healthy` (Task 3 bonus)
The `postgres` service defines a healthcheck using `pg_isready`. Both `pgadmin` and `adminer`
declare:
```yaml
depends_on:
  postgres:
    condition: service_healthy
```
This guarantees pgAdmin and Adminer only start **after** PostgreSQL is actually ready to accept
connections, not merely after its container process has started. This was verified in the actual
`docker compose up -d` run — see `task7_compose_up.txt` in the evidence log, where Postgres
reaches `Healthy` before Adminer/pgAdmin proceed.

### 3c. Environment variables via `.env` (Task 6)
All credentials (`POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`,
`PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`) plus port numbers and the Postgres image
tag are defined in `.env` and referenced in `docker-compose.yml` via `${VARIABLE}` substitution.
No credential is hardcoded in the Compose file.

## 4. How to Run

```bash
# from the student-course-management/ directory
docker compose up -d          # start all 3 services
docker compose ps             # list compose-managed containers
docker ps                     # list all containers system-wide

docker network ls             # confirm std-net-L1S22BSCS0025 exists
docker volume ls              # confirm std-volume-L1S22BSCS0025 exists

docker compose down           # stop & remove containers/network, KEEP the volume
docker compose down -v        # (only if you want a full reset, destroys the volume too)
```

- pgAdmin: http://localhost:5050 (login: value of `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD`)
- Adminer: http://localhost:8080 (System: PostgreSQL, Server: `postgres`, User/Password/DB: from `.env`)

## 5. Verification Performed

### 5a. `docker compose ps` / `docker ps`
All three containers (`postgres-std-L1S22BSCS0025`, `pgadmin-std-L1S22BSCS0025`,
`adminer-std-L1S22BSCS0025`) came up successfully, with Postgres reporting `(healthy)`.

### 5b. Network inspection — explanation
`docker network inspect std-net-L1S22BSCS0025` shows:
- **Driver: bridge**, exactly as declared, confirming Compose created a custom bridge network
  (not the default `bridge` network).
- An **IPAM subnet** (`172.18.0.0/16`) automatically assigned by Docker, from which each
  container received a private IP (Postgres `172.18.0.3`, Adminer `172.18.0.2`, pgAdmin
  `172.18.0.4`).
- A **`Containers`** map listing all three service containers attached to this one network,
  which is what allows them to reach each other by container name/IP.
- **Labels** (`com.docker.compose.project`, `com.docker.compose.network`, etc.) proving the
  network is Compose-managed, not manually created — Compose stamps these labels onto every
  resource it owns so `docker compose down` knows exactly what to clean up.
- This confirms Compose's embedded DNS + bridge networking is what lets pgAdmin/Adminer reach
  Postgres using the service name `postgres` as hostname, never a hardcoded IP.

### 5c. Volume inspection — explanation
`docker volume inspect std-volume-L1S22BSCS0025` shows:
- **Driver: local**, the default driver, storing data on the host filesystem.
- **Mountpoint**: `/var/lib/docker/volumes/std-volume-L1S22BSCS0025/_data` — the actual host-side
  directory backing the volume, independent of any single container's lifecycle.
- **Labels** again show `com.docker.compose.project`/`com.docker.compose.volume`, confirming
  Compose owns and tracks this volume.
- Because this Mountpoint lives outside any container, removing/recreating the `postgres`
  container (e.g. via `docker compose down` + `up`) does **not** delete the actual PostgreSQL
  data files — only `docker compose down -v` or `docker volume rm` would.

### 5d. Inter-container DNS resolution
Ran `docker exec adminer-std-L1S22BSCS0025 getent hosts postgres` from inside the Adminer
container — it resolved the service name `postgres` to the Postgres container's internal IP
(`172.19.0.2`), confirming Compose's embedded DNS resolver works as expected and no hardcoded
IPs are needed for inter-service communication.

### 5e. Data persistence (Task 8b)
1. Created the `student` table and inserted 3 rows via a GUI tool (pgAdmin Query Tool / Adminer
   SQL command), per Task 8a — **not** via any SQL script file.
2. Ran `docker compose down` (no `-v` flag) — all three containers and the network were removed;
   `docker ps` confirmed no running containers, while `docker volume ls` still showed
   `std-volume-L1S22BSCS0025`.
3. Ran `docker compose up -d` again — all three services came back up cleanly (Compose recreated
   the network from its declaration; the volume was reused, not recreated).
4. Reconnected via pgAdmin/Adminer and ran `SELECT * FROM student;` — all 3 previously inserted
   rows were still present, proving the named volume persisted the PostgreSQL data directory
   across the full container-removal / recreation cycle.

## 6. Key Takeaway

The core skill this assignment tests is knowing that `docker compose down` (no flag) removes
containers + networks but **keeps** named volumes, while `docker compose down -v` destroys the
volume as well. Only the former was used during the persistence test in Task 8b, which is exactly
why the `student` table's data survived a full stack teardown and restart.
