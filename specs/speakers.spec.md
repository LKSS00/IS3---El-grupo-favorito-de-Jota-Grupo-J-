# Speakers & Presentations Spec

## 1. Objetivo y Contexto

El módulo de speakers y presentaciones permite gestionar los expositores y sus presentaciones/trabajos en la plataforma **Academic Event Manager**. Los speakers pueden enviar abstracts de sus presentaciones para revisión, y los organizadores pueden aprobar o rechazarlos. El módulo también gestiona la información de perfil de los speakers (bio, foto, redes sociales) y su asignación a eventos y horarios específicos.

**Estados de presentación:** `pending`, `approved`, `rejected`, `changes_requested`.

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-SP1: Speaker envía abstract de presentación

**Como** speaker asignado a un evento, **quiero** enviar el abstract de mi presentación, **para** que el organizador lo revise y apruebe.

**Criterios de aceptación:**
- Solo usuarios con rol `speaker` en el evento pueden enviar abstracts
- Los campos requeridos son: `title`, `abstract`, `topic`, `durationMinutes`
- El abstract debe tener entre 200 y 2000 caracteres
- Se puede adjuntar un archivo PDF opcional (máximo 5MB)
- Una vez enviado, el estado inicial es `pending`
- Un speaker puede enviar múltiples abstracts para un mismo evento

### HU-SP2: Organizador revisa y aprueba/rechaza abstracts

**Como** organizador de un evento, **quiero** revisar los abstracts enviados, **para** decidir cuáles presentaciones formarán parte del evento.

**Criterios de aceptación:**
- Solo el organizador del evento (o admin) puede revisar abstracts
- El organizador puede ver todos los abstracts con su estado actual
- Al aprobar: cambia estado a `approved`, se asigna opcionalmente a un horario (Schedule)
- Al rechazar: cambia estado a `rejected`, debe incluir un motivo
- Al solicitar cambios: cambia estado a `changes_requested`, debe incluir comentarios específicos
- Se notifica al speaker automáticamente sobre el cambio de estado

### HU-SP3: Speaker gestiona su perfil de speaker

**Como** speaker, **quiero** completar y editar mi perfil profesional, **para** que los asistentes conozcan mi trayectoria.

**Criterios de aceptación:**
- El speaker puede editar su bio, foto de perfil, institución, cargo
- El speaker puede agregar enlaces a redes sociales (LinkedIn, ORCID, personal website)
- La foto debe ser una URL válida o subida al servidor (máximo 2MB)
- El perfil es visible públicamente en la página del evento
- Los campos se guardan en la tabla `SpeakerEvent` y opcionalmente en el perfil de User

### HU-SP4: Organizador gestiona speakers del evento

**Como** organizador, **quiero** asignar y gestionar los speakers de mi evento, **para** conformar la agenda académica.

**Criterios de aceptación:**
- El organizador puede invitar usuarios existentes como speakers del evento
- El organizador puede remover speakers del evento (solo si no tienen presentaciones aprobadas)
- Al invitar un speaker, se crea un registro en `SpeakerEvent` y se asigna el rol `speaker` al usuario
- El organizador puede ver el listado de speakers con sus presentaciones
- Si el usuario no existe, se puede enviar una invitación por email para registrarse

### HU-SP5: Participante ve agenda con speakers

**Como** participante, **quiero** ver la agenda del evento con los speakers y sus presentaciones, **para** decidir qué sesiones asistir.

**Criterios de aceptación:**
- La agenda muestra horarios (Schedule) con título, hora, speaker asignado
- Al hacer clic en un speaker, se ve su perfil y sus presentaciones aprobadas
- Las presentaciones con estado `approved` son las únicas visibles para participantes
- Se muestra título, abstract resumido, duración y biografía breve del speaker

---

## 3. Requisitos Funcionales y Reglas de Negocio

### RF-SP1: Envío de abstracts
- Endpoint: `POST /api/v1/events/:id/presentations`
- Campos: `title`, `abstract`, `topic`, `durationMinutes`, `coAuthors` (array opcional), `pdfUrl` (opcional)
- Validación de longitud de abstract: 200-2000 caracteres
- Validación de duración: entre 10 y 180 minutos
- El speakerId se toma del usuario autenticado con rol speaker en el evento

### RF-SP2: Listado de abstracts (organizador)
- Endpoint: `GET /api/v1/events/:id/presentations`
- Filtros: `status` (pending, approved, rejected, changes_requested), `speakerId`
- Soporte de paginación: `page` (default 1), `limit` (default 20, máx 100)
- Solo organizador del evento o admin puede ver todos los abstracts

### RF-SP3: Revisión de abstract
- Endpoint: `PATCH /api/v1/presentations/:id/review`
- Campos: `status` (approved/rejected/changes_requested), `reviewComments` (requerido si rejected o changes_requested), `scheduleId` (opcional, si approved)
- Solo organizador del evento o admin puede revisar
- Al cambiar estado, se envía notificación automática al speaker

### RF-SP4: Gestión de speakers en evento
- Endpoint: `POST /api/v1/events/:id/speakers` (invitar speaker)
- Endpoint: `DELETE /api/v1/events/:id/speakers/:speakerId` (remover speaker)
- Endpoint: `GET /api/v1/events/:id/speakers` (listar speakers del evento)
- Al invitar: crear registro en `SpeakerEvent`, asignar rol `speaker` al usuario

### RF-SP5: Perfil de speaker
- Endpoint: `PATCH /api/v1/speakers/me/profile` (editar propio perfil)
- Endpoint: `GET /api/v1/speakers/:id/profile` (ver perfil público)
- Campos: `bio`, `institution`, `position`, `photoUrl`, `linkedinUrl`, `orcidUrl`, `websiteUrl`

### RF-SP6: Agenda con speakers
- Endpoint: `GET /api/v1/events/:id/schedule` (ya existe en Schedule, extender para incluir speaker y presentation)
- La respuesta incluye datos del speaker y su presentación aprobada si existe

### RF-SP7: Modelo de datos
```typescript
interface Presentation {
  id: string;
  eventId: string;
  speakerId: string;
  title: string;
  abstract: string;
  topic: string;
  durationMinutes: number;
  coAuthors?: string[];
  pdfUrl?: string;
  status: "pending" | "approved" | "rejected" | "changes_requested";
  reviewComments?: string;
  scheduleId?: string;
  submittedAt: Date;
  reviewedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

interface SpeakerProfile {
  userId: string;
  bio?: string;
  institution?: string;
  position?: string;
  photoUrl?: string;
  linkedinUrl?: string;
  orcidUrl?: string;
  websiteUrl?: string;
}
```

### RB-SP1: Un abstract por revisión a la vez
Un speaker no puede tener más de un abstract en estado `pending` para el mismo evento simultáneamente.

### RB-SP2: No eliminar speakers con presentaciones aprobadas
Un speaker no puede ser removido de un evento si tiene presentaciones en estado `approved`, hasta que el evento finalice.

### RB-SP3: Abstract editable solo en estado pending
Un speaker solo puede editar su abstract si está en estado `pending` o `changes_requested`.

### RB-SP4: Integración con Schedule
Al aprobar un abstract, el organizador puede opcionalmente asignarlo a un horario existente (Schedule). Si no hay horario, se puede crear uno nuevo.

### RB-SP5: Notificaciones automáticas
Al cambiar el estado de un abstract, se envía notificación automática al speaker usando el módulo de Notifications.

---

## 4. Restricciones Técnicas del Módulo

- **Validación:** esquemas Zod para `Presentation`, `SpeakerProfile`, `ReviewPresentation`
- **Respuestas de error:** seguir formato de `Contracts.md` (`NOT_FOUND`, `INSUFFICIENT_PERMISSIONS`, `VALIDATION_ERROR`, `CONFLICT`)
- **Paginación:** seguir formato estándar con `meta` (page, limit, total, totalPages)
- **Máximo 200 líneas por archivo** (siguiendo estándares del proyecto)
- **Máximo 3 niveles de anidamiento** por función
- **Convención de Commits:** Conventional Commits (`feat:`, `fix:`, etc.)
- **Persistencia:** tabla `presentations` nueva en PostgreSQL, extender `SpeakerEvent` si es necesario
- **Archivos:** los PDFs se almacenan en un servicio de almacenamiento (local o S3) con URLs firmadas
- **Relaciones:** Presentation → Event (N:1), Presentation → User/speaker (N:1), Presentation → Schedule (N:1)

---

## 5. Plan de Tareas

| # | Tarea | Tipo | Prioridad | Dependencias |
|---|-------|------|-----------|--------------|
| 1 | Crear esquema Prisma para Presentation | Backend | Alta | - |
| 2 | Extender SpeakerEvent con campos de perfil si es necesario | Backend | Media | 1 |
| 3 | Implementar esquemas Zod para validación de presentations | Backend | Alta | - |
| 4 | Implementar esquemas Zod para perfil de speaker | Backend | Media | - |
| 5 | Implementar endpoint POST /api/v1/events/:id/presentations | Backend | Alta | 1, 3 |
| 6 | Verificar rol speaker en el evento antes de permitir envío | Backend | Alta | 5 |
| 7 | Implementar endpoint GET /api/v1/events/:id/presentations | Backend | Alta | 1 |
| 8 | Implementar paginación y filtros para listado | Backend | Alta | 7 |
| 9 | Implementar endpoint PATCH /api/v1/presentations/:id/review | Backend | Alta | 1, 3 |
| 10 | Implementar lógica de cambio de estado y notificaciones | Backend | Alta | 9 |
| 11 | Implementar endpoints de gestión de speakers en evento | Backend | Alta | 1 |
| 12 | Implementar asignación de rol speaker al invitar | Backend | Alta | 11 |
| 13 | Implementar endpoints de perfil de speaker | Backend | Media | 4 |
| 14 | Implementar upload de PDF para abstracts | Backend | Media | 5 |
| 15 | Extender endpoint GET /api/v1/events/:id/schedule con speaker y presentation | Backend | Media | 1, 9 |
| 16 | Proteger endpoints con middlewares de roles | Backend | Alta | 5-15 |
| 17 | Integrar notificaciones al cambiar estado de abstract | Backend | Alta | 10, Módulo Notifications |
| 18 | Crear tests unitarios para validaciones | Test | Alta | 3, 4 |
| 19 | Crear tests de integración para endpoints | Test | Alta | 5-16 |
| 20 | Crear tests para flujo de revisión de abstracts | Test | Alta | 9-10 |
| 21 | Crear formulario de envío de abstract en frontend | Frontend | Alta | 5 |
| 22 | Crear vista de revisión para organizador | Frontend | Alta | 9 |
| 23 | Crear vista de perfil de speaker | Frontend | Media | 13 |
| 24 | Crear vista de agenda con speakers y presentaciones | Frontend | Alta | 15 |

---

## 6. Estrategia de Verificación

### Tests Unitarios
- Esquemas Zod: verificar que aceptan presentations válidas y rechazan inválidas
- Validación de longitud de abstract: verificar límites 200-2000 caracteres
- Validación de duración: verificar límites 10-180 minutos
- Lógica de estados: verificar transiciones válidas e inválidas

### Tests de Integración
- POST /presentations: envío exitoso, no es speaker (403), abstract muy corto (400)
- GET /presentations: listado paginado correcto, filtros por status, acceso no autorizado (403)
- PATCH /presentations/:id/review: aprobar, rechazar, solicitar cambios, no autorizado (403)
- POST /events/:id/speakers: invitar speaker, ya es speaker (409), usuario no existe
- DELETE /events/:id/speakers/:speakerId: remover, tiene presentations aprobadas (409)

### Tests E2E
- Flujo completo: speaker envía abstract → organizador revisa → aprueba → se asigna a horario → participante ve en agenda
- Flujo de rechazo: speaker envía abstract → organizador rechaza con motivo → speaker recibe notificación → speaker edita y reenvía
- Verificar que un speaker de otro evento no puede enviar abstract a este evento

### Criterios de Calidad
- Cobertura de tests mínima: 80%
- Todas las respuestas siguen formato de `Contracts.md`
- Zero errores de validación no manejados
- Linting y typecheck sin errores
- Notificaciones se envían correctamente al cambiar estado de abstract
- PDFs se suben y almacenan correctamente
- La agenda muestra correctamente speakers y presentaciones aprobadas
