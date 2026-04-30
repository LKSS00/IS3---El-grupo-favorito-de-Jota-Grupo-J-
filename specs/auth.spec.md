# Auth Spec

## Endpoints

| Método | Endpoint | Descripción | Autenticación |
|--------|----------|-------------|---------------|
| POST | `/api/v1/auth/login` | Iniciar sesión | No |
| POST | `/api/v1/auth/register` | Crear cuenta | No |
| POST | `/api/v1/auth/refresh` | Renovar token | No |
| POST | `/api/v1/auth/logout` | Cerrar sesión | Sí |

---

## POST /api/v1/auth/login

### Request Body

```json
{
  "email": "usuario@ejemplo.com",
  "password": "contraseña_segura"
}
```

### Validación (Zod)

```typescript
const loginSchema = z.object({
  email: z.string().email("Formato de email inválido"),
  password: z.string().min(8, "La contraseña debe tener al menos 8 caracteres"),
});
```

### Response (200)

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

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `VALIDATION_ERROR` | 400 | Email o contraseña inválidos |
| `AUTH_REQUIRED` | 401 | Credenciales incorrectas |
| `INTERNAL_SERVER_ERROR` | 500 | Error del servidor |

---

## POST /api/v1/auth/register

### Request Body

```json
{
  "email": "usuario@ejemplo.com",
  "password": "contraseña_segura",
  "confirmPassword": "contraseña_segura",
  "firstName": "Juan",
  "lastName": "Pérez"
}
```

### Validación (Zod)

```typescript
const registerSchema = z
  .object({
    email: z.string().email("Formato de email inválido"),
    password: z
      .string()
      .min(8, "La contraseña debe tener al menos 8 caracteres")
      .regex(/[A-Z]/, "Debe contener al menos una mayúscula")
      .regex(/[0-9]/, "Debe contener al menos un número")
      .regex(/[^a-zA-Z0-9]/, "Debe contener al menos un carácter especial"),
    confirmPassword: z.string(),
    firstName: z.string().min(2, "El nombre debe tener al menos 2 caracteres"),
    lastName: z.string().min(2, "El apellido debe tener al menos 2 caracteres"),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Las contraseñas no coinciden",
    path: ["confirmPassword"],
  });
```

### Response (201)

```json
{
  "success": true,
  "data": {
    "user": {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "email": "usuario@ejemplo.com",
      "firstName": "Juan",
      "lastName": "Pérez",
      "role": "participant"
    }
  },
  "message": "Usuario creado exitosamente"
}
```

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `VALIDATION_ERROR` | 400 | Datos inválidos o contraseñas no coinciden |
| `CONFLICT` | 409 | Email ya registrado |

---

## POST /api/v1/auth/refresh

### Request Body

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

### Validación (Zod)

```typescript
const refreshSchema = z.object({
  refreshToken: z.string().min(1, "Refresh token requerido"),
});
```

### Response (200)

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

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `VALIDATION_ERROR` | 400 | Token no proporcionado |
| `INVALID_TOKEN` | 401 | Token expirado o inválido |

---

## POST /api/v1/auth/logout

### Headers

```
Authorization: Bearer <token>
```

### Response (200)

```json
{
  "success": true,
  "message": "Sesión cerrada exitosamente"
}
```

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `AUTH_REQUIRED` | 401 | No se proporcionó token |
| `INVALID_TOKEN` | 401 | Token inválido |

---

## JWT Payload

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

### Configuración de Tokens

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| Access Token Expiration | 3600s (1 hora) | Duración del access token |
| Refresh Token Expiration | 604800s (7 días) | Duración del refresh token |
| Algorithm | HS256 | Algoritmo de firma |
| Secret | `process.env.JWT_SECRET` | Variable de entorno |

---

## Flujo de Autenticación

```
1. Usuario se registra → Rol por defecto: participant
2. Login exitoso → Access Token + Refresh Token
3. Access Token usado en cada request protegida
4. Access Token expirado → Usar Refresh Token para obtener nuevo par
5. Logout → Invalidar tokens activos
6. Token rotation: cada refresh genera nuevo par de tokens
```

---

## Seguridad

- **Password Hashing:** bcrypt con salt rounds = 12
- **Token Rotation:** Cada uso de refresh token genera nuevo access + refresh token
- **Invalidación en Logout:** Los tokens se invalidan al cerrar sesión
- **Rate Limiting:** Máximo 5 intentos de login por minuto por IP
- **CORS:** Orígenes permitidos configurados por entorno
