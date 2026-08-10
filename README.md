# Actividad-IAM
## Video : 
https://drive.google.com/drive/folders/1_ycWP2WYVu81d7hfb-vPW81mRN3i6sAl?usp=sharing
# Proyecto Micro Seguridad — Módulo IAM

Módulo de **Identidad y Acceso (IAM)** del sistema "Design Software". Se encarga de autenticar usuarios, administrar roles y permisos (RBAC), y controlar quién puede hacer qué dentro de la plataforma.

El proyecto está dividido en 4 partes, cada una en su propio repositorio (aquí vienen comprimidas en `.zip`):

| Carpeta | Qué es | Tecnología |
|---|---|---|
| `design-software-iam-db-back` | API / lógica de negocio (backend) | Go + Gin |
| `design-software-iam-db-front` | Interfaz web (frontend) | Angular 21 |
| `design-software-iam-db` | Scripts de la base de datos | PostgreSQL + Liquibase |
| `docker-infra` | Infraestructura compartida (base de datos y herramientas) | Docker Compose |

---

## 1. ¿Qué hace este módulo?

- **Login / logout** con JWT (token de acceso + refresh token).
- **Recuperación de contraseña** (olvidé mi contraseña / reset por correo).
- **Gestión de usuarios**: crear, listar, editar, bloquear/desbloquear, cambiar estado.
- **Roles y permisos (RBAC)**: crear roles, asignarles "features" (permisos), asignar roles a usuarios.
- **Excepciones puntuales de acceso** ("scope overrides"): dar o quitar un permiso puntual a un usuario sin tocar su rol.
- **Auditoría**: historial de inicios de sesión.
- **Catálogo**: módulos y funcionalidades (features) del sistema, usados para armar los permisos.
- **Dashboards por rol**: el frontend muestra una vista distinta según el rol del usuario (Admin, Coordinador, Director de Centro, Líder de Área, Instructor, Aprendiz, etc.).

---

## 2. Arquitectura general

```
┌────────────────────┐        ┌────────────────────┐        ┌──────────────────────┐
│   Frontend (Angular) │ ───▶ │   Backend (Go/Gin)   │ ───▶ │  PostgreSQL (schemas)  │
│   iam-web  :4200      │      │   iam-service :8001   │      │  identity, rbac, etc.  │
└────────────────────┘        └────────────────────┘        └──────────────────────┘
```

- El **frontend** consume la API del backend en `http://localhost:8001` (configurable).
- El **backend** sigue una arquitectura por capas (hexagonal/clean): `domain` → `application` (casos de uso) → `infrastructure` (Postgres, SMTP, seguridad) → `api` (handlers HTTP).
- La **base de datos** se versiona con **Liquibase** (no se crean las tablas a mano) y usa varios *schemas* de PostgreSQL para separar responsabilidades.

Este módulo (`iam`) es uno de varios módulos del sistema "design-software" (hay otros como academic, actors, audit, document, monitoring, reference, scheduling, training), pero todos comparten la **misma base de datos**, orquestada desde `docker-infra`.

---

## 3. Backend — `design-software-iam-db-back`

**Stack:** Go 1.26, framework [Gin](https://github.com/gin-gonic/gin), driver `lib/pq` para PostgreSQL, JWT con `golang-jwt`.

### Estructura de carpetas (dentro de `iam-service/`)

```
cmd/
  api/          → punto de entrada del servidor HTTP (main.go)
  seed/         → script para crear el usuario admin inicial y datos base
internal/
  api/
    dto/        → objetos de transferencia de datos (request/response)
    http/
      handlers/    → controladores HTTP (uno por recurso)
      middleware/  → autenticación, CORS, verificación de rol/permiso
      router.go    → define todas las rutas
  application/  → casos de uso (un archivo por acción: crear rol, asignar feature, etc.)
  domain/       → entidades de negocio, reglas, interfaces de repositorio
  infrastructure/
    persistence/postgres/  → implementación real de los repositorios (SQL)
    security/               → hashing de contraseñas, JWT
    notification/           → envío de correos (SMTP)
    messaging/, trainingcenter/
pkg/
  config/         → carga de variables de entorno
  jwtvalidator/    → validación de JWT reutilizable
```

### Endpoints principales de la API

**Autenticación** (`/auth`)
- `POST /auth/login`
- `POST /auth/refresh`
- `POST /auth/logout`
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `GET /auth/me` (requiere sesión)

**Usuarios** (`/users`, requiere rol `SYSTEM_ADMIN` salvo `/users/me`)
- `GET /users/me`
- `POST /users` · `GET /users` · `GET /users/:id` · `PUT /users/:id`
- `PATCH /users/:id/status` · `PATCH /users/:id/unlock`
- `GET /users/:id/roles` · `POST /users/:id/roles` · `DELETE /users/:id/roles/:roleId`
- `GET /users/:id/scope-overrides` · `POST /users/:id/scope-overrides` · `DELETE /users/:id/scope-overrides/:overrideId`

**Catálogo, roles y permisos** (requiere rol `SYSTEM_ADMIN`)
- `GET/POST /modules` · `PUT /modules/:id` · `GET /modules/:id/features`
- `GET/POST /features` · `PUT /features/:id`
- `GET/POST /roles` · `PUT /roles/:id` · `DELETE /roles/:id`
- `GET/POST /roles/:id/features` · `DELETE /roles/:id/features/:featureId` · `PUT /roles/:id/features/batch`
- `GET /training-centers`
- `GET /authz/check` → verifica si un usuario tiene un permiso específico

**Auditoría** (`/audit`, requiere el permiso `AUDIT_LOGIN_VIEW`)
- `GET /audit/logins`

**Salud**
- `GET /health`

### Variables de entorno que necesita (`iam-service/.env.*`)

```
POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
JWT_SECRET
SEED_ADMIN_EMAIL, SEED_ADMIN_PASSWORD   (solo para el script de seed)
SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASSWORD, SMTP_FROM
FRONTEND_RESET_URL, FRONTEND_LOGIN_URL
```
Hay un archivo `.env` por ambiente: `.env.develop`, `.env.qa`, `.env.staging`, `.env.main`.

### Cómo correrlo

**Opción A — con Docker (recomendada):**
```bash
cd design-software-iam-db-back/iam-service
docker compose up --build
```
Esto levanta el servicio en el puerto **8001** y también ofrece un contenedor `iam-seed` para poblar datos base:
```bash
docker compose run --rm iam-seed
```
> Nota: este `docker-compose.yml` espera que exista la red `design-software_default` y la base de datos ya corriendo (ver sección de Infraestructura más abajo).

**Opción B — local con Go instalado:**
```bash
cd design-software-iam-db-back/iam-service
go mod download
go run ./cmd/api
```

---

## 4. Frontend — `design-software-iam-db-front`

**Stack:** Angular 21 (standalone components, *lazy loading* por ruta), TypeScript, RxJS.

### Estructura de carpetas (dentro de `iam-web/src/app/`)

```
core/
  guards/        → auth-guard (sesión activa) y role-guard (rol autorizado)
  interceptors/  → jwt-interceptor (agrega el token a cada petición)
  services/      → auth.ts, token-storage.ts
features/
  auth/          → login, forgot-password, reset-password
  users/         → listado, creación, detalle, overrides de un usuario
  roles/         → listado, detalle, asignación, matriz de permisos
  catalog/       → módulos y features
  audit/         → historial de logins
  dashboard/     → un dashboard distinto por cada rol
  forbidden/     → página de acceso denegado
models/          → interfaces TypeScript de cada entidad
shared/navbar/   → barra de navegación
```

### Roles con dashboard propio
`SYSTEM_ADMIN`, `COORDINATOR`, `CENTER_DIRECTOR`, `AREA_LEADER`, además de vistas para `ADMIN_STAFF`, `INSTRUCTOR` y `APRENDIZ`. El acceso a cada ruta se protege con `authGuard` (¿hay sesión?) y `roleGuard` (¿el rol tiene permiso para esa ruta?).

### Configuración de la API
En `src/environments/environment.ts`:
```ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8001',
};
```
Si el backend corre en otro host/puerto, hay que actualizar este archivo (y su equivalente de producción).

### Cómo correrlo
```bash
cd design-software-iam-db-front/iam-web
npm install
npm start        # equivalente a "ng serve" → http://localhost:4200
```
Otros comandos útiles:
```bash
npm run build     # build de producción
npm test          # tests con Vitest
```

---

## 5. Base de datos — `design-software-iam-db`

Versionada con **Liquibase**. No se ejecutan los `.sql` a mano: Liquibase lee el `changelog/changelog-master.yaml` y aplica los cambios en orden.

### Schemas que crea
- `identity` → tabla `user`
- `rbac_catalog` → tablas `module`, `feature`
- `rbac` → tablas `role`, `role_feature`, `user_role`, `user_scope_override`
- `session` → tablas `refresh_token`, `password_reset_request`
- `identity_audit` → tabla `audit_login`

### Organización de las carpetas
```
01_ddl/    → creación de estructuras (extensiones, schemas, tipos, tablas, alters, índices, vistas, funciones, triggers...)
02_dml/    → datos (inserts, updates, deletes, upserts, patches)
03_dcl/    → roles y permisos de base de datos
04_tcl/    → control transaccional
05_rollbacks/  → reversos de cada cambio anterior (mismo orden que 01-04)
changelog/changelog-master.yaml → archivo maestro que Liquibase ejecuta
```

Esta carpeta normalmente **no se corre sola**: se monta como volumen dentro del contenedor `liquibase-iam` definido en `docker-infra`.

---

## 6. Infraestructura compartida — `docker-infra`

Levanta la base de datos PostgreSQL única del sistema y ofrece un contenedor de Liquibase por cada módulo (incluido `iam`).

```bash
cd docker-infra
# 1. Levantar la base de datos
docker compose up postgres -d

# 2. Aplicar los cambios de este módulo (IAM) a la base de datos
docker compose --profile tooling run --rm liquibase-iam update
```

Variables de entorno necesarias (`docker-infra/.env.*`):
```
POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_PORT
```

> Importante: los repos de cada módulo (`design-software-iam-db`, `design-software-academic-management-db`, etc.) deben estar como carpetas **hermanas** de `docker-infra` (mismo nivel), porque el `docker-compose.yml` los referencia con rutas relativas (`../design-software-iam-db`).

---

## 7. Cómo levantar todo el módulo de punta a punta

```bash
# 1) Base de datos
cd docker-infra
docker compose up postgres -d
docker compose --profile tooling run --rm liquibase-iam update

# 2) Backend
cd ../design-software-iam-db-back/iam-service
docker compose up --build -d
docker compose run --rm iam-seed        # crea el usuario admin inicial

# 3) Frontend
cd ../../design-software-iam-db-front/iam-web
npm install
npm start
```

Luego entrar a `http://localhost:4200` y loguearse con el usuario admin creado en el seed (`SEED_ADMIN_EMAIL` / `SEED_ADMIN_PASSWORD` del `.env` del backend).

---

## 8. Notas rápidas

- Los `.zip` incluyen las carpetas `.git`, `node_modules` y `.angular` (caché) de cada repo — no son necesarias para entender el código, solo aumentan el peso del archivo.
- El backend expone `/health` para verificar que está corriendo.
- El acceso a la mayoría de endpoints de administración requiere el rol `SYSTEM_ADMIN`; el endpoint de auditoría de logins requiere el permiso específico `AUDIT_LOGIN_VIEW`.
