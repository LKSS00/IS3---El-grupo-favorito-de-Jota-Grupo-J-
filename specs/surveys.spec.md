# Surveys Spec

## Endpoints

| Método | Endpoint | Descripción | Autenticación |
|--------|----------|-------------|---------------|
| POST | `/api/v1/events/:id/surveys` | Enviar encuesta | Sí |
| GET | `/api/v1/events/:id/surveys` | Listar respuestas | Sí (organizer) |
| GET | `/api/v1/events/:id/surveys/stats` | Estadísticas de encuesta | Sí (organizer) |

---

## POST /api/v1/events/:id/surveys

### Descripción

Permite a un participante registrado enviar su evaluación de satisfacción post-evento.

### Restricciones

- Solo usuarios **inscritos y acreditados** en el evento pueden responder
- **Una encuesta por usuario por evento**
- Disponible solo **después de la fecha de fin del evento**
- No editable una vez enviada

### Request Body

```json
{
  "responses": [
    {
      "questionId": "q1",
      "question": "Calificación general del evento",
      "type": "rating",
      "value": 5
    },
    {
      "questionId": "q2",
      "question": "¿Qué le pareció la organización?",
      "type": "rating",
      "value": 4
    },
    {
      "questionId": "q3",
      "question": "Comentarios adicionales",
      "type": "text",
      "value": "Excelente evento, muy bien organizado"
    },
    {
      "questionId": "q4",
      "question": "¿Recomendaría este evento?",
      "type": "boolean",
      "value": true
    }
  ]
}
```

### Validación (Zod)

```typescript
const surveyResponseSchema = z.object({
  questionId: z.string().uuid(),
  question: z.string().min(1, "La pregunta es requerida"),
  type: z.enum(["rating", "text", "boolean"]),
  value: z.union([
    z.number().min(1).max(5), // rating
    z.string(),                // text
    z.boolean(),               // boolean
  ]),
});

const surveySchema = z.object({
  responses: z
    .array(surveyResponseSchema)
    .min(1, "Debe incluir al menos una respuesta")
    .max(20, "Máximo 20 preguntas por encuesta"),
});
```

### Response (201)

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "eventId": "e1f2g3h4-i5j6-7890-abcd-ef1234567890",
    "userId": "u1v2w3x4-y5z6-7890-abcd-ef1234567890",
    "responses": [
      {
        "questionId": "q1",
        "question": "Calificación general del evento",
        "type": "rating",
        "value": 5
      }
    ],
    "submittedAt": "2025-06-20T18:30:00.000Z"
  },
  "message": "Encuesta enviada exitosamente"
}
```

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `NOT_FOUND` | 404 | Evento no encontrado |
| `AUTH_REQUIRED` | 401 | No autenticado |
| `INSUFFICIENT_PERMISSIONS` | 403 | No inscrito en el evento |
| `CONFLICT` | 409 | Ya envió encuesta para este evento |
| `VALIDATION_ERROR` | 400 | Datos inválidos |
| `CONFLICT` | 409 | El evento aún no ha finalizado |

---

## GET /api/v1/events/:id/surveys

### Descripción

Lista todas las respuestas de encuestas para un evento. Solo accesible para el organizador.

### Query Params

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `page` | `number` | 1 | Página |
| `limit` | `number` | 20 | Por página (máx: 100) |

### Response (200)

```json
{
  "success": true,
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "userId": "u1v2w3x4-y5z6-7890-abcd-ef1234567890",
      "responses": [
        {
          "questionId": "q1",
          "type": "rating",
          "value": 5
        }
      ],
      "submittedAt": "2025-06-20T18:30:00.000Z"
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

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `NOT_FOUND` | 404 | Evento no encontrado |
| `INSUFFICIENT_PERMISSIONS` | 403 | No es organizador del evento |

---

## GET /api/v1/events/:id/surveys/stats

### Descripción

Estadísticas agregadas de las encuestas de un evento. Solo accesible para el organizador.

### Response (200)

```json
{
  "success": true,
  "data": {
    "totalResponses": 45,
    "responseRate": 0.75,
    "averageRating": 4.3,
    "ratingDistribution": {
      "1": 2,
      "2": 3,
      "3": 5,
      "4": 15,
      "5": 20
    },
    "topKeywords": ["organización", "contenido", "expositores"],
    "sentimentScore": 0.85
  }
}
```

### Errores

| Código | HTTP | Escenario |
|--------|------|-----------|
| `NOT_FOUND` | 404 | Evento no encontrado |
| `INSUFFICIENT_PERMISSIONS` | 403 | No es organizador del evento |

---

## Modelo de Datos (Survey)

```typescript
interface Survey {
  id: string;           // UUID
  eventId: string;      // UUID - FK a Event
  userId: string;       // UUID - FK a User
  responses: SurveyResponse[];
  submittedAt: Date;
  createdAt: Date;
  updatedAt: Date;
}

interface SurveyResponse {
  questionId: string;
  question: string;
  type: "rating" | "text" | "boolean";
  value: number | string | boolean;
}
```

### Relaciones

```
Survey ────< SurveyResponse
   │
   ├──> Event (1:1)
   └──> User (1:1)
```

---

## Preguntas por Defecto

| # | Pregunta | Tipo | Requerida |
|---|----------|------|-----------|
| 1 | Calificación general del evento | rating (1-5) | Sí |
| 2 | Calidad del contenido | rating (1-5) | Sí |
| 3 | Calidad de los expositores | rating (1-5) | Sí |
| 4 | Organización del evento | rating (1-5) | Sí |
| 5 | ¿Recomendaría este evento? | boolean | Sí |
| 6 | Comentarios adicionales | text | No |
| 7 | Sugerencias de mejora | text | No |

---

## Integración con Reports

- Las estadísticas de encuestas se incluyen en los **informes del evento** (`GET /api/v1/events/:id/reports`)
- El `sentimentScore` se calcula mediante análisis básico de texto en respuestas abiertas
- `responseRate` = total de encuestas enviadas / total de participantes acreditados
