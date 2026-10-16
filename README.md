# Gestión de Apartamentos Turísticos

Aplicación web full-stack para la gestión operativa de un conjunto de apartamentos
de alquiler turístico: reservas, limpiezas y facturación de las limpiezas.

---

## El problema

Una pequeña gestora de apartamentos turísticos llevaba la operativa en hojas de
cálculo y mensajes sueltos. Eso generaba tres problemas concretos:

- **Visibilidad de las reservas.** Sin una vista única del mes era difícil saber
  qué apartamento estaba ocupado y cuándo, y detectar solapamientos.
- **Coordinación de las limpiezas.** El hueco de limpieza depende del check-out de
  una reserva y del check-in de la siguiente. Calcularlo a mano, por apartamento y
  día, es lento y propenso a errores.
- **Facturación de las limpiezas.** Cada limpieza se factura por horas y tarifa, y
  el recibo hay que generarlo, archivarlo y confirmar que se ha pagado.

La aplicación cubre esas tres áreas con control de acceso por rol: **`admin`**
(operativa completa) y **`limpiadora`** (su trabajo y sus recibos).

---

## Funcionalidades

**Reservas y calendario**

- Vista mensual por apartamento con las reservas dispuestas en carriles sin solapamiento.
- Filtros por apartamento y por estado; alta, edición y borrado de reservas.
- Horas de entrada y salida pactadas por reserva, con hora estándar por defecto.
- Bonificación eléctrica por noche, configurable por apartamento.

**Limpiezas**

- Cálculo de las oportunidades de limpieza a partir de los huecos entre un check-out
  y el siguiente check-in.
- Tipos de limpieza con tarifa por hora, gestionables desde el panel de administración.
- Vista de organización de limpiezas por día y apartamento.

**Facturación**

- Alta de factura de limpieza (horas × tarifa), rectificación y nota de cancelación.
- Recibo previsualizado en Angular y **replicado en una plantilla Jinja2** que el
  backend renderiza a PDF con **Chromium headless**, de modo que el PDF archivado es
  idéntico a lo que el usuario vio en pantalla.
- El PDF se sube por SSH/SCP a un **NAS Synology**. La subida es *best-effort* con
  cola de reintentos: cada documento lleva estado
  (`Pendiente` → `Procesando` → `Completado` / `Error`), número de intentos,
  `next_retry_at` y último error. Un worker horario reprocesa lo que quedó pendiente.
- Confirmación de pago por parte de administración y de la limpiadora.

**Búsquedas y usuarios**

- Búsqueda de reservas, de disponibilidad y por apartamento, con panel de estadísticas.
- Exportación a Excel de la plantilla de resumen del **Modelo 210** (alquiler turístico).
- Gestión de usuarios, autenticación JWT con *refresh*, cambio obligatorio de la
  contraseña inicial y recuperación de contraseña por email.

---

## Arquitectura

### Backend — Clean Architecture

Cuatro capas con la dependencia apuntando siempre **hacia dentro**: cada capa conoce
a las de dentro, nunca a las de fuera.

```
┌────────────────────────────────────────────────────────┐
│  api/            Routers FastAPI, schemas, manejo de   │  ← HTTP
│                  excepciones, inyección de dependencias│
├────────────────────────────────────────────────────────┤
│  application/    Casos de uso + INTERFACES de los      │  ← Orquestación
│                  servicios que necesitan (puertos)     │
├────────────────────────────────────────────────────────┤
│  domain/         Entidades y reglas de negocio,        │  ← Núcleo
│                  excepciones, interfaces de repositorio│
└────────────────────────────────────────────────────────┘
                            ▲  implementa los puertos
┌───────────────────────────┴────────────────────────────┐
│  infrastructure/  Adaptadores concretos: SQLAlchemy,   │  ← Detalles
│                   JWT, passlib, SMTP, Chromium, SCP    │
└────────────────────────────────────────────────────────┘
```

- **`domain/`** — entidades Pydantic (`Booking`, `Bill`, `BillDocument`, `Apartment`,
  `User`, `CleaningType`), reglas de negocio y excepciones de dominio.
  No importa nada de las demás capas.
- **`application/`** — un caso de uso por operación (`CreateBillUseCase`,
  `GenerateAndStoreBillDocumentUseCase`, `LoginUseCase`…). Aquí viven los **puertos**:
  `IBillRepository`, `IFileStorage`, `IEmailSender`, `ITokenManager`,
  `IPasswordManager`, `IBillPdfRenderer`. La capa depende de las interfaces, nunca
  de sus implementaciones.
- **`infrastructure/`** — los **adaptadores** que cumplen esos puertos:
  `SQLAlchemy*Repository`, `JwtTokenManager`, `PasslibPasswordManager`,
  `SmtpEmailSender`, `ChromiumBillPdfRenderer`, `SynologyFileStorage`.
- **`api/`** — routers versionados bajo `/api/v1`, schemas de entrada/salida y el
  mapeo de excepción de dominio → respuesta HTTP. **`api/dependencies.py` es el
  *composition root*:** el único punto donde se decide qué implementación concreta
  recibe cada caso de uso, mediante `Depends` de FastAPI.

Esa inversión de dependencias es lo que permite probar los casos de uso con dobles
en lugar de necesitar MySQL, un NAS y un servidor SMTP reales.

### Frontend — Angular por features

```
frontend/src/app/
├── auth/         AuthService, interceptor HTTP, guards, TokenService
├── config/       Permisos por rol, configuración del banner
├── models/       Interfaces TypeScript del dominio
├── services/     Un servicio HTTP por recurso (bookings, bills, apartments…)
├── pages/        Una carpeta por pantalla, con sus componentes hijos
│   ├── calendar/              (calendar-header, week-row, booking-bar)
│   ├── searches/              (booking, availability y apartment search, stats)
│   └── cleaning-organization/, bills/, admin/, account/, logging/…
├── shared/       Componentes reutilizables (bill-receipt, modales, formularios)
├── pipes/        Transformaciones de presentación
└── directives/
```

Componentes *standalone* y **Signals API** de Angular 19 para el estado de las vistas.

### Cómo se comunican backend y frontend

|            | Desarrollo                                            | Producción                                                                     |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------------------------------ |
| **SPA**    | `ng serve` en `:4200`                                 | Estáticos compilados, servidos por Caddy                                        |
| **API**    | Uvicorn en `:8000`                                    | Contenedor `backend`, sin puerto público                                        |
| **Puente** | `proxy.conf.json` reenvía `/api/*` → `127.0.0.1:8000` | Caddy hace `reverse_proxy /api/* → backend:8000` por la red interna de Compose  |

`environment.apiUrl` está **vacío en ambos entornos**: el frontend siempre llama a
rutas relativas (`/api/v1/...`) y quien resuelve el destino es el proxy. Así no hay
URLs de backend incrustadas en el bundle, y en producción no hace falta CORS porque
SPA y API comparten origen.

La autenticación viaja en `Authorization: Bearer <access_token>`. El interceptor HTTP
adjunta el token, intenta un *refresh* cuando la respuesta es 401 y cierra sesión si
ese refresh falla.

---

## Stack técnico

| Capa                | Tecnología                                                                  |
| ------------------- | --------------------------------------------------------------------------- |
| **Frontend**        | Angular 19.2 · TypeScript 5.7 · RxJS 7.8 · SCSS · Signals API               |
| **Backend**         | FastAPI 0.115 · Python 3.13 · Pydantic 2.9 · SQLAlchemy 2.0 · Alembic       |
| **Base de datos**   | MySQL 8.0 (PyMySQL)                                                         |
| **Autenticación**   | PyJWT (access + refresh) · passlib con bcrypt                               |
| **Documentos**      | Jinja2 + Chromium headless (HTML → PDF) · paramiko/scp (NAS Synology)       |
| **Infraestructura** | Docker · Docker Compose · Caddy (TLS automático)                            |
| **Calidad**         | Ruff · mypy · pytest + coverage · ESLint · Prettier · Jest · GitHub Actions |
| **Gestores**        | pip (backend) · pnpm 10.33 sobre Node 22.22 (frontend)                      |

El backend tiene 47 ficheros de test repartidos en `unit/`, `integration/` y `e2e/`,
el frontend 36 *specs* de Jest, y el esquema se versiona con 6 migraciones de Alembic.

---

## Puesta en marcha en local

### Requisitos

- Python 3.13 · Node 22.22 (ver [`.nvmrc`](.nvmrc)) · pnpm 10.33 · Docker
- Chromium o Chrome instalado, si quieres generar los PDF de factura

### 1 · Base de datos

```bash
docker compose up -d db      # MySQL 8.0 en :3306
```

### 2 · Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env               # rellenar DB_*, JWT_SECRET_KEY y DEFAULT_PASSWORD
alembic upgrade head               # crea el esquema
./start.sh                         # migra y arranca Uvicorn
```

API en `http://localhost:8000`; documentación interactiva en `/docs` (solo con `DEBUG=true`).

Variables mínimas para arrancar: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME`,
`JWT_SECRET_KEY` y `DEFAULT_PASSWORD`. Si dejas `SMTP_HOST` vacío, los enlaces de
recuperación de contraseña se imprimen por consola en lugar de enviarse por email.
Todas las opciones están documentadas en [`backend/.env.example`](backend/.env.example).

Crear el primer usuario administrador:

```bash
python scripts/create_admin_user.py
```

### 3 · Frontend

```bash
cd frontend
pnpm install
pnpm start
```

Aplicación en `http://localhost:4200`; las llamadas a `/api/*` llegan al backend por proxy.

### Pruebas y calidad

```bash
# Backend
cd backend
ruff check . && ruff format --check .
python -m mypy api/ application/ domain/ infrastructure/ config.py main.py \
  --config-file pyproject.toml --explicit-package-bases
python -m pytest tests/ --cov --cov-report=term-missing

# Frontend
cd frontend
pnpm lint && pnpm exec prettier --check "src/**/*.{ts,html,scss}"
pnpm typecheck && pnpm test:ci && pnpm build
```

Ambos conjuntos se ejecutan en GitHub Actions
([`backend-ci.yml`](.github/workflows/backend-ci.yml) y
[`frontend-ci.yml`](.github/workflows/frontend-ci.yml)); los tests de backend corren
contra un servicio MySQL real.

---

## Capturas de pantalla


| Vista                                                             | Captura     |
| ----------------------------------------------------------------- | ----------- |
| **Calendario mensual** — reservas por apartamento y semana         | <img width="1136" height="403" alt="image" src="https://github.com/user-attachments/assets/a62d6b9c-e333-473e-b441-f8f82f3a6096" />
 |
| **Organización de limpiezas** — huecos entre check-out y check-in  | _pendiente_ |
| **Recibo de factura** — previsualización que replica el PDF        | _pendiente_ |
| **Búsquedas** — reservas, disponibilidad y estadísticas            | _pendiente_ |
| **Panel de administración** — usuarios, apartamentos y tarifas     | _pendiente_ |

---

## Despliegue

La producción se orquesta con `docker-compose.prod.yml` sobre una MySQL externa
(no contenedorizada):

| Servicio          | Rol                                                                   |
| ----------------- | --------------------------------------------------------------------- |
| `migrate`         | Ejecuta `alembic upgrade head` antes de levantar el resto             |
| `backend`         | FastAPI + Chromium, sin puerto público                                |
| `bill-docs-retry` | Worker horario que reintenta la sincronización de PDFs con el NAS     |
| `web`             | SPA de Angular servida por Caddy, con TLS automático y proxy `/api/*` |

```bash
cp .env.production.example .env.production   # rellenar con valores reales
chmod 600 .env.production
docker compose -f docker-compose.prod.yml config       # validar
docker compose -f docker-compose.prod.yml up -d --build
```

`docker-compose.yml` y `docker-compose.test.yml` son **solo** para desarrollo local y CI.



---

## Licencia

[MIT](LICENSE).
