# ADR-0004: Modelo de Aislamiento Multi-Tenant (Discriminador `tenant_id` vs Esquemas Separados)

> **Estado:** Aceptado  
> **Fecha:** Mayo 2026  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Base de Datos / Seguridad / Multi-Tenancy  

---

## 1. Contexto y Planteamiento del Problema

ValVic es un SaaS B2B donde múltiples clínicas veterinarias y centros médicos independientes comparten la infraestructura central. Dado que la plataforma maneja datos médicos sensibles (fichas clínicas, historiales de consultas, RUTs de pacientes y datos de contacto), **la separación lógica de datos entre clientes es un requerimiento crítico de seguridad y cumplimiento legal** (Ley N° 19.628 de Protección de la Vida Privada en Chile).

Un fallo en el aislamiento que permita que la Clínica A visualice las conversaciones, precios o pacientes de la Clínica B significaría la destrucción inmediata de la reputación comercial y demandas legales severas.

---

## 2. Opciones Consideradas

| Modelo Multi-Tenant | Descripción | Ventajas | Desventajas |
| :--- | :--- | :--- | :--- |
| **Opción A: Base de Datos Separada por Tenant** | Cada clínica tiene su propia instancia de base de datos física o lógica. | Aislamiento físico absoluto; cero riesgo de fuga relacional. | Inviable en la capa Always Free de OCI (requeriría decenas de instancias de DB); costo de administración de migraciones insostenible. |
| **Opción B: Esquema de Base de Datos Separado por Tenant** | Una sola base de datos, pero cada clínica tiene su propio schema (ej. `schema_clinica_spotvet`, `schema_clinica_conzoolvet`). | Buen aislamiento; comandos DDL aislados. | Dificultad extrema en migraciones masivas (ejecutar 24 scripts por cada nuevo cliente); complejidad para realizar analítica global de la plataforma. |
| **Opción C: Esquema Compartido con Discriminador `tenant_id` y Filtro Obligatorio en ORM (Elegida)** | Todas las tablas tienen una columna `tenant_id` indexada. Cada consulta de la aplicación filtra obligatoriamente por el tenant autenticado. | Eficiencia máxima de recursos; compatible con Always Free; migraciones DDL únicas y centralizadas; ideal para analítica agregada. | Riesgo de fuga si un desarrollador olvida incluir `WHERE tenant_id = :id` en una consulta manual. |

---

## 3. Decisión de Arquitectura

Se decide implementar la **Opción C: Esquema Compartido con Columna Discriminadora `tenant_id` Reforzada por Software**:

1. **Resolución Perimetral del Tenant (`tenant_resolver.py`):**
   * En peticiones entrantes de Meta (WhatsApp): El `tenant_id` se resuelve de forma determinista buscando el `phone_number_id` o el `WABA_ID` en la tabla maestra `tenants`.
   * En peticiones de usuarios del CRM: El `tenant_id` se extrae del token JWT de sesión verificado por Clerk en los headers HTTP.

2. **Gobernanza a Nivel de ORM (SQLModel):**
   * Todos los modelos de dominio (`Patient`, `Appointment`, `Conversation`, `CatalogItem`, `Lead`) heredan de una clase base que incluye:
     ```python
     tenant_id: str = Field(index=True, nullable=False)
     ```
   * Toda función de acceso a datos en la capa de servicios (`services/` y `tools.py`) recibe obligatoriamente el `tenant_id` como primer argumento:
     ```python
     statement = select(Patient).where(
         Patient.tenant_id == tenant_id,
         Patient.id == patient_id
     )
     ```

3. **Restricción de Integridad Referencial:**
   Las claves foráneas compuestas garantizan que un registro secundario (ej. una cita) pertenezca forzosamente al mismo `tenant_id` que el registro primario (el paciente y el médico), impidiendo asignaciones cruzadas por error lógico.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Escalabilidad Operativa con Costo Fijo $0:** El sistema puede atender a cientos de clínicas dentro de la misma instancia de Oracle Database 23ai sin generar sobrecarga administrativa de schemas.
* **Migraciones Rápidas y Atómicas:** Los 24 archivos DDL se ejecutan una sola vez para toda la plataforma, simplificando radicalmente el proceso de despliegue y CI/CD.
* **Facilidad para Reportería Agregada:** Permite a los administradores de ValVic generar métricas globales de adopción (ej. volumen total de mensajes procesados, tasa de conversión promedio) mediante simples agrupaciones por `tenant_id`.

### Consecuencias Negativas / Mitigación:
* **Riesgo de Omisión en Consultas:** Para mitigar el riesgo de que una nueva función olvide filtrar por `tenant_id`, se implementó una suite de pruebas de seguridad perimetral (`test_multi_tenant_isolation.py`) que ejecuta ataques simulados entre tenants intentando acceder a IDs de otras organizaciones, garantizando respuestas `404 Not Found`.

---

## 5. Cumplimiento y Verificación

* **Suite de Pruebas de Aislamiento:** `pytest backend/api/tests/test_multi_tenant_isolation.py` verifica que un usuario autenticado en `tenant_A` reciba error al intentar leer o modificar recursos pertenecientes a `tenant_B`.