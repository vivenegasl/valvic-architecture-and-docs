# Especificación de Arquitectura de Sistemas (Extracto Público)

Este documento detalla la topología de infraestructura, el flujo de interconexión de datos y las políticas de seguridad física y lógica aplicadas en el ecosistema **ValVic**.

---

## 1. Topología de Infraestructura (OCI Always Free)

El backend y la capa de datos de ValVic operan íntegramente bajo la capa de servicios **Always Free** de **Oracle Cloud Infrastructure (OCI)**, optimizando los costos de infraestructura a $0 sin comprometer el rendimiento ni la seguridad.

```
                    [ Tráfico HTTPS (Puerto 443) ]
                                  │
                                  ▼
                     [ OCI VCN Security List ]
                                  │
                                  ▼
                      [ VM Compute ARM Ampere ]
              ┌──────────────────────────────────────┐
              │  Ubuntu Server 22.04 LTS (aarch64)   │
              │                                      │
              │  [ Nginx SSL (Let's Encrypt) ]       │
              │                 │                    │
              │           (Proxy Local)              │
              │                 ▼                    │
              │  [ FastAPI (Uvicorn - Port 8001) ]   │
              │       │                       │      │
              │  (SQL Client)             (HTTPS)    │
              │       ▼                       ▼      │
              └───────┼───────────────────────┼──────┘
                      │                       │
                      ▼                       ▼
          [ Oracle Database 23ai ]    [ Anthropic Claude API ]
          (AI Vector Search & CRM)     (Orquestación Híbrida)
```

### Componentes Físicos y Lógicos de la Instancia:
1. **VM Compute Ampere A1 (ARM64):**
   * **Especificaciones:** 4 OCPUs ARM Ampere, 24 GB de memoria RAM y 200 GB de almacenamiento en bloque.
   * **Sistema Operativo:** Ubuntu Server 22.04 LTS optimizado para arquitecturas `aarch64`.
2. **Servidor Web y Proxy Reverso (Nginx):**
   * Redirige las peticiones entrantes desde los puertos estándar HTTP (`80`) y HTTPS (`443`) de forma interna hacia la API en el puerto `8001`.
   * Implementa cifrado fuerte de transporte TLS 1.3 con renovación automática de certificados Let's Encrypt mediante `certbot`.
3. **Servicio FastAPI (valvic-vicky.service):**
   * Demonio administrado por `systemd` que ejecuta la API web con `uvicorn`. Se configura con reinicio automático (`Restart=always`) ante excepciones críticas de sistema.
   * Las variables de entorno críticas y claves se inyectan en caliente desde un archivo de entorno protegido (`/opt/valvic/app/.env`), con permisos de lectura exclusivos para el usuario del servicio.

---

## 2. Persistencia y Almacenamiento de Datos

La persistencia se encuentra estrictamente desacoplada del código funcional, garantizando consistencia relacional, almacenamiento de estados de conversación y capacidades de IA vectorial en el mismo motor.

* **Oracle Database 23ai (Free Tier):**
   * **Rol:** Motor relacional multi-tenant principal. Almacena catálogos de servicios, registros de tenants, perfiles de pacientes y clientes, así como la base de leads y oportunidades del embudo comercial (pipeline de prospección y ventas).
   * **Esquema de Embudo Comercial (Leads & CRM):** Almacena el historial y estado de cada oportunidad (`nuevo`, `mensaje_listo`, `contactado`, `interesado`, `cierre_senal`, `pago_confirmado`) junto con el mensaje personalizado generado por el Agente Prospector y la bitácora de ofertas de Vicky.
   * **AI Vector Search:** Almacena los embeddings vectoriales de la base de conocimiento clínica (en la tabla `knowledge_base`) para realizar búsquedas semánticas nativas a gran velocidad y recuperar contexto relevante antes de llamar al LLM.
   * **LangGraph Checkpoints:** Almacena el estado persistente de los grafos conversacionales en la base de datos relacional, permitiendo reanudar o auditar conversaciones en cualquier paso.
* **Fallback de Desarrollo (SQLite Local):**
   * En entornos de desarrollo locales (`localhost`), el backend inicializa automáticamente una base de datos SQLite para acelerar las pruebas del equipo técnico, simulando el esquema relacional de Oracle mediante wrappers de base de datos adaptables.

---

## 3. Seguridad de Datos y Transporte

La seguridad perimetral de ValVic se implementa de manera defensiva en múltiples niveles:

### A. Validación y Autenticación en la Frontera (Ingress):
* **Firma Digital Meta Webhooks (HMAC SHA-256):**  
  Cada petición enviada por Meta a `/webhook/whatsapp` o `/webhook/messenger` incluye el header `X-Hub-Signature-256`. El backend realiza un cálculo criptográfico de tipo `HMAC-SHA256` utilizando el cuerpo bruto del request y la clave secreta de la app (`META_APP_SECRET`). Si la firma calculada no coincide exactamente con el encabezado recibido, la petición es abortada inmediatamente con un código `403 Forbidden`, mitigando ataques de suplantación.
* **Seguridad Perimetral en Red (Firewalls):**  
  * **UFW (Ubuntu local):** Bloquea las conexiones directas desde el exterior hacia el puerto `8001`. Solo el proxy inverso local (`nginx` en `127.0.0.1`) tiene permiso para interactuar con la API.
  * **OCI Security List (VCN):** Restringe las reglas de ingreso de red exclusivamente a los puertos de Nginx (`80`, `443`), reduciendo la superficie de exposición ante escaneos de puertos no autorizados.

### B. Consumo Seguro de APIs de Inteligencia Artificial (Egress):
* **Comunicaciones Cifradas:**  
  La comunicación con las APIs de Anthropic Claude y Google Gemini se realiza exclusivamente a través de túneles HTTPS con suites de cifrado modernas (TLS 1.2 y TLS 1.3), garantizando la confidencialidad de la información e impidiendo ataques de interceptación (Man-in-the-Middle).
* **Manejo de Secretos:**  
  Las llaves de API (`ANTHROPIC_API_KEY`, `GOOGLE_AI_API_KEY`) nunca se escriben en el sistema de archivos del monorepo público ni se registran en los logs. Se almacenan únicamente en el archivo `.env` del servidor de producción, el cual está excluido de git por política estricta de `.gitignore`.

### C. Resiliencia de Operación, Tareas de Fondo y Backups:
* **Tolerancia a Fallos de LLM (Failover Chain):**  
  El software detecta de forma autónoma errores de timeout, cuotas o caídas de los proveedores de LLM. La llamada es reintentada instantáneamente sobre el proveedor de respaldo (de Gemini a Claude, o viceversa) siguiendo la cadena de failover registrada en el componente de gestión de prompts.
* **Ejecución Programada de Prospección (Jobs/Cron):**  
  Las campañas de captación outbound del Agente Prospector se ejecutan en segundo plano mediante tareas programadas (Jobs/Cron). El sistema valida los flags de suscripción activos (`prospector_active`) de cada tenant antes de iniciar el scraping y la pre-calificación masiva para garantizar el aislamiento SaaS.
* **Copias de Respaldo Automatizadas:**  
  Un proceso en segundo plano programado mediante `cron` ejecuta un volcado de base de datos a las 03:00 AM diariamente. El archivo comprimido resultante es enviado y almacenado en un bucket privado de **Oracle Object Storage** con encriptación en reposo, garantizando un Punto de Recuperación Objetivo (RPO) inferior a 24 horas.
