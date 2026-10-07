# Research & Technical Decisions: Primer Incremento TaskControl

**Feature**: `001-core-task-auth` (Gestión básica de tareas con autenticación)  
**Date**: 2026-10-06  
**Status**: Completed  

---

## 1. Arquitectura Monolítica y Estructura en Capas

### Contexto y Requisitos
El **Principio I** de la constitución exige un monolito estricto (un solo repositorio, un solo proceso, una sola base de datos relacional). El **Principio II** exige una separación estricta en 4 capas sin saltos: Modelos $\rightarrow$ Servicios (Dominio) $\rightarrow$ Blueprints (HTTP) $\rightarrow$ Presentación (Jinja2 + JS).

### Decisión
Estructurar el código bajo un paquete central `src/taskcontrol/` utilizando el patrón **Application Factory** (`create_app`) de Flask:
- `models/`: Clases SQLAlchemy puras que representan el esquema de base de datos.
- `services/`: Módulos con la lógica de negocio pura (validación de reglas de tarea, transiciones de estado, hashing, orquestación de transacciones). No importan `request`, `g`, ni `Response`.
- `blueprints/`: Controladores HTTP divididos por contexto funcional (`auth.py` y `tasks.py`). Extraen parámetros de la petición HTTP, validan sesión y delegan inmediatamente en la capa de servicios.
- `templates/` y `static/`: Renderizado del lado del servidor con Jinja2 y JavaScript vanilla mínimo para apoyo visual.

### Alternativas Consideradas
- *Estructura plana (todo en app.py)*: Rechazada por violar el Principio II y dificultar las pruebas unitarias aisladas de la lógica de negocio.
- *Arquitectura hexagonal / Clean Architecture con interfaces abstractas y repositorios genéricos*: Rechazada por sobreingeniería prematura violando el Principio V (*Simplicidad sobre generalidad prematura*). SQLAlchemy ya actúa como Data Mapper / Unit of Work.

---

## 2. Gestión de Seguridad y Hashing de Contraseñas (HU-12)

### Contexto y Requisitos
El requerimiento funcional FR-003 y el Principio VII exigen que ninguna contraseña se almacene en texto plano y que se apliquen algoritmos de derivación robustos. Además, la clarificación formalizada exige una longitud mínima de 8 caracteres con al menos una letra y un número.

### Decisión
Utilizar `werkzeug.security` con `generate_password_hash(method='scrypt')` y `check_password_hash()`.
- Algoritmo: `scrypt` (estándar nativo recomendado en Python 3.11+ y Werkzeug moderno, resistente a ataques por hardware acelerado).
- Validación en `AuthService`: Expresión regular previa al hash para verificar `len >= 8` y presencia de `[a-zA-Z]` y `[0-9]`.
- Normalización de correo: `email.strip().lower()` antes de cualquier consulta o almacenamiento para garantizar unicidad insensible a mayúsculas.

### Alternativas Consideradas
- *bcrypt con paquete C externo*: Requiere compiladores C en Windows y configuraciones adicionales de dependencias; `werkzeug.security` con `scrypt` proporciona la misma solidez criptográfica sin dependencias nativas problemáticas.
- *Validación en frontend únicamente*: Prohibida explícitamente por el Principio VII.

---

## 3. Autenticación y Manejo de Sesión (HU-13)

### Contexto y Requisitos
El Principio VII y la clarificación de sesión exigen verificación de sesión activa en el backend en cada endpoint que modifique o consulte datos privados, utilizando una sesión de navegador estándar (no persistente) que se destruya al cerrar el navegador o cerrar sesión.

### Decisión
- **Mecanismo de sesión**: Sesiones firmadas criptográficamente del lado del cliente utilizando la cookie de sesión nativa de Flask protegida con `SECRET_KEY`.
- **Configuración de cookies**:
  - `SESSION_COOKIE_HTTPONLY = True`: Impide el acceso a la cookie desde scripts del cliente (mitiga XSS).
  - `SESSION_COOKIE_SAMESITE = 'Lax'`: Protección contra ataques de Cross-Site Request Forgery (CSRF).
  - `SESSION_PERMANENT = False`: La cookie expira al cerrar la sesión del navegador.
- **Protección de rutas en backend**: Un decorador reutilizable `@login_required` en la capa de blueprints que verifica `session.get('user_id')`. Si no existe sesión activa, redirige inmediatamente a `/login` con un mensaje flash o responde 401 Unauthorized.
- **Aislamiento de recursos**: Los endpoints de tareas siempre pasan el `current_user_id` obtenido de la sesión al `TaskService`, asegurando que ninguna consulta ni mutación opere fuera del ámbito del usuario autenticado.

---

## 4. Auditoría Estructurada Centralizada (Principio VIII)

### Contexto y Requisitos
El Principio VIII exige que toda mutación sobre tareas (creación, cambio de estado, edición) genere un registro de auditoría estructurado con `actor`, `action`, `entity`, `timestamp` y detalles desde el primer incremento. No debe duplicarse esta lógica en cada ruta.

### Decisión
Centralizar la auditoría en la capa de servicios mediante un módulo dedicado `audit_service.py`:
1. **Persistencia**: Se persiste una fila en la tabla relacional `audit_logs` con `actor_id`, `action`, `entity_type='Task'`, `entity_id`, `details` (JSON o dict serializado) y `timestamp` (UTC).
2. **Logging en aplicación**: Simultáneamente, se emite un registro estructurado a través de `logging.getLogger('taskcontrol.audit')` formateado como JSON para trazabilidad en consola y archivos de log.
3. **Punto de integración**: `TaskService` invoca `AuditService.record_event()` dentro de la misma transacción atómica de base de datos antes del `commit`. Las rutas HTTP nunca invocan auditoría directamente.

### Alternativas Consideradas
- *Hooks / signals de SQLAlchemy (`after_insert`, `after_update`)*: Añaden complejidad oculta y dificultan identificar el `actor` de negocio cuando se opera fuera del ciclo de petición HTTP o en tests de servicio.
- *Llamadas manuales en cada función de Blueprint*: Rechazadas por violar el principio DRY y permitir que una ruta olvide auditar una acción.

---

## 5. Estrategia de Migraciones de Base de Datos (Principio VI)

### Contexto y Requisitos
El Principio VI prohíbe modificaciones manuales directas a la base de datos y exige migraciones versionadas y reproducibles mediante Flask-Migrate / Alembic.

### Decisión
- Configurar `Flask-Migrate` vinculado a la instancia única de `SQLAlchemy`.
- Directorio de migraciones en la raíz del repositorio: `migrations/`.
- Flujo de inicialización:
  1. `flask db init`: Genera el entorno de Alembic (se ejecuta una única vez al configurar el repositorio).
  2. `flask db migrate -m "initial_schema_users_tasks_audit"`: Detecta los modelos `User`, `Task` y `AuditLog` y genera el script versionado.
  3. `flask db upgrade`: Aplica los cambios sobre el motor de base de datos relacional (SQLite local `taskcontrol.db`).

---

## 6. Estrategia de Pruebas y TDD (Principio IV)

### Contexto y Requisitos
El Principio IV impone Test-First (TDD) bloqueante para la lógica de negocio y transiciones de estado.

### Decisión
Estructurar las pruebas con `pytest` en dos niveles estrictamente delimitados:
1. **Pruebas de Unidad / Dominio (`tests/unit/`)**:
   - Validan `AuthService`, `TaskService` y `AuditService`.
   - Se ejecutan contra una base de datos SQLite en memoria (`sqlite:///:memory:`).
   - Prueban: validación de contraseñas, unicidad de correos, transiciones permitidas e inválidas de tareas, bloqueo de reapertura de tareas completadas, y registro de auditoría.
   - **Regla bloqueante**: Estas pruebas deben escribirse en rojo (Red) antes de escribir la lógica del servicio correspondiente.
2. **Pruebas de Integración y Rutas (`tests/integration/`)**:
   - Validan los Blueprints HTTP utilizando el `test_client` de Flask.
   - Prueban: cumplimiento del contrato HTTP (códigos de estado, redirecciones, cookies de sesión, headers), validación de formularios y rechazo de acceso no autenticado o acceso cruzado entre cuentas.
