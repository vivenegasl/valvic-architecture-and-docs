# ADR-0004: Modelo de Aislamiento Multi-Tenant (Discriminador `tenant_id` vs Esquemas Separados)

> **Estado:** Arquitectura Objetivo (To-Be) Aprobada para Refactorización  
> **Fecha:** Mayo 2026 (Revisión Enterprise To-Be: Noviembre 2026)  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Base de Datos / Seguridad / Multi-Tenancy  

---

## 1. Contexto y Planteamiento del Problema

ValVic es un SaaS B2B donde múltiples clínicas veterinarias y centros médicos independientes comparten la infraestructura central. Dado que la plataforma maneja datos médicos sensibles (fichas clínicas, historiales de consultas, RUTs de pacientes y datos de contacto), **la separación lógica de datos entre clientes es un requerimiento crítico de seguridad y cumplimiento legal** (Ley N° 19.628 de Protección de la Vida Privada en Chile).

En la versión prototipo inicial (As-Is), el aislamiento se intentó resolver mediante **disciplina manual de código**: cada desarrollador debía recordar incluir manualmente la cláusula `.where(Model.tenant_id == tenant_id)` en cada una de las consultas en Python.

**Riesgo de diseño:** Confiar en que los desarrolladores o proveedores recuerden escribir un `where` a mano en 100% de las funciones es una receta para la fuga de datos por error humano.

---

## 2. Opciones Consideradas

| Modelo Multi-Tenant | Descripción | Ventajas | Desventajas |
| :--- | :--- | :--- | :--- |
| **Opción A: Base de Datos Separada por Tenant** | Cada clínica tiene su propia instancia de base de datos física o lógica. | Aislamiento físico absoluto; cero riesgo de fuga relacional. | Inviable a escala en costos de infraestructura; complejidad de administración de migraciones insostenible. |
| **Opción B: Filtro Manual en Código de Aplicación (As-Is Inicial)** | Escribir a mano `WHERE tenant_id = :id` en cada consulta de la aplicación. | Rápido de prototipar. | **Riesgo crítico de error humano:** Si un desarrollador olvida el filtro en una consulta, expone datos de todos los clientes. |
| **Opción C: Esquema Compartido con Row-Level Security (RLS) y Filtros Globales de Sesión (Decisión To-Be v2.1)** | Todas las tablas tienen una columna `tenant_id` indexada, pero el aislamiento se delega a las herramientas del motor: **Row-Level Security (RLS) nativo en BD** y **filtros globales automáticos en el ORM**. | **Seguridad por diseño (No reinventar la rueda):** El motor de base de datos o el ORM inyectan el filtro de forma automática e invisible. Es físicamente imposible que una consulta acceda a datos de otro tenant, incluso si el desarrollador no pone el `where`. | Requiere configurar la política RLS en el DDL de la base de datos. |

---

## 3. Decisión de Arquitectura: Seguridad por Diseño

Se decide implementar la **Opción C: Row-Level Security (RLS) y Filtros Automáticos de Sesión**:

1. **Resolución Perimetral del Tenant (`tenant_resolver.py`):**
   * El `tenant_id` se extrae en el gateway a partir del token JWT de sesión verificado o del identificador telefónico del webhook de Meta.

2. **Inyección Automática a Nivel de Motor (Row-Level Security / RLS):**
   En lugar de escribir `WHERE tenant_id` en Python a mano, se establece la política nativa en la base de datos:
   ```sql
   -- Política de seguridad a nivel de motor:
   ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation_policy ON patients
     USING (tenant_id = CURRENT_SETTING('app.current_tenant_id'));
   ```
   * Al abrir la conexión, el middleware ejecuta una sola vez `SET LOCAL app.current_tenant_id = :tenant_id`. A partir de ese milisegundo, **cualquier SELECT, UPDATE o DELETE ejecutado en esa sesión está forzosamente acotado a ese tenant a nivel de kernel de base de datos**, garantizando inmunidad ante errores de código.

3. **Restricción de Integridad Referencial:**
   Las claves foráneas compuestas garantizan que un registro secundario (ej. una cita) pertenezca forzosamente al mismo `tenant_id` que el registro primario, impidiendo asignaciones cruzadas.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Erradicación del Error Humano:** Cero riesgo de que un desarrollador o proveedor olvide filtrar un query. La seguridad reside en el motor, no en la memoria del programador.
* **Simplicidad de Código:** Las funciones de negocio se concentran en la lógica clínica sin ensuciar cada línea con validaciones repetitivas de tenant.
* **Escalabilidad y Costo Fijo:** Cientos de organizaciones operan de forma segura dentro de la misma instancia de base de datos.

### Consecuencias Negativas / Mitigación:
* **Contexto de Sesión Obligatorio:** Requiere que el middleware de conexión siempre inicialice la variable de sesión del tenant antes de despachar consultas, lo cual se valida mediante pruebas automáticas en el pipeline de CI/CD.

---

## 5. Criterios de Aceptación y Plan de Verificación (Refactorización To-Be)

* **Pruebas de Penetración de Tenant (DoD):** En las pruebas de integración, se debe ejecutar una consulta `SELECT * FROM patients` sin cláusula WHERE explícita y certificar que la base de datos devuelva exclusivamente los registros del tenant contextualizado en la sesión.