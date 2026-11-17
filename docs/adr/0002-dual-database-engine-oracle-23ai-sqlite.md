# ADR-0002: Paridad Dev/Prod Estricta — Deprecación de SQLite y Adopción de Contenedores Idénticos (Twelve-Factor App #10)

> **Estado:** Arquitectura Objetivo (To-Be) Aprobada para Refactorización  
> **Fecha:** Mayo 2026 (Revisión Enterprise To-Be: Noviembre 2026)  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Persistencia / Base de Datos / DevOps / Testing  

---

## 1. Contexto y Planteamiento del Problema

En las primeras fases de ValVic se implementó una estrategia de persistencia dual:
* **Producción:** Oracle Database 23ai Always Free (con soporte nativo para AI Vector Search y transacciones pesimistas).
* **Desarrollo y Tests Unitarios:** SQLite local y en memoria (`sqlite+aiosqlite:///:memory:`) con el fin de ejecutar las pruebas de Pytest de forma ultrarrápida (en menos de 10 segundos) sin dependencias de red.

Aunque esta estrategia permitió iterar con rapidez en los primeros días, **en la práctica se convirtió en una de las fuentes principales de deuda técnica y fragilidad del sistema**.

---

## 2. Los Problemas Reales del Anti-Patrón SQLite en Desarrollo

La disparidad de motores violó directamente el **Factor X de Twelve-Factor App (Dev/Prod Parity)**, provocando fallas graves:

1. **Incompatibilidad de Tipos Vectoriales:** SQLite no tiene soporte nativo para el tipo `VECTOR` de Oracle. Las pruebas de búsqueda semántica de información clínica se ejecutaban sobre *mocks* artificiales, dejando sin probar el motor real de embeddings en desarrollo.
2. **Mecánica de Bloqueos Opuesta:** SQLite bloquea el archivo de base de datos a nivel global (`database is locked`), mientras que Oracle bloquea a nivel de fila (`SELECT ... FOR UPDATE`). Probar concurrencia sobre SQLite daba falsos positivos y falsos negativos.
3. **Invalidez de las Migraciones DDL:** Los 24 archivos DDL diseñados para Oracle nunca se ejecutaban en los tests de SQLite (`SQLModel.metadata.create_all()` autogeneraba un esquema plano), lo que impedía validar si las claves foráneas compuestas, secuencias y restricciones de integridad funcionaban en producción.
4. **Acumulación de Deuda y Scripts Parche:** Para hacer pasar los tests en SQLite sin romper Oracle, se generaron múltiples scripts de rescate (`fix_all_tests.py`, `convert_tests_v3.py`), aumentando la complejidad cognitiva del repositorio.

---

## 3. Opciones Consideradas

| Alternativa | Ventajas | Desventajas |
| :--- | :--- | :--- |
| **Opción A: Mantener Motor Dual (SQLite / Oracle)** | Tests en memoria rápidos (8 segundos). | **Anti-patrón de DevOps comprobado:** Falsa sensación de seguridad, deuda técnica en scripts de prueba y riesgo inaceptable de errores no detectados en producción. |
| **Opción B: Conectar Tests Locales a Oracle Cloud Remoto** | Motor idéntico. | Latencia de red de 300 ms por query, dependencia de conexión a internet y riesgo de contaminar datos de prueba en la nube. |
| **Opción C: Paridad Dev/Prod Estricta con Contenedores Docker Locales (Decisión Enterprise v2.1)** | **100% de paridad:** El mismo motor relacional (Oracle 23ai Free o PostgreSQL 16) se ejecuta en un contenedor local para desarrollo y CI/CD. | Requiere que el entorno de desarrollo tenga Docker instalado y consuma ~2 GB de RAM. |

---

## 4. Decisión de Arquitectura y Lección de Liderazgo

Se decide **deprecar definitivamente SQLite** y adoptar la **Opción C: Paridad Dev/Prod Estricta mediante Contenedores Docker (Twelve-Factor App Factor X)**:

1. **Un Solo Motor, Una Sola Verdad:**
   * Tanto en desarrollo local como en el pipeline de CI/CD (GitHub Actions) y en los servidores de producción, la aplicación se conecta **exactamente al mismo motor relacional**.
   * Se elimina la bifurcación condicional en `engine.py` (`if "pytest" in sys.modules`). El código de conexión es universal.

2. **Ejecución de Migraciones Reales en cada Test:**
   * La suite de pruebas automatizadas ya no utiliza `create_all()`. Ejecuta secuencialmente los **24 scripts DDL reales** sobre el contenedor de pruebas efímero (vía Testcontainers o Docker Compose), garantizando que cualquier error de sintaxis, índice o restricción se detecte antes de fusionar una rama en Git.

3. **Pruebas de Concurrencia e Invariantes Reales:**
   * Las pruebas de estrés del motor de agenda prueban el `SELECT FOR UPDATE` sobre el motor real, asegurando empíricamente que la invariante de negocio **Zero Double-Booking** se cumpla en condiciones idénticas a producción.

---

## 5. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Cero Errores Sorpresa en Producción:** Garantiza paridad absoluta entre lo que se prueba en local/CI y lo que corre en producción, eliminando inconsistencias de dialectos o tipos vectoriales.
* **Purga de Deuda Técnica:** Permite erradicar los scripts parches (`fix_all_tests*`) y bifurcaciones en el ORM, simplificando radicalmente la base de código.
* **Validación de AI Vector Search en Desarrollo:** Los desarrolladores podrán probar la similitud de coseno y los embeddings clínicos directamente en sus máquinas.

### Consecuencias Negativas / Mitigación:
* **Sobrecarga de Tiempo en Tests (Estimada):** Se proyecta que levantar un contenedor local de base de datos añada ~15-20 segundos al inicio de la suite de pruebas. Este costo es despreciable comparado con la confiabilidad matemática del software entregado.

---

## 6. Criterios de Aceptación y Plan de Verificación (Refactorización To-Be)

* **Validación de Migraciones en CI/CD:** El pipeline de integración continua deberá ejecutar secuencialmente los 24 archivos DDL sobre el contenedor efímero antes de correr la suite de pruebas.
* **Eliminación de Mocks Vectoriales:** Las pruebas de búsqueda semántica deberán ejecutarse directamente contra las funciones vectoriales del motor sin requerir emulación en memoria.