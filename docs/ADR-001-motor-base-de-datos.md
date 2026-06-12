
---

**Título:** ADR-001 Elección de PostgreSQL como motor de base de datos
**Estado:** Aceptado
**Fecha:** 2026-06-12
**Decisores:** Leandro Krauss, Junior
**Relacionado:** project.md, Spec-Database.md

**Contexto**

* **Qué problema se está resolviendo:** Definición del motor de persistencia principal para el sistema de gestión de eventos académicos.
* **Qué restricciones aplican (negocio, técnica, legal):** El sistema requiere integridad transaccional estricta para la gestión de registros de asistentes, roles y encuestas.
* **Qué datos de proyecto sustentan la decisión:** La estructura de datos analizada en las *specs* (Eventos, Oradores, Usuarios) presenta relaciones complejas fuertemente tipadas que se benefician de un esquema relacional.

**Decisión**

* **Qué se decide exactamente:** Adoptar PostgreSQL como el motor de base de datos relacional principal del sistema.
* **Alcance:** Cubre la persistencia de los módulos de usuarios, eventos, oradores, encuestas y registros. No excluye el uso futuro de bases en memoria para caché.

**Alternativas consideradas**

* **Opción A: PostgreSQL:**
* **Pros:** Excelente soporte para transacciones ACID, tipos de datos avanzados, fuerte cumplimiento de estándares SQL e integridad referencial.
* **Contras:** El escalamiento horizontal nativo es más complejo que en soluciones NoSQL.


* **Opción B: MongoDB (NoSQL):**
* **Pros:** Esquema flexible y escalado horizontal nativo.
* **Contras:** Falta de *joins* nativos eficientes y consistencia eventual que no se alinea con la precisión requerida para la matriculación a eventos.



**Consecuencias**

* **Beneficios esperados:** Consistencia absoluta en los datos críticos del dominio y facilidad para realizar reportes estructurados.
* **Costos o riesgos que se aceptan:** Las migraciones de esquema requerirán una planificación cuidadosa para no generar tiempos de inactividad.
* **Impacto en operación y equipo:** El equipo utilizará el ecosistema estándar de herramientas relacionales (pgAdmin, psql, scripts de migración en C# o el stack definido).

**Plan de implementación**

* **Pasos mínimos para ejecutarla:** Configurar la cadena de conexión en el entorno local, establecer el esquema inicial basado en las *specs* y configurar el motor en el entorno de despliegue.

**Dependencias**

* Configuración de infraestructura (Docker/Hosting) para alojar la instancia de base de datos.

**Métrica de éxito**

* Despliegue exitoso del esquema inicial sin errores de integridad referencial.

**Triggers de revisión**

* **Qué condiciones obligan a reabrir esta ADR:** Si las consultas de lectura alcanzan cuellos de botella que requieran evaluar fragmentación (*sharding*) o bases NoSQL secundarias.
* **Fecha sugerida de revisión:** 2026-12-01

---
