# ADR-0001: Desacoplamiento de Ingress de Webhooks — De Cola SQL a Broker Estándar (Redis)

> **Estado:** Aprobado / Evolucionado a Estándar Enterprise (v2.1)  
> **Fecha:** Mayo 2026 (Revisión Enterprise: Noviembre 2026)  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Ingress / Mensajería Asíncrona / Resiliencia Operativa  

---

## 1. Contexto y Planteamiento del Problema

La integración con Meta Cloud API (WhatsApp Business v25.0) impone una restricción técnica crítica:
* Cada petición POST de webhook enviada por Meta al endpoint `/webhook/whatsapp` debe ser respondida con un código de estado HTTP `200 OK` en **menos de 3 segundos**.
* Si el servidor tarda más de 3 segundos o responde con error (5xx/4xx), Meta asume falla de conectividad e inicia una política agresiva de reintentos exponenciales.
* Si múltiples reintentos fallan, Meta desactiva automáticamente el webhook en la consola de desarrollador, incomunicando a todas las clínicas asociadas.

En la versión inicial (V1), el backend ejecutaba de forma síncrona el flujo conversacional completo al recibir el webhook: validaba la firma HMAC, invocaba al modelo de lenguaje (LLM), consultaba la base de datos y despachaba la respuesta a WhatsApp dentro de la misma petición HTTP. Ante picos de tráfico o lentitud en las APIs de IA (Gemini/Claude con latencias de 1.5 a 4 segundos), se producían timeouts generalizados y duplicidad masiva de mensajes.

---

## 2. Opciones Consideradas

| Alternativa | Descripción | Ventajas | Desventajas |
| :--- | :--- | :--- | :--- |
| **Opción A: Ejecución Síncrona en el Request** | Procesar todo el flujo dentro de la conexión HTTP abierta por Meta. | Simple de implementar; cero componentes extra. | Inviable. Rompe el SLA de 3 segundos de Meta ante cualquier fluctuación de red o latencia de LLM. |
| **Opción B: Cola Transaccional en Base de Datos (`SKIP LOCKED`)** *(Solución de Transición Inicial)* | Persistir el payload en la tabla `sys_webhook_queue` y procesar con workers en el `lifespan` de FastAPI usando `SELECT FOR UPDATE SKIP LOCKED`. | Evitaba instalar componentes adicionales en la capa Always Free de OCI. | **Complejidad accidental:** Sobrecarga de I/O en la base de datos relacional, acoplamiento de workers al ciclo de vida del servidor web y necesidad de programar manualmente reintentos, backoffs y DLQs. |
| **Opción C: Broker de Mensajería Estándar (Redis / RabbitMQ) con Ingress Gateway Stateless (Decisión Definitiva Enterprise v2.1)** | Un endpoint mínimo valida la firma HMAC, envía el mensaje a una cola en memoria en Redis en <5 ms y retorna `200 OK`. Flotas de workers desacoplados consumen de Redis. | **Estándar industrial indiscutible:** Resuelve de fábrica reintentos exponenciales, *Dead-Letter Queues* (DLQ), persistencia, métricas y escalabilidad horizontal sin tocar la base de datos relacional. | Requiere un servicio de Redis en Docker o administrado en la nube. |

---

## 3. Decisión de Arquitectura y Lección de Liderazgo

Se decide formalizar la transición hacia la **Opción C: Ingress Gateway Stateless con Broker Estándar (Redis)**:

1. **Reconocimiento de la Complejidad Accidental:**
   * La solución de transición con `sys_webhook_queue` y `SKIP LOCKED` demostró ser funcional para validar el concepto, pero violaba el principio de no reinventar la rueda en sistemas de producción reales. 
   * Tratar una base de datos relacional transaccional como broker de mensajería degrada el rendimiento de disco y obliga a mantener lógica de reintentos hecha a mano.

2. **Arquitectura Definitiva Enterprise (v2.1):**
   * **Ingress Stateless:** El microservicio de Ingress valida la firma HMAC-SHA256 (`X-Hub-Signature-256`), asigna una clave de idempotencia (`idempotency_key = SHA256(msg_id)`) y publica el evento en la cola `queue:whatsapp:inbound` de Redis con `LPUSH` / `XADD`.
   * **Tiempo de Respuesta:** Reducido a **menos de 15 ms** (p95 < 25 ms), garantizando cumplimiento absoluto del SLA de Meta.
   * **Consumo Desacoplado:** Los workers de IA (Google ADK) se ejecutan en contenedores independientes que consumen de Redis con `BRPOP` / *Consumer Groups*, con reintentos automáticos configurados de fábrica.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Desacoplamiento Absoluto:** Si la base de datos de pacientes o la API de Gemini sufren lentitud, el Ingress Gateway sigue respondiendo a Meta en milisegundos sin acumular hilos HTTP bloqueados.
* **Eliminación de Código Frágil:** Se elimina la necesidad de mantener tablas temporales de colas en SQL, reduciendo la superficie de mantenimiento.
* **Capacidad de Escalabilidad Horizontal:** Si el volumen de mensajes crece 10x, basta con levantar 3 réplicas adicionales del contenedor de workers en Docker sin tocar el Gateway.

### Consecuencias Negativas / Mitigación:
* **Componente Adicional de Infraestructura:** Se requiere un contenedor de Redis. Dado que Redis consume menos de 40 MB de RAM en reposo, su impacto en la VM de OCI es insignificante comparado con la estabilidad operativa que aporta.

---

## 5. Cumplimiento y Verificación

* **Test de Ingress:** Peticiones simuladas de webhooks retornan HTTP 200 en menos de 15 ms.
* **Tolerancia a Caídas:** Al detener intencionalmente el worker de IA, los mensajes se acumulan de forma segura en Redis sin pérdida de datos y se procesan en orden en cuanto el contenedor se levanta.