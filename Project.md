# Academic Event Manager

## Visión General

**Academic Event Manager** es una aplicación web diseñada para facilitar la organización, gestión y seguimiento de eventos académicos como congresos, jornadas, cursos, charlas y talleres. La plataforma permite a organizadores administrar eventos de manera integral, mientras que los participantes pueden inscribirse, interactuar y obtener certificaciones de forma autónoma.

### Objetivos del Proyecto

- Centralizar la gestión de eventos académicos en una plataforma accesible desde cualquier dispositivo
- Facilitar la inscripción y acreditación de participantes
- Automatizar la generación de certificados e informes
- Proveer herramientas de feedback post-evento mediante encuestas y comentarios

### Funcionalidades Principales

| Módulo | Descripción |
|--------|-------------|
| Gestión de eventos | Creación, edición y publicación de eventos con tipo, fecha, cupo y fechas límite |
| Inscripción de participantes | Registro autónomo o gestionado por personal del evento |
| Gestión de roles | Control de acceso basado en roles: Organizador, Participante, Disertante |
| Acreditación | Validación y confirmación de asistencia al evento |
| Feedback post-evento | Comentarios y encuestas de satisfacción |
| Certificados | Generación automática de certificados (asistencia, aprobación, autor/expositor) |
| Informes | Reportes del evento incluyendo agenda, métricas de asistencia y participación |

---

## Stack Tecnológico

### Frontend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Next.js** | 15.x | Framework React con SSR/SSG, routing y API routes |
| **TypeScript** | 5.x | Tipado estático para mayor robustez |
| **Tailwind CSS** | 4.x | Framework de utilidades para diseño responsive |
| **shadcn/ui** | latest | Componentes UI accesibles y personalizables |
| **React Hook Form + Zod** | latest | Gestión de formularios y validación de esquemas |

### Backend

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **Node.js** | 22.x (LTS) | Entorno de ejecución |
| **Express.js** | 4.x / **Hono** | Framework HTTP para API RESTful |
| **TypeScript** | 5.x | Tipado estático consistente con frontend |
| **Prisma ORM** | 6.x | ORM type-safe para gestión de base de datos |
| **JWT + bcrypt** | latest | Autenticación y hashing de contraseñas |
| **Zod** | latest | Validación de datos de entrada |

### Base de Datos

| Tecnología | Versión | Propósito |
|------------|---------|-----------|
| **PostgreSQL** | 16.x | Base de datos relacional principal |
| **Redis** | 7.x | Caché, sesiones y colas de tareas |

### Infraestructura y DevOps

| Tecnología | Propósito |
|------------|-----------|
| **Docker + Docker Compose** | Contenedores para desarrollo y producción |
| **GitHub Actions** | CI/CD pipeline |
| **Vercel / Railway** | Deployment (frontend + backend) |

---

## Estándares de Codificación

### Clean Code & Principios

1. **SOLID**: Aplicar los cinco principios de diseño orientado a objetos
2. **DRY**: Evitar duplicación de lógica; extraer funciones y componentes reutilizables
3. **KISS**: Mantener soluciones simples y directas; sobre-ingeniería solo cuando sea justificable
4. **YAGNI**: No implementar funcionalidades que no se necesitan actualmente

### Convenciones de Nomenclatura

| Elemento | Convención | Ejemplo |
|----------|------------|---------|
| Archivos fuente | `kebab-case` | `event-registration.tsx` |
| Componentes React | `PascalCase` | `EventCard.tsx` |
| Funciones/Variables | `camelCase` | `getEventById()` |
| Constantes | `UPPER_SNAKE_CASE` | `MAX_PARTICIPANTS` |
| Interfaces/Tipos | `PascalCase` | `EventDTO`, `UserRole` |
| Rutas de API | `kebab-case` | `/api/v1/event-registration` |

### Estructura de Código

- **Máximo 200 líneas por archivo** (componentes u handlers)
- **Máximo 3 niveles de anidamiento** por función
- **Funciones de un solo propósito** con nombres descriptivos
- **Comentarios solo para el "por qué"**, no para el "qué" (el código debe ser autoexplicativo)
- **Manejo de errores centralizado** con middleware dedicado

### Git Workflow

| Regla | Descripción |
|-------|-------------|
| Commits atómicos | Un commit = un cambio lógico |
| Conventional Commits | `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:` |
| Branches | `main` (producción), `develop` (integración), `feat/*`, `fix/*` |
| Pull Requests | Requieren al menos 1 review antes de merge |

---

## Estructura de Carpetas

```
academic-event-manager/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Pipeline de integración continua
│       └── deploy.yml                # Pipeline de despliegue
│
├── apps/
│   ├── web/                          # Frontend (Next.js)
│   │   ├── public/                   # Assets estáticos
│   │   ├── src/
│   │   │   ├── app/                  # App Router (rutas y layouts)
│   │   │   │   ├── (auth)/           # Grupo: login, registro, recuperación
│   │   │   │   ├── (events)/         # Grupo: listado, detalle, inscripción
│   │   │   │   ├── (dashboard)/      # Grupo: paneles por rol
│   │   │   │   ├── api/              # API routes (BFF pattern)
│   │   │   │   └── layout.tsx
│   │   │   ├── components/           # Componentes reutilizables
│   │   │   │   ├── ui/               # Componentes base (shadcn/ui)
│   │   │   │   ├── events/           # Componentes específicos de eventos
│   │   │   │   ├── auth/             # Componentes de autenticación
│   │   │   │   └── certificates/     # Componentes de certificados
│   │   │   ├── hooks/                # Custom React hooks
│   │   │   ├── lib/                  # Utilidades y configuración
│   │   │   ├── services/             # Clientes de API
│   │   │   ├── store/                # Estado global (Zustand)
│   │   │   ├── types/                # Tipos TypeScript compartidos
│   │   │   └── utils/                # Funciones auxiliares
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   └── package.json
│   │
│   └── api/                          # Backend (Node.js + Express/Hono)
│       ├── src/
│       │   ├── config/               # Configuración (env, db, redis)
│       │   ├── modules/              # Módulos por dominio
│       │   │   ├── auth/             # Autenticación y autorización
│       │   │   ├── events/           # CRUD de eventos
│       │   │   ├── registrations/    # Inscripciones y cupos
│       │   │   ├── roles/            # Gestión de roles
│       │   │   ├── certificates/     # Generación de certificados
│       │   │   ├── surveys/          # Encuestas y feedback
│       │   │   └── reports/          # Informes y agenda
│       │   ├── common/               # Código compartido entre módulos
│       │   │   ├── middleware/       # Middleware (auth, validation, error)
│       │   │   ├── utils/            # Utilidades
│       │   │   └── types/            # Tipos compartidos
│       │   ├── database/
│       │   │   ├── prisma/
│       │   │   │   ├── schema.prisma # Esquema de base de datos
│       │   │   │   └── migrations/   # Migraciones
│       │   │   └── seed.ts           # Datos de prueba
│       │   └── index.ts              # Entry point
│       └── package.json
│
├── packages/                         # Paquetes compartidos (monorepo)
│   ├── types/                        # Tipos TypeScript compartidos
│   ├── validators/                   # Esquemas Zod compartidos
│   └── config/                       # Configs compartidas (tsconfig, eslint)
│
├── docker/
│   ├── docker-compose.yml            # Servicios locales (PostgreSQL, Redis)
│   └── Dockerfile                    # Imágenes de producción
│
├── docs/
│   ├── api/                          # Documentación de API
│   ├── architecture/                 # Decisiones arquitectónicas
│   └── guides/                       # Guías de desarrollo
│
├── .env.example
├── .gitignore
├── package.json                      # Root package.json (workspaces)
├── turbo.json                        # Configuración de Turborepo (si aplica)
└── README.md
```

---

## Modelo de Datos Preliminar (Entidades Principales)

```
User ────────────────< Registration >────────── Event
  │                        │                     │
  │                        │                     ├── Schedule
  │                        │                     ├── Speaker (role)
  │                        │                     └── Survey
  │                        │
  │                        └── Certificate
  │
  └── Role (Organizer / Participant / Speaker)
```

---

## Próximos Pasos

1. Definir esquema completo de Prisma con relaciones y constraints
2. Diseñar wireframes de las vistas principales
3. Configurar entorno de desarrollo con Docker Compose
4. Implementar módulo de autenticación (primer entregable)
5. Establecer pipeline CI/CD básico
