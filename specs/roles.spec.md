# Roles Spec

## 1. Objetivo y Contexto

El módulo de roles gestiona el control de acceso basado en roles (RBAC) para la plataforma **Academic Event Manager**. Define los permisos de cada tipo de usuario sobre los recursos del sistema y proporciona middleware para proteger rutas. Los roles determinan qué acciones puede realizar un usuario en cada módulo de la plataforma.

**Roles disponibles:** `admin`, `organizer`, `speaker`, `participant`.

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-R1: Admin gestiona usuarios y roles

**Como** administrador de la plataforma, **quiero** asignar y revocar roles a los usuarios, **para** controlar su nivel de acceso.

**Criterios de aceptación:**
- El admin puede crear, leer, actualizar y eliminar cualquier usuario
- El admin puede asignar roles adicionales a cualquier usuario
- El admin puede eliminar cualquier evento de la plataforma
- El admin tiene acceso a métricas globales de todos los eventos

### HU-R2: Organizador gestiona sus eventos

**Como** organizador, **quiero** crear y administrar mis propios eventos, **para** gestionar inscripciones, acreditar participantes y generar informes.

**Criterios de aceptación:**
- Puede crear, editar y eliminar sus propios eventos
- Puede ver inscripciones y acreditar participantes de sus eventos
- Puede ver encuestas y generar informes de sus eventos
- No puede modificar eventos de otros organizadores
- Puede generar certificados para asistentes de sus eventos

### HU-R3: Disertante accede a sus eventos asignados

**Como** disertante, **quiero** ver los eventos donde fui asignado, **para** consultar detalles y descargar mi certificado.

**Criterios de aceptación:**
- Puede ver solo los eventos donde fue asignado como speaker
- Puede gestionar su perfil
- Puede descargar su certificado de disertante
- No puede modificar eventos ni inscripciones

### HU-R4: Participante se inscribe y gestiona su experiencia

**Como** participante, **quiero** inscribirme a eventos, ver mis certificados y enviar encuestas, **para** participar activamente en la plataforma.

**Criterios de aceptación:**
- Puede inscribirse a eventos públicos
- Puede cancelar su propia inscripción
- Puede ver y descargar sus certificados
- Puede enviar encuestas post-evento (una por evento)
- Solo puede ver sus propios datos, no los de otros usuarios

### HU-R5: Usuario con múltiples roles cambia de contexto

**Como** usuario con múltiples roles, **quiero** cambiar mi rol activo, **para** acceder a las funcionalidades correspondientes a cada rol.

**Criterios de aceptación:**
- Puede tener múltiples roles asignados simultáneamente
- El JWT contiene un único rol activo
- Puede cambiar su rol activo mediante `POST /api/v1/users/me/role`
- Solo puede activar roles que le fueron asignados previamente

---

## 3. Requisitos Funcionales y Reglas de Negocio

### RF-R1: Jerarquía de permisos
```
admin (4) > organizer (3) > speaker (2) > participant (1)
```
- `admin` tiene implícitamente todos los permisos de roles inferiores
- La comparación se realiza mediante función `hasPermission(userRole, requiredRole)`

### RF-R2: Middlewares de protección
- `requireRole(role)`: verifica rol exacto
- `requireAnyRole(roles[])`: verifica al menos uno de los roles
- `requireOwnerOrAdmin(ownerIdField)`: verifica propiedad o rol admin

### RF-R3: Tabla de permisos por módulo

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

### RF-R4: Modelo de asignación de roles
```typescript
interface UserRoleAssignment {
  id: string;
  userId: string;
  role: UserRole;
  assignedBy: string;
  assignedAt: Date;
  isActive: boolean;
}
```

### RB-R1: Un rol no asignado no puede activarse
Solo roles previamente asignados al usuario pueden ser activados.

### RB-R2: El rol activo se incluye en el JWT
El payload del JWT contiene el campo `role` con el rol activo del usuario.

### RB-R3: Admin siempre tiene acceso
El rol `admin` bypass todas las verificaciones de propiedad en middlewares.

---

## 4. Restricciones Técnicas del Módulo

- **Enum UserRole:** definido como tipo TypeScript `admin | organizer | speaker | participant`
- **Middlewares:** implementar como funciones de orden superior que retornan middleware de Express/Hono
- **Respuestas de error:** código `INSUFFICIENT_PERMISSIONS` (403) para accesos denegados
- **JWT:** el rol se extrae del token decodificado en `req.user.role`
- **Máximo 200 líneas por archivo** (siguiendo estándares del proyecto)
- **Máximo 3 niveles de anidamiento** por función
- **Convención de Commits:** Conventional Commits (`feat:`, `fix:`, etc.)
- **Persistencia:** las asignaciones de roles se almacenan en tabla intermedia User-Role
- **Sin hardcodeo de roles:** los permisos se definen en configuración, no en lógica dispersa

---

## 5. Plan de Tareas

| # | Tarea | Tipo | Prioridad | Dependencias |
|---|-------|------|-----------|--------------|
| 1 | Definir enum UserRole en tipos compartidos | Backend | Alta | - |
| 2 | Crear esquema Prisma para UserRoleAssignment | Backend | Alta | 1 |
| 3 | Implementar middleware requireRole | Backend | Alta | 1 |
| 4 | Implementar middleware requireAnyRole | Backend | Alta | 1 |
| 5 | Implementar middleware requireOwnerOrAdmin | Backend | Alta | 1 |
| 6 | Implementar función hasPermission (jerarquía) | Backend | Alta | 1 |
| 7 | Implementar endpoint POST /api/v1/users/me/role | Backend | Alta | 2 |
| 8 | Implementar endpoint de asignación de roles (admin) | Backend | Alta | 2 |
| 9 | Proteger rutas de eventos con middlewares de roles | Backend | Alta | 3-5 |
| 10 | Proteger rutas de inscripciones con middlewares | Backend | Alta | 3-5 |
| 11 | Proteger rutas de certificados con middlewares | Backend | Alta | 3-5 |
| 12 | Proteger rutas de encuestas con middlewares | Backend | Alta | 3-5 |
| 13 | Proteger rutas de informes con middlewares | Backend | Alta | 3-5 |
| 14 | Crear tests unitarios para middlewares | Test | Alta | 3-6 |
| 15 | Crear tests de integración para rutas protegidas | Test | Alta | 9-13 |
| 16 | Implementar selector de rol en frontend (dashboard) | Frontend | Media | 7 |
| 17 | Ocultar UI según rol del usuario | Frontend | Media | - |

---

## 6. Estrategia de Verificación

### Tests Unitarios
- `requireRole`: verifica que solo el rol especificado pasa, otros reciben 403
- `requireAnyRole`: verifica que al menos uno de los roles pasa
- `requireOwnerOrAdmin`: verifica que el dueño o admin pasa, otros reciben 403
- `hasPermission`: verifica jerarquía (admin > organizer > speaker > participant)

### Tests de Integración
- Un organizer puede crear eventos pero no eliminar eventos de otros
- Un participant puede inscribirse pero no ver inscripciones de otros
- Un admin puede acceder a todos los endpoints
- Un speaker solo ve eventos asignados
- Cambiar rol activo refleja el nuevo rol en el JWT

### Tests E2E
- Flujo: admin asigna rol organizer → usuario crea evento → participante se inscribe → organizador acredita
- Verificar que cada rol ve solo lo que le corresponde en la UI
- Verificar que los middlewares bloquean correctamente accesos no autorizados

### Criterios de Calidad
- Cobertura de tests mínima: 80%
- Todos los endpoints protegidos devuelven `INSUFFICIENT_PERMISSIONS` (403) correctamente
- Zero accesos indebidos detectados en pruebas de penetración
- Linting y typecheck sin errores
- Middlewares documentados con ejemplos de uso
