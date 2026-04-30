# Roles Spec

## Enum UserRole

| Valor | Descripción |
|-------|-------------|
| `admin` | Administrador de la plataforma |
| `organizer` | Organizador de eventos |
| `speaker` | Disertante / Expositor |
| `participant` | Participante |

---

## Permisos por Rol

### Admin

| Recurso | Crear | Leer | Actualizar | Eliminar |
|---------|-------|------|------------|----------|
| Usuarios | Sí | Sí | Sí | Sí |
| Eventos | Sí | Sí | Sí | Sí |
| Inscripciones | Sí | Sí | Sí | Sí |
| Encuestas | No | Sí | No | No |
| Certificados | No | Sí | No | No |
| Roles | Sí | Sí | Sí | Sí |

- Acceso total a la plataforma
- Gestión de usuarios y asignación de roles
- Puede eliminar cualquier evento
- Acceso a métricas globales

### Organizer

| Recurso | Crear | Leer | Actualizar | Eliminar |
|---------|-------|------|------------|----------|
| Eventos propios | Sí | Sí | Sí | Sí |
| Eventos de otros | No | Sí (públicos) | No | No |
| Inscripciones (sus eventos) | Sí | Sí | Sí | No |
| Acreditación (sus eventos) | Sí | Sí | Sí | No |
| Encuestas (sus eventos) | No | Sí | No | No |
| Certificados (sus eventos) | Sí | Sí | No | No |
| Informes (sus eventos) | Sí | Sí | No | No |

- CRUD completo de sus propios eventos
- Acreditar participantes en sus eventos
- Ver inscripciones y estadísticas de sus eventos
- Generar certificados e informes

### Speaker

| Recurso | Crear | Leer | Actualizar | Eliminar |
|---------|-------|------|------------|----------|
| Eventos asignados | No | Sí | No | No |
| Perfil | No | Sí | Sí | No |
| Certificado de disertante | No | Sí | No | No |

- Ver eventos donde fue asignado como disertante
- Gestionar su perfil
- Descargar certificado de disertante

### Participant

| Recurso | Crear | Leer | Actualizar | Eliminar |
|---------|-------|------|------------|----------|
| Eventos | No | Sí (públicos) | No | No |
| Inscripciones propias | Sí | Sí | No | Sí (cancelar) |
| Certificados propios | No | Sí | No | No |
| Encuestas | Sí | No | No | No |

- Inscribirse a eventos públicos
- Cancelar su propia inscripción
- Descargar sus certificados
- Enviar encuestas post-evento

---

## Middleware de Roles

### requireRole(role)

Verifica que el usuario tenga exactamente el rol especificado.

```typescript
function requireRole(role: UserRole) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user.role;
    if (userRole !== role) {
      return res.status(403).json({
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "No tiene permisos para realizar esta acción",
        },
      });
    }
    next();
  };
}
```

**Uso:**

```typescript
app.post("/api/v1/events", requireRole("organizer"), createEventHandler);
```

### requireAnyRole(roles[])

Verifica que el usuario tenga al menos uno de los roles especificados.

```typescript
function requireAnyRole(roles: UserRole[]) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user.role;
    if (!roles.includes(userRole)) {
      return res.status(403).json({
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "No tiene permisos para realizar esta acción",
        },
      });
    }
    next();
  };
}
```

**Uso:**

```typescript
app.delete("/api/v1/events/:id", requireAnyRole(["organizer", "admin"]), deleteEventHandler);
```

### requireOwnerOrAdmin(resourceOwnerId)

Verifica que el usuario sea el dueño del recurso o admin.

```typescript
function requireOwnerOrAdmin(ownerIdField: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.user.sub;
    const userRole = req.user.role;

    if (userRole === "admin") return next();

    const resourceOwnerId = req[ownerIdField];
    if (userId !== resourceOwnerId) {
      return res.status(403).json({
        success: false,
        error: {
          code: "INSUFFICIENT_PERMISSIONS",
          message: "No tiene permisos para realizar esta acción",
        },
      });
    }
    next();
  };
}
```

---

## Múltiples Roles por Usuario

Un usuario puede tener **múltiples roles** asociados, pero el JWT contiene un **rol activo**.

### Modelo User-Role

```typescript
interface UserRoleAssignment {
  id: string;           // UUID
  userId: string;       // UUID - FK a User
  role: UserRole;
  assignedBy: string;   // UUID - FK a User (admin)
  assignedAt: Date;
  isActive: boolean;
}
```

### Cambio de Rol Activo

El rol activo se determina en el momento del login y se incluye en el JWT. Para cambiar de rol activo:

```
POST /api/v1/users/me/role
{
  "role": "organizer"
}
```

Solo roles previamente asignados al usuario pueden ser activados.

---

## Jerarquía de Permisos

```
admin > organizer > speaker > participant
```

- `admin` tiene implícitamente todos los permisos de roles inferiores
- `organizer` tiene permisos de `participant` + gestión de eventos
- `speaker` tiene permisos de `participant` + acceso a contenido de disertante

### Función de Comparación

```typescript
function hasPermission(userRole: UserRole, requiredRole: UserRole): boolean {
  const hierarchy: Record<UserRole, number> = {
    admin: 4,
    organizer: 3,
    speaker: 2,
    participant: 1,
  };
  return hierarchy[userRole] >= hierarchy[requiredRole];
}
```

---

## Casos de Error

| Código | HTTP | Escenario |
|--------|------|-----------|
| `INSUFFICIENT_PERMISSIONS` | 403 | Rol no tiene permisos para la acción |
| `INVALID_TOKEN` | 401 | Token inválido o expirado |
| `AUTH_REQUIRED` | 401 | No autenticado |
| `NOT_FOUND` | 404 | Recurso no encontrado (oculta información si no tiene permisos) |

---

## Protección de Rutas por Módulo

| Módulo | Endpoint | Roles Permitidos |
|--------|----------|-----------------|
| Auth | `/api/v1/auth/*` | Todos (públicos excepto logout) |
| Events | `GET /api/v1/events` | Todos |
| Events | `POST /api/v1/events` | organizer, admin |
| Events | `PATCH /api/v1/events/:id` | organizer (propio), admin |
| Events | `DELETE /api/v1/events/:id` | organizer (propio), admin |
| Registrations | `POST /api/v1/events/:id/registrations` | participant, admin |
| Registrations | `GET /api/v1/events/:id/registrations` | organizer (propio), admin |
| Accreditation | `POST /api/v1/events/:id/accredit` | organizer (propio), admin |
| Certificates | `GET /api/v1/certificates` | Todos (propios) |
| Certificates | `POST /api/v1/certificates/generate` | organizer (propios), admin |
| Surveys | `POST /api/v1/events/:id/surveys` | participant |
| Surveys | `GET /api/v1/events/:id/surveys` | organizer (propio), admin |
| Reports | `GET /api/v1/events/:id/reports` | organizer (propio), admin |
