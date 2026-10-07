# Implementation Plan: Gestión Básica de Tareas con Autenticación de Usuarios

**Branch**: `001-core-task-auth` | **Date**: 2026-10-06 | **Spec**: [spec.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/spec.md)

**Input**: Especificación de la funcionalidad desde `specs/001-core-task-auth/spec.md` (HU-01 a HU-04 y HU-12 a HU-13 del backlog).

---

## Summary

Implementar el primer incremento funcional de TaskControl como un monolito estricto en Python con Flask. El incremento abarca el registro seguro de usuarios con contraseñas hasheadas (`scrypt`), control de sesiones en el backend (`login_required`), creación de tareas con validación en servidor, listado y filtrado por estado con orden descendente por creación, transiciones de estado controladas con bloqueo de reapertura de tareas completadas, y edición preservando el estado de la tarea. Toda mutación sobre tareas genera un registro de auditoría estructurado e inmutable centralizado en la capa de servicios. La arquitectura sigue una separación estricta de 4 capas (Modelos, Servicios, Blueprints y Jinja2/JS) con desarrollo Test-First (TDD) bloqueante para la lógica de dominio.

---

## Technical Context

**Language/Version**: Python 3.11+  
**Primary Dependencies**: Flask 3.x, Flask-SQLAlchemy 3.x, Flask-Migrate (Alembic), Werkzeug 3.x, Jinja2  
**Storage**: Base de datos relacional gestionada vía SQLAlchemy ORM (SQLite local para desarrollo y pruebas; PostgreSQL compatible para producción)  
**Testing**: `pytest`, `pytest-cov` (ejecución de pruebas de servicios con SQLite en memoria)  
**Target Platform**: Servidor web monolítico ejecutable localmente en Windows y Linux  
**Project Type**: Aplicación web monolítica con renderizado en servidor y JavaScript vanilla auxiliar  
**Performance Goals**: Tiempo de respuesta de endpoints menor a 100 ms para operaciones de tareas  
**Constraints**:
- Cumplimiento innegociable de los 8 principios de la constitución del proyecto.
- Ningún microservicio ni cola de mensajería (Principio I).
- Lógica de negocio estrictamente desacoplada del contexto HTTP (Principio II).
- Contratos de endpoints respetados rigurosamente (Principio III).
- TDD obligatorio y bloqueante antes de escribir código de dominio (Principio IV).
- Migraciones versionadas con Alembic para cambios de esquema (Principio VI).
- Validación/sanitización y verificación de sesión activa en backend en cada endpoint (Principio VII).
- Registro de auditoría con actor, acción, entidad y marca temporal en cada mutación (Principio VIII).  
**Scale/Scope**: Primer incremento focalizado exclusivamente en 6 historias de usuario (*Must*): HU-01, HU-02, HU-03, HU-04, HU-12 y HU-13.

---

## Constitution Check

*GATE: Evaluación previa y posterior al diseño contra la constitución de TaskControl.*

| Principio | Estado | Justificación y Cumplimiento en el Plan |
|---|---|---|
| **I. Monolito por diseño** | **PASS** | Un único repositorio, un único proceso web (`run.py` / Flask) y una única base de datos relacional. No se incluyen microservicios ni colas. |
| **II. Separación en capas** | **PASS** | Estructura con 4 capas explícitas: `models/`, `services/`, `blueprints/` y `templates/` + `static/`. Las rutas nunca tocan el ORM directamente; los servicios no conocen `request` ni `Response`. |
| **III. Contrato explícito backend-JS** | **PASS** | Diseñados y documentados en [contracts/auth-contracts.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/contracts/auth-contracts.md) y [contracts/task-contracts.md](file:///G:/INFO%2003-02-2021/DOCUMENTOS/Jehg/jehg/UNIVERSIDAD/6%20Semestre/Pagina%20Disruptivas/specs/001-core-task-auth/contracts/task-contracts.md) con rutas, métodos, payloads de entrada/salida y códigos de error. |
| **IV. Test-first en lógica de negocio** | **PASS** | La suite `tests/unit/` define pruebas en rojo antes de la implementación para `AuthService` y `TaskService` (transiciones válidas e inválidas, bloqueo de reaperturas, validaciones). Condición bloqueante. |
| **V. Simplicidad sobre generalidad prematura** | **PASS** | Filosofía YAGNI: sin repositorios genéricos artificiales ni motores de plugins. Solo código necesario para las 6 historias del incremento. |
| **VI. Integridad de datos y migraciones** | **PASS** | Configuración de Flask-Migrate con directorio `migrations/` versionado. Esquema relacional con claves foráneas e índices. |
| **VII. Seguridad por defecto** | **PASS** | Hashing con `scrypt`, cookies de sesión HttpOnly y SameSite=Lax, validación exhaustiva en backend contra XSS (texto plano) y decorador `@login_required` para verificación de sesión en cada endpoint de tareas. |
| **VIII. Observabilidad mínima viable** | **PASS** | `AuditService` centralizado que persiste en la tabla `audit_logs` y emite log estructurado en JSON con actor, acción, entidad y timestamp para toda creación, edición o transición de tareas. |

---

## Project Structure

### Documentation (este incremento)

```text
specs/001-core-task-auth/
├── spec.md              # Especificación funcional validada y clarificada
├── plan.md              # Este plan de implementación técnica
├── research.md          # Investigación técnica y decisiones arquitectónicas (Fase 0)
├── data-model.md        # Definición de entidades SQLAlchemy y máquina de estados (Fase 1)
├── quickstart.md        # Guía de inicialización y escenarios de validación (Fase 1)
├── contracts/           # Contratos formales de endpoints HTTP (Fase 1)
│   ├── auth-contracts.md # Contratos para HU-12 y HU-13
│   └── task-contracts.md # Contratos para HU-01 a HU-04
└── checklists/
    └── requirements.md  # Checklist de calidad de requerimientos
```

### Source Code (Estructura Monolítica del Repositorio)

```text
src/
└── taskcontrol/
    ├── __init__.py              # Application Factory (create_app)
    ├── config.py                # Configuración por entornos (Development, Testing, Production)
    ├── extensions.py            # Instancias de SQLAlchemy y Flask-Migrate
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
    │   ├── auth.py              # Rutas /register, /login, /logout
    │   ├── tasks.py             # Rutas /tasks, /tasks/new, /tasks/<id>/edit, /tasks/<id>/status
    │   └── decorators.py        # Decorador @login_required para verificación estricta en backend
    │
    ├── templates/               # Capa 4A: Presentación en servidor (Jinja2)
    │   ├── base.html            # Plantilla maestra (layout común, alertas flash, navbar)
    │   ├── auth/
    │   │   ├── login.html       # Vista de inicio de sesión
    │   │   └── register.html    # Vista de registro de usuario
    │   └── tasks/
    │       ├── index.html       # Listado y filtrado de tareas
    │       ├── new.html         # Formulario de creación de tarea
    │       └── edit.html        # Formulario de edición de tarea
    │
    └── static/                  # Capa 4B: Recursos estáticos del cliente
        ├── css/
        │   └── styles.css       # Estilos visuales de la interfaz
        └── js/
            └── main.js          # JavaScript vanilla mínimo (apoyo a interacción sin framework SPA)

migrations/                      # Scripts versionados de Alembic / Flask-Migrate
tests/
├── conftest.py                  # Fixtures compartidas (app, cliente de pruebas, base de datos en memoria)
├── unit/                        # Pruebas unitarias de dominio (TDD mandatorio antes de implementar servicios)
│   ├── test_auth_service.py     # Pruebas de reglas de usuario y credenciales
│   ├── test_task_service.py     # Pruebas de reglas de tareas y máquina de estados
│   └── test_audit_service.py    # Pruebas de emisión y persistencia de eventos de auditoría
└── integration/                 # Pruebas de integración HTTP y contratos
    ├── test_auth_routes.py      # Verificación de endpoints de autenticación y sesiones
    └── test_task_routes.py      # Verificación de endpoints de tareas y control de acceso

run.py                           # Punto de entrada unificado para levantar la aplicación localmente
requirements.txt                 # Dependencias fijadas del proyecto
```

**Decisión de Estructura**:
Se seleccionó una estructura monolítica empaquetada bajo `src/taskcontrol/` con separación estricta en 4 capas según el Principio II. No se fragmenta en proyectos frontend/backend independientes, ya que la presentación se sirve directamente mediante Jinja2 y JavaScript vanilla en el mismo proceso.

---

## Estrategia de Implementación por Capas

### 1. Capa de Modelos (`models/`)
- Definición declarativa de `User`, `Task` y `AuditLog` utilizando `db.Model`.
- Restricciones relacionales: `user_id` en `tasks` con clave foránea; `email` único e indexado.
- Enumeración de estados en `Task`: `pending`, `in_progress`, `completed`.

### 2. Capa de Servicios (`services/`) — TDD Bloqueante
- **`AuthService`**:
  - `register_user(email, password)`: Normaliza correo, valida complejidad ($\ge 8$ caracteres, letra y número), valida unicidad, genera hash `scrypt` y persiste.
  - `authenticate_user(email, password)`: Busca por email y verifica hash con `check_password_hash`.
- **`TaskService`**:
  - `create_task(user_id, title, description, due_date)`: Valida título no vacío (1-200 caracteres), sanitiza texto plano, asigna estado inicial `pending`, persiste y delega en `AuditService`.
  - `get_user_tasks(user_id, status_filter='all')`: Consulta tareas aisladas estrictamente por `user_id`, ordenadas por `created_at DESC`.
  - `update_task(user_id, task_id, title, description, due_date)`: Verifica propiedad, valida campos, actualiza atributos preservando el estado y delega en `AuditService`.
  - `change_task_status(user_id, task_id, new_status)`: Verifica propiedad, valida transición contra la máquina de estados (bloqueando reaperturas desde `completed`), actualiza estado y delega en `AuditService`.
- **`AuditService`**:
  - `record_task_event(actor_id, action, task_id, details=None)`: Inserta fila en `audit_logs` y emite log estructurado JSON vía logger `taskcontrol.audit`.

### 3. Capa de Blueprints y Seguridad (`blueprints/`)
- Implementación de `auth.py` para registro, login y logout.
- Implementación de `tasks.py` para gestión de tareas.
- `decorators.py`: Implementa `@login_required` para verificar sesión activa en backend en todas las rutas de tareas antes de invocar la capa de servicios.

### 4. Capa de Presentación (`templates/` y `static/`)
- Plantillas Jinja2 limpias y semánticas con manejo de errores y mensajes flash.
- Filtro por estado mediante parámetros query convencionales (`?status=all|pending|in_progress|completed`).
- JavaScript vanilla mínimo para confirmaciones o interacción liviana sin framework pesado.

---

## Complexity Tracking

| Violación Constitucional Potencial | Por Qué se Requiere | Alternativa Más Simple Rechazada |
|---|---|---|
| *Ninguna* | No se detectaron violaciones; el diseño cumple el 100% de la constitución. | N/A |
