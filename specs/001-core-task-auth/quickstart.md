# Quickstart & Guía de Validación: TaskControl (Incremento 1)

**Feature**: `001-core-task-auth`  
**Date**: 2026-10-06  
**Stack**: Python 3.11+, Flask, SQLAlchemy, Flask-Migrate, pytest  

---

## 1. Prerrequisitos

- Python 3.11 o superior instalado.
- Entorno virtual de Python configurado.

---

## 2. Puesta en Marcha con un Solo Comando (Principio I & Restricciones)

### 2.1 Instalación de dependencias
```bash
python -m venv .venv
# En Windows PowerShell:
.venv\Scripts\Activate.ps1
# O en bash:
source .venv/bin/activate

pip install -r requirements.txt
```

### 2.2 Variables de entorno mínimas (`.env`)
Crear un archivo `.env` en la raíz (no versionado):
```env
FLASK_APP=src/taskcontrol:create_app
FLASK_ENV=development
SECRET_KEY=dev-secret-key-taskcontrol-monolith
DATABASE_URL=sqlite:///taskcontrol.db
```

### 2.3 Inicialización de Base de Datos y Migraciones (Principio VI)
```bash
# Aplicar migraciones iniciales para crear users, tasks y audit_logs
flask db upgrade
```

### 2.4 Ejecución de la Aplicación
```bash
python run.py
# La aplicación queda disponible en http://127.0.0.1:5000
```

---

## 3. Ejecución de Pruebas Automatizadas (TDD - Principio IV)

Para verificar el cumplimiento del principio Test-First y los contratos antes o después de la implementación:

```bash
# 1. Ejecutar pruebas unitarias de la capa de dominio (servicios y reglas bloqueantes)
pytest tests/unit/ -v

# 2. Ejecutar pruebas de integración de rutas HTTP y contratos
pytest tests/integration/ -v

# 3. Ejecutar toda la suite con reporte de cobertura
pytest --cov=src/taskcontrol tests/
```

---

## 4. Escenarios de Validación Extremo a Extremo (Manual o curl)

### Escenario 1: Registro de Usuario (HU-12)
1. Navegar a `http://127.0.0.1:5000/register`.
2. Intentar registrar `usuario@ejemplo.com` con contraseña `corta` (menos de 8 caracteres).
   - **Resultado esperado**: Error de validación; el sistema rechaza el formulario.
3. Registrar `usuario@ejemplo.com` con `Password123` (8+ caracteres, letra y número).
   - **Resultado esperado**: Redirección a `/login` con mensaje de bienvenida; la contraseña queda hasheada con `scrypt` en la base de datos.

### Escenario 2: Inicio y Control de Sesión (HU-13)
1. Navegar a `http://127.0.0.1:5000/tasks` sin autenticación previa.
   - **Resultado esperado**: Redirección inmediata a `/login` (Principio VII).
2. En `/login`, ingresar `usuario@ejemplo.com` y `Password123`.
   - **Resultado esperado**: Redirección a `/tasks`, cookie de sesión HttpOnly establecida.

### Escenario 3: Creación y Listado de Tareas (HU-01, HU-02)
1. En `/tasks`, hacer clic en "Nueva Tarea" o navegar a `/tasks/new`.
2. Registrar una tarea con título `"Mi primera tarea"`, descripción `"Detalles de prueba"` y fecha `"2026-10-20"`.
   - **Resultado esperado**: Redirección a `/tasks`. La tarea aparece en la lista con estado `"pendiente"`. Se genera una fila en `audit_logs` con `action='TASK_CREATED'`.
3. Crear una segunda tarea con fecha pasada o sin fecha.
   - **Resultado esperado**: La tarea más reciente se ubica al principio del listado (orden descendente por fecha de creación).

### Escenario 4: Transición de Estados (HU-03)
1. En la lista de tareas, cambiar el estado de la tarea de `"pendiente"` a `"en progreso"`.
   - **Resultado esperado**: El estado se actualiza en pantalla y en la base de datos; se crea un registro de auditoría `TASK_STATUS_CHANGED`.
2. Cambiar de `"en progreso"` a `"completada"`.
   - **Resultado esperado**: El estado se actualiza a completada; auditoría registrada.
3. Intentar volver de `"completada"` a `"pendiente"`.
   - **Resultado esperado**: La acción es bloqueada con mensaje de error (no se permite reapertura directa en este incremento).

### Escenario 5: Edición y Aislamiento de Usuario (HU-04, HU-02)
1. Editar la tarea creada modificando su título a `"Mi primera tarea (editada)"`.
   - **Resultado esperado**: Se guarda el cambio, se preserva el estado y se registra evento `TASK_UPDATED`.
2. Iniciar sesión con un segundo usuario `otro@ejemplo.com`.
   - **Resultado esperado**: Su listado de tareas está completamente vacío. Intentar acceder a `/tasks/1/edit` del primer usuario responde `404 Not Found` o `403 Forbidden` (cero fuga de información).
