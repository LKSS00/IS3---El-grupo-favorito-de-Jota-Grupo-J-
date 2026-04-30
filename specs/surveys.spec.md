# Surveys Spec

## 1. Objetivo y Contexto

El módulo de encuestas permite recolectar feedback post-evento de los participantes en la plataforma **Academic Event Manager**. Los organizadores pueden ver las respuestas y estadísticas agregadas para evaluar la calidad de sus eventos. Las encuestas alimentan los informes del evento con métricas de satisfacción.

**Tipos de preguntas:** rating (1-5), texto abierto, booleano.

---

## 2. Historias de Usuario y Criterios de Aceptación

### HU-S1: Participante envía encuesta post-evento

**Como** participante acreditado, **quiero** completar una encuesta de satisfacción después del evento, **para** compartir mi experiencia y sugerencias.

**Criterios de aceptación:**
- Solo usuarios inscritos y acreditados en el evento pueden responder
- Solo una encuesta por usuario por evento
- La encuesta está disponible solo después de la fecha de fin del evento
- Una vez enviada, la encuesta no es editable
- Las preguntas requeridas deben responderse obligatoriamente
- Se devuelven los 7 campos de respuesta: 4 ratings, 1 booleano, 2 textos

### HU-S2: Organizador ve respuestas de encuestas

**Como** organizador de un evento, **quiero** ver todas las respuestas de encuestas, **para** analizar el feedback de los participantes.

**Criterios de aceptación:**
- Solo el organizador del evento (o admin) puede ver las respuestas
- Las respuestas se muestran paginadas (default: 20 por página)
- Se muestran respuestas anonimizadas (userId visible pero sin datos personales)
- Si no hay respuestas, se indica claramente

### HU-S3: Organizador ve estadísticas de encuestas

**Como** organizador, **quiero** ver estadísticas agregadas de las encuestas, **para** tener una visión rápida del nivel de satisfacción.

**Criterios de aceptación:**
- Muestra total de respuestas y tasa de respuesta (respuestas / acreditados)
- Muestra promedio general de ratings (escala 1-5)
- Muestra distribución de ratings (cuántos 1, 2, 3, 4, 5)
- Muestra palabras clave más frecuentes en comentarios
- Muestra score de sentimiento (0-1) calculado sobre respuestas de texto
- Solo accesible para organizador del evento o admin

---

## 3. Requisitos Funcionales y Reglas de Negocio

### RF-S1: Envío de encuestas
- Endpoint: `POST /api/v1/events/:id/surveys`
- Máximo 20 preguntas por encuesta
- Las respuestas se validan con esquema Zod según tipo

### RF-S2: Listado de respuestas
- Endpoint: `GET /api/v1/events/:id/surveys`
- Soporte de paginación: `page` (default 1), `limit` (default 20, máx 100)
- Formato de respuesta estándar con `meta` de paginación

### RF-S3: Estadísticas
- Endpoint: `GET /api/v1/events/:id/surveys/stats`
- Respuesta incluye: `totalResponses`, `responseRate`, `averageRating`, `ratingDistribution`, `topKeywords`, `sentimentScore`

### RF-S4: Preguntas por defecto

| # | Pregunta | Tipo | Requerida |
|---|----------|------|-----------|
| 1 | Calificación general del evento | rating (1-5) | Sí |
| 2 | Calidad del contenido | rating (1-5) | Sí |
| 3 | Calidad de los expositores | rating (1-5) | Sí |
| 4 | Organización del evento | rating (1-5) | Sí |
| 5 | ¿Recomendaría este evento? | boolean | Sí |
| 6 | Comentarios adicionales | text | No |
| 7 | Sugerencias de mejora | text | No |

### RF-S5: Modelo de datos
```typescript
interface Survey {
  id: string;
  eventId: string;
  userId: string;
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

### RB-S1: Una encuesta por usuario por evento
Si un usuario ya envió encuesta para un evento, se devuelve 409 `CONFLICT`.

### RB-S2: Disponible solo post-evento
La encuesta solo puede enviarse después de la fecha de fin del evento.

### RB-S3: No editable
Una vez enviada, la encuesta no puede modificarse ni eliminarse.

### RB-S4: Integración con Reports
Las estadísticas de encuestas se incluyen en los informes del evento (`GET /api/v1/events/:id/reports`). `responseRate` = encuestas enviadas / participantes acreditados.

---

## 4. Restricciones Técnicas del Módulo

- **Validación:** esquemas Zod para `SurveyResponse` y `Survey`, validar tipo de respuesta según tipo de pregunta
- **Respuestas de error:** seguir formato de `Contracts.md` (`NOT_FOUND`, `INSUFFICIENT_PERMISSIONS`, `CONFLICT`, `VALIDATION_ERROR`)
- **Paginación:** seguir formato estándar con `meta` (page, limit, total, totalPages)
- **Máximo 200 líneas por archivo** (siguiendo estándares del proyecto)
- **Máximo 3 niveles de anidamiento** por función
- **Convención de Commits:** Conventional Commits (`feat:`, `fix:`, etc.)
- **Persistencia:** respuestas se almacenan como JSON en campo de base de datos o tabla separada SurveyResponse
- **Relaciones:** Survey → Event (N:1), Survey → User (N:1), Survey → SurveyResponse (1:N)
- **Anonimización:** en listados públicos, no exponer datos personales del encuestado

---

## 5. Plan de Tareas

| # | Tarea | Tipo | Prioridad | Dependencias |
|---|-------|------|-----------|--------------|
| 1 | Crear esquema Prisma para Survey y SurveyResponse | Backend | Alta | - |
| 2 | Implementar esquemas Zod para validación de encuestas | Backend | Alta | - |
| 3 | Implementar endpoint POST /api/v1/events/:id/surveys | Backend | Alta | 1, 2 |
| 4 | Verificar inscripción y acreditación antes de permitir encuesta | Backend | Alta | 3 |
| 5 | Verificar que el evento haya finalizado | Backend | Alta | 4 |
| 6 | Implementar endpoint GET /api/v1/events/:id/surveys | Backend | Alta | 1 |
| 7 | Implementar paginación para listado de encuestas | Backend | Alta | 6 |
| 8 | Implementar endpoint GET /api/v1/events/:id/surveys/stats | Backend | Alta | 1 |
| 9 | Calcular ratingDistribution y averageRating | Backend | Media | 8 |
| 10 | Calcular topKeywords de respuestas de texto | Backend | Media | 8 |
| 11 | Calcular sentimentScore | Backend | Media | 8 |
| 12 | Proteger endpoints con middlewares de roles | Backend | Alta | 3-11 |
| 13 | Crear tests unitarios para validaciones | Test | Alta | 2 |
| 14 | Crear tests de integración para endpoints | Test | Alta | 3-12 |
| 15 | Crear formulario de encuesta en frontend | Frontend | Alta | 3 |
| 16 | Crear vista de estadísticas para organizador | Frontend | Media | 8 |

---

## 6. Estrategia de Verificación

### Tests Unitarios
- Esquemas Zod: verificar que aceptan respuestas válidas y rechazan inválidas
- Cálculo de `averageRating`: verificar promedio correcto con distintos inputs
- Cálculo de `ratingDistribution`: verificar conteo correcto por valor
- Cálculo de `responseRate`: verificar división encuestas/acreditados

### Tests de Integración
- POST /surveys: envío exitoso, duplicado (409), no acreditado (403), evento no finalizado (409)
- GET /surveys: listado paginado correcto, acceso no autorizado (403)
- GET /surveys/stats: estadísticas correctas con datos de prueba, acceso no autorizado (403)

### Tests E2E
- Flujo completo: evento finaliza → participante acreditado envía encuesta → organizador ve respuestas y estadísticas
- Verificar que un participante no acreditado no puede enviar encuesta
- Verificar que un organizador de otro evento no puede ver las encuestas

### Criterios de Calidad
- Cobertura de tests mínima: 80%
- Todas las respuestas siguen formato de `Contracts.md`
- Zero errores de validación no manejados
- Linting y typecheck sin errores
- Estadísticas calculadas correctamente con datos de prueba
