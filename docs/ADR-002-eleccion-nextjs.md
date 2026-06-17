
**Título:** ADR-002 Elección de Next.js como framework de Frontend
**Estado:** Aceptado
**Fecha:** 2026-06-17
**Decisores:** Leandro Krauss, Ledesma Junior 
**Relacionado:** Project.md, specs/ (Events, Registration, Auth, Surveys, Speakers, Roles)

**Contexto**

* **Qué problema se está resolviendo:** Definición del framework de frontend para la aplicación web de gestión de eventos académicos, que debe ser moderna, responsive y mantenible.
* **Qué restricciones aplican (negocio, técnica, legal):** El sistema debe ser accesible desde cualquier dispositivo, soportar renderizado del lado del servidor para SEO y rendimiento, y permitir una estructura de rutas clara para los distintos módulos (eventos, autenticación, dashboard).
* **Qué datos de proyecto sustentan la decisión:** El equipo tiene experiencia previa con React y ecosistema JavaScript. Las funcionalidades requieren SSR para listados públicos de eventos y formularios interactivos para inscripciones y encuestas.

**Decisión**

* **Qué se decide exactamente:** Adoptar Next.js 15 como framework de frontend utilizando App Router, Server Components, y API Routes como patrón BFF (Backend For Frontend).
* **Alcance:** Cubre toda la interfaz de usuario de la plataforma, incluyendo los módulos de autenticación, eventos, inscripciones, certificados, encuestas, roles y reportes. Las API Routes de Next.js se limitarán al patrón BFF; la lógica de negocio residirá en el backend separado (Express/Hono).

**Alternativas consideradas**

* **Opción A: Next.js 15:**
  * **Pros:** SSR/SSG nativo, App Router con layouts anidados, Server Components para reducir JavaScript del lado del cliente, integración con TypeScript y Tailwind CSS, comunidad activa y ecosistema maduro.
  * **Contras:** Mayor curva de aprendizaje inicial frente a una SPA simple, bundling más complejo si no se usan correctamente los Server Components.

* **Opción B: React SPA con Vite:**
  * **Pros:** Configuración inicial más simple, ideal para aplicaciones puramente cliente, menor consumo de recursos del servidor.
  * **Contras:** Sin SSR nativo (requiere soluciones adicionales como React Helmet para SEO), peor rendimiento percibido en primera carga, más complejidad para rutas dinámicas.

* **Opción C: Angular:**
  * **Pros:** Framework completo con CLI robusto, tipado fuerte nativo con TypeScript.
  * **Contras:** Mayor boilerplate, curva de aprendizaje más pronunciada, ecosistema menos flexible para integraciones con librerías de UI modernas, menor afinidad del equipo.

**Consecuencias**

* **Beneficios esperados:** Mejor SEO para eventos públicos, carga inicial rápida gracias a SSR, estructura de rutas mantenible con App Router, desarrollo ágil con Server Components y el ecosistema React.
* **Costos o riesgos que se aceptan:** Dependencia de Vercel como plataforma de deploy recomendada, aunque es posible deployar en otros entornos Node.js. El abuso de "use client" puede degradar el rendimiento.
* **Impacto en operación y equipo:** El equipo deberá capacitarse en Server Components y patrones de Next.js 15. Las builds pueden ser más lentas que una SPA tradicional.

**Plan de implementación**

* **Pasos mínimos para ejecutarla:** Configurar el proyecto con `create-next-app`, definir la estructura de carpetas según `Project.md`, establecer el layout base con shadcn/ui y Tailwind CSS, implementar las rutas principales del App Router (auth, events, dashboard).

**Dependencias**

* Node.js 22 LTS como entorno de ejecución.
* Configuración de variables de entorno para API backend y autenticación.

**Métrica de éxito**

* Lighthouse score > 90 en rendimiento y SEO para las páginas públicas de eventos.
* Tiempo de carga inicial < 2 segundos en conexiones de banda ancha.

**Triggers de revisión**

* **Qué condiciones obligan a reabrir esta ADR:** Si el rendimiento en rutas con muchos datos dinámicos se degrada y las estrategias de caché de Next.js no son suficientes, o si la arquitectura de Server Components se vuelve inmanejable.
* **Fecha sugerida de revisión:** 2026-12-01
