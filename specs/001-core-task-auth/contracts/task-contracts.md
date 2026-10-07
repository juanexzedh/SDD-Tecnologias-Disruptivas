# Contratos HTTP: Gestión de Tareas (HU-01 a HU-04)

**Feature**: `001-core-task-auth`  
**Date**: 2026-10-06  
**Principio Constitucional**: III (Contrato explícito previo a la implementación) y VII (Seguridad por defecto)  

---

## 1. Listado y Filtrado de Tareas (`HU-02`)

### 1.1 `GET /tasks`
- **Descripción**: Retorna las tareas pertenecientes al usuario autenticado, ordenadas por defecto por fecha de creación descendente (`created_at DESC`), con opción de filtrado por estado.
- **Autenticación requerida**: **SÍ** (`@login_required`). Si no está autenticado, redirige (302) a `/login` o responde 401.
- **Parámetros de Consulta (Query String)**:
  - `status` (opcional, string): Valores admitidos: `'all'`, `'pending'`, `'in_progress'`, `'completed'`. Por defecto: `'all'`.
- **Respuestas**:
  - `200 OK` (Navegador): Renderiza plantilla `tasks/index.html` con la lista de tareas del usuario actual.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "data": {
        "tasks": [
          {
            "id": 10,
            "title": "Configurar entorno de desarrollo",
            "description": "Instalar dependencias y ejecutar migraciones iniciales.",
            "due_date": "2026-10-15",
            "status": "pending",
            "created_at": "2026-10-06T14:30:00Z"
          }
        ],
        "filter": "all",
        "total": 1
      }
    }
    ```
  - `401 Unauthorized`:
    ```json
    {
      "status": "error",
      "code": "UNAUTHORIZED",
      "message": "Se requiere autenticación activa para consultar tareas."
    }
    ```

---

## 2. Creación de Tareas (`HU-01`)

### 2.1 `GET /tasks/new`
- **Descripción**: Muestra la interfaz con el formulario para registrar una nueva tarea.
- **Autenticación requerida**: **SÍ** (`@login_required`).
- **Respuestas**:
  - `200 OK`: Renderiza plantilla `tasks/new.html`.
  - `302 Found`: Redirección a `/login` si no está autenticado.

### 2.2 `POST /tasks`
- **Descripción**: Crea una nueva tarea en estado `'pending'` asociada al usuario autenticado y genera un evento de auditoría (`TASK_CREATED`).
- **Autenticación requerida**: **SÍ** (`@login_required`).
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "title": "Implementar endpoints de tareas",
    "description": "Texto plano sin etiquetas HTML ni scripts.",
    "due_date": "2026-10-12"
  }
  ```
- **Validaciones en Backend**:
  - `title`: Obligatorio, cadena no vacía (trim), longitud de 1 a 200 caracteres.
  - `description`: Opcional, texto plano (hasta 2000 caracteres). Caracteres especiales son escapados para neutralizar XSS; se preservan saltos de línea.
  - `due_date`: Opcional, formato de fecha válido (`YYYY-MM-DD`). Se admiten fechas pasadas, presentes o futuras.
- **Respuestas**:
  - `302 Found` (Navegador): Redirección a `/tasks` con mensaje flash `"Tarea creada exitosamente."`.
  - `201 Created` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Tarea creada exitosamente",
      "data": {
        "id": 1,
        "title": "Implementar endpoints de tareas",
        "description": "Texto plano sin etiquetas HTML ni scripts.",
        "due_date": "2026-10-12",
        "status": "pending",
        "created_at": "2026-10-06T15:00:00Z"
      }
    }
    ```
  - `400 Bad Request`:
    ```json
    {
      "status": "error",
      "code": "VALIDATION_ERROR",
      "message": "El título es obligatorio y no puede estar vacío.",
      "errors": {
        "title": ["El título es obligatorio."]
      }
    }
    ```
  - `401 Unauthorized`: Acceso denegado por falta de sesión activa.

---

## 3. Edición de Tareas (`HU-04`)

### 3.1 `GET /tasks/<int:task_id>/edit`
- **Descripción**: Muestra la interfaz con el formulario prellenado para editar una tarea existente.
- **Autenticación requerida**: **SÍ** (`@login_required`).
- **Respuestas**:
  - `200 OK`: Renderiza plantilla `tasks/edit.html` con los datos de la tarea.
  - `404 Not Found` / `403 Forbidden`: Si la tarea no existe o no pertenece al usuario autenticado.

### 3.2 `POST /tasks/<int:task_id>/edit`
- **Descripción**: Actualiza los atributos de una tarea (título, descripción, fecha límite) preservando su estado actual y registrando el evento de auditoría (`TASK_UPDATED`).
- **Autenticación requerida**: **SÍ** (`@login_required`).
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "title": "Título actualizado de la tarea",
    "description": "Nueva descripción en texto plano.",
    "due_date": "2026-10-20"
  }
  ```
- **Respuestas**:
  - `302 Found` (Navegador): Redirección a `/tasks` con mensaje flash `"Tarea actualizada exitosamente."`.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Tarea actualizada exitosamente",
      "data": {
        "id": 1,
        "title": "Título actualizado de la tarea",
        "description": "Nueva descripción en texto plano.",
        "due_date": "2026-10-20",
        "status": "pending",
        "updated_at": "2026-10-06T15:30:00Z"
      }
    }
    ```
  - `400 Bad Request`: Título vacío, longitud excedida o formato de fecha inválido.
  - `403 Forbidden` / `404 Not Found`: Tarea no encontrada o perteneciente a otro usuario (cero fuga de datos).

---

## 4. Transición de Estado de Tareas (`HU-03`)

### 4.1 `POST /tasks/<int:task_id>/status`
- **Descripción**: Modifica el estado del ciclo de vida de la tarea según la máquina de estados permitida y genera un evento de auditoría (`TASK_STATUS_CHANGED`).
- **Autenticación requerida**: **SÍ** (`@login_required`).
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "status": "in_progress"
  }
  ```
- **Valores admitidos para `status`**: `'pending'`, `'in_progress'`, `'completed'`.
- **Transiciones válidas**:
  - `pending` $\rightarrow$ `in_progress` | `completed`
  - `in_progress` $\rightarrow$ `pending` | `completed`
- **Transiciones rechazadas**:
  - `completed` $\rightarrow$ cualquier estado (rechazada con 400 en este incremento; requiere reapertura HU-06).
  - Estado actual $\rightarrow$ mismo estado (rechazada con 400).
- **Respuestas**:
  - `302 Found` (Navegador): Redirección a `/tasks` tras actualizar el estado.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Estado de la tarea actualizado exitosamente",
      "data": {
        "id": 1,
        "old_status": "pending",
        "new_status": "in_progress"
      }
    }
    ```
  - `400 Bad Request`:
    ```json
    {
      "status": "error",
      "code": "INVALID_STATE_TRANSITION",
      "message": "No se permite cambiar el estado de una tarea completada en este incremento."
    }
    ```
  - `403 Forbidden` / `404 Not Found`: Tarea no encontrada o no perteneciente al usuario autenticado.
