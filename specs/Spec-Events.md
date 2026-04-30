# Spec-Events - Academic Event Manager

Este documento define las especificaciones funcionales y técnicas del módulo de gestión de eventos, incluyendo CRUD, filtros, manejo de cupos y fechas límite.

---

## Descripción del Módulo

El módulo de eventos permite a los organizadores crear, editar, publicar y gestionar eventos académicos. Los eventos son visibles públicamente (una vez publicados) y pueden filtrarse por tipo, fecha y estado.

---

## Reglas de Negocio

### Reglas Generales

| Regla | Descripción |
|-------|-------------|
| **Publicación** | Un evento solo es visible públicamente cuando `isPublished = true` |
| **Cupo** | Si `maxCapacity` está definido, las inscripciones no pueden superarlo |
| **Cupo mínimo** | Si `minCapacity` está definido, el evento requiere ese mínimo para realizarse |
| **Fechas de inscripción** | Si `registrationStart` y `registrationEnd` están definidos, solo se permite inscribirse dentro de ese rango |
| **Coherencia de fechas** | `startDate` debe ser anterior a `endDate`; `registrationEnd` no puede ser posterior a `startDate` |
| **Slug único** | El `slug` se genera automáticamente desde el título y debe ser único |
| **Tipo de evento** | Obligatorio; define la categoría del evento (course, conference, talk, etc.) |
| **Ubicación** | Si `isOnline = true`, se puede omitir `location` pero se recomienda `onlineUrl` |

### Estados del Evento

| Estado | Condición | Descripción |
|--------|-----------|-------------|
| **Borrador** | `isPublished = false` | El evento no es visible públicamente |
| **Publicado** | `isPublished = true` y `startDate > ahora` | Visible, acepta inscripciones si está en el rango |
| **En curso** | `isPublished = true` y `startDate <= ahora <= endDate` | Evento activo |
| **Finalizado** | `isPublished = true` y `endDate < ahora` | Evento pasado |
| **Cancelado** | `isPublished = false` después de publicado | Evento cancelado por el organizador |

---

## Endpoints

### Listar Eventos Públicos

**GET** `/api/v1/events`

**Query Params:**

| Parámetro | Tipo | Requerido | Default | Descripción |
|-----------|------|-----------|---------|-------------|
| `type` | `EventType` | No | - | Filtrar por tipo |
| `status` | `string` | No | `all` | `upcoming`, `ongoing`, `past` |
| `page` | `number` | No | 1 | Página |
| `limit` | `number` | No | 20 | Items por página (max: 100) |
| `search` | `string` | No | - | Búsqueda por título/descripción |

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "title": "Congreso de Tecnología 2025",
      "slug": "congreso-tecnologia-2025",
      "type": "conference",
      "description": "Evento anual sobre las últimas tendencias en tecnología.",
      "banner": "https://cdn.example.com/banners/congreso-2025.jpg",
      "location": "Universidad Nacional",
      "isOnline": false,
      "startDate": "2025-08-15T09:00:00.000Z",
      "endDate": "2025-08-17T18:00:00.000Z",
      "registrationStart": "2025-06-01T00:00:00.000Z",
      "registrationEnd": "2025-08-10T23:59:59.000Z",
      "minCapacity": 50,
      "maxCapacity": 300,
      "currentRegistrations": 142,
      "isPublished": true,
      "status": "upcoming",
      "organizer": {
        "id": "org-id-uuid",
        "firstName": "María",
        "lastName": "García"
      },
      "createdAt": "2025-05-01T10:00:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

---

### Obtener Detalle de Evento

**GET** `/api/v1/events/:id`

**Path Params:**

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `id` | `UUID` | ID del evento |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "Congreso de Tecnología 2025",
    "slug": "congreso-tecnologia-2025",
    "type": "conference",
    "description": "Evento anual sobre las últimas tendencias en tecnología.",
    "banner": "https://cdn.example.com/banners/congreso-2025.jpg",
    "location": "Universidad Nacional",
    "isOnline": false,
    "onlineUrl": null,
    "startDate": "2025-08-15T09:00:00.000Z",
    "endDate": "2025-08-17T18:00:00.000Z",
    "registrationStart": "2025-06-01T00:00:00.000Z",
    "registrationEnd": "2025-08-10T23:59:59.000Z",
    "minCapacity": 50,
    "maxCapacity": 300,
    "currentRegistrations": 142,
    "isPublished": true,
    "status": "upcoming",
    "organizer": {
      "id": "org-id-uuid",
      "firstName": "María",
      "lastName": "García"
    },
    "speakers": [
      {
        "id": "speaker-id-uuid",
        "firstName": "Carlos",
        "lastName": "López",
        "bio": "Investigador en IA con 15 años de experiencia."
      }
    ],
    "schedule": [
      {
        "id": "schedule-id-uuid",
        "title": "Apertura del Congreso",
        "description": "Presentación inaugural.",
        "startTime": "2025-08-15T09:00:00.000Z",
        "endTime": "2025-08-15T10:00:00.000Z",
        "location": "Auditorio Principal",
        "speaker": {
          "firstName": "María",
          "lastName": "García"
        }
      }
    ],
    "createdAt": "2025-05-01T10:00:00.000Z",
    "updatedAt": "2025-05-15T14:30:00.000Z"
  }
}
```

**Error (404):**
```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "El evento solicitado no existe o no está publicado"
  }
}
```

---

### Crear Evento

**POST** `/api/v1/events`

**Autenticación:** Requerida (rol `organizer` o `admin`)

**Request:**
```json
{
  "title": "Congreso de Tecnología 2025",
  "description": "Evento anual sobre las últimas tendencias en tecnología.",
  "type": "conference",
  "location": "Universidad Nacional",
  "isOnline": false,
  "onlineUrl": null,
  "minCapacity": 50,
  "maxCapacity": 300,
  "registrationStart": "2025-06-01T00:00:00.000Z",
  "registrationEnd": "2025-08-10T23:59:59.000Z",
  "startDate": "2025-08-15T09:00:00.000Z",
  "endDate": "2025-08-17T18:00:00.000Z",
  "isPublished": false
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "Congreso de Tecnología 2025",
    "slug": "congreso-tecnologia-2025",
    "type": "conference",
    "isPublished": false,
    "createdAt": "2025-05-01T10:00:00.000Z"
  },
  "message": "Evento creado exitosamente como borrador"
}
```

**Errores:**

| Código | HTTP | Condición |
|--------|------|-----------|
| `AUTH_REQUIRED` | 401 | No autenticado |
| `INSUFFICIENT_PERMISSIONS` | 403 | No tiene rol organizer/admin |
| `VALIDATION_ERROR` | 400 | Datos inválidos |
| `CONFLICT` | 409 | Slug duplicado |

---

### Actualizar Evento

**PATCH** `/api/v1/events/:id`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Request:**
```json
{
  "title": "Congreso de Tecnología 2025 - Actualizado",
  "maxCapacity": 400,
  "registrationEnd": "2025-08-12T23:59:59.000Z"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "title": "Congreso de Tecnología 2025 - Actualizado",
    "slug": "congreso-tecnologia-2025-actualizado",
    "maxCapacity": 400,
    "updatedAt": "2025-05-15T14:30:00.000Z"
  },
  "message": "Evento actualizado exitosamente"
}
```

**Errores:**

| Código | HTTP | Condición |
|--------|------|-----------|
| `NOT_FOUND` | 404 | Evento no existe |
| `INSUFFICIENT_PERMISSIONS` | 403 | No es el organizador |
| `CONFLICT` | 409 | No se puede modificar evento ya iniciado |

---

### Publicar Evento

**POST** `/api/v1/events/:id/publish`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Validaciones Pre-Publicación:**

| Validación | Descripción |
|------------|-------------|
| Título y descripción obligatorios | El evento debe tener contenido mínimo |
| Fechas definidas | `startDate` y `endDate` deben estar definidos |
| Coherencia de fechas | `startDate < endDate` |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "isPublished": true,
    "slug": "congreso-tecnologia-2025",
    "publishedAt": "2025-05-20T08:00:00.000Z"
  },
  "message": "Evento publicado exitosamente"
}
```

---

### Despublicar Evento

**POST** `/api/v1/events/:id/unpublish`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Validaciones:**

| Validación | Descripción |
|------------|-------------|
| No iniciado | No se puede despublicar si `startDate <= ahora` |
| Sin inscripciones confirmadas | Si tiene inscripciones, se debe notificar antes |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "isPublished": false
  },
  "message": "Evento despublicado exitosamente"
}
```

---

### Eliminar Evento

**DELETE** `/api/v1/events/:id`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Validaciones:**

| Validación | Descripción |
|------------|-------------|
| Sin inscripciones | No se puede eliminar si tiene inscripciones confirmadas |
| No iniciado | No se puede eliminar si ya comenzó |

**Response (200):**
```json
{
  "success": true,
  "message": "Evento eliminado exitosamente"
}
```

---

### Gestionar Agenda (Schedule)

#### Crear Item de Agenda

**POST** `/api/v1/events/:id/schedule`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Request:**
```json
{
  "title": "Keynote: El futuro de la IA",
  "description": "Charla magistral sobre tendencias en inteligencia artificial.",
  "speakerId": "speaker-id-uuid",
  "startTime": "2025-08-15T09:00:00.000Z",
  "endTime": "2025-08-15T10:30:00.000Z",
  "location": "Auditorio Principal"
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "schedule-id-uuid",
    "title": "Keynote: El futuro de la IA",
    "startTime": "2025-08-15T09:00:00.000Z",
    "endTime": "2025-08-15T10:30:00.000Z"
  },
  "message": "Item de agenda creado exitosamente"
}
```

#### Listar Agenda

**GET** `/api/v1/events/:id/schedule`

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "schedule-id-uuid",
      "title": "Keynote: El futuro de la IA",
      "description": "Charla magistral sobre tendencias en inteligencia artificial.",
      "startTime": "2025-08-15T09:00:00.000Z",
      "endTime": "2025-08-15T10:30:00.000Z",
      "location": "Auditorio Principal",
      "speaker": {
        "firstName": "Carlos",
        "lastName": "López"
      }
    }
  ]
}
```

---

## Validaciones

### Creación/Actualización

| Campo | Regla |
|-------|-------|
| `title` | Requerido, 3-200 caracteres |
| `description` | Requerido, mínimo 20 caracteres |
| `type` | Requerido, valor válido del enum `EventType` |
| `startDate` | Requerido, debe ser futuro al crear |
| `endDate` | Requerido, debe ser >= `startDate` |
| `registrationStart` | Opcional, si se define debe ser < `registrationEnd` |
| `registrationEnd` | Opcional, si se define debe ser <= `startDate` |
| `minCapacity` | Opcional, entero positivo, <= `maxCapacity` |
| `maxCapacity` | Opcional, entero positivo, >= `minCapacity` |
| `location` | Requerido si `isOnline = false` |
| `onlineUrl` | Requerido si `isOnline = true` |

---

## Flujo de Estados

```
                    ┌───────────┐
                    │  Borrador  │
                    └─────┬─────┘
                          │ publish()
                          ▼
                    ┌─────────────┐
          ┌────────<│  Publicado   │─────────┐
          │         └──────┬──────┘         │
          │                │                │
    unpublish()      startDate <= ahora   cancel()
          │                │                │
          │                ▼                ▼
          │         ┌────────────┐    ┌───────────┐
          │         │ En curso   │    │ Cancelado │
          │         └──────┬─────┘    └───────────┘
          │                │
          │          endDate < ahora
          │                │
          │                ▼
          │         ┌────────────┐
          └─────────│ Finalizado │
                    └────────────┘
```

---

## Casos de Uso

### Caso 1: Crear evento sin cupo ni fechas límite

```json
{
  "title": "Charla sobre Microservicios",
  "description": "Introducción a arquitectura de microservicios.",
  "type": "talk",
  "isOnline": true,
  "onlineUrl": "https://meet.example.com/charla",
  "startDate": "2025-09-01T18:00:00.000Z",
  "endDate": "2025-09-01T20:00:00.000Z"
}
```

### Caso 2: Evento con cupo y fechas de inscripción

```json
{
  "title": "Curso de PostgreSQL Avanzado",
  "description": "Curso intensivo de 3 días sobre PostgreSQL.",
  "type": "course",
  "location": "Facultad de Ingeniería",
  "minCapacity": 10,
  "maxCapacity": 30,
  "registrationStart": "2025-07-01T00:00:00.000Z",
  "registrationEnd": "2025-07-25T23:59:59.000Z",
  "startDate": "2025-08-01T09:00:00.000Z",
  "endDate": "2025-08-03T17:00:00.000Z"
}
```

### Caso 3: Filtrar eventos futuros

```
GET /api/v1/events?status=upcoming&type=conference&page=1&limit=10
```

### Caso 4: Filtrar eventos pasados

```
GET /api/v1/events?status=past&search=tecnologia
```
