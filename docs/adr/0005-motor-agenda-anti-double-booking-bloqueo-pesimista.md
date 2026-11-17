# ADR-0005: Motor de Agenda Anti-Double-Booking Mediante Bloqueo Pesimista (`FOR UPDATE`)

> **Estado:** Aceptado  
> **Fecha:** Mayo 2026  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Backend / Transacciones / Concurrencia / Operaciones Clínicas  

---

## 1. Contexto y Planteamiento del Problema

Uno de los mayores dolores operacionales en las clínicas de salud y veterinarias es la **sobreventa de horas médicas (Double-Booking)**:
* Dos pacientes intentan agendar simultáneamente a través de WhatsApp a las 11:00 AM para el mismo médico o box de atención.
* La recepcionista humana en el CRM está agendando presencialmente mientras el asistente de IA ("Val") está confirmando la cita por chat en paralelo.

Si el software no controla la concurrencia a nivel de base de datos con aislamiento transaccional estricto, ambos pacientes reciben confirmación de la cita, produciendo un conflicto crítico en sala de espera, demoras asistenciales y quejas formales de pacientes.

---

## 2. Opciones Consideradas

| Mecanismo de Control | Funcionamiento | Ventajas | Desventajas |
| :--- | :--- | :--- | :--- |
| **Opción A: Validación en Memoria (Application-Level Check)** | La aplicación consulta si el slot está libre y, si lo está, procede a insertar. | Sencillo de programar. | **Inseguro bajo concurrencia (Race Condition / Check-Then-Act):** Dos peticiones concurrentes leen simultáneamente que el slot está "libre" y ambas ejecutan el INSERT. |
| **Opción B: Bloqueo Optimista (Optimistic Locking vía `version_id`)** | Se lee el registro con un número de versión. Al guardar, se verifica que la versión no haya cambiado. Si cambió, se aborta y se reintenta. | No bloquea la base de datos; alta performance en escenarios de baja contención. | Experiencia de usuario deficiente en WhatsApp: si la reserva falla en el commit, el paciente debe esperar a que el bot reintente o le pida otra hora tras haberle dicho que sí. |
| **Opción C: Bloqueo Pesimista Transaccional (`SELECT ... FOR UPDATE`) con Transacción Atómica (Elegida)** | La transacción adquiere un cerrojo exclusivo sobre el recurso de disponibilidad médica antes de confirmar. Cualquier otra petición debe esperar o ser rechazada de inmediato. | **Garantía matemática absoluta:** Es físicamente imposible el solapamiento o la sobreventa de citas. Consistencia ACID inmediata. | Bloquea temporalmente el registro durante la transacción (requiere transacciones ultracortas <100ms). |

---

## 3. Decisión de Arquitectura

Se decide implementar la **Opción C: Bloqueo Pesimista Atómico a Nivel de Base de Datos**:

1. **Procedimiento Atómico de Reserva (`agenda_service.py`):**
   Dentro de una transacción aislada gestionada por SQLAlchemy/SQLModel:
   ```python
   async with db.transaction() as session:
       # 1. Adquirir bloqueo exclusivo sobre el profesional y el horario
       query = (
           select(AppointmentSlot)
           .where(
               AppointmentSlot.tenant_id == tenant_id,
               AppointmentSlot.professional_id == professional_id,
               AppointmentSlot.slot_time == requested_time,
               AppointmentSlot.status == "AVAILABLE"
           )
           .with_for_update()  # <-- Bloqueo pesimista exclusivo
       )
       result = await session.exec(query)
       slot = result.first()
       
       if not slot:
           raise SlotAlreadyBookedException("El horario ya fue reservado.")
           
       # 2. Mutar estado de forma atómica en el mismo commit
       slot.status = "BOOKED"
       slot.patient_id = patient.id
       slot.booked_via = "WHATSAPP_AI"
       
       await session.commit()
   ```

2. **Ventana Transaccional Ultracorta:**
   * La llamada al modelo de lenguaje (LLM) y la redacción del mensaje empático se realizan **antes** o **después** de abrir la transacción de base de datos.
   * La transacción con cerrojo pesimista dura únicamente entre **10 y 25 milisegundos**, evitando bloqueos prolongados o acumulación de hilos en espera.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Invariante de Negocio Garantizada:** Cero posibilidad matemática de sobreventa de horas (Zero Double-Booking) tanto entre múltiples pacientes concurrentes en WhatsApp como entre el bot de IA y la secretaria humana en el panel web.
* **Integridad Referencial Absoluta:** La confirmación emitida al paciente por WhatsApp solo se envía una vez que el commit en la base de datos se completó con éxito.

### Consecuencias Negativas / Limitaciones:
* **Manejo de Reintentos:** Si dos transacciones intentan bloquear el mismo slot exacto al mismo milisegundo, la segunda transacción esperará brevemente y luego recibirá una excepción `SlotAlreadyBookedException`. El asistente conversacional captura esta excepción y le responde amablemente al segundo usuario: *"Ese horario acaba de ser tomado hace un instante, ¿te acomodaría 30 minutos más tarde?"*.

---

## 5. Cumplimiento y Verificación

* **Test de Concurrencia:** `pytest backend/api/tests/test_concurrency_booking.py` lanza 20 corrutinas concurrentes intentando reservar el mismo horario; exactamente 1 corrutina tiene éxito (`BOOKED`) y las otras 19 son rechazadas de forma limpia con código controlado.