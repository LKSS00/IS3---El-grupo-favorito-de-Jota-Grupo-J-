# Auth Spec

## 1. Objetivo y Contexto

El módulo de autenticación gestiona la identidad de los usuarios en la plataforma **Academic Event Manager**. Proporciona mecanismos de registro, login, renovación de tokens y cierre de sesión mediante JWT (JSON Web Tokens). Es el primer módulo a implementar ya que todos los demás módulos dependen de él para el control de acceso.

**Tecnologías:** bcrypt (hashing), JWT (tokens), Zod (validación), Prisma ORM (persistencia).

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-A1: Registro de usuario

**Como** usuario nuevo, **quiero** crear una cuenta con mis datos personales, **para** poder acceder a la plataforma.

**Criterios de aceptación:**
- El email debe tener formato válido y no estar registrado previamente
- La contraseña debe tener mínimo 8 caracteres, una mayúscula, un número y un carácter especial
- Se requiere confirmación de contraseña que coincida con la original
- El nombre y apellido deben tener al menos 2 caracteres
- Al registrarse exitosamente, el usuario recibe el rol `participant` por defecto
- Si el email ya existe, se devuelve error 409 con código `CONFLICT`

### HU-A2: Inicio de sesión

**Como** usuario registrado, **quiero** iniciar sesión con mi email y contraseña, **para** acceder a mi cuenta y funcionalidades protegidas.

**Criterios de aceptación:**
- El sistema valida email y contraseña contra la base de datos
- Si las credenciales son correctas, se devuelve access token + refresh token
- El JWT incluye `sub` (UUID), `email`, `role`, `iat`, `exp`
- Si las credenciales son incorrectas, se devuelve 401 con código `AUTH_REQUIRED`
- Los datos de entrada se validan con Zod antes de procesar

### HU-A3: Renovación de token

**Como** usuario con sesión activa, **quiero** que mi sesión se renueve automáticamente, **para** no tener que volver a loguearme cuando el access token expire.

**Criterios de aceptación:**
- El refresh token se envía en el body de la petición
- Si el refresh token es válido, se genera un nuevo par de tokens (token rotation)
- Si el refresh token está expirado o es inválido, se devuelve 401 con `INVALID_TOKEN`
- El access token tiene expiración de 3600s (1 hora)
- El refresh token tiene expiración de 604800s (7 días)

### HU-A4: Cierre de sesión

**Como** usuario autenticado, **quiero** cerrar mi sesión, **para** invalidar mis tokens activos y proteger mi cuenta.

**Criterios de aceptación:**
- Requiere header `Authorization: Bearer <token>`
- Los tokens activos se invalidan al cerrar sesión
- Se devuelve 200 con mensaje de éxito
- Sin token válido se devuelve 401

---

## 3. Requisitos Funcionales y Reglas de Negocio

### RF-A1: Registro
- El email debe ser único en la plataforma
- Las contraseñas se almacenan hasheadas con bcrypt (salt rounds = 12)
- No se devuelve la contraseña ni el hash en ninguna respuesta

### RF-A2: Login
- Máximo 5 intentos de login por minuto por IP (rate limiting)
- Se compara el hash de la contraseña ingresada con el almacenado

### RF-A3: Token Management
- Access Token: duración 3600s, firmado con HS256
- Refresh Token: duración 604800s
- Token rotation: cada refresh genera nuevo par de tokens
- Secret del JWT se obtiene de `process.env.JWT_SECRET`

### RF-A4: JWT Payload
```json
{
  "sub": "UUID del usuario",
  "email": "email del usuario",
  "role": "rol activo (participant|organizer|speaker|admin)",
  "iat": "timestamp de emisión",
  "exp": "timestamp de expiración"
}
```

### RB-A1: Rol por defecto
Todo usuario nuevo recibe el rol `participant` al registrarse.

### RB-A2: Invalidación de tokens
Al hacer logout, los tokens se invalidan y no pueden usarse para refresh.

### RB-A3: CORS
Los orígenes permitidos se configuran por variable de entorno según el entorno (dev/prod).

---

## 4. Restricciones Técnicas del Módulo

- **Password Hashing:** bcrypt con salt rounds = 12, nunca almacenar passwords en texto plano
- **JWT:** algoritmo HS256, secret en variable de entorno, nunca hardcodear
- **Validación:** todos los inputs se validan con esquemas Zod antes de procesar
- **Rate Limiting:** máximo 5 intentos de login por minuto por IP
- **Máximo 200 líneas por archivo** (siguiendo estándares del proyecto)
- **Máximo 3 niveles de anidamiento** por función
- **Convención de Commits:** Conventional Commits (`feat:`, `fix:`, etc.)
- **Respuestas de error:** seguir formato definido en `Contracts.md` (códigos `AUTH_REQUIRED`, `INVALID_TOKEN`, `VALIDATION_ERROR`)
- **Headers estándar:** `Content-Type: application/json`, `Authorization: Bearer <token>`

---

## 5. Plan de Tareas

| # | Tarea | Tipo | Prioridad | Dependencias |
|---|-------|------|-----------|--------------|
| 1 | Configurar esquema Prisma para User (id, email, password, firstName, lastName, role) | Backend | Alta | - |
| 2 | Implementar esquemas Zod (loginSchema, registerSchema, refreshSchema) | Backend | Alta | - |
| 3 | Implementar servicio de hash de contraseñas con bcrypt | Backend | Alta | 1 |
| 4 | Implementar servicio de generación de JWT (access + refresh) | Backend | Alta | 1 |
| 5 | Implementar endpoint POST /api/v1/auth/register | Backend | Alta | 2, 3 |
| 6 | Implementar endpoint POST /api/v1/auth/login | Backend | Alta | 2, 3, 4 |
| 7 | Implementar endpoint POST /api/v1/auth/refresh | Backend | Alta | 4 |
| 8 | Implementar endpoint POST /api/v1/auth/logout | Backend | Alta | - |
| 9 | Implementar middleware de autenticación (verifyToken) | Backend | Alta | 4 |
| 10 | Implementar middleware de rate limiting para login | Backend | Media | 6 |
| 11 | Configurar CORS por entorno | Backend | Media | - |
| 12 | Crear tests unitarios para servicios de auth | Test | Alta | 3, 4 |
| 13 | Crear tests de integración para endpoints de auth | Test | Alta | 5-8 |
| 14 | Crear formulario de registro en frontend (React Hook Form + Zod) | Frontend | Alta | 5 |
| 15 | Crear formulario de login en frontend (React Hook Form + Zod) | Frontend | Alta | 6 |
| 16 | Implementar manejo de tokens en frontend (almacenamiento, refresh) | Frontend | Alta | 6, 7 |
| 17 | Implementar interceptor HTTP para incluir token en requests | Frontend | Alta | 9 |

---

## 6. Estrategia de Verificación

### Tests Unitarios
- Servicio de hashing: verificar que bcrypt genera hash y que la comparación funciona
- Servicio JWT: verificar generación de tokens con payload correcto y expiración
- Esquemas Zod: verificar que las validaciones rechazan datos inválidos y aceptan válidos

### Tests de Integración
- POST /register: crear usuario exitosamente, verificar email duplicado (409)
- POST /login: login exitoso con credenciales correctas, login fallido (401)
- POST /refresh: refresh válido genera nuevos tokens, refresh expirado retorna 401
- POST /logout: logout invalida tokens, logout sin token retorna 401

### Tests E2E
- Flujo completo: registro → login → access protegido → refresh → logout
- Verificar que tokens inválidos son rechazados en endpoints protegidos
- Verificar rate limiting con múltiples intentos de login

### Criterios de Calidad
- Cobertura de tests mínima: 80%
- Todos los endpoints devuelven respuestas en formato de `Contracts.md`
- Zero vulnerabilidades de seguridad (passwords no expuestos, tokens seguros)
- Linting y typecheck sin errores
