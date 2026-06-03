# Reglas Globales del Agente — BootCamp AI Data Engineering

## Regla operativa para Labs (Lab-0, Lab-1, ...)
Cuando se te pida preparar cualquier lab en este bootcamp, debes:
1. Leer el archivo `mini-data-platform/dossier-minidataplatform.md` como única fuente de verdad para puertos, credenciales y relaciones entre contenedores.
2. Antes del trabajo del lab (especialmente antes del notebook y cualquier operación contra PostgreSQL), asegurarte de que el entorno Docker Compose esté levantado y saludable usando el compose `mini-data-platform/airflow.docker-compose.yml`.
3. Respetar coherencia de credenciales: `PG_PASSWORD` (externo) debe coincidir con `POSTGRES_PASSWORD` (interno) y mantener la coherencia con `SHARED_PASSWORD`/`SUPERSET_PASSWORD`/`ADMIN_PWD`.
4. Para PostgreSQL compartida, usa el servicio y puerto definidos en el dossier.

## Integración con skills de Labs
Si el objetivo es preparar/ejecutar un lab, debes cargar la skill correspondiente (de forma on-demand) para obtener el flujo específico del lab:
- Lab-0 → skill `lab-0`
- Lab-1 → skill `lab-1`
- Lab-2 → skill `lab-2`

La skill complementa las reglas operativas anteriores; no reemplaza la fuente de verdad del dossier.
