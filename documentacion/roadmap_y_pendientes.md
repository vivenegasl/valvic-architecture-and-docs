# ValVic — Estado de Desarrollo, Roadmap & Tareas Pendientes

> **Informe de Ingeniería y Plan de Despliegue v2.0.**  
> *Actualizado: Junio 2026.*

---

## 1. Resumen Ejecutivo del Estado del Sistema

El ecosistema **ValVic** ha completado su refactorización a la arquitectura v2, consolidando el backend en **FastAPI + Google ADK + SQLModel**, el frontend de gestión en **Next.js 15 (CRM)** y la suite de marketing/sitios en **Astro (Landing & Client Factory)**.

### Métricas Clave de Ingeniería:
* **Cobertura de Tests:** +150 tests en Pytest (143 pasando en suite unitaria/mocked con SQLite in-memory).
* **Modelos de Datos:** 44 modelos SQLModel en 13 archivos de dominio.
* **DDL de Base de Datos:** 24 archivos de migración DDL secuenciales para Oracle Database 23ai.
* **Endpoints API:** 28 enrutadores REST con tipado estricto Pydantic v2.
* **Cero SQL Crudo:** 100% de operaciones de base de datos a través del ORM SQLModel / SQLAlchemy AsyncSession.
* **Costo de Servidores:** $0 USD/mes operando bajo la capa Always Free de Oracle Cloud Infrastructure (OCI).

---

## 2. Matriz de Módulos y Estado de Implementación

| # | Módulo / Funcionalidad | Entorno Local (Dev) | Producción (OCI) | Notas Técnicas |
|---|------------------------|:-------------------:|:----------------:|----------------|
| **1** | **API Core FastAPI + Google ADK** | ✅ Operativo | 🟡 Listo para Deploy | ValRouter unificado, control de intents y tools |
| **2** | **Webhooks Meta (WhatsApp / IG / Messenger)** | ✅ Operativo | 🟡 Listo para Deploy | Validación HMAC-SHA256 y payload v25.0 |
| **3** | **Cola Asíncrona `sys_webhook_queue`** | ✅ Operativo | 🟡 DDL `20` Listo | Worker in-process con `SKIP LOCKED` |
| **4** | **Motor de Agenda Anti-Double-Booking** | ✅ Operativo | 🟡 DDL `03` Listo | Bloqueo transaccional de slots y control de sobrecupos |
| **5** | **Recordatorios de Citas 24h** | ✅ Operativo | 🟡 DDL `11` Listo | Programación y despacho automático de avisos |
| **6** | **Campos Dinámicos EAV (Pacientes/Clínica)** | ✅ Operativo | 🟡 DDL `12` Listo | Extensibilidad sin alterar esquema relacional |
| **7** | **Pipeline Oportunidades & Lead Scoring** | ✅ Operativo | 🟡 DDL `13` Listo | Calificación automática de prospectos 1–10 |
| **8** | **Analytics & Atribución Comercial** | ✅ Operativo | 🟡 DDL `14` Listo | Registro de eventos UTM, fuentes y conversión |
| **9** | **Knowledge Base + AI Vector Search** | ✅ Operativo | 🟡 DDL `15` Listo | Búsqueda semántica con embeddings clínicos |
| **10** | **Catálogo de Servicios y Productos** | ✅ Operativo | 🟡 DDL `16` Listo | Listas de precios, duraciones y categorías |
| **11** | **Playbooks & Automatización** | ✅ Operativo | 🟡 DDL `17` Listo | Triggers y reglas de negocio condicionales |
| **12** | **Integraciones Externas (HubSpot / Pipedrive)** | ✅ Operativo | 🟡 DDL `18` Listo | Sincronización bidireccional de leads |
| **13** | **Billing & Pagos (Mercado Pago / PayPal / Redbank)** | ✅ Operativo | 🟡 DDL `19` Listo | Generación de links de pago y webhooks |
| **14** | **Prospector Autónomo + Scheduler In-Process** | ✅ Operativo | 🟡 DDL `05` Listo | Lead sourcing con Google Places y copy B2B |
| **15** | **CRM Next.js 15 (Dashboard)** | ✅ Compila (`build` ✅) | 🔴 Pendiente Deploy | Rutas de chats, agenda, pipeline y botón dorado |
| **16** | **Client Factory (Astro Multi-Tenant)** | ✅ Compila (`build` ✅) | ✅ Operativo | Generación de `clientes.valvic.cl` y `spotvet.cl` |
| **17** | **Portal Paciente (Autogestión OTP)** | ✅ Operativo | 🟡 DDL `21` Listo | Autenticación ligera sin contraseñas |
| **18** | **Landing Page Comercial (Astro)** | ✅ Compila (`build` ✅) | ✅ Operativo | Conexión con demo interactiva de CRM |
| **19** | **Compliance GDPR & Meta Data Deletion** | ✅ Operativo | 🟡 Listo para Deploy | Endpoints de borrado y verificación de estado |
| **20** | **Observabilidad & Logging Estructurado** | ✅ Operativo | 🟡 Listo para Deploy | Request IDs correlacionados y trazabilidad |

*Leyenda: ✅ = Completado y probado | 🟡 = Código listo, pendiente despliegue en servidor OCI | 🔴 = Tarea activa prioritaria*

---

## 3. Tareas Pendientes y Bloqueadores de Producción (Fase 0)

Para completar el lanzamiento comercial a producción (Go-Live), se han priorizado las siguientes acciones técnicas:

### Prioridad 1 — Despliegue de Infraestructura Cloud (OCI VM Ampere)
- [ ] **Aprovisionamiento de Instancia OCI:** Confirmar VM Ampere A1 (4 OCPU, 24 GB RAM) en región `sa-santiago-1` o equivalente Always Free.
- [ ] **Configuración de Caddy / Nginx SSL:** Configurar reverse proxy con certificados Let's Encrypt para `api.valvic.cl` y `crm.valvic.cl`.
- [ ] **Ejecución de DDLs Oracle 23ai:** Correr `./backend/scripts/run_oracle_ddl.sh` (scripts `01` a `24`) en la base de datos Oracle Autonomous 23ai.

### Prioridad 2 — Despliegue de Frontends
- [ ] **Deploy de CRM en Vercel / Docker OCI:** Desplegar `frontend/crm` apuntando a `NEXT_PUBLIC_API_URL=https://api.valvic.cl` y configurar variables de producción de Clerk (`CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`).
- [ ] **Deploy de Client Factory & Landing:** Configurar pipeline de build estático en Cloudflare Pages / Vercel para `valvic.cl` y `clientes.valvic.cl`.

### Prioridad 3 — Configuración de Meta Cloud API en Producción
- [ ] **Homologación de Webhook Meta:** Configurar URL `https://api.valvic.cl/webhook/whatsapp` y verificar token con la App de Meta for Developers.
- [ ] **Plantillas de Mensajes WhatsApp Business:** Enviar a revisión las plantillas oficiales para campañas outbound y recordatorios de citas de 24 horas.

### Prioridad 4 — Tareas de Mejora Continua y Post-Lanzamiento (Fase 1 / Backlog)
- [ ] **Módulo de Campañas Masivas CRM:** Finalizar interfaz visual de creación de campañas de WhatsApp con segmentación de tags.
- [ ] **Auditoría de Uso y Rate Limiting Avanzado:** Implementar límite dinámico de consumo de tokens por plan de suscripción en `sys_usage_tracking`.
- [ ] **Migración de Scripts Legacy:** Depurar scripts históricos que dependían de MySQL, asegurando 100% de compatibilidad con Oracle 23ai / SQLite.

---

## 4. Registro de Decisiones de Arquitectura (ADR) Recientes

1. **ADR-01: Migración a Google ADK (Agent Development Kit)**  
   *Contexto:* La versión anterior utilizaba una mezcla de CrewAI y LangGraph, lo que generaba sobrecarga de dependencias y latencias elevadas.  
   *Decisión:* Se unificó todo el flujo conversacional en Google ADK con el enrutador `ValRouter`, reduciendo el tiempo de inferencia y facilitando la inyección directa de herramientas (`tools`) para agenda y catálogo.

2. **ADR-02: Workers Lifespan In-Process vs Celery / Redis**  
   *Contexto:* Instalar y mantener Redis + Celery aumentaba el costo y la complejidad operativa de la instancia OCI.  
   *Decisión:* Se implementaron tareas en segundo plano mediante `asyncio` dentro del ciclo `lifespan` de FastAPI, utilizando la tabla `sys_webhook_queue` con `SELECT FOR UPDATE SKIP LOCKED` para garantizar concurrencia segura sin dependencias externas.

3. **ADR-03: Dual Database Strategy (Oracle 23ai + SQLite)**  
   *Contexto:* Necesidad de ejecutar pruebas unitarias ultrarrápidas en local y CI sin requerir conexión constante a Oracle Cloud.  
   *Decisión:* SQLModel con un patrón singleton en `engine.py` detecta automáticamente si se ejecutan tests (`"pytest" in sys.modules`) o desarrollo local, instanciando SQLite, mientras que en producción activa el pool de alta disponibilidad de Oracle 23ai.
