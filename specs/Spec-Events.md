# Spec: Módulo de Gestión de Eventos (Events)

## 1. Objetivo y Contexto
El objetivo de este módulo es permitir a los organizadores crear, editar, publicar y gestionar eventos académicos (congresos, cursos, charlas, etc.)[cite: 9]. Este es el módulo central del sistema, ya que provee los datos principales que los usuarios consumirán para informarse y, posteriormente, inscribirse[cite: 9].

## 2. Historias de Usuario y Criterios de Aceptación
**HU-01: Creación de Evento**
Como organizador quiero crear un evento en la plataforma para comenzar a planificar su difusión[cite: 2, 9].
*   **CA1:** Al crearse, el evento debe quedar con el estado `isPublished = false` (Borrador)[cite: 9].
*   **CA2:** El sistema debe generar automáticamente un `slug` único basado en el título del evento[cite: 9].
*   **CA3:** El sistema debe rechazar la creación si `startDate` no es menor que `endDate`[cite: 9].

**HU-02: Publicación de Evento**
Como organizador quiero publicar un evento en estado borrador para que sea visible al público[cite: 2, 9].
*   **CA1:** El evento solo puede publicarse si tiene definidos los campos obligatorios (`title`, `description`, `type`, `startDate`, `endDate`)[cite: 9].
*   **CA2:** Una vez publicado, el evento debe aparecer en el listado público de eventos[cite: 9].

**HU-03: Búsqueda y Filtrado**
Como usuario público quiero ver el listado de eventos y filtrarlos para encontrar los de mi interés[cite: 2, 9].
*   **CA1:** El listado general solo debe mostrar eventos con `isPublished = true`[cite: 9].
*   **CA2:** El usuario debe poder filtrar los resultados por `status` (upcoming, ongoing, past) y por `type` (course, conference, etc.)[cite: 9].

## 3. Requisitos Funcionales y Reglas de Negocio
*   **Validación de Fechas de Inscripción:** Si se definen `registrationStart` y `registrationEnd`, estas deben ser coherentes entre sí y `registrationEnd` no puede ser posterior a `startDate`[cite: 9].
*   **Control de Cupos:** Si se define `minCapacity`, debe ser siempre menor o igual a `maxCapacity`[cite: 9].
*   **Eliminación Segura:** Un evento no puede ser eliminado ni despublicado si ya comenzó o si tiene inscripciones confirmadas[cite: 9].
*   **Endpoints requeridos a implementar:**
    *   GET `/api/v1/events` (Público, con filtros)[cite: 9].
    *   GET `/api/v1/events/:id` (Detalle público)[cite: 9].
    *   POST `/api/v1/events` (Crear borrador)[cite: 9].
    *   PATCH `/api/v1/events/:id` (Actualizar evento)[cite: 9].
    *   POST `/api/v1/events/:id/publish` y `/unpublish` (Cambio de estado)[cite: 9].
    *   DELETE `/api/v1/events/:id` (Eliminar)[cite: 9].

## 4. Restricciones técnicas específicas de este módulo
*   **Seguridad y Roles:** Los endpoints de creación, modificación, publicación y eliminación deben estar protegidos por middleware, verificando que el usuario tenga rol `organizer` o `admin`[cite: 9].
*   **Propiedad del Recurso:** Un organizador solo puede editar o eliminar los eventos que él mismo haya creado (`organizerId == userId`)[cite: 9].
*   **Paginación:** El endpoint público de listado debe implementar obligatoriamente parámetros de paginación (`page`, `limit`) para no saturar la base de datos[cite: 9].

## 5. Modelo de datos de este módulo
La entidad principal a utilizar será `Event`, la cual interactúa con el esquema de Prisma ORM[cite: 8].
```prisma
model Event {
  id              String        @id @default(uuid()) @db.Uuid
  title           String
  description     String        @db.Text
  type            EventType
  slug            String        @unique
  banner          String?
  location        String?
  isOnline        Boolean       @default(false) @map("is_online")
  onlineUrl       String?       @map("online_url")
  minCapacity     Int?          @map("min_capacity")
  maxCapacity     Int?          @map("max_capacity")
  registrationStart DateTime?   @map("registration_start")
  registrationEnd   DateTime?   @map("registration_end")
  startDate       DateTime      @map("start_date")
  endDate         DateTime      @map("end_date")
  isPublished     Boolean       @default(false) @map("is_published")
  organizerId     String        @map("organizer_id") @db.Uuid
  createdAt       DateTime      @default(now()) @map("created_at")
  updatedAt       DateTime      @updatedAt @map("updated_at")

  organizer     User             @relation("EventOrganizer", fields: [organizerId], references: [id])
  
  @@index([type])
  @@index([isPublished])
  @@index([startDate, endDate])
  @@map("events")
}