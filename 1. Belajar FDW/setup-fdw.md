# Belajar FDW

### 1. setup source db dulu

pertama buat role di db source
```sql
CREATE ROLE usr_app_db_master LOGIN PASSWORD 'pass_master';
CREATE ROLE usr_fdw_db_master LOGIN PASSWORD 'pass_fdw';
```


cek owner database
```sql
-- cek owner database
ALTER DATABASE db_master OWNER TO usr_app_db_master;

SELECT datname, pg_get_userbyid(datdba) AS owner
FROM pg_database
WHERE datname = 'db_master';

SELECT datname, pg_get_userbyid(datdba) AS owner FROM pg_database;
-- end
```

grant user untuk fdw agar digunakan buat reader di source
```sql
GRANT CONNECT ON DATABASE db_master TO usr_fdw_db_master;
GRANT USAGE ON SCHEMA public TO usr_fdw_db_master;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO usr_fdw_db_master;

-- tabel baru yang dibuat app_master otomatis bisa dibaca usr_fdw_db_master

ALTER DEFAULT PRIVILEGES FOR ROLE usr_app_db_master IN SCHEMA public
GRANT SELECT ON TABLES TO usr_fdw_db_master;	
```

(optional) kalau object bukan pemilik user source
```sql
DO $$
DECLARE r record;
BEGIN
  -- tabel
  FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = 'public' LOOP
    EXECUTE format('ALTER TABLE public.%I OWNER TO usr_app_db_master', r.tablename);
  END LOOP;
  -- sequence (SERIAL membuat sequence terpisah)
  FOR r IN SELECT sequencename FROM pg_sequences WHERE schemaname = 'public' LOOP
    EXECUTE format('ALTER SEQUENCE public.%I OWNER TO usr_app_db_master', r.sequencename);
  END LOOP;
  -- view
  FOR r IN SELECT viewname FROM pg_views WHERE schemaname = 'public' LOOP
    EXECUTE format('ALTER VIEW public.%I OWNER TO usr_app_db_master', r.viewname);
  END LOOP;
END $$;
```

### 2. Setup di db target

mulai setup
```sql
CREATE ROLE usr_app_db_integration LOGIN PASSWORD 'pass_integration';
ALTER DATABASE db_integration OWNER TO usr_app_db_integration;

CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER db_master_server
  FOREIGN DATA WRAPPER postgres_fdw -- <--ini extension
  OPTIONS (host 'db', port '5432', dbname 'db_master');

CREATE USER MAPPING FOR usr_app_db_integration
  SERVER db_master_server
  OPTIONS (user 'usr_fdw_db_master', password 'pass_fdw');

GRANT USAGE ON FOREIGN SERVER db_master_server TO usr_app_db_integration;
```

perhatian kalau db target mau akses db source:
```sql
CREATE USER MAPPING FOR usr_app_db_integration SERVER db_master_server
OPTIONS (user 'usr_fdw_db_master', password 'pass_fdw');

GRANT USAGE ON FOREIGN SERVER db_master_server TO usr_app_db_integration;  -- hanya perlu untuk IMPORT / CREATE FOREIGN TABLE
GRANT SELECT ON users, animals TO usr_app_db_integration;
```

cek server yang telah dibuat
```sql
SELECT srvname, srvoptions FROM pg_foreign_server;
```

cek user mapping yang telah dibuat
```sql
SELECT srvname, usename, umoptions FROM pg_user_mappings;
```

### recap di db target
#### A. Sebagai postgres atau superuser
```sql
-- 1. role app + owner database
CREATE ROLE usr_app_db_integration LOGIN PASSWORD 'pass_integration';
ALTER DATABASE db_integration OWNER TO usr_app_db_integration;

-- 2. extension (butuh superuser)
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- 3. server: alamat master (host harus jalur yang meminta password)
CREATE SERVER db_master_server
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'db', port '5432', dbname 'db_master');

-- 4. user mapping: role lokal -> role remote
CREATE USER MAPPING FOR usr_app_db_integration
  SERVER db_master_server
  OPTIONS (user 'usr_fdw_db_master', password 'pass_fdw');

-- 5. izin memakai server (untuk IMPORT / CREATE FOREIGN TABLE)
GRANT USAGE ON FOREIGN SERVER db_master_server TO usr_app_db_integration;
```

#### B. Sebagai usr_app_db_integration (app user)
```sql
-- 6. import foreign table dari master
IMPORT FOREIGN SCHEMA public FROM SERVER db_master_server INTO public;
```
ini hanya dilakukan sekali jalan (snapshot). jika ada table baru atau struktur baru jalankan berikut

#### C. cek tiap tahap
```sql
SELECT datname, pg_get_userbyid(datdba) AS owner
FROM pg_database WHERE datname = 'db_integration';                    -- owner = usr_app_db_integration

SELECT extname FROM pg_extension WHERE extname = 'postgres_fdw';     -- ada

SELECT srvname, srvoptions FROM pg_foreign_server;                    -- host=db,port=5432,dbname=db_master

SELECT srvname, usename, umoptions FROM pg_user_mappings;             -- ada baris usr_app_db_integration

SELECT has_server_privilege('usr_app_db_integration','db_master_server','USAGE');  -- t

SELECT foreign_table_name FROM information_schema.foreign_tables;     -- users, animals, ...

SELECT * FROM users LIMIT 1;                                          -- koneksi FDW hidup
```


note:
> - lebih banyak eksekusi di db target terkait grant, akses, owner, dsb
> - IMPORT dijalankan oleh role yang punya user mapping

### Kegiatan Developer

#### A. create table
di db source:
```sql
CREATE TABLE foods (id SERIAL PRIMARY KEY, name TEXT NOT NULL);
INSERT INTO foods (name) VALUES ('Whiskas'), ('Pedigree');

-- cek FDW boleh baca
SELECT has_table_privilege('usr_fdw_db_master', 'public.foods', 'SELECT');   -- harus t
```
Kalau `f`:`GRANT SELECT ON foods TO usr_fdw_db_master;`

Di db target:
```sql
SELECT * FROM foods;   -- ERROR: relation "foods" does not exist

IMPORT FOREIGN SCHEMA public LIMIT TO (foods)
  FROM SERVER db_master_server INTO public;

SELECT * FROM foods;   -- 2 baris
```
Cek Terdaftar:
```sql
SELECT foreign_table_name FROM information_schema.foreign_tables
WHERE foreign_table_name = 'foods';
```

#### B. Kolom Baru
di db source:
```sql
ALTER TABLE users ADD COLUMN email TEXT;
UPDATE users SET email = 'fiqih@example.com' WHERE name = 'Fiqih';
```
di db target:\
cara 1
```sql
SELECT email FROM users;   -- ERROR: column "email" does not exist

ALTER FOREIGN TABLE users ADD COLUMN email TEXT;

SELECT id, name, email FROM users;   -- email terbaca
```
cara 2
```sql
DROP FOREIGN TABLE users;
IMPORT FOREIGN SCHEMA public LIMIT TO (users)
    FROM SERVER db_master_server INTO public;
GRANT SELECT ON users TO usr_app_db_integration;
```

#### C. Banyak perubahan sekaligus: resync
Kalau tidak tahu persis apa yang berubah, drop semua foreign table lalu import ulang (tabel lokal seperti pets tidak tersentuh):

```sql
BEGIN;
DO $$
DECLARE r record;
BEGIN
  FOR r IN SELECT foreign_table_name AS t
           FROM information_schema.foreign_tables
           WHERE foreign_table_schema = 'public' LOOP
    EXECUTE format('DROP FOREIGN TABLE public.%I CASCADE', r.t);
  END LOOP;
END $$;
IMPORT FOREIGN SCHEMA public FROM SERVER db_master_server INTO public;
COMMIT;
```


