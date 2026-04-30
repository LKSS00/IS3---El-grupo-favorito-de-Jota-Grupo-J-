# Contracts - Academic Event Manager

Este documento define los contratos de la API, incluyendo formatos de respuesta, esquema de autenticación y tipos de datos comunes compartidos entre frontend y backend.

---

## Formato de Respuestas

### Respuesta Exitosa (200/201)

```json
{
  "success": true,
  "data": { ... },
  "message": "Operación realizada exitosamente",
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `success` | `boolean` | Sí | Siempre `true` en respuestas exitosas |
| `data` | `object \| array \| null` | Sí | Payload de la respuesta |
| `message` | `string` | No | Mensaje descriptivo opcional |
| `meta` | `object` | No | Metadatos de paginación (solo en listados) |

### Respuesta de Error (4xx/5xx)

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Los datos proporcionados no son válidos",
    "details": [
      {
        "field": "email",
        "message": "El formato de email no es válido"
      }
    ]
  }
}
```

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `success` | `boolean` | Sí | Siempre `false` en respuestas de error |
| `error.code` | `string` | Sí | Código de error machine-readable |
| `error.message` | `string` | Sí | Mensaje descriptivo human-readable |
| `error.details` | `array \| null` | No | Detalles adicionales (ej: campos con error) |

### Códigos de Error

| Código | HTTP | Descripción |
|--------|------|-------------|
| `AUTH_REQUIRED` | 401 | No se proporcionaron credenciales |
| `INVALID_TOKEN` | 401 | Token expirado o inválido |
| `INSUFFICIENT_PERMISSIONS` | 403 | El usuario no tiene permisos suficientes |
| `NOT_FOUND` | 404 | Recurso no encontrado |
| `VALIDATION_ERROR` | 400 | Datos de entrada inválidos |
| `CONFLICT` | 409 | Conflicto (ej: cupo agotado, inscripción duplicada) |
| `INTERNAL_SERVER_ERROR` | 500 | Error interno del servidor |

---

## Autenticación

### Esquema

**Bearer Token (JWT)**

```
Authorization: Bearer <token>
```

### Login

**POST** `/api/v1/auth/login`

**Request:**
```json
{
  "email": "usuario@ejemplo.com",
  "password": "contraseña_segura"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600,
    "tokenType": "Bearer",
    "user": {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "usuario@ejemplo.com",
      "firstName": "Juan",
      "lastName": "Pérez",
      "role": "participant"
    }
  }
}
```

### Refresh Token

**POST** `/api/v1/auth/refresh`

**Request:**
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
    "expiresIn": 3600,
    "tokenType": "Bearer"
  }
}
```

### Logout

**POST** `/api/v1/auth/logout`

**Headers:** `Authorization: Bearer <token>`

**Response (200):**
```json
{
  "success": true,
  "message": "Sesión cerrada exitosamente"
}
```

### Payload del JWT

```json
{
  "sub": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "usuario@ejemplo.com",
  "role": "organizer",
  "iat": 1714435200,
  "exp": 1714438800
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `sub` | `string (UUID)` | ID del usuario |
| `email` | `string` | Email del usuario |
| `role` | `string` | Rol activo del usuario |
| `iat` | `number` | Timestamp de emisión |
| `exp` | `number` | Timestamp de expiración |

---

## Tipos de Datos Comunes

### UUID

Todos los identificadores de recursos usan **UUID v4** como formato estándar.

```
Formato: xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
Ejemplo: a1b2c3d4-e5f6-4789-abcd-ef1234567890
```

### Fechas

Todas las fechas se transmiten en formato **ISO 8601** con zona horaria UTC.

```
Fecha: 2025-06-15
Fecha y hora: 2025-06-15T14:30:00.000Z
```

### Enums Comunes

**UserRole**

| Valor | Descripción |
|-------|-------------|
| `admin` | Administrador de la plataforma |
| `organizer` | Organizador de eventos |
| `speaker` | Disertante / Expositor |
| `participant` | Participante |

**EventType**

| Valor | Descripción |
|-------|-------------|
| `course` | Curso |
| `conference` | Congreso |
| `seminar` | Seminario |
| `workshop` | Taller |
| `symposium` | Simposio |
| `talk` | Charla |

**RegistrationStatus**

| Valor | Descripción |
|-------|-------------|
| `pending` | Inscripción pendiente |
| `confirmed` | Inscripción confirmada |
| `cancelled` | Inscripción cancelada |
| `waitlist` | En lista de espera |

**CertificateType**

| Valor | Descripción |
|-------|-------------|
| `attendance` | Certificado de asistencia |
| `approval` | Certificado de aprobación |
| `speaker` | Certificado de disertante |
| `author` | Certificado de autor |

### Tipos Reutilizables

```typescript
// Identificador universal
type UUID = string; // UUID v4

// Fecha ISO
type ISODate = string; // "2025-06-15T14:30:00.000Z"

// Paginación
type PaginationParams = {
  page: number;       // default: 1
  limit: number;      // default: 20, max: 100
};

// Respuesta base
type ApiResponse<T> = {
  success: boolean;
  data: T;
  message?: string;
  meta?: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
};

// Error base
type ApiError = {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Array<{ field: string; message: string }>;
  };
};
```

---

## Headers Estándar

| Header | Requerido | Descripción |
|--------|-----------|-------------|
| `Authorization` | Sí (endpoints protegidos) | `Bearer <token>` |
| `Content-Type` | Sí | `application/json` |
| `Accept` | No | `application/json` |
| `Accept-Language` | No | `es`, `en` (default: `es`) |

---

## Endpoints Base (Resumen)

| Método | Endpoint | Descripción | Autenticación |
|--------|----------|-------------|---------------|
| POST | `/api/v1/auth/login` | Iniciar sesión | No |
| POST | `/api/v1/auth/register` | Crear cuenta | No |
| POST | `/api/v1/auth/refresh` | Renovar token | No |
| POST | `/api/v1/auth/logout` | Cerrar sesión | Sí |
| GET | `/api/v1/events` | Listar eventos públicos | No |
| GET | `/api/v1/events/:id` | Detalle de evento | No |
| POST | `/api/v1/events` | Crear evento | Sí (organizer) |
| PATCH | `/api/v1/events/:id` | Actualizar evento | Sí (organizer) |
| DELETE | `/api/v1/events/:id` | Eliminar evento | Sí (organizer/admin) |
| POST | `/api/v1/events/:id/registrations` | Inscribirse a evento | Sí |
| GET | `/api/v1/events/:id/registrations` | Listar inscripciones | Sí (organizer) |
| POST | `/api/v1/events/:id/accredit` | Acreditar participante | Sí (organizer) |
| GET | `/api/v1/certificates` | Listar certificados del usuario | Sí |
| POST | `/api/v1/certificates/generate` | Generar certificado | Sí |
| POST | `/api/v1/events/:id/surveys` | Enviar encuesta | Sí |
| GET | `/api/v1/events/:id/reports` | Generar informe del evento | Sí (organizer) |
