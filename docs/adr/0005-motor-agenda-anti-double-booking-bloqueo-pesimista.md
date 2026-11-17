# ADR-0005: Prevención de Double-Booking — De Lógica Procedural Artesanal a Restricciones Declarativas y Locks Distribuidos (Redis)

> **Estado:** Arquitectura Objetivo (To-Be) Aprobada para Refactorización  
> **Fecha:** Mayo 2026 (Revisión Enterprise To-Be: Noviembre 2026)  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Concurrencia / Operaciones Clínicas / Persistencia / Redis  

---

## 1. Contexto y Planteamiento del Problema

Uno de los mayores dolores operacionales en clínicas veterinarias y centros de salud es la **sobreventa de horas médicas (Double-Booking)**:
* Dos pacientes intentan agendar simultáneamente a través de WhatsApp a las 11:00 AM para el mismo médico.
* La recepcionista humana en el CRM está agendando presencialmente mientras el asistente de IA ("Val") está confirmando la cita por chat en paralelo.

En la versión prototipo inicial (As-Is), se intentó resolver esto mediante **lógica procedural en código Python**: funciones de aplicación que consultaban disponibilidad, abrían transacciones manuales e intentaban coordinar bloqueos en memoria (`asyncio.Lock`). 

**Falla de diseño identificada:** Escribir código procedural en Python para controlar la concurrencia es **reinventar la rueda** de forma frágil. En un entorno con múltiples réplicas de la API o múltiples workers de fondo, los locks en memoria no funcionan y las validaciones a nivel de aplicación sufren de condiciones de carrera (*Check-Then-Act*).

---

## 2. Opciones Consideradas

| Alternativa | Descripción | Ventajas | Desventajas |
| :--- | :--- | :--- | :--- |
| **Opción A: Validación Procedural en Aplicación (As-Is Inicial)** | Código Python que consulta `if is_free(): insert()`. | Fácil de programar inicialmente. | **Anti-patrón de concurrencia:** Vulnerable a condiciones de carrera. Reinventa la lógica de exclusión mutua de forma defectuosa. |
| **Opción B: Bloqueo Pesimista Procedural (`SELECT ... FOR UPDATE` artesanal)** | Código en la API que adquiere locks de fila manualmente en cada transacción. | Garantiza exclusión si la transacción está bien aislada. | Sobrecarga de código transaccional procedural en Python; propenso a deadlocks si los recursos no se ordenan; mantiene conexiones abiertas a la base de datos innecesariamente. |
| **Opción C: Solución Estándar de la Industria — Restricción Declarativa en Base de Datos + Locks Temporales en Redis (Decisión To-Be v2.1)** | Delegar la garantía de no-solapamiento en el **motor de base de datos** (Restricciones Únicas / Exclusion Constraints) y gestionar la retención temporal de horas mediante **Redis con TTL**. | **Cero código procedural inventado:** La base de datos garantiza a nivel de kernel C++ que dos citas jamás colisionen. Redis gestiona la reserva temporal (5 min) de forma nativa con expiración automática. | Requiere modelar la restricción en el DDL y disponer de Redis. |

---

## 3. Decisión de Arquitectura: No Reinventar la Rueda

Se decide adoptar la **Opción C**, utilizando herramientas estándar de la industria que resuelven el problema de raíz:

```mermaid
flowchart TD
    Paciente["Paciente en WhatsApp / Recepcionista"] -->|1. Intención de Reserva| Worker["Worker / API"]
    
    Worker -->|2. Adquirir Hold Temporal (SET NX EX 300)| Redis[("Redis Distributed Lock<br/>(TTL 5 min)")]
    
    Redis -->|Hold Exitoso| Negociacion["Confirmación con Paciente"]
    Redis -->|Slot Ocupado| Rechazo["Ofrecer siguiente horario libre"]
    
    Negociacion -->|3. Confirmación Definitiva (INSERT)| DB[(Base de Datos Enterprise)]
    
    subgraph DB_Engine["Garantía a Nivel de Motor (Kernel)"]
        Constraint["CONSTRAINT unique_slot_professional<br/>UNIQUE (tenant_id, professional_id, slot_time)<br/>o EXCLUDE USING GIST"]
    end
    
    DB --> DB_Engine
```

### 1. Garantía Declarativa en Base de Datos (Integridad en el Motor)
En lugar de programar validaciones en Python, la regla de negocio se delega al motor relacional mediante DDL:
```sql
-- Restricción a nivel de base de datos que hace físicamente imposible el solapamiento:
ALTER TABLE appointments 
ADD CONSTRAINT uq_appointment_slot 
UNIQUE (tenant_id, professional_id, appointment_time);
```
* **Ventaja:** Si dos transacciones intentaran confirmar el mismo horario simultáneamente, el motor de base de datos rechaza la segunda de forma atómica e instantánea con un error de violación de integridad referencial. **Cero líneas de lógica de bloqueo procedural en la aplicación.**

### 2. Bloqueo Temporal Distribuido en Redis (Reserva Hold de 5 minutos)
Cuando un paciente en WhatsApp manifiesta la intención de agendar ("Me sirve a las 11:00 AM"), el slot no debe quedar libre para otros usuarios mientras el paciente confirma sus datos:
* En vez de escribir tablas de locks temporales en SQL o tareas cron de limpieza, se utiliza la primitiva nativa de Redis:
  ```
  SET slot:hold:{tenant_id}:{prof_id}:{timestamp} {session_id} NX EX 300
  ```
  * `NX`: Solo adquiere el lock si no existe (atómico).
  * `EX 300`: Expiración automática a los 5 minutos.
* **Resultado:** Si el paciente abandona la conversación o se corta la conexión, **Redis libera el horario automáticamente sin necesidad de scripts de limpieza ni tareas en segundo plano**.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Erradicación de Complejidad Accidental:** Se elimina por completo el código artesanal de bloqueos en Python (`_locks[key] = asyncio.Lock()`).
* **Resiliencia Distribuida:** Funciona de forma transparente tanto con un solo proceso como con 10 réplicas del worker distribuidas en la nube.
* **Rendimiento Máximo:** La verificación de disponibilidad se resuelve en la memoria ultrarrápida de Redis (<2 ms), reduciendo la carga de lectura en la base de datos relacional.

### Consecuencias Negativas / Mitigación:
* **Dependencia de Redis:** El ciclo de hold temporal depende de la disponibilidad de Redis. En caso de caída de Redis, el sistema puede operar en modo degradado apoyándose exclusivamente en la restricción declarativa de la base de datos.

---

## 5. Criterios de Aceptación y Plan de Verificación (Refactorización To-Be)

* **Validación de la Restricción Declarativa:** Una prueba automatizada debe intentar insertar directamente dos citas en el mismo horario y verificar que la base de datos arroje una violación de restricción única (`IntegrityError`).
* **Validación de Expiración en Redis (Hold TTL):** Comprobar que un slot bloqueado temporalmente quede liberado automáticamente tras transcurrir los 300 segundos sin requerir intervención de la aplicación.