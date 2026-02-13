# PostgreSQL Reset Superuser Password (Windows) — Forgot Password Method

## Reminder
Use this method if:
- Forgot the `postgres` password
- Cannot login using pgAdmin or psql

---

## 1. Find pg_hba.conf

```sh
C:\Program Files\PostgreSQL<version>\data\pg_hba.conf
```

---

## 2. Temporarily Disable Password Authentication
- Open **pg_hba.conf** using Notepad (Run as Administrator).
- Find line and change to

```sh
local   all   postgres   md5

change to

local   all   postgres   trust
```

- What This Does
  - `md5` → Requires password  
  - `trust` → Allows login without password (Temporary only)

---

## 3. Restart PostgreSQL Service
- Service Name: postgresql-x64-<version>
  - Note: Version number may differ (e.g. 13, 15, 16)

---

## Step 4 — Login and Reset Password

Open Command Prompt:
- Type `psql -U postgres`
- Type `ALTER USER postgres WITH PASSWORD 'NewPassword';`

---

## Step 5 — Restore Security (VERY IMPORTANT)

Edit **pg_hba.conf** again.

Change back:

```sh
trust → md5   (or scram-sha-256)
```

---

## Step 6 — Restart PostgreSQL Again

