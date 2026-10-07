# Research & Technical Decisions: Primer Incremento TaskControl

**Feature**: `001-core-task-auth` (Gestión básica de tareas con autenticación)  
**Date**: 2026-10-06  
**Status**: Completed (Actualizado con correcciones de CSRF, Secretos y CLI)  

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

## 2. Gestión de Seguridad, Hashing y Manejo de Secretos (Principio VII)

### Contexto y Requisitos
El requerimiento funcional FR-003 y el Principio VII exigen que ninguna contraseña se almacene en texto plano, que se apliquen algoritmos de derivación robustos y que **ningún secreto se almacene en el repositorio**.

### Decisión sobre Hashing
Utilizar `werkzeug.security` con `generate_password_hash(method='scrypt')` y `check_password_hash()`.
- Algoritmo: `scrypt` (estándar nativo recomendado en Python 3.11+ y Werkzeug moderno, resistente a ataques por hardware acelerado).
- Validación en `AuthService`: Expresión regular previa al hash para verificar `len >= 8` y presencia de `[a-zA-Z]` y `[0-9]`.
- Normalización de correo: `email.strip().lower()` antes de cualquier consulta o almacenamiento para garantizar unicidad insensible a mayúsculas.

### Decisión sobre `SECRET_KEY`
- **Cero secretos en el repositorio**: La clave criptográfica `SECRET_KEY` se lee exclusivamente desde variables de entorno (`os.environ.get('SECRET_KEY')`).
- **Archivo de referencia `.env.example`**: Versionado en la raíz con valores de plantilla (`SECRET_KEY=cambiar-por-clave-secreta-en-produccion`, `DATABASE_URL=sqlite:///taskcontrol.db`), mientras que el archivo `.env` real está ignorado en `.gitignore`.
- **Estrategia por entornos (`config.py`)**:
  - `DevelopmentConfig`: Lee `SECRET_KEY` del entorno o `.env`. Si no está definida, emite advertencia de desarrollo con fallback temporal no seguro solo para desarrollo local.
  - `ProductionConfig`: Valida obligatoriamente `os.environ['SECRET_KEY']` y lanza un error fatal (`ValueError`) en el arranque si no está configurada.
  - `TestingConfig`: Define explícitamente `SECRET_KEY = 'test-secret-key-taskcontrol-testing'` dentro del código de pruebas para garantizar que la suite automatizada sea 100% reproducible y no dependa de archivos externos `.env`.

---

## 3. Protección CSRF y Manejo de Sesión (Principios V y VII)

### Contexto y Requisitos
El Principio VII exige seguridad por defecto en todos los endpoints que modifiquen datos. Los navegadores envían cookies de sesión automáticamente, lo que expone a formularios `POST` (registro, login, logout, creación, edición, cambio de estado de tareas) a ataques de Cross-Site Request Forgery (CSRF). El Principio V exige simplicidad y evitar capas innecesarias.

### Decisión de Protección CSRF
Adoptar `CSRFProtect` proveniente de la biblioteca estándar `Flask-WTF`.
- **Justificación respecto al Principio V (Simplicidad)**:
  No se introducen clases de formulario complejas (`FlaskForm` / WTForms) con validadores duplicados o meta-esquemas pesados. Se integra únicamente la extensión `CSRFProtect(app)` en su modalidad independiente y liviana:
  1. Genera tokens criptográficos por sesión mediante `{{ csrf_token() }}` disponible globalmente en Jinja2.
  2. En cada formulario HTML `POST`, se inserta un campo oculto: `<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">`.
  3. En solicitudes asíncronas con JavaScript (fetch), se envía mediante la cabecera HTTP estándar `X-CSRFToken`.
  4. La extensión intercepta automáticamente toda petición `POST`, `PUT`, `DELETE` antes de que alcance el blueprint. Si el token está ausente o es inválido, rechaza la solicitud de inmediato con código **`400 Bad Request`** y mensaje de error específico (`CSRF_TOKEN_MISSING` / `CSRF_TOKEN_INVALID`).
- **Estrategia en Pruebas de Integración**:
  - En `TestingConfig`, se establece `WTF_CSRF_ENABLED = False` para las pruebas funcionales de rutas y contratos de negocio, permitiendo probar la lógica HTTP sin sobrecarga de parsing de tokens en cada test.
  - Se implementa una suite de prueba específica (`test_csrf.py`) con `WTF_CSRF_ENABLED = True` para verificar exhaustivamente que cualquier petición `POST` sin token o con token alterado sea rechazada con `400 Bad Request`.

### Decisión de Sesión
- **Mecanismo de sesión**: Cookies de sesión firmadas nativas de Flask protegidas con `SECRET_KEY`.
- **Configuración de cookies**:
  - `SESSION_COOKIE_HTTPONLY = True`: Mitiga robo de sesión por XSS.
  - `SESSION_COOKIE_SAMESITE = 'Lax'`: Primera barrera de defensa contra CSRF.
  - `SESSION_PERMANENT = False`: La cookie expira al cerrar la sesión del navegador.
- **Protección de rutas en backend**: Decorador `@login_required` en la capa de blueprints para verificar `session.get('user_id')`. Si no existe sesión, redirige a `/login` o responde 401.

---

## 4. Configuración del Entorno y Ejecución desde la Raíz (Restricciones del Stack)

### Contexto y Requisitos
Las restricciones de la constitución exigen que el entorno pueda levantarse desde la raíz con comandos directos estándar:
```bash
pip install -r requirements.txt
flask db upgrade
flask run
```
Con la estructura en paquete `src/taskcontrol/`, el comando `flask` requiere resolver `FLASK_APP` sin requerir que el usuario configure variables de entorno manualmente.

### Decisión
1. **Dependencia `python-dotenv`**: Incluida en `requirements.txt`. El CLI de `flask` integra soporte nativo para `python-dotenv` y busca automáticamente archivos `.flaskenv` y `.env` en la raíz al invocar cualquier comando `flask`.
2. **Archivo `.flaskenv` versionado en la raíz**:
   ```env
   FLASK_APP=src/taskcontrol:create_app
   FLASK_ENV=development
   ```
3. **Resolución de rutas en `create_app`**:
   En `src/taskcontrol/__init__.py`, se garantiza que `src` esté en `sys.path` al inicio de la carga (`sys.path.insert(0, os.path.abspath('src'))`), permitiendo que tanto `flask db upgrade` como `flask run` resuelvan los módulos de la aplicación y las migraciones limpiamente desde la raíz del repositorio sin pasos previos.

---

## 5. Auditoría Estructurada Centralizada (Principio VIII)

### Contexto y Requisitos
El Principio VIII exige que toda mutación sobre tareas (creación, cambio de estado, edición) genere un registro de auditoría estructurado con `actor`, `action`, `entity`, `timestamp` y detalles desde el primer incremento. No debe duplicarse esta lógica en cada ruta.

### Decisión
Centralizar la auditoría en la capa de servicios mediante un módulo dedicado `audit_service.py`:
1. **Persistencia**: Se persiste una fila en la tabla relacional `audit_logs` con `actor_id`, `action`, `entity_type='Task'`, `entity_id`, `details` (JSON serializado) y `timestamp` (UTC).
2. **Logging en aplicación**: Simultáneamente, se emite un registro estructurado a través de `logging.getLogger('taskcontrol.audit')` formateado como JSON para trazabilidad en consola y archivos de log.
3. **Punto de integración**: `TaskService` invoca `AuditService.record_event()` dentro de la misma transacción atómica de base de datos antes del `commit`. Las rutas HTTP nunca invocan auditoría directamente.

---

## 6. Estrategia de Migraciones de Base de Datos (Principio VI)

### Decisión
- Configurar `Flask-Migrate` vinculado a la instancia única de `SQLAlchemy`.
- Directorio de migraciones en la raíz del repositorio: `migrations/`.
- Flujo de inicialización:
  1. `flask db init`: Genera el entorno de Alembic.
  2. `flask db migrate -m "initial_schema_users_tasks_audit"`: Detecta los modelos `User`, `Task` y `AuditLog` y genera el script versionado.
  3. `flask db upgrade`: Aplica los cambios sobre el motor de base de datos relacional (`taskcontrol.db`).

---

## 7. Estrategia de Pruebas y TDD (Principio IV)

### Decisión
Estructurar las pruebas con `pytest` en dos niveles estrictamente delimitados:
1. **Pruebas de Unidad / Dominio (`tests/unit/`)**:
   - Validan `AuthService`, `TaskService` y `AuditService` con TDD bloqueante (pruebas en rojo antes del código).
   - Base de datos SQLite en memoria (`sqlite:///:memory:`).
2. **Pruebas de Integración y Rutas (`tests/integration/`)**:
   - Validan Blueprints HTTP, control de acceso (`@login_required`), aislamiento por usuario y contratos de respuesta.
   - Incluyen `test_csrf.py` para probar la barrera de seguridad de tokens CSRF.
