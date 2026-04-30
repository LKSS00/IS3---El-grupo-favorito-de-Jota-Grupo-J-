# Spec-Database - Academic Event Manager

Este documento define el esquema completo de la base de datos PostgreSQL utilizando Prisma ORM, incluyendo entidades, relaciones, constraints e índices.

---

## Diagrama Entidad-Relación

```
┌─────────────┐       ┌──────────────────┐       ┌─────────────┐
│    User     │──────<│  Registration    │>──────│   Event     │
└─────────────┘       └──────────────────┘       └─────────────┘
      │                        │                        │
      │                        │                        ├── Schedule
      │                        │                        ├── SpeakerEvent
      │                        │                        ├── Survey
      │                        │                        └── Comment
      │                        │
      │                        ├── Certificate
      │                        └── SurveyResponse
      │
      ├── UserRole
      └── PasswordResetToken
```

---

## Schema de Prisma

```prisma
// Configuración del generador
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Enums
enum UserRoleType {
  admin
  organizer
  speaker
  participant
}

enum EventType {
  course
  conference
  seminar
  workshop
  symposium
  talk
}

enum RegistrationStatus {
  pending
  confirmed
  cancelled
  waitlist
}

enum CertificateType {
  attendance
  approval
  speaker
  author
}

// ─────────────────────────────────────────────
// AUTENTICACIÓN Y USUARIOS
// ─────────────────────────────────────────────

model User {
  id            String    @id @default(uuid()) @db.Uuid
  email         String    @unique
  passwordHash  String    @map("password_hash")
  firstName     String    @map("first_name")
  lastName      String    @map("last_name")
  phone         String?
  avatar        String?
  isActive      Boolean   @default(true) @map("is_active")
  createdAt     DateTime  @default(now()) @map("created_at")
  updatedAt     DateTime  @updatedAt @map("updated_at")

  // Relaciones
  roles               UserRole[]
  registrations       Registration[]
  organizedEvents     Event[]            @relation("EventOrganizer")
  speakerEvents       SpeakerEvent[]
  certificates        Certificate[]
  surveysCreated      Survey[]           @relation("SurveyCreator")
  surveyResponses     SurveyResponse[]
  comments            Comment[]
  passwordResetTokens PasswordResetToken[]

  @@index([email])
  @@index([isActive])
  @@map("users")
}

model UserRole {
  id        String        @id @default(uuid()) @db.Uuid
  userId    String        @map("user_id") @db.Uuid
  role      UserRoleType
  eventId   String?       @map("event_id") @db.Uuid  // null = rol global, con valor = rol por evento
  createdAt DateTime      @default(now()) @map("created_at")

  user  User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  event Event?  @relation(fields: [eventId], references: [id], onDelete: Cascade)

  @@unique([userId, role, eventId])
  @@index([userId])
  @@index([eventId])
  @@map("user_roles")
}

model PasswordResetToken {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  token     String   @unique
  expiresAt DateTime @map("expires_at")
  used      Boolean  @default(false)
  createdAt DateTime @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([token])
  @@index([userId])
  @@map("password_reset_tokens")
}

// ─────────────────────────────────────────────
// EVENTOS
// ─────────────────────────────────────────────

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
  createdAt       DateTime      @default(now()) @map("created_at")
  updatedAt       DateTime      @updatedAt @map("updated_at")

  // Relaciones
  organizerId   String           @map("organizer_id") @db.Uuid
  organizer     User             @relation("EventOrganizer", fields: [organizerId], references: [id])

  registrations   Registration[]
  speakers        SpeakerEvent[]
  schedules       Schedule[]
  surveys         Survey[]
  certificates    Certificate[]
  comments        Comment[]

  @@index([type])
  @@index([isPublished])
  @@index([startDate, endDate])
  @@index([registrationStart, registrationEnd])
  @@map("events")
}

model Schedule {
  id          String   @id @default(uuid()) @db.Uuid
  eventId     String   @map("event_id") @db.Uuid
  title       String
  description String?  @db.Text
  speakerId   String?  @map("speaker_id") @db.Uuid
  startTime   DateTime @map("start_time")
  endTime     DateTime @map("end_time")
  location    String?
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  event   Event  @relation(fields: [eventId], references: [id], onDelete: Cascade)
  speaker User?  @relation(fields: [speakerId], references: [id], onDelete: SetNull)

  @@index([eventId])
  @@index([startTime])
  @@map("schedules")
}

model SpeakerEvent {
  id        String   @id @default(uuid()) @db.Uuid
  eventId   String   @map("event_id") @db.Uuid
  speakerId String   @map("speaker_id") @db.Uuid
  bio       String?  @db.Text
  createdAt DateTime @default(now()) @map("created_at")

  event   Event @relation(fields: [eventId], references: [id], onDelete: Cascade)
  speaker User  @relation(fields: [speakerId], references: [id], onDelete: Cascade)

  @@unique([eventId, speakerId])
  @@index([eventId])
  @@index([speakerId])
  @@map("speaker_events")
}

// ─────────────────────────────────────────────
// INSCRIPCIONES
// ─────────────────────────────────────────────

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

  certificates Certificate[]
  surveyResponses SurveyResponse[]

  @@unique([userId, eventId])
  @@index([userId])
  @@index([eventId])
  @@index([status])
  @@map("registrations")
}

// ─────────────────────────────────────────────
// ACREDITACIÓN
// ─────────────────────────────────────────────

model Accreditation {
  id              String    @id @default(uuid()) @db.Uuid
  registrationId  String    @map("registration_id") @db.Uuid
  accreditedAt    DateTime  @default(now()) @map("accredited_at")
  accreditedBy    String    @map("accredited_by") @db.Uuid
  qrCode          String?   @unique @map("qr_code")
  notes           String?   @db.Text

  registration Registration @relation(fields: [registrationId], references: [id], onDelete: Cascade)
  accreditedBy User         @relation(fields: [accreditedBy], references: [id])

  @@index([registrationId])
  @@index([qrCode])
  @@map("accreditations")
}

// ─────────────────────────────────────────────
// CERTIFICADOS
// ─────────────────────────────────────────────

model Certificate {
  id              String          @id @default(uuid()) @db.Uuid
  userId          String          @map("user_id") @db.Uuid
  eventId         String          @map("event_id") @db.Uuid
  registrationId  String          @map("registration_id") @db.Uuid
  type            CertificateType
  title           String
  description     String?         @db.Text
  fileUrl         String?         @map("file_url")
  issuedAt        DateTime        @default(now()) @map("issued_at")
  verificationCode String         @unique @map("verification_code")

  user         User         @relation(fields: [userId], references: [id])
  event        Event        @relation(fields: [eventId], references: [id])
  registration Registration @relation(fields: [registrationId], references: [id])

  @@index([userId])
  @@index([eventId])
  @@index([type])
  @@index([verificationCode])
  @@map("certificates")
}

// ─────────────────────────────────────────────
// ENCUESTAS Y FEEDBACK
// ─────────────────────────────────────────────

model Survey {
  id          String   @id @default(uuid()) @db.Uuid
  eventId     String   @map("event_id") @db.Uuid
  createdBy   String   @map("created_by") @db.Uuid
  title       String
  description String?  @db.Text
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  event     Event            @relation(fields: [eventId], references: [id], onDelete: Cascade)
  creator   User             @relation("SurveyCreator", fields: [createdBy], references: [id])
  questions SurveyQuestion[]
  responses SurveyResponse[]

  @@index([eventId])
  @@index([isActive])
  @@map("surveys")
}

model SurveyQuestion {
  id         String   @id @default(uuid()) @db.Uuid
  surveyId   String   @map("survey_id") @db.Uuid
  text       String
  type       String   // "rating", "text", "multiple_choice", "boolean"
  required   Boolean  @default(false)
  options    Json?    // ["Muy bueno", "Bueno", "Regular", "Malo"] para multiple_choice
  order      Int      @default(0)

  survey    Survey             @relation(fields: [surveyId], references: [id], onDelete: Cascade)
  answers   SurveyAnswer[]

  @@index([surveyId])
  @@map("survey_questions")
}

model SurveyResponse {
  id             String   @id @default(uuid()) @db.Uuid
  surveyId       String   @map("survey_id") @db.Uuid
  registrationId String   @map("registration_id") @db.Uuid
  respondentId   String   @map("respondent_id") @db.Uuid
  submittedAt    DateTime @default(now()) @map("submitted_at")

  survey       Survey          @relation(fields: [surveyId], references: [id], onDelete: Cascade)
  registration Registration    @relation(fields: [registrationId], references: [id], onDelete: Cascade)
  respondent   User            @relation(fields: [respondentId], references: [id])
  answers      SurveyAnswer[]

  @@unique([surveyId, registrationId])
  @@index([surveyId])
  @@index([respondentId])
  @@map("survey_responses")
}

model SurveyAnswer {
  id             String   @id @default(uuid()) @db.Uuid
  responseId     String   @map("response_id") @db.Uuid
  questionId     String   @map("question_id") @db.Uuid
  value          String   @db.Text  // respuesta libre, número como string, o índice de opción
  valueNumeric   Float?   @map("value_numeric") // para ratings 1-5

  response  SurveyResponse @relation(fields: [responseId], references: [id], onDelete: Cascade)
  question  SurveyQuestion @relation(fields: [questionId], references: [id], onDelete: Cascade)

  @@unique([responseId, questionId])
  @@index([responseId])
  @@index([questionId])
  @@map("survey_answers")
}

// ─────────────────────────────────────────────
// COMENTARIOS
// ─────────────────────────────────────────────

model Comment {
  id        String   @id @default(uuid()) @db.Uuid
  eventId   String   @map("event_id") @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  content   String   @db.Text
  rating    Int?     // 1-5
  parentId  String?  @map("parent_id") @db.Uuid  // para respuestas anidadas
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  event  Event    @relation(fields: [eventId], references: [id], onDelete: Cascade)
  user   User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  parent Comment? @relation("CommentReplies", fields: [parentId], references: [id])
  replies Comment[] @relation("CommentReplies")

  @@index([eventId])
  @@index([userId])
  @@index([rating])
  @@map("comments")
}
```

---

## Índices y Consideraciones de Performance

### Índices Compuestos

```prisma
// Búsqueda de eventos publicados por fecha
@@index([isPublished, startDate, type])

// Inscripciones por evento y estado
@@index([eventId, status], map: "idx_registration_event_status")

// Certificados por usuario y tipo
@@index([userId, type], map: "idx_certificate_user_type")

// Encuestas activas por evento
@@index([eventId, isActive], map: "idx_survey_event_active")
```

### Consideraciones

| Aspecto | Detalle |
|---------|---------|
| **UUID** | Todos los IDs usan `@db.Uuid` para consistencia y seguridad |
| **Soft Delete** | No se implementa; se usa `isActive` en User y `isPublished` en Event |
| **Auditoría** | `createdAt` y `updatedAt` en todas las entidades principales |
| **Paginación** | Índices en campos de filtro común para soporte de `OFFSET/LIMIT` eficiente |
| **JSON** | `options` en `SurveyQuestion` usa tipo `Json` para flexibilidad |
| **Constraints** | `@@unique` en combinaciones que deben ser irrepetibles (ej: user+event en Registration) |

---

## Migraciones

### Orden de Aplicación

1. `001_init_enums` - Creación de enums
2. `002_users_and_auth` - User, UserRole, PasswordResetToken
3. `003_events_and_schedules` - Event, Schedule, SpeakerEvent
4. `004_registrations` - Registration, Accreditation
5. `005_certificates` - Certificate
6. `006_surveys_and_feedback` - Survey, SurveyQuestion, SurveyResponse, SurveyAnswer
7. `007_comments` - Comment

### Seed Data

```typescript
// Usuarios base
- admin@academicevents.com / admin
- organizer@ejemplo.com / organizer123
- speaker@ejemplo.com / speaker123
- participant@ejemplo.com / participant123

// Eventos de ejemplo
- Congreso de Tecnología 2025 (conference)
- Curso de Introducción a PostgreSQL (course)
- Charla: Buenas prácticas en Clean Code (talk)
```
