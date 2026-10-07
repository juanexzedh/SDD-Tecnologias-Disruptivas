# Contratos HTTP: Autenticación y Cuentas (HU-12, HU-13)

**Feature**: `001-core-task-auth`  
**Date**: 2026-10-06  
**Principio Constitucional**: III (Contrato explícito previo a la implementación)  

---

## 1. Registro de Usuario (`HU-12`)

### 1.1 `GET /register`
- **Descripción**: Muestra la interfaz con el formulario de registro de nuevos usuarios.
- **Autenticación requerida**: No. Si ya existe sesión activa, redirige (302) a `/tasks`.
- **Códigos de Respuesta**:
  - `200 OK`: Renderiza plantilla `auth/register.html`.

### 1.2 `POST /register`
- **Descripción**: Procesa la creación de una nueva cuenta de usuario.
- **Autenticación requerida**: No.
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "email": "usuario@ejemplo.com",
    "password": "Password123"
  }
  ```
- **Validaciones en Backend**:
  - `email`: Obligatorio, formato válido de correo, normalizado con trim y minúsculas.
  - `password`: Obligatorio, longitud mínima de 8 caracteres, al menos 1 letra y al menos 1 número.
- **Respuestas**:
  - `302 Found` (Navegador): Redirección a `/login` con mensaje flash informativo `"Cuenta creada exitosamente. Por favor, inicia sesión."`.
  - `201 Created` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Usuario registrado exitosamente",
      "data": {
        "id": 1,
        "email": "usuario@ejemplo.com"
      }
    }
    ```
  - `400 Bad Request`:
    ```json
    {
      "status": "error",
      "code": "VALIDATION_ERROR",
      "message": "La contraseña debe tener al menos 8 caracteres y contener al menos una letra y un número.",
      "errors": {
        "password": ["Debe contener al menos 8 caracteres, una letra y un número."]
      }
    }
    ```
  - `409 Conflict`:
    ```json
    {
      "status": "error",
      "code": "EMAIL_ALREADY_REGISTERED",
      "message": "El correo electrónico ya se encuentra registrado."
    }
    ```

---

## 2. Inicio y Cierre de Sesión (`HU-13`)

### 2.1 `GET /login`
- **Descripción**: Muestra la interfaz del formulario de inicio de sesión.
- **Autenticación requerida**: No. Si ya existe sesión activa, redirige (302) a `/tasks`.
- **Códigos de Respuesta**:
  - `200 OK`: Renderiza plantilla `auth/login.html`.

### 2.2 `POST /login`
- **Descripción**: Valida credenciales e inicializa la sesión de usuario activa.
- **Autenticación requerida**: No.
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "email": "usuario@ejemplo.com",
    "password": "Password123"
  }
  ```
- **Respuestas**:
  - `302 Found` (Navegador): Establece cookie de sesión firmada `session['user_id'] = user.id` y redirige a `/tasks`.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Sesión iniciada exitosamente",
      "data": {
        "id": 1,
        "email": "usuario@ejemplo.com"
      }
    }
    ```
  - `400 Bad Request`:
    ```json
    {
      "status": "error",
      "code": "MISSING_FIELDS",
      "message": "Correo y contraseña son requeridos."
    }
    ```
  - `401 Unauthorized`:
    ```json
    {
      "status": "error",
      "code": "INVALID_CREDENTIALS",
      "message": "Correo o contraseña incorrectos."
    }
    ```

### 2.3 `POST /logout`
- **Descripción**: Finaliza y destruye la sesión activa en el servidor.
- **Autenticación requerida**: Sí (`@login_required`). Si no está autenticado, responde 302 a `/login`.
- **Payload de Entrada**: Ninguno.
- **Respuestas**:
  - `302 Found` (Navegador): Elimina los datos de sesión (`session.clear()`) y redirige a `/login` con mensaje flash.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Sesión finalizada exitosamente"
    }
    ```
