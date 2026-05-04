# Notifications Spec

## 1. Objetivo y Contexto

El módulo de notificaciones permite enviar y gestionar notificaciones en la plataforma **Academic Event Manager**. Las notificaciones pueden ser in-app (guardadas en base de datos) y por email (usando un servicio SMTP). El sistema notifica automáticamente eventos importantes como confirmaciones de registro, promociones de lista de espera, disponibilidad de certificados y recordatorios de eventos.

**Canales soportados:** in-app, email.

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-N1: Usuario recibe notificación in-app

**Como** usuario de la plataforma, **quiero** recibir notificaciones dentro de la aplicación, **para** estar al tanto de eventos importantes sin depender del email.

**Criterios de aceptación:**
- Las notificaciones se guardan en base de datos con estado `read: false` por defecto
- El usuario puede ver todas sus notificaciones no leídas y leídas
- El usuario puede marcar una notificación como leída individualmente
- El usuario puede marcar todas las notificaciones como leídas
- Las notificaciones se muestran ordenadas por fecha de creación (más recientes primero)
- Se incluye un indicador de no leídas en la UI

### HU-N2: Usuario recibe notificación por email

**Como** usuario de la plataforma, **quiero** recibir notificaciones importantes por email, **para** no perderme información crítica cuando no estoy en la aplicación.

**Criterios de aceptación:**
- El email se envía de forma asíncrona (no bloquea la respuesta de la API)
- El usuario puede configurar qué notificaciones recibe por email
- Se usa una plantilla HTML profesional con logo y colores de la plataforma
- Se incluye un enlace directo a la acción relacionada en la plataforma
- Si el envío de email falla, se registra el error pero no se interrumpe el flujo

### HU-N3: Sistema envía notificaciones automáticas

**Como** organizador, **quiero** que el sistema envíe notificaciones automáticas, **para** mantener a los participantes informados sin intervención manual.

**Criterios de aceptación:**
- Al confirmar una inscripción → notificar al participante
- Al promover de waitlist → notificar al participante
- Al generar un certificado → notificar al usuario
- Al cancelar un evento → notificar a todos los inscritos
- 24 horas antes del evento → recordatorio a inscritos confirmados
- Las notificaciones automáticas se disparan desde los respectivos módulos

### HU-N4: Usuario gestiona sus preferencias de notificación

**Como** usuario, **quiero** configurar qué notificaciones recibo y por qué canal, **para** evitar spam y recibir solo lo que me interesa.

**Criterios de aceptación:**
- El usuario puede habilitar/deshabilitar notificaciones in-app por tipo
- El usuario puede habilitar/deshabilitar notificaciones por email por tipo
- Las preferencias se guardan por usuario
- Por defecto, todas las notificaciones están habilitadas

---

## 3. Requisitos Funcionales y Reglas de Negocio

### RF-N1: Envío de notificaciones
- Endpoint: `POST /api/v1/notifications/send`
- Soporta envío a un usuario específico o a múltiples usuarios (array de userIds)
- Soporta tipos: `registration_confirmed`, `waitlist_promoted`, `certificate_available`, `event_reminder`, `event_cancelled`, `survey_available`
- Campos requeridos: `userId`, `type`, `title`, `message`
- Campo opcional: `link` (enlace a la acción en la plataforma)

### RF-N2: Listado de notificaciones
- Endpoint: `GET /api/v1/notifications/me`
- Soporte de paginación: `page` (default 1), `limit` (default 20, máx 100)
- Filtros opcionales: `read` (true/false), `type`
- Formato de respuesta estándar con `meta` de paginación
- Incluye conteo de no leídas en la respuesta

### RF-N3: Marcar como leída
- Endpoint: `PATCH /api/v1/notifications/:id/read`
- Solo el dueño de la notificación puede marcarla como leída
- Endpoint para marcar todas: `POST /api/v1/notifications/read-all`

### RF-N4: Eliminar notificación
- Endpoint: `DELETE /api/v1/notifications/:id`
- Solo el dueño de la notificación puede eliminarla
- Eliminación física (no soft delete)

### RF-N5: Preferencias de notificación
- Endpoint: `GET /api/v1/notifications/preferences/me`
- Endpoint: `PATCH /api/v1/notifications/preferences/me`
- Estructura de preferencias por tipo y canal

### RF-N6: Modelo de datos
```typescript
interface Notification {
  id: string;
  userId: string;
  type: NotificationType;
  title: string;
  message: string;
  link?: string;
  read: boolean;
  readAt?: Date;
  channel: "in_app" | "email" | "both";
  createdAt: Date;
}

type NotificationType =
  | "registration_confirmed"
  | "waitlist_promoted"
  | "certificate_available"
  | "event_reminder"
  | "event_cancelled"
  | "survey_available"
  | "registration_cancelled"
  | "event_updated";
```

### RB-N1: No duplicados críticos
Para notificaciones de tipo `certificate_available` y `event_reminder`, verificar que no se haya enviado una notificación igual en las últimas 24 horas.

### RB-N2: Respetar preferencias
Antes de enviar una notificación, verificar las preferencias del usuario. Si tiene deshabilitado ese tipo para ese canal, no enviar.

### RB-N3: Notificaciones asíncronas
El envío de emails debe hacerse de forma asíncrona usando una cola (Redis/Bull) para no bloquear la respuesta de la API.

### RB-N4: Integración con otros módulos
Los módulos de Events, Registrations, Certificates y Surveys deben llamar al servicio de notificaciones al ocurrir eventos relevantes.

---

## 4. Restricciones Técnicas del Módulo

- **Validación:** esquemas Zod para `Notification`, `NotificationPreferences`
- **Respuestas de error:** seguir formato de `Contracts.md` (`NOT_FOUND`, `INSUFFICIENT_PERMISSIONS`, `VALIDATION_ERROR`)
- **Paginación:** seguir formato estándar con `meta` (page, limit, total, totalPages)
- **Máximo 200 líneas por archivo** (siguiendo estándares del proyecto)
- **Máximo 3 niveles de anidamiento** por función
- **Convención de Commits:** Conventional Commits (`feat:`, `fix:`, etc.)
- **Persistencia:** tabla `notifications` en PostgreSQL
- **Email:** usar nodemailer con configuración SMTP desde variables de entorno
- **Cola de emails:** Redis + Bull para procesamiento asíncrono
- **Relaciones:** Notification → User (N:1)

---

## 5. Plan de Tareas

| # | Tarea | Tipo | Prioridad | Dependencias |
|---|-------|------|-----------|--------------|
| 1 | Crear esquema Prisma para Notification y NotificationPreferences | Backend | Alta | - |
| 2 | Configurar nodemailer y variables de entorno SMTP | Backend | Alta | - |
| 3 | Configurar Redis y Bull para cola de emails | Backend | Alta | 2 |
| 4 | Implementar esquemas Zod para validación de notificaciones | Backend | Alta | - |
| 5 | Implementar endpoint POST /api/v1/notifications/send | Backend | Alta | 1, 4 |
| 6 | Implementar servicio de envío de emails asíncrono | Backend | Alta | 3 |
| 7 | Implementar endpoint GET /api/v1/notifications/me | Backend | Alta | 1 |
| 8 | Implementar paginación y filtros para listado | Backend | Alta | 7 |
| 9 | Implementar endpoint PATCH /api/v1/notifications/:id/read | Backend | Alta | 1 |
| 10 | Implementar endpoint POST /api/v1/notifications/read-all | Backend | Media | 9 |
| 11 | Implementar endpoint DELETE /api/v1/notifications/:id | Backend | Media | 1 |
| 12 | Implementar endpoints de preferencias GET y PATCH | Backend | Media | 1 |
| 13 | Integrar notificaciones en módulo de Registrations | Backend | Alta | 5 |
| 14 | Integrar notificaciones en módulo de Certificates | Backend | Alta | 5 |
| 15 | Integrar notificaciones en módulo de Events (recordatorios) | Backend | Media | 5 |
| 16 | Proteger endpoints con middlewares de autenticación | Backend | Alta | 5-15 |
| 17 | Crear tests unitarios para validaciones | Test | Alta | 4 |
| 18 | Crear tests de integración para endpoints | Test | Alta | 5-16 |
| 19 | Crear tests para servicio de emails | Test | Media | 6 |
| 20 | Crear vista de notificaciones en frontend | Frontend | Alta | 7 |
| 21 | Crear componente de campanita con badge de no leídas | Frontend | Alta | 20 |
| 22 | Crear vista de preferencias de notificación | Frontend | Media | 12 |

---

## 6. Estrategia de Verificación

### Tests Unitarios
- Esquemas Zod: verificar que aceptan notificaciones válidas y rechazan inválidas
- Servicio de notificaciones: verificar que respeta preferencias del usuario
- Filtro de duplicados: verificar que no envía notificaciones duplicadas en ventana de 24h

### Tests de Integración
- POST /notifications/send: envío exitoso, usuario no existe (404), datos inválidos (400)
- GET /notifications/me: listado paginado correcto, filtros por read y type
- PATCH /notifications/:id/read: marcar como leída, no autorizado (403), no existe (404)
- POST /notifications/read-all: marcar todas como leídas correctamente
- DELETE /notifications/:id: eliminar, no autorizado (403)

### Tests E2E
- Flujo completo: inscripción confirmada → notificación in-app creada → usuario la ve → la marca como leída
- Flujo de email: evento cancelado → notificación por email enviada a todos los inscritos
- Verificar que un usuario no puede marcar como leída la notificación de otro usuario

### Criterios de Calidad
- Cobertura de tests mínima: 80%
- Todas las respuestas siguen formato de `Contracts.md`
- Zero errores de validación no manejados
- Linting y typecheck sin errores
- Emails se envían asíncronamente sin bloquear la API
- Preferencias de usuario se respetan correctamente
