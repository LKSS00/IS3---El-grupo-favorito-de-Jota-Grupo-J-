# Spec-Registration - Academic Event Manager

Este documento define las especificaciones funcionales y técnicas del módulo de inscripciones, incluyendo inscripción autónoma, inscripción por personal del evento, manejo de cupos, fechas límite y lista de espera.

---

## Descripción del Módulo

El módulo de inscripciones gestiona el proceso mediante el cual los participantes se registran en eventos académicos. Soporta dos modalidades de inscripción (autónoma y gestionada por organizador), control de cupo mínimo/máximo, validación de fechas límite y sistema de lista de espera.

---

## Reglas de Negocio

### Reglas Generales

| Regla | Descripción |
|-------|-------------|
| **Unicidad** | Un usuario solo puede inscribirse una vez por evento (`userId + eventId` únicos) |
| **Evento publicado** | Solo se puede inscribir en eventos publicados (`isPublished = true`) |
| **Fechas de inscripción** | Si `registrationStart` y `registrationEnd` están definidos, la inscripción solo es válida dentro de ese rango |
| **Cupo máximo** | Si `maxCapacity` está alcanzado, nuevas inscripciones van a `waitlist` |
| **Cupo mínimo** | El evento se confirma cuando las inscripciones confirmadas alcanzan `minCapacity` |
| **Estado inicial** | Toda inscripción comienza en `pending` y pasa a `confirmed` tras validación automática o del organizador |
| **Inscripción por organizador** | El organizador puede inscribir participantes sin pasar por validaciones de cupo ni fechas |

### Estados de Inscripción

| Estado | Descripción | Transiciones posibles |
|--------|-------------|----------------------|
| `pending` | Inscripción recibida, pendiente de confirmación | `confirmed`, `cancelled` |
| `confirmed` | Inscripción confirmada, participante aceptado | `cancelled` |
| `cancelled` | Inscripción cancelada por usuario u organizador | `confirmed` (reactivación) |
| `waitlist` | Cupo lleno, en lista de espera | `confirmed` (si se libera lugar), `cancelled` |

---

## Endpoints

### Inscribirse a Evento (Autónoma)

**POST** `/api/v1/events/:id/registrations`

**Autenticación:** Requerida (cualquier rol)

**Request:**
```json
{
  "notes": "Soy estudiante de Ingeniería en Sistemas."
}
```

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "reg-id-uuid",
    "eventId": "event-id-uuid",
    "userId": "user-id-uuid",
    "status": "confirmed",
    "attended": false,
    "registeredAt": "2025-06-15T10:30:00.000Z"
  },
  "message": "Inscripción realizada exitosamente"
}
```

**Errores:**

| Código | HTTP | Condición |
|--------|------|-----------|
| `AUTH_REQUIRED` | 401 | No autenticado |
| `NOT_FOUND` | 404 | Evento no existe |
| `CONFLICT` | 409 | Ya inscrito en este evento |
| `VALIDATION_ERROR` | 400 | Inscripción fuera de fechas permitidas |

### Inscribir Participante (Organizador)

**POST** `/api/v1/events/:id/registrations/admin`

**Autenticación:** Requerida (rol `organizer` del evento o `admin`)

**Request:**
```json
{
  "userEmail": "nuevo@ejemplo.com",
  "firstName": "Ana",
  "lastName": "Martínez",
  "notes": "Inscripción manual por organización."
}
```

**Lógica:**

| Escenario | Comportamiento |
|-----------|----------------|
| Usuario existe | Se crea la inscripción vinculada al usuario |
| Usuario no existe | Se crea cuenta temporal y luego la inscripción |
| Cupo lleno | Permite inscripción por encima del cupo (override) |
| Fuera de fechas | Permite inscripción sin respetar fechas límite |

**Response (201):**
```json
{
  "success": true,
  "data": {
    "id": "reg-id-uuid",
    "eventId": "event-id-uuid",
    "userId": "user-id-uuid",
    "status": "confirmed",
    "attended": false,
    "registeredBy": "organizer-id-uuid",
    "registeredAt": "2025-06-15T10:30:00.000Z"
  },
  "message": "Participante inscrito exitosamente por el organizador"
}
```

---

### Listar Inscripciones de un Evento

**GET** `/api/v1/events/:id/registrations`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Query Params:**

| Parámetro | Tipo | Requerido | Default | Descripción |
|-----------|------|-----------|---------|-------------|
| `status` | `string` | No | `all` | Filtrar por estado |
| `page` | `number` | No | 1 | Página |
| `limit` | `number` | No | 20 | Items por página |

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "reg-id-uuid",
      "user": {
        "id": "user-id-uuid",
        "firstName": "Juan",
        "lastName": "Pérez",
        "email": "juan@ejemplo.com"
      },
      "status": "confirmed",
      "attended": false,
      "notes": "Soy estudiante de Ingeniería en Sistemas.",
      "registeredAt": "2025-06-15T10:30:00.000Z"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 142,
    "totalPages": 8
  }
}
```

---

### Obtener Mis Inscripciones

**GET** `/api/v1/users/me/registrations`

**Autenticación:** Requerida

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "reg-id-uuid",
      "event": {
        "id": "event-id-uuid",
        "title": "Congreso de Tecnología 2025",
        "type": "conference",
        "startDate": "2025-08-15T09:00:00.000Z",
        "endDate": "2025-08-17T18:00:00.000Z",
        "status": "upcoming"
      },
      "status": "confirmed",
      "attended": false,
      "registeredAt": "2025-06-15T10:30:00.000Z"
    }
  ]
}
```

---

### Cancelar Inscripción

**DELETE** `/api/v1/events/:id/registrations/:registrationId`

**Autenticación:** Requerida (usuario propio u organizador)

**Validaciones:**

| Validación | Descripción |
|------------|-------------|
| Propiedad | El usuario debe ser el dueño de la inscripción o el organizador |
| Evento no iniciado | No se puede cancelar si el evento ya comenzó |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "reg-id-uuid",
    "status": "cancelled"
  },
  "message": "Inscripción cancelada exitosamente"
}
```

---

### Confirmar Inscripción (Organizador)

**PATCH** `/api/v1/events/:id/registrations/:registrationId/status`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Request:**
```json
{
  "status": "confirmed"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "reg-id-uuid",
    "status": "confirmed"
  },
  "message": "Inscripción confirmada exitosamente"
}
```

---

### Gestionar Lista de Espera

**GET** `/api/v1/events/:id/registrations/waitlist`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Response (200):**
```json
{
  "success": true,
  "data": [
    {
      "id": "reg-id-uuid",
      "user": {
        "firstName": "Ana",
        "lastName": "Martínez",
        "email": "ana@ejemplo.com"
      },
      "status": "waitlist",
      "registeredAt": "2025-06-20T14:00:00.000Z",
      "waitlistPosition": 1
    }
  ],
  "meta": {
    "total": 8
  }
}
```

#### Promover de Lista de Espera

**POST** `/api/v1/events/:id/registrations/waitlist/promote`

**Autenticación:** Requerida (organizador del evento o `admin`)

**Lógica:** Promueve al primer usuario en la lista de espera a `confirmed`.

**Request:**
```json
{
  "registrationId": "reg-id-uuid"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "reg-id-uuid",
    "status": "confirmed",
    "promotedAt": "2025-07-01T09:00:00.000Z"
  },
  "message": "Participante promovido de lista de espera"
}
```

---

## Validaciones de Inscripción

### Flujo de Validación (Inscripción Autónoma)

```
1. ¿Usuario autenticado? ──NO──> Error: AUTH_REQUIRED
        │ SÍ
        ▼
2. ¿Evento existe y publicado? ──NO──> Error: NOT_FOUND
        │ SÍ
        ▼
3. ¿Ya inscrito? ──SÍ──> Error: CONFLICT
        │ NO
        ▼
4. ¿Dentro de fechas de inscripción? ──NO──> Error: VALIDATION_ERROR
        │ SÍ o no definido
        ▼
5. ¿Cupo disponible? ──NO──> Estado: waitlist
        │ SÍ
        ▼
6. Crear inscripción → Estado: confirmed
```

### Tabla de Reglas de Cupo

| Condición | Resultado |
|-----------|-----------|
| `maxCapacity` no definido | Inscripción directa → `confirmed` |
| `maxCapacity` definido y disponible | Inscripción → `confirmed` |
| `maxCapacity` alcanzado | Inscripción → `waitlist` |
| `minCapacity` alcanzado | Evento se confirma automáticamente |
| Inscripción por organizador | Override de cupo → `confirmed` |

---

## Casos de Uso

### Caso 1: Inscripción autónoma con cupo disponible

```
POST /api/v1/events/a1b2c3d4/registrations
Body: { "notes": "Me interesa el tema de IA." }
→ Estado: confirmed
```

### Caso 2: Inscripción autónoma con cupo lleno

```
POST /api/v1/events/a1b2c3d4/registrations
Body: {}
→ Estado: waitlist (maxCapacity alcanzado)
```

### Caso 3: Inscripción fuera de fecha

```
POST /api/v1/events/a1b2c3d4/registrations
→ Error: VALIDATION_ERROR (registrationEnd ya pasó)
```

### Caso 4: Organizador inscribe participante manualmente

```
POST /api/v1/events/a1b2c3d4/registrations/admin
Body: {
  "userEmail": "invitado@ejemplo.com",
  "firstName": "Roberto",
  "lastName": "Fernández"
}
→ Estado: confirmed (override de cupo y fechas)
```

### Caso 5: Promover desde lista de espera

```
POST /api/v1/events/a1b2c3d4/registrations/waitlist/promote
Body: { "registrationId": "waitlist-reg-id" }
→ Estado: waitlist → confirmed
```
