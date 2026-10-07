# Contratos HTTP: Autenticación y Cuentas (HU-12, HU-13)

**Feature**: `001-core-task-auth`  
**Date**: 2026-10-06  
**Principio Constitucional**: III (Contrato explícito previo a la implementación) y VII (Seguridad por defecto con protección CSRF)  

---

## 1. Registro de Usuario (`HU-12`)

### 1.1 `GET /register`
- **Descripción**: Muestra la interfaz con el formulario de registro de nuevos usuarios.
- **Autenticación requerida**: No. Si ya existe sesión activa, redirige (302) a `/tasks`.
- **Códigos de Respuesta**:
  - `200 OK`: Renderiza plantilla `auth/register.html` conteniendo el token CSRF (`{{ csrf_token() }}`).

### 1.2 `POST /register`
- **Descripción**: Procesa la creación de una nueva cuenta de usuario.
- **Autenticación requerida**: No.
- **Protección CSRF**: Requerida (Principio VII). Debe enviarse `csrf_token` como campo de formulario o cabecera `X-CSRFToken`.
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "csrf_token": "IjNkYj...token_firmado_csrf",
    "email": "usuario@ejemplo.com",
    "password": "Password123"
  }
  ```
- **Validaciones en Backend**:
  - `csrf_token`: Obligatorio y válido contra la sesión actual.
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
  - `400 Bad Request` (Fallo de Validación CSRF):
    ```json
    {
      "status": "error",
      "code": "CSRF_ERROR",
      "message": "Token CSRF ausente o inválido."
    }
    ```
  - `400 Bad Request` (Fallo de Validación de Negocio):
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
  - `200 OK`: Renderiza plantilla `auth/login.html` conteniendo el token CSRF.

### 2.2 `POST /login`
- **Descripción**: Valida credenciales e inicializa la sesión de usuario activa.
- **Autenticación requerida**: No.
- **Protección CSRF**: Requerida (Principio VII). Debe enviarse `csrf_token` como campo de formulario o cabecera `X-CSRFToken`.
- **Payload de Entrada** (`application/x-www-form-urlencoded` o `application/json`):
  ```json
  {
    "csrf_token": "IjNkYj...token_firmado_csrf",
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
  - `400 Bad Request` (Fallo CSRF o campos incompletos):
    ```json
    {
      "status": "error",
      "code": "CSRF_ERROR",
      "message": "Token CSRF ausente o inválido."
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
- **Protección CSRF**: Requerida (Principio VII). Se invoca mediante formulario POST con `csrf_token` para evitar desautenticación forzada por CSRF.
- **Payload de Entrada**:
  ```json
  {
    "csrf_token": "IjNkYj...token_firmado_csrf"
  }
  ```
- **Respuestas**:
  - `302 Found` (Navegador): Elimina los datos de sesión (`session.clear()`) y redirige a `/login` con mensaje flash.
  - `200 OK` (API/JSON):
    ```json
    {
      "status": "success",
      "message": "Sesión finalizada exitosamente"
    }
    ```
  - `400 Bad Request` (Fallo CSRF): Token ausente o manipulado.
