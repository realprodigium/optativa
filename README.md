# Proyecto Final V1 | Vulnerabilidades deliberadas

**Google OAuth + FastAPI (versión inicial insegura)**

## overview

version inicial de una webapp sencilla que utiliza auth federada con Google OAuth 2.0, desarrollada con fastapi en el backend y HTML/JS puro en el frontend.

El objetivo de esta primera --v no es la seguridad, sino contar con un sistema funcional pero deliberadamente inseguro, similar a implementaciones reales mal diseñadas, con el fin de identificar, analizar y posteriormente mitigar vulnerabilidades o en sintesis tener un sistema seguro, basandonos en OWASP TOP10.

---

## Descripcion

* Backend monolítico en FastAPI.
* Autenticación delegada a Google OAuth.
* Uso de `authlib` para el flujo OAuth.
* Renderizado de `index.html` con un botón de login.
* Almacenamiento del token OAuth completo en la sesión del servidor.
* Secrets y claves definidos directamente en el código.

No se implementa:

* Autorización por usuario.
* Control de permisos.
* Endurecimiento de sesiones.
* Separación por capas.
* Manejo seguro de secretos.

---

## Vulnerabilidades identificadas

### Exposición de secretos en el código fuente

El `CLIENT_ID`, `CLIENT_SECRET` de Google OAuth y la `SECRET_KEY` de la aplicación están definidos directamente en el archivo Python.

```python
CLIENT_ID = "..."
CLIENT_SECRET = "..."
SECRET_KEY = "app-secret-key"
```

**Riesgo**
Cualquier persona con acceso al repositorio puede:

* Suplantar la app frente a Google.
* Emitir tokens OAuth validos
* Firmar o falsificar cookies de session

**OWASP**

* API8: Security Misconfiguration
* A2: Cryptographic Failures

**Impacto**
Compromiso total de la autenticación federada y de las sesiones de usuario.

---

### Gestion insegura de sesiones

**Descripción**
La aplicación utiliza `SessionMiddleware` sin configuraciones adicionales de seguridad.
Las cookies de sesión no tienen flags como `HttpOnly`, `Secure` o `SameSite`, y la clave de firmado es débil y pública.

```python
app.add_middleware(SessionMiddleware, secret_key=SECRET_KEY)
```

**Riesgo**

* Session hijacking.
* Session fixation.
* Robo de identidad del usuario autenticado.

**OWASP**

* API2: Broken Authentication

**Impacto**
Un atacante podría reutilizar o secuestrar sesiones activas y acceder a información del usuario autenticado.

---

### Exposición excesiva de tokens OAuth

**Descripción**
El token OAuth completo (access token, id token, refresh token y userinfo) se almacena directamente en la sesión del servidor.

```python
request.session["user"] = token
```

No existe:

* control de expiración
* validación manual de claims
* reducción de información almacenada

**Riesgo**
Si la sesión es comprometida:

* acceso directo a APIs de Google en nombre del usuario
* exposición de datos personales

**OWASP**

* API2: Broken Authentication
* A3: Sensitive Data Exposure

**Impacto**
Acceso no autorizado a recursos del usuario en Google (perfil, email, APIs asociadas).

---

### Falta de validación del flujo OAuth (CSRF / state)

**Descripción**
El callback de OAuth (`/authorize`) no valida explícitamente el parámetro `state`, dejando el flujo expuesto a ataques de tipo CSRF.

```python
token = await oauth.google.authorize_access_token(request)
```

**Riesgo**
Un atacante podría forzar la asociación de una sesión válida con un usuario distinto.

**OWASP**

* API5: Broken Function Level Authorization

**Impacto**
Confusión de identidad y autenticación incorrecta de usuarios.

---

## por ahora

Aunque se utiliza un proveedor confiable como Google para la auth, el sistema demuestra que delegarla no garantiza seguridad por sí sola.
Las principales vulnerabilidades no provienen de Google, sino de cómo la aplicación maneja sesiones, tokens y secretos.

Este tipo de errores son comunes en aplicaciones reales desarrolladas bajo presión o desconocimiento de prácticas de seguridad.

...............