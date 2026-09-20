# AGENTS.md

> Monorepo **Attention** — plataforma de e-commerce de moda con probador virtual AR (UAGRM · Sistemas de Información II). Cada subdirectorio raíz es un **proyecto independiente con su propio `.git`**; ninguno se construye desde aquí. Entrá siempre al directorio del subproyecto antes de tocar nada.

---

## 0. Layout del monorepo

| Directorio | Stack | Entry | Cómo correrlo |
|---|---|---|---|
| `1p_si2_backend/` | Python 3.14 · FastAPI · SQLAlchemy · Alembic · Pydantic Settings | `app/main.py` | `cd 1p_si2_backend && .venv\Scripts\activate && python scripts\run.py` (uvicorn reload en `127.0.0.1:8000`) |
| `1p_si2_frontend/` | Angular 22 standalone · Tailwind 4 · Signals · pnpm | `src/app/` | `cd 1p_si2_frontend && pnpm start` (sirve en `4200`; build SSR → `dist/frontend/browser`) |
| `1p_si2_mobile/` | Flutter 3.13+ · Dart · `http` · `flutter_secure_storage` | `lib/main.dart` | `cd 1p_si2_mobile && flutter run` (sondéo de host automático, ver §4) |
| `1p_si2_agents/` | Markdown + steering + specs + ADRs | — | Workspace **SDD** (sin código). Leer antes de implementar cualquier CU |
| `.atl/` | Skill registry | — | Solo para OpenCode; indexá skills desde acá |

**Reglas transversales:**

- No commitees en la raíz. Cada subproyecto tiene su historial git propio: `cd` primero, después `git ...`.
- Cambios de contrato (endpoints `/api/v1/...`, modelos compartidos) requieren alinear **backend + frontend + mobile**. El backend manda — los otros dos consumen.
- `.env` del backend vive en `1p_si2_backend/.env` y **no** se versiona (está en `.gitignore`).

---

## 1. `1p_si2_backend/` — FastAPI

### Comandos clave (desde la raíz del backend)

| Tarea | Comando |
|---|---|
| Activar venv | `.venv\Scripts\activate` (Windows) |
| Servir en dev | `python scripts\run.py` (uvicorn con reload) |
| Verificar config pre-deploy | `python verify_config.py` |
| Crear migración | `alembic revision --autogenerate -m "<msg>"` (desde `1p_si2_backend/`) |
| Aplicar migraciones | `alembic upgrade head` |
| Smoke test de un CU | `python scripts\smoke\smoke_cu<N>.py` (no requiere servidor) |
| Tests unitarios | `python -m pytest tests\unit\` *(directorios vacíos hoy: `.gitkeep`)* |

### Estructura real (no inventar nombres)

```
app/
├── main.py                # FastAPI app, CORS, routers por CU con prefijo /api/v1/<recurso>
├── core/                  # config (pydantic-settings), database (Base compartido), security (JWT)
├── modules/               # usuarios, empresa, compras, inventario, probador, ventas, delivery
├── api/v1/endpoints/      # auth, users, roles, branches, categories, products, variantes,
│                          # reservas, inventario, ventas, probador, suppliers, dashboard, company, catalogo
└── schemas/               # Pydantic por CU
alembic/
├── env.py                 # Importa TODOS los modules.models (mismo orden que main.py)
└── versions/              # Migraciones
scripts/
├── run.py                 # Uvicorn dev
└── smoke/                 # Smoke tests por CU (CU8, CU14, CU22, CU15+21)
```

### Quirks que un agente se pierde

- **`alembic.ini` omite `timezone=` a propósito.** El venv local (Python 3.14, Windows) no tiene `tzdata` y Alembic 1.19.1 llama a `ZoneInfo(...)` si la opción existe → crash. No la agregues "para estar completos". El comentario en el archivo lo explica.
- **La URL de DB se resuelve en cascada** (idéntica en `app/core/database.py` y `alembic/env.py`): `os.environ["DATABASE_URL"]` → `Settings.DATABASE_URL` del `.env` → partes `DB_HOST/DB_PORT/DB_USER/DB_PASSWORD/DB_NAME` → fallback local `localhost/tienda_ropa`. Dialecto: `postgresql://...` (psycopg2, **no** `postgresql+psycopg://`).
- **CORS_ORIGINS acepta JSON o CSV.** Ejemplo: `CORS_ORIGINS=["https://1p-si2-frontend.vercel.app","http://localhost:4200"]`. El parser vive en `app/core/config.py:parse_cors_origins`. `main.py` además acepta string CSV como cinturón de seguridad.
- **JWT**: `SECRET_KEY` por defecto es **insegura** (`clave_super_secreta_atention_CAMBIAR_EN_PRODUCCION`). `verify_config.py` lo detecta y aborta. En Render va como env var.
- **Nuevo módulo de modelo**: hay que importarlo en **dos lugares** (`app/main.py` y `alembic/env.py`, mismo orden) para que su tabla se registre en `Base.metadata` y Alembic la vea.
- **Routers ya cableados** (ver `app/main.py`): no agregar prefijos nuevos a mano — los prefijos contractuales están fijos (`/api/v1/auth`, `/api/v1/productos`, `/api/v1/reservas`, `/api/v1/inventario`, `/api/v1/ventas`, `/api/v1/probador-virtual`, `/api/v1/dashboard`, etc.).
- **Deploy**: Render (`attention-backend-czw9.onrender.com`) y BD en Supabase. Variables de entorno documentadas en `RENDER_ENV_VARS.md`.

### Antes de hacer PR en backend

1. `python verify_config.py` (debe terminar con 0 errores críticos).
2. `alembic upgrade head` contra la DB de prueba.
3. `python scripts\smoke\smoke_cu<N>.py` del CU tocado (los que apliquen).
4. Si tocás `app/modules/*/models.py`, regenerar migración: `alembic revision --autogenerate -m "..."` y revisar el diff.

---

## 2. `1p_si2_frontend/` — Angular 22

> ⚠️ El archivo `1p_si2_frontend/frontend.md` (CLAUDE.md legacy) está **desactualizado**: dice "Angular 17" y rutas como `/admin/...`, `/gerente/...`, `/vendedor/...`, `/tienda/...`. El código real usa Angular **22** y estructura `src/app/core/`, `src/app/features/`, `src/app/shared/`, `src/app/layouts/`. **Confiar en `package.json` y en el código, no en `frontend.md`.**

### Comandos clave

| Tarea | Comando |
|---|---|
| Instalar deps | `pnpm install` (NO npm, NO yarn — `packageManager: pnpm@11.20.0`) |
| Dev server | `pnpm start` |
| Build producción (SSR) | `pnpm run build` → `dist/frontend/browser/` |
| Tests | `pnpm test` (Vitest + jsdom) |
| Formato | `prettier` (config en `.prettierrc`) |

### Estructura real

```
src/app/
├── core/
│   ├── guards/            # auth.guard, guest.guard
│   ├── interceptors/      # jwt.interceptor, error.interceptor
│   ├── models/            # Interfaces TS (una por entidad)
│   └── services/          # api.ts (base), auth.service.ts, carrito, probador, catalogo-tienda
├── features/              # auth, users, roles, company, branches, categories, suppliers,
│                          # inventario, reservas, probador-virtual, dashboard, catalogo, proximamente
├── shared/components/     # badge, confirm-dialog, layout
├── layouts/               # admin-layout, gerente-layout, vendedor-layout, cliente-layout (tienda)
└── environments/          # environment.ts (apiUrl → http://localhost:8000/api/v1) y .prod.ts
```

### Convenciones de código

- Todos los componentes son **standalone** (`standalone: true`).
- Nombre: `{feature}-{tipo}.component.ts` (p.ej. `login-form.component.ts`).
- HTTP solo desde services. Componentes jamás llaman `HttpClient` directo.
- Tailwind 4 (PostCSS) — sin CSS inline. Clases base documentadas en `frontend.md` (siguen vigentes: botones, input, card, badges).
- Idioma: **español** en UI, validaciones, mensajes de error.
- Modelos con `snake_case` para campos de la API (`id_usuario`, `nombre_rol`) y PascalCase para interfaces.

### Deploy (Vercel)

- `vercel.json` define build: `pnpm run build`, output: `dist/frontend/browser`.
- Rewrites: `/(.*)` → `/index.csr.html` (SPA routing).
- API base prod: configurable en `environment.prod.ts` (debe apuntar a Render).

---

## 3. `1p_si2_mobile/` — Flutter

### Comandos clave

| Tarea | Comando |
|---|---|
| Instalar deps | `flutter pub get` |
| Análisis estático | `flutter analyze` (debe dar 0 errores, 0 warnings) |
| Tests | `flutter test` (todos en verde) |
| Run debug | `flutter run` |
| Build APK | `flutter build apk --release` |
| Instalar deps Android | `cd android && .\gradlew.bat` (Windows) |

### Estructura canónica (Clean Feature-First)

> El código actual todavía tiene `lib/screens/*.dart` legacy (login, home, catalog). Las features nuevas van en `lib/features/<feature>/` con la forma `data/` (models + service) + `presentation/` (controllers + widgets + screens). **No anclar features nuevas a `lib/screens/`.** Detalle completo en `1p_si2_agents/steering/architecture.md`.

```
lib/
├── main.dart              # MyApp + _SessionGate
├── core/
│   ├── config/            # ApiConfig.resolveBaseUrl() — ver §4
│   ├── constants/         # AppColors (paleta Attention)
│   ├── network/           # Interceptores HTTP
│   ├── storage/           # SecureStorageService (JWT cifrado en KeyStore/Keychain)
│   └── theme/             # AppTheme (Material 3)
├── shared/                # utils, widgets (LoadingView, ErrorRetryView, EmptyView)
└── features/<feature>/
    ├── data/              # models/ + <feature>_service.dart
    └── presentation/      # controllers/ + widgets/ + screens/
```

### Reglas innegociables (mobile)

1. **Tokens solo en `FlutterSecureStorage`.** Prohibido `SharedPreferences` para JWT o passwords.
2. **Colores vía `AppColors` o `Theme.of(context).colorScheme`.** Prohibido hardcodear hex en widgets.
3. **Tres estados en vistas asíncronas**: `Loading` (spinner), `Data` (con Pull-to-Refresh), `Error` (vista en español con botón "Reintentar").
4. **Texto en español neutro formal**, validaciones y mensajes incluidos.
5. **Exclusividad de rol `C` (Cliente)**: si `nombre_rol != 'C'`, abortar login sin persistir token, mostrar *"Esta aplicación es exclusiva para clientes. Ingresa desde la plataforma web."* Validar también en `_SessionGate` al restaurar sesión.
6. **Resolución de host SIEMPRE vía `ApiConfig.resolveBaseUrl()`** — nunca URL hardcoded.

### 4. Conectividad multi-host (`ApiConfig`)

La app sondea candidatos con timeout 1200ms y memoiza el primero que responda:

1. `http://10.0.2.2:8000/api/v1` — emulador Android oficial
2. `http://localhost:8000/api/v1` — dispositivo físico USB con `adb reverse tcp:8000 tcp:8000`
3. `http://192.168.0.2:8000/api/v1` — dispositivo físico en LAN Wi-Fi (ajustar a tu IP)

Detalles y consecuencias en `1p_si2_agents/decisions/ADR-002_multi_host_network.md`.

### 5. Antes de hacer commit en mobile

1. `flutter analyze` → 0 errores, 0 warnings.
2. `flutter test` → todos en verde.
3. Si tocás un CU, marcá `[x]` en `1p_si2_agents/planning/tasks.md` y referenciá la spec.

---

## 6. Workspace SDD: `1p_si2_agents/`

**Ciclo obligatorio** para implementar un CU en cualquier subproyecto:

```
SPEC → REVIEW → PLAN → IMPLEMENT → TEST → VERIFY → DOCUMENT → COMMIT
```

| Carpeta | Qué hay |
|---|---|
| `steering/vision_and_scope.md` | Propósito, roles (ASU/GS/V/D/**C**), los 15 CUs |
| `steering/tech_stack.md` | Stack Flutter y **auditoría de drift** (lo que falta por implementar) |
| `steering/architecture.md` | Clean Feature-First, capas por feature, multi-host, SessionGate |
| `agents/mobile_developer.md` | Reglas innegociables + plantilla canónica HTTP (try/catch exhaustivo) |
| `agents/prompt_maestro.md` | Prompts estandarizados por subproyecto |
| `specs/<NN_modulo>/*.md` | Una spec por CU (Gherkin + contratos API) |
| `planning/master_plan.md` | Plan maestro de 10 fases |
| `planning/backlog.md` | Matriz del backlog de los 15 CUs |
| `planning/tasks.md` | WBS con checklist `[ ]` / `[/]` / `[x]` |
| `decisions/ADR-*.md` | ADRs aceptados (state mgmt, multi-host, rol C) |
| `testing/test_plan.md` | Matriz de pruebas de los 15 CUs |

**Reglas del ciclo:**

- **Antes de tocar código de un CU**, leé `specs/<NN_modulo>/<cu>_spec.md` y `planning/tasks.md`.
- El estado de cada task va en `tasks.md` (no en commits, no en issues sueltos).
- Si descubrís un nuevo constraint arquitectónico, agregá un ADR en `decisions/`.

---

## 7. Convenciones comunes a los 3 subproyectos

- **Contrato API mandatorio**: `{"status": "success|error", "data": {...}, "message": "..."}` (envelope estándar del backend).
- **Sin `Co-Authored-By` ni atribución de IA en commits.** Conventional Commits en español o inglés según el subproyecto (revisá `git log --oneline -10` antes del primer commit).
- **Conventional Commits**: revisar el estilo del repo antes de escribir el primer mensaje. Hoy hay pocos commits de referencia — usá `feat|fix|chore|docs|refactor(scope): descripción`.
- **Migraciones de Alembic**: el autogenerate detecta diffs entre `Base.metadata` y la DB. Si agregás modelos, importarlos primero en `app/main.py` y `alembic/env.py`, después `alembic revision --autogenerate`. **Revisar siempre el SQL generado** antes de commitear.
- **Secrets**: nada de credenciales hardcodeadas. `SECRET_KEY`, `DATABASE_URL`, `CORS_ORIGINS` siempre por env var o `.env` (que va en `.gitignore`).

---

## 8. Verificación rápida por subproyecto

```bash
# Backend
cd 1p_si2_backend && .venv\Scripts\activate
python verify_config.py
python scripts\run.py    # uvicorn reload en :8000

# Frontend
cd 1p_si2_frontend
pnpm install
pnpm start               # ng serve en :4200

# Mobile
cd 1p_si2_mobile
flutter pub get
flutter analyze
flutter test
flutter run              # emulador o dispositivo
```

Si algo falla, primero `cd` al subproyecto correcto — los errores de "módulo no encontrado" suelen ser eso.
