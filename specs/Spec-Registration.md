# Spec: Módulo de Inscripciones (Registration)

## 1. Objetivo y Contexto
El objetivo de este módulo es gestionar el proceso mediante el cual los participantes se registran en eventos académicos[cite: 10]. Se debe soportar dos modalidades de inscripción: una autónoma por parte del usuario y otra gestionada manualmente por el personal del evento (organizador/admin)[cite: 10].

## 2. Historias de Usuario y Criterios de Aceptación
**HU-01: Inscripción autónoma**
Como participante registrado quiero inscribirme a un evento público para asegurar mi lugar[cite: 2, 10].
*   **CA1:** El usuario solo puede inscribirse si el evento está publicado (`isPublished = true`)[cite: 10].
*   **CA2:** Si el evento tiene `registrationStart` y `registrationEnd`, la inscripción debe rechazarse con error si está fuera de ese rango[cite: 10].
*   **CA3:** Si las inscripciones confirmadas alcanzan el `maxCapacity`, la nueva inscripción debe guardarse automáticamente con el estado `waitlist`[cite: 10].
*   **CA4:** Un usuario no puede inscribirse dos veces al mismo evento (combinación de `userId` y `eventId` debe ser única)[cite: 10].

**HU-02: Inscripción por parte del organizador**
Como organizador quiero inscribir a un participante manualmente para gestionar excepciones[cite: 2, 10].
*   **CA1:** La inscripción manual debe ignorar las validaciones de `maxCapacity` (override de cupo)[cite: 10].
*   **CA2:** La inscripción manual debe ignorar las validaciones de fechas límite de inscripción[cite: 10].

## 3. Requisitos Funcionales y Reglas de Negocio
*   **Estados de inscripción:** Toda inscripción debe manejar los estados `pending`, `confirmed`, `cancelled` o `waitlist`[cite: 10].
*   **Flujo por defecto:** Toda inscripción autónoma comienza en `pending` y debe pasar a `confirmed` automáticamente si hay cupo disponible[cite: 10].
*   **Promoción de Waitlist:** Se debe proveer una lógica para promover al primer usuario en lista de espera a `confirmed` si se libera un cupo[cite: 10].
*   **Endpoints requeridos a implementar:** 
    *   POST `/api/v1/events/:id/registrations` (Autónoma)[cite: 10].
    *   POST `/api/v1/events/:id/registrations/admin` (Organizador)[cite: 10].
    *   GET `/api/v1/events/:id/registrations` (Listado por evento)[cite: 10].
    *   DELETE `/api/v1/events/:id/registrations/:registrationId` (Cancelar)[cite: 10].
    *   POST `/api/v1/events/:id/registrations/waitlist/promote` (Promover)[cite: 10].

## 4. Restricciones técnicas específicas de este módulo
*   **Autenticación:** Todos los endpoints de inscripción requieren autenticación mediante token JWT[cite: 10].
*   **Autorización:** Los endpoints marcados como "Admin/Organizador" deben verificar mediante middleware que el usuario logueado es el dueño del evento o un administrador del sistema[cite: 10].
*   **Transacciones de BD:** La verificación de cupo (`maxCapacity`) y la posterior inserción del registro deben manejarse dentro de una transacción para evitar condiciones de carrera.

## 5. Modelo de datos de este módulo
La entidad principal a utilizar será `Registration` integrada a Prisma ORM[cite: 8].
```prisma
model Registration {
  id          String             @id @default(uuid()) @db.Uuid
  userId      String             @map("user_id") @db.Uuid
  eventId     String             @map("event_id") @db.Uuid
  status      RegistrationStatus @default(pending)
  attended    Boolean            @default(false)
  notes       String?            @db.Text
  createdAt   DateTime           @default(now()) @map("created_at")
  updatedAt   DateTime           @updatedAt @map("updated_at")

  user  User  @relation(fields: [userId], references: [id], onDelete: Cascade)
  event Event @relation(fields: [eventId], references: [id], onDelete: Cascade)

  @@unique([userId, eventId])
  @@index([userId])
  @@index([eventId])
  @@index([status])
  @@map("registrations")
}