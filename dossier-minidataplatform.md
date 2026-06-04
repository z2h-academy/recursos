# Dossier Técnico — mini-data-platform

## 1. Visión General

Plataforma de datos local basada en Docker Compose que orquesta servicios para ingesta, almacenamiento object storage, visualización y monitoreo. El único archivo de despliegue es **`airflow.docker-compose.yml`**. Las configuraciones sensibles se externalizan en archivos `.env` dentro de `config/`.

Estructura actual del proyecto:

```
mini-data-platform/
├── .env
├── .gitignore
├── airflow.docker-compose.yml
└── config/
    ├── airflow_container.env
    ├── airflow_database.env
    ├── jupyter.env          # No referenciado en el compose (legacy)
    ├── minio.env
    ├── pgadmin.env
    ├── postgres.env
    ├── shared_database.env
    ├── superset_container.env
    └── superset_database.env
```

> **Nota:** Los directorios `dags/`, `logs/`, `notebooks/`, `plugins/`, `Proyectos/` y `python_scripts/` fueron eliminados. Docker los recrea automáticamente al iniciar los servicios si no existen.

---

## 2. Servicios Definidos

| Servicio | Imagen | Container_name | Puerto(s) | Propósito |
|---|---|---|---|---|
| `postgres` | `postgres:13` | — | — | Base de datos del scheduler de Airflow (CeleryResultBackend + metadata) |
| `postgres-2` | `3xploiterx/postgres:13` | `postgres-container` | `5432:5432` | Base de datos compartida (DAGs, Superset, datos de aplicación) |
| `redis` | `redis:latest` | — | `6379` (expose) | Broker de Celery |
| `airflow-webserver` | `z2hx/airflow` | — | `7777:8080` | UI de Airflow |
| `airflow-scheduler` | `z2hx/airflow` | — | — | Scheduler de DAGs |
| `airflow-worker` | `z2hx/airflow` | — | — | Ejecutor Celery Worker |
| `airflow-triggerer` | `z2hx/airflow` | — | — | Triggerer para DAGs deferrables |
| `airflow-init` | `z2hx/airflow` | — | — | One-shot: migraciones + creación usuario admin |
| `airflow-cli` | `z2hx/airflow` | — | — | CLI (perfil `debug`) |
| `flower` | `z2hx/airflow` | — | `5555:5555` | Monitoreo Celery (perfil `flower`) |
| `minio` | `quay.io/minio/minio:RELEASE.2022-08-26T19-53-15Z` | `minio` | `9000:9000` (API), `9001:9001` (Console) | Almacenamiento S3-compatible |
| `pgadmin` | `dpage/pgadmin4` | `pgadmin` | `5050:80` | Administración visual de PostgreSQL |

---

## 3. Dependencias entre Servicios

```
postgres ──┐
           ├── redis ──┐
           │           │
    postgres-2         │
           │           │
           ▼           ▼
    ┌───── airflow-init ─────┐
    │         │              │
    ▼         ▼              ▼
airflow-webserver / scheduler / worker / triggerer
    │
    ▼
  flower (perfil opcional flower)
```

- `airflow-init` debe completarse con éxito (`condition: service_completed_successfully`) antes de que webserver/scheduler/worker/triggerer se inicien.
- `postgres` y `redis` requieren `service_healthy` como condición para todos los servicios Airflow.
- `postgres-2`, `minio` y `pgadmin` son independientes y no tienen dependencias declaradas.

---

## 4. Archivos de Configuración (`config/`)

### 4.1. `airflow_container.env`
```env
NEWS_DATA=/opt/airflow/notebooks
```
Ruta interna para notebooks dentro del contenedor Airflow.

### 4.2. `airflow_database.env`
```env
AIRFLOW_PASSWORD=airflow
```
Password del usuario `airflow` en `postgres-2`.

### 4.3. `minio.env`
```env
MINIO_ROOT_USER=minio-access-key
MINIO_ROOT_PASSWORD=minio-secret-key
```
Access key y secret key de administrador de MinIO.

### 4.4. `postgres.env`
```env
POSTGRES_PASSWORD=changeme1234
SHARED_PASSWORD=changeme1234
POSTGRES_USER=postgres
POSTGRES_DB=postgres
```
Superusuario de `postgres-2`. Incluye `SHARED_PASSWORD` para que el contenedor reconozca la variable compartida.

### 4.5. `pgadmin.env`
```env
PGADMIN_DEFAULT_EMAIL=pgadmin4@pgadmin.org
PGADMIN_DEFAULT_PASSWORD=admin
```
Credenciales de ingreso a pgAdmin.

### 4.6. `shared_database.env`
```env
SHARED_PASSWORD=changeme1234
```
**Fuente única de verdad** para la contraseña compartida entre bases.

### 4.7. `superset_container.env`
```env
ADMIN_PWD=changeme1234
SUP_META_DB_URI=postgres://superset:changeme1234@postgres/superset
```
Password del admin de Superset + URI de metadatos que apunta a `postgres-2`.

### 4.8. `superset_database.env`
```env
SUPERSET_PASSWORD=changeme1234
```
Password del usuario `superset` dentro de `postgres-2`.

### 4.9. `jupyter.env` (legacy — no referenciado)
```env
JUPYTER_ENABLE_LAB=yes
JUPYTER_PASSWORD=xploiter
NEWS_DATA=/home/jovyan/work
```
No hay servicio Jupyter en el compose actual. Se mantiene por si se reincorpora.

---

## 5. Archivo `.env` (Raíz)

```env
AIRFLOW_UID=1000

PG_USER=postgres
PG_PASSWORD=changeme1234
PG_HOST=postgres-container
PG_PORT=5432
PG_DB=ny_taxi
PG_TABLE_NAME=yellow_taxi_data
PG_URL=d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet

LOCAL_PARQUET_FILE_PATH=/opt/airflow/Proyectos/input
```

Variables de conexión a `postgres-2` para scripts externos. `PG_PASSWORD` debe coincidir con `POSTGRES_PASSWORD` de `config/postgres.env`.

---

## 6. Mapeo de Variables Críticas — Fuente Única de Verdad

| Concepto | Variable | Archivo Fuente | Servicios que la consumen |
|---|---|---|---|
| Password shared (cross-db) | `SHARED_PASSWORD` | `config/shared_database.env` | postgres-2, Superset |
| Password superuser postgres | `POSTGRES_PASSWORD` | `config/postgres.env` | postgres-2 |
| Password airflow db | `AIRFLOW_PASSWORD` | `config/airflow_database.env` | postgres-2 |
| Password superset db | `SUPERSET_PASSWORD` | `config/superset_database.env` | postgres-2 |
| Password admin superset | `ADMIN_PWD` | `config/superset_container.env` | Superset |
| MinIO access key | `MINIO_ROOT_USER` | `config/minio.env` | minio |
| MinIO secret key | `MINIO_ROOT_PASSWORD` | `config/minio.env` | minio |
| pgAdmin email / password | `PGADMIN_DEFAULT_*` | `config/pgadmin.env` | pgadmin |
| UID Airflow | `AIRFLOW_UID` | `.env` | airflow-* |
| Conexión PostgreSQL externa | `PG_HOST/PG_PORT/PG_USER/PG_PASSWORD` | `.env` | scripts externos |

**Regla de coherencia:** `POSTGRES_PASSWORD`, `SHARED_PASSWORD`, `SUPERSET_PASSWORD` y `ADMIN_PWD` deben tener el mismo valor (`changeme1234` en entorno actual).

---

## 7. Variables AIRFLOW__ Hardcodeadas en el YAML

| Variable | Valor |
|---|---|
| `AIRFLOW__CORE__EXECUTOR` | `CeleryExecutor` |
| `AIRFLOW__DATABASE__SQL_ALCHEMY_CONN` | `postgresql+psycopg2://airflow:airflow@postgres/airflow` |
| `AIRFLOW__CELERY__RESULT_BACKEND` | `db+postgresql://airflow:airflow@postgres/airflow` |
| `AIRFLOW__CELERY__BROKER_URL` | `redis://:@redis:6379/0` |
| `AIRFLOW__CORE__FERNET_KEY` | `''` |
| `AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION` | `true` |
| `AIRFLOW__CORE__LOAD_EXAMPLES` | `false` |
| `AIRFLOW__API__AUTH_BACKENDS` | `airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session` |
| `AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK` | `true` |

---

## 8. Carga Múltiple de Env Files por Servicio

| Servicio | Env Files Cargados |
|---|---|
| `airflow-webserver`, `scheduler`, `worker`, `triggerer`, `init`, `cli`, `flower` | `airflow_container.env` + `minio.env` + `shared_database.env` |
| `postgres-2` | `postgres.env` + `superset_database.env` + `airflow_database.env` + `shared_database.env` |
| `minio` | `minio.env` |
| `pgadmin` | `pgadmin.env` |

No hay colisión de nombres entre variables de distintos archivos.

---

## 9. Conexiones entre Servicios

| Origen | Destino | Propósito |
|---|---|---|
| airflow-* | `postgres:5432` | Metadata / Celery result backend |
| airflow-* | `redis:6379` | Celery broker |
| airflow-* | `postgres-container:5432` | Operaciones de DAG via SQLAlchemy |
| minio | — | API S3 `:9000`, Console `:9001` |
| pgadmin | `postgres-container:5432` | Admin UI (configuración manual del usuario) |

---

## 10. Puertos Expuestos al Host

| Puerto Host | Servicio |
|---|---|
| `5432` | postgres-2 (PostgreSQL compartida) |
| `7777` | airflow-webserver |
| `5050` | pgadmin |
| `9000` | minio (API S3) |
| `9001` | minio (Console UI) |
| `5555` | flower (solo perfil `flower`) |

---

## 11. Volúmenes Persistidos

| Volumen | Monta en | Ruta |
|---|---|---|
| `postgres-db-volume` | postgres | `/var/lib/postgresql/data` |
| `postgres_volume` | postgres-2 | `/var/lib/postgresql/data/` |
| `minio_volume` | minio | `/data` |
| `pgadmin_volume` | pgadmin | `/var/lib/pgadmin` |

---

## 12. Perfiles Opcionales

- `debug` → `airflow-cli`
- `flower` → `flower` (monitoreo Celery, puerto `5555:5555`)

---

## 13. Recomendaciones

1. **No hardcodear credenciales en scripts** — usar siempre `.env` para scripts externos y `config/*.env` para servicios Docker.
2. **`PG_PASSWORD` en `.env` debe coincidir** con `POSTGRES_PASSWORD` en `config/postgres.env`.
3. Al cambiar la contraseña compartida, actualizar **simultáneamente**: `shared_database.env`, `postgres.env`, `superset_database.env` y `superset_container.env`.
4. `jupyter.env` es legacy — no está referenciado en el compose pero se conserva por si se añade Jupyter más adelante.
5. Para adjuntar un nuevo servicio a la base compartida, cargar `shared_database.env` (obtiene `SHARED_PASSWORD`) y usar `PG_HOST=postgres-container`.
6. Para adjuntar un nuevo servicio a MinIO, cargar `minio.env` (obtiene access/secret key).
