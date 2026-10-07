# Implementation Plan: Gestión Básica de Tareas con Autenticación de Usuarios

**Branch**: `001-core-task-auth` | **Date**: 2026-10-06 | **Spec**: [spec.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/spec.md)

**Input**: Especificación de la funcionalidad desde `specs/001-core-task-auth/spec.md` (HU-01 a HU-04 y HU-12 a HU-13 del backlog).

---

## Summary

Implementar el primer incremento funcional de TaskControl como un monolito estricto en Python con Flask. El incremento abarca el registro seguro de usuarios con contraseñas hasheadas (`scrypt`), control de sesiones en el backend (`login_required`), protección obligatoria contra Cross-Site Request Forgery (CSRF) mediante `CSRFProtect` en todos los formularios mutables (`POST`), gestión estricta de secretos mediante variables de entorno (cero secretos en el repositorio), creación de tareas con validación en servidor, listado y filtrado por estado con orden descendente por creación, transiciones de estado controladas con bloqueo de reapertura de tareas completadas, y edición preservando el estado de la tarea. Toda mutación sobre tareas genera un registro de auditoría estructurado e inmutable centralizado en la capa de servicios. La arquitectura sigue una separación estricta de 4 capas (Modelos, Servicios, Blueprints y Jinja2/JS) con desarrollo Test-First (TDD) bloqueante para la lógica de dominio y arranque inmediato desde la raíz mediante `.flaskenv`.

---

## Technical Context

**Language/Version**: Python 3.11+  
**Primary Dependencies**:
- Web & Core: Flask 3.x, Jinja2, Werkzeug 3.x
- Persistencia & Migraciones: Flask-SQLAlchemy 3.x, Flask-Migrate (Alembic)
- Seguridad & Entorno: Flask-WTF (específicamente `CSRFProtect`), `python-dotenv`  
**Storage**: Base de datos relacional gestionada vía SQLAlchemy ORM (SQLite local para desarrollo y pruebas; PostgreSQL compatible para producción)  
**Testing**: `pytest`, `pytest-cov` (ejecución de pruebas de servicios con SQLite en memoria y clave aislada de testing)  
**Target Platform**: Servidor web monolítico ejecutable localmente en Windows y Linux  
**Project Type**: Aplicación web monolítica con renderizado en servidor y JavaScript vanilla auxiliar  
**Performance Goals**: Tiempo de respuesta de endpoints menor a 100 ms para operaciones de tareas  
**Constraints**:
- Cumplimiento innegociable de los 8 principios de la constitución del proyecto.
- Ningún microservicio ni cola de mensajería (Principio I).
- Lógica de negocio estrictamente desacoplada del contexto HTTP (Principio II).
- Contratos de endpoints respetados rigurosamente, incluyendo tokens CSRF en formularios `POST` (Principio III).
- TDD obligatorio y bloqueante antes de escribir código de dominio (Principio IV).
- Simplicidad arquitectónica: protección CSRF liviana con `CSRFProtect` sin acoplarse a formularios pesados de WTForms (Principio V).
- Migraciones versionadas con Alembic para cambios de esquema (Principio VI).
- Seguridad por defecto: validación/sanitización en servidor, verificación de sesión activa en backend en cada endpoint mutable, contraseñas con hash `scrypt`, protección CSRF universal y `SECRET_KEY` obligatoria leída de variable de entorno sin secretos en el repositorio (Principio VII).
- Registro de auditoría con actor, acción, entidad y marca temporal en cada mutación centralizado en servicios (Principio VIII).
- Entorno levantable con un solo comando desde la raíz mediante `.flaskenv` (Restricciones del stack).  
**Scale/Scope**: Primer incremento focalizado exclusivamente en 6 historias de usuario (*Must*): HU-01, HU-02, HU-03, HU-04, HU-12 y HU-13.

---

## Constitution Check

*GATE: Evaluación previa y posterior al diseño contra la constitución de TaskControl.*

| Principio | Estado | Justificación y Cumplimiento en el Plan |
|---|---|---|
| **I. Monolito por diseño & Entorno ejecutable** | **PASS** | Un único repositorio, proceso y base de datos. Se configura `.flaskenv` versionado en la raíz con `FLASK_APP=src/taskcontrol:create_app`, permitiendo que `pip install -r requirements.txt`, `flask db upgrade` y `flask run` funcionen desde la raíz de inmediato. |
| **II. Separación en capas** | **PASS** | Estructura con 4 capas explícitas: `models/`, `services/`, `blueprints/` y `templates/` + `static/`. Las rutas nunca tocan el ORM directamente; los servicios no conocen `request` ni `Response`. |
| **III. Contrato explícito backend-JS** | **PASS** | Diseñados en [contracts/auth-contracts.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/contracts/auth-contracts.md) y [contracts/task-contracts.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/contracts/task-contracts.md), especificando rutas, métodos, payloads de entrada (incluyendo `csrf_token` obligatorio en todo `POST`), payloads de salida y códigos de error (incluyendo `400 CSRF_ERROR`). |
| **IV. Test-first en lógica de negocio** | **PASS** | La suite `tests/unit/` define pruebas en rojo antes de la implementación para `AuthService` y `TaskService` (transiciones válidas e inválidas, bloqueo de reaperturas, validaciones). Condición bloqueante. |
| **V. Simplicidad sobre generalidad prematura** | **PASS** | Filosofía YAGNI: Protección CSRF resuelta mediante la extensión ligera `CSRFProtect(app)` de Flask-WTF sin introducir clases ni validadores de formulario pesados. Sin repositorios genéricos artificiales. |
| **VI. Integridad de datos y migraciones** | **PASS** | Configuración de Flask-Migrate con directorio `migrations/` versionado. Esquema relacional con claves foráneas e índices. |
| **VII. Seguridad por defecto & Gestión de secretos** | **PASS** | Hashing con `scrypt`, cookies de sesión HttpOnly y SameSite=Lax, protección CSRF en todos los métodos mutables, `SECRET_KEY` obligatoria leída de variable de entorno (con plantilla `.env.example` y valor estático aislado solo en `TestingConfig`), y decorador `@login_required` para verificación de sesión en cada endpoint de tareas. |
| **VIII. Observabilidad mínima viable** | **PASS** | `AuditService` centralizado que persiste en la tabla `audit_logs` y emite log estructurado en JSON con actor, acción, entidad y timestamp para toda creación, edición o transición de tareas. |

---

## Project Structure

### Documentation (este incremento)

```text
specs/001-core-task-auth/
├── spec.md              # Especificación funcional validada y clarificada
├── plan.md              # Este plan de implementación técnica (actualizado)
├── research.md          # Investigación técnica y decisiones arquitectónicas (Fase 0)
├── data-model.md        # Definición de entidades SQLAlchemy y máquina de estados (Fase 1)
├── quickstart.md        # Guía de inicialización y escenarios de validación (Fase 1)
├── contracts/           # Contratos formales de endpoints HTTP (Fase 1)
│   ├── auth-contracts.md # Contratos para HU-12 y HU-13 (con CSRF)
│   └── task-contracts.md # Contratos para HU-01 a HU-04 (con CSRF)
└── checklists/
    └── requirements.md  # Checklist de calidad de requerimientos
```

### Source Code (Estructura Monolítica del Repositorio)

```text
.flaskenv                        # Configuración CLI versionada (FLASK_APP=src/taskcontrol:create_app)
.env.example                     # Plantilla de variables de entorno sin secretos reales
.gitignore                       # Exclusión de .env, .venv, *.pyc, taskcontrol.db
requirements.txt                 # Dependencias fijadas del proyecto (incluye python-dotenv, Flask-WTF)
run.py                           # Punto de entrada alternativo para levantar la aplicación

src/
└── taskcontrol/
    ├── __init__.py              # Application Factory (create_app, sys.path setup)
    ├── config.py                # Configuración por entornos (Development, Testing, Production)
    ├── extensions.py            # Instancias de SQLAlchemy, Flask-Migrate y CSRFProtect
    │
    ├── models/                  # Capa 1: Modelos de datos relacionales (SQLAlchemy)
    │   ├── __init__.py
    │   ├── user.py              # Modelo User (id, email, password_hash, created_at)
    │   ├── task.py              # Modelo Task (id, user_id, title, description, due_date, status, timestamps)
    │   └── audit_log.py         # Modelo AuditLog (id, timestamp, actor_id, action, entity_id, details)
    │
    ├── services/                # Capa 2: Lógica de negocio y dominio (TDD bloqueante)
    │   ├── __init__.py
    │   ├── auth_service.py      # Registro, verificación de contraseñas y hashing
    │   ├── task_service.py      # Creación, listado filtrado, edición y máquina de transiciones
    │   └── audit_service.py     # Registro centralizado de eventos de auditoría y log estructurado
    │
    ├── blueprints/              # Capa 3: Controladores y rutas HTTP (Flask)
    │   ├── __init__.py
    │   ├── auth.py              # Rutas /register, /login, /logout (protegidas con CSRF)
    │   ├── tasks.py             # Rutas /tasks, /tasks/new, /tasks/<id>/edit, /tasks/<id>/status
    │   └── decorators.py        # Decorador @login_required para verificación estricta en backend
    │
    ├── templates/               # Capa 4A: Presentación en servidor (Jinja2)
    │   ├── base.html            # Layout común con alerts flash, navbar y meta CSRF si aplica
    │   ├── auth/
    │   │   ├── login.html       # Vista de inicio de sesión con {{ csrf_token() }}
    │   │   └── register.html    # Vista de registro de usuario con {{ csrf_token() }}
    │   └── tasks/
    │       ├── index.html       # Listado y filtrado de tareas
    │       ├── new.html         # Formulario de creación de tarea con {{ csrf_token() }}
    │       └── edit.html        # Formulario de edición de tarea con {{ csrf_token() }}
    │
    └── static/                  # Capa 4B: Recursos estáticos del cliente
        ├── css/
        │   └── styles.css       # Estilos visuales de la interfaz
        └── js/
            └── main.js          # JavaScript vanilla mínimo (envío de header X-CSRFToken si fetch)

migrations/                      # Scripts versionados de Alembic / Flask-Migrate
tests/
├── conftest.py                  # Fixtures compartidas (app, client, DB en memoria, testing SECRET_KEY)
├── unit/                        # Pruebas unitarias de dominio (TDD mandatorio antes de implementar servicios)
│   ├── test_auth_service.py     # Pruebas de reglas de usuario y credenciales
│   ├── test_task_service.py     # Pruebas de reglas de tareas y máquina de estados
│   └── test_audit_service.py    # Pruebas de emisión y persistencia de eventos de auditoría
└── integration/                 # Pruebas de integración HTTP y contratos
    ├── test_auth_routes.py      # Verificación de endpoints de autenticación y sesiones
    ├── test_task_routes.py      # Verificación de endpoints de tareas y control de acceso
    └── test_csrf.py             # Verificación de rechazo 400 por omisión o manipulación de token CSRF
```

---

## Estrategia de Implementación por Capas y Seguridad

### 1. Gestión de Configuración y Secretos (Principio VII)
- `config.py` define clases de configuración:
  - `BaseConfig`: `SQLALCHEMY_TRACK_MODIFICATIONS = False`, `SESSION_COOKIE_HTTPONLY = True`, `SESSION_COOKIE_SAMESITE = 'Lax'`, `SESSION_PERMANENT = False`.
  - `DevelopmentConfig`: Lee `SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-insecure-key-warning')`.
  - `ProductionConfig`: `SECRET_KEY = os.environ['SECRET_KEY']` (lanza `ValueError` si está ausente).
  - `TestingConfig`: `TESTING = True`, `SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'`, `WTF_CSRF_ENABLED = False` (para pruebas de rutas normales), `SECRET_KEY = 'test-secret-key-taskcontrol-testing'` (fija y reproducible en el código de test sin requerir `.env`).
- Archivo [.env.example](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/.env.example): Provee plantilla sin secretos reales para el desarrollador.
- Archivo `.flaskenv`: Configura `FLASK_APP=src/taskcontrol:create_app` para habilitar `flask db upgrade` y `flask run` directamente desde la raíz del proyecto.

### 2. Protección CSRF (Principios V y VII)
- Inicialización en `extensions.py`: `csrf = CSRFProtect()`.
- En `create_app()`: `csrf.init_app(app)`.
- Manejador de errores `@app.errorhandler(CSRFError)`: Retorna código `400 Bad Request` con mensaje `"Token CSRF ausente o inválido"` estructurado (JSON o flash HTML).
- En plantillas Jinja2: Todo `<form method="POST">` incluye `<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">`.
- En peticiones asíncronas con JavaScript: Inclusión de la cabecera `X-CSRFToken` leída desde una metaetiqueta o cookie.

### 3. Capa de Modelos (`models/`)
- Definición de `User`, `Task` y `AuditLog` con SQLAlchemy.
- Restricciones relacionales, claves foráneas, índices de unicidad en `email` y estados de tareas.

### 4. Capa de Servicios (`services/`) — TDD Bloqueante
- `AuthService`: Registro, normalización, validación ($\ge 8$ caracteres, letra y número) y hashing con `scrypt`.
- `TaskService`: CRUD, filtrado por usuario autenticado, orden descendente por fecha de creación, máquina de estados con bloqueo estricto de reaperturas desde `completed`, y delegación en `AuditService`.
- `AuditService`: Registro centralizado e inmutable en `audit_logs` y log estructurado en JSON.

### 5. Capa de Blueprints y Rutas (`blueprints/`)
- Rutas HTTP protegidas con `@login_required` para verificación de sesión en backend.
- Extracción de parámetros y delegación inmediata en servicios sin lógica de negocio en las rutas.

### 6. Estrategia de Pruebas (Principios IV y VII)
- **Unitarias (`tests/unit/`)**: Prueban exclusivamente `AuthService`, `TaskService` y `AuditService` con SQLite en memoria. Se escriben primero en rojo (TDD).
- **Integración (`tests/integration/`)**: Prueban blueprints HTTP usando `test_client`.
  - En `test_auth_routes.py` y `test_task_routes.py`: Se ejecutan con `WTF_CSRF_ENABLED = False` para centrarse en contratos de ruta y códigos de estado.
  - En `test_csrf.py`: Se ejecuta con `WTF_CSRF_ENABLED = True` para validar que toda petición `POST` sin token o con token falso sea rechazada con `400 Bad Request`.

---

## Complexity Tracking

| Violación Constitucional Potencial | Por Qué se Requiere | Alternativa Más Simple Rechazada |
|---|---|---|
| *Ninguna* | No se detectaron violaciones; el diseño cumple el 100% de la constitución. | N/A |
