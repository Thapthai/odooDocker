# Odoo 18 + PostgreSQL (Localhost) คู่มือเริ่มใหม่

คู่มือนี้ใช้สำหรับรัน Odoo ใน Docker และเชื่อม PostgreSQL ที่ติดตั้งอยู่บนเครื่อง Windows (localhost)

## โครงสร้างที่ใช้

- Compose file: `docker/docker-compose.yml`
- Env file: `.env`
- Custom addons: `addons/`

## 1) เตรียมค่าใน `.env`

ตัวอย่างค่าที่ใช้งานได้:

```env
POSTGRES_HOST=host.docker.internal
POSTGRES_PORT=5432
POSTGRES_USER=erptest
POSTGRES_PASSWORD=2076
```

หมายเหตุ:
- Odoo 18 ไม่อนุญาตให้ใช้ DB user ชื่อ `postgres`
- ต้องใช้ user แยก เช่น `erptest` หรือ `odoo`

## 2) เตรียม PostgreSQL บนเครื่อง

เชื่อมเข้า PostgreSQL ด้วยบัญชีแอดมิน แล้วรัน:

```sql
CREATE ROLE erptest WITH LOGIN PASSWORD '2076' CREATEDB;
```

ถ้ามี role อยู่แล้ว:

```sql
ALTER ROLE erptest WITH LOGIN PASSWORD '2076' CREATEDB;
```

## 3) รัน Odoo

รันจาก root โปรเจกต์ (`...\odoo`):

```powershell
docker compose --env-file .env -f docker/docker-compose.yml up -d
```

เปิดใช้งาน:

- http://localhost:8069

## 4) สร้างฐานข้อมูลใหม่ (กรณีจำรหัสเดิมไม่ได้)

เปิด:

- http://localhost:8069/web/database/manager

แล้วสร้าง DB ใหม่ด้วยชื่อใหม่ เช่น `erptest_new`

## 5) ล้างและเริ่มใหม่ทั้งหมด (ทางลัด)

### 5.1 หยุด container

```powershell
docker compose --env-file .env -f docker/docker-compose.yml down
```

### 5.2 ลบฐานเก่าใน PostgreSQL (ถ้าต้องการ)

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'erptest'
  AND pid <> pg_backend_pid();
```

```sql
DROP DATABASE IF EXISTS erptest;
```

### 5.3 สร้างฐานใหม่เองด้วย SQL (optional)

```sql
CREATE DATABASE erptest OWNER erptest TEMPLATE template1;
```

### 5.4 สตาร์ท Odoo อีกครั้ง

```powershell
docker compose --env-file .env -f docker/docker-compose.yml up -d
```

## 6) คำสั่งตรวจสอบเร็ว

```powershell
docker ps -a --filter name=odoo-app
docker logs odoo-app --tail 100
```

## 7) ปัญหาที่เจอบ่อย

- `Using the database user 'postgres' is a security risk, aborting`
  - เปลี่ยน `POSTGRES_USER` เป็น user อื่น เช่น `erptest`

- `Internal Server Error` ที่ `/web/database/manager` หรือ `/web/database/create`
  - มักเกิดจาก DB สร้างค้างหรือไม่ initialize
  - แก้โดยลบ DB ที่มีปัญหาแล้วสร้างใหม่ หรือ initialize ด้วย:

```powershell
docker exec odoo-app odoo -d erptest -i base --stop-after-init --db_host=host.docker.internal --db_port=5432 --db_user=erptest --db_password=2076
```

- `collations with different collate and ctype values`
  - เป็นปัญหา locale ของ DB ที่เสีย
  - ลบ DB ที่เสีย แล้วสร้างใหม่จาก `template1`

## 8) คำแนะนำใช้งานจริง

- dev: ใช้รหัสง่ายได้ แต่ควรจำให้ได้
- production: เปลี่ยนรหัสผ่านให้ปลอดภัย, backup DB สม่ำเสมอ, และอย่าใช้ user สิทธิ์สูงเกินจำเป็น
