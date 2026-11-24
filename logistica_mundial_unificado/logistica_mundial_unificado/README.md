# Logística Mundial

Aplicación de logística: **Angular** (frontend) + **Node.js/Express** (API) + **MySQL**.
Lista para ejecutarse con Docker Compose y para desplegarse en **Red Hat OpenShift Service on AWS (ROSA)**.

## Estructura

| Carpeta | Contenido |
|---|---|
| `frontend/` | Angular 20, servido por Nginx sin privilegios (puerto 8080) que reenvía `/api` al backend |
| `backend/` | API REST (Express + mysql2) en el puerto 3000 |
| `database/init.sql` | Tablas, datos de demostración y vista SQL |
| `deploy/openshift/` | Manifiestos: builds desde GitHub, MySQL, backend, frontend, Route y Job de inicialización |
| `deploy/secret.example.yaml` | Plantilla del Secret (sin claves reales) |
| `documentacion/referencia-react/` | Panel React original, solo como referencia (no se ejecuta) |
| `docker-compose.yml` | Entorno local completo |

## Ejecución local (Docker)

```bash
cp .env.example .env        # y cambia las contraseñas
docker compose up --build
```

- App: http://localhost:8080 · API: http://localhost:3000/api · Salud: http://localhost:3000/api/health
- Reiniciar la base desde cero (borra datos): `docker compose down -v && docker compose up --build`

## Ejecución local sin Docker

Requisitos: Node.js 20+, MySQL 8+.

```bash
# 1) Base de datos
mysql -u root -p < database/init.sql
# 2) API
cd backend && cp .env.example .env && npm install && npm run dev
# 3) Frontend (otra terminal)
cd frontend && npm install && npm start     # http://localhost:4200
```

## Subir a GitHub

```bash
git init && git add . && git commit -m "Logística Mundial"
git branch -M main
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git
git push -u origin main
```

Antes de subir: genera y sube los `package-lock.json` (`npm install` dentro de `frontend/` y `backend/`) para builds reproducibles. Nunca subas `.env` ni `deploy/openshift/secret.yaml` (ya están en `.gitignore`).

## Desplegar en OpenShift (ROSA)

Requisitos: `oc` instalado, sesión iniciada (`oc login ...`) y el repositorio en GitHub.

1. **Proyecto**
   ```bash
   oc new-project logistica-mundial
   ```
2. **Repositorio**: edita `deploy/openshift/01-builds.yaml` y cambia `https://github.com/TU_USUARIO/TU_REPOSITORIO.git` por tu URL (si es privado, crea el `github-token` indicado en el archivo y descomenta `sourceSecret`).
3. **Secret de base de datos** (elige contraseñas propias):
   ```bash
   oc create secret generic logistica-db \
     --from-literal=MYSQL_ROOT_PASSWORD='CAMBIAR' \
     --from-literal=MYSQL_DATABASE='logistica_mundial' \
     --from-literal=MYSQL_USER='logistica' \
     --from-literal=MYSQL_PASSWORD='CAMBIAR'
   ```
4. **SQL de inicialización** (alternativa: `oc apply -f deploy/logistica-all-in-one.yaml` ya lo incluye todo):
   ```bash
   oc create configmap logistica-init-sql --from-file=init.sql=database/init.sql
   ```
5. **Aplicar manifiestos y compilar**:
   ```bash
   oc apply -f deploy/openshift/
   oc start-build backend --follow
   oc start-build frontend --follow
   ```
6. **Verificar y abrir**:
   ```bash
   oc get pods
   oc logs job/db-init
   oc get route frontend -o jsonpath='https://{.spec.host}{"\n"}'
   ```

Los Deployments se actualizan solos cuando termina un build nuevo. Para actualizar la app: `git push` y luego `oc start-build backend` / `oc start-build frontend`.

## Antes de producción

- No hay autenticación/autorización en la API: añade login (por ejemplo OpenShift OAuth proxy o JWT) antes de exponerla a usuarios reales.
- `database/init.sql` carga datos de demostración; el Job `db-init` debe ejecutarse una sola vez.
- MySQL corre como un único pod con un volumen de 5 Gi: configura copias de seguridad, o usa Amazon RDS para MySQL en producción.
- Ajusta réplicas, recursos y la clase de almacenamiento según tu clúster.
