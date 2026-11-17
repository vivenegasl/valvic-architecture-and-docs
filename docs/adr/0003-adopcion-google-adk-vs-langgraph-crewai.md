# ADR-0003: Adopción de Google ADK y Reemplazo de Frameworks Pesados (LangGraph / CrewAI)

> **Estado:** Aceptado  
> **Fecha:** Mayo 2026  
> **Decisores:** Víctor Venegas Lindzay  
> **Área:** Inteligencia Artificial / Orquestación Agéntica / Latencia  

---

## 1. Contexto y Planteamiento del Problema

El asistente conversacional de ValVic ("Val") es responsable de atender a pacientes en WhatsApp e Instagram: responder preguntas sobre servicios, precios y horarios, verificar la disponibilidad de médicos y bloquear citas en tiempo real mediante herramientas (*tool calling*).

En las primeras versiones prototípicas de ValVic se experimentó con frameworks populares del ecosistema Python para agentes:
* **CrewAI:** Orientado a colaboración multi-agente basada en roles.
* **LangChain / LangGraph:** Orientado a grafos de estado cíclicos y cadenas de razonamiento.

**Problemas Detectados en Producción:**
1. **Sobrecarga Excesiva de Dependencias:** Ambos frameworks arrastraban cientos de dependencias de terceros, generando conflictos constantes de versiones con Pydantic v2 y aumentando el tamaño de la imagen Docker en más de 800 MB.
2. **Latencia Inaceptable:** El overhead interno de abstracción de LangChain y los bucles de reflexión de CrewAI añadían entre **1.5 y 3.0 segundos de latencia adicional** sólo en la orquestación en memoria, antes de siquiera recibir los tokens del modelo de lenguaje.
3. **Complejidad de Depuración:** Rastrear el estado interno de un grafo en LangGraph ante una respuesta inesperada al usuario resultaba engorroso y opaco.

---

## 2. Opciones Consideradas

| Alternativa | Latencia Interna | Sobrecarga de Dependencias | Facilidad de Control de Tools |
| :--- | :---: | :---: | :--- |
| **Opción A: Mantener CrewAI / LangGraph** | Alta (+1500 ms) | Muy Alta (+150 paquetes) | Media (Opaco ante errores de ejecución de herramientas). |
| **Opción B: Implementación Artesanal con API Nativa de OpenAI/Anthropic** | Muy Baja (<50 ms) | Mínima (Sólo SDK oficial) | Alta, pero requiere programar manualmente el bucle de ejecución de tools y el historial. |
| **Opción C: Google ADK (Agent Development Kit) con Gemini 2.5 Flash (Elegida)** | Ultra Baja (<100 ms) | Ligera (Diseñado para producción) | Excelente: Soporte nativo para `LlmAgent`, inyección declarativa de `tools` tipadas y context-awareness multi-tenant. |

---

## 3. Decisión de Arquitectura

Se decide adoptar **Google ADK (Agent Development Kit)** como el motor estándar de orquestación conversacional de ValVic:

1. **Estructura Declarativa con `LlmAgent`:**
   En lugar de grafos complejos y cajas negras, se definen agentes atómicos especializados en `backend/api/adk/agents.py`:
   - `sales_agent`: Agente principal de atención y ventas con inyección de herramientas clínicas (`ALL_SALES_TOOLS`).
   - `prospector_agent`: Agente enfocado exclusivamente en la prospección B2B y redacción persuasiva en frío.
   - `copilot_agent`: Agente asistente para el operador humano en el CRM.

2. **Modelo Primario Gemini 2.5 Flash con Fallback a Claude 3.5 Sonnet:**
   - **Primario:** `Gemini 2.5 Flash` por su latencia ultra-baja (primer token en <350 ms), ventana de contexto masiva y bajo costo por millón de tokens.
   - **Fallback Dinámico:** En caso de error HTTP 429 (límite de cuota) o timeout en la API de Google, el `LLMManager` conmuta de forma transparente hacia Anthropic Claude sin interrumpir la sesión del paciente.

3. **Inyección Segura de Herramientas de Negocio (`tools.py`):**
   Las herramientas disponibles para el agente conversacional son funciones asíncronas fuertemente tipadas:
   * `consultar_catalogo(query)`: Búsqueda de servicios y precios con filtrado estricto por `tenant_id`.
   * `consultar_disponibilidad(fecha, profesional_id)`: Inspección de slots libres en agenda.
   * `agendar_cita(paciente_rut, slot_id)`: Invocación del motor transaccional con bloqueo pesimista.

---

## 4. Consecuencias y Compensaciones

### Consecuencias Positivas:
* **Reducción Drástica de Latencia Conversacional:** El tiempo total desde que el paciente envía un mensaje en WhatsApp hasta que recibe la respuesta con agenda bajó de **4.8 segundos a 1.2 segundos**.
* **Reducción de Deuda Técnica y Dependencias:** Se eliminaron más de 80 paquetes huérfanos del entorno Python, estabilizando el build y acelerando el tiempo de arranque de la API.
* **Control Absoluto del Estado Conversacional:** Google ADK permite serializar y persistir el estado de la conversación directamente en la base de datos relacional de forma determinista.

### Consecuencias Negativas / Limitaciones:
* **Acoplamiento al SDK de Google ADK:** Se asume dependencia de las convenciones de Google ADK. Sin embargo, dado que las herramientas de negocio (`tools.py`) son funciones puras de Python desacopladas del SDK, migrar a otra interfaz en el futuro requiere únicamente adaptar el wrapper de invocación.

---

## 5. Cumplimiento y Verificación

* **Pruebas de Tool Calling:** `backend/api/tests/test_adk_workflow.py` valida la correcta ejecución de herramientas de catálogo y agenda con mocking de respuestas de Gemini.
* **Tolerancia a Fallos:** La prueba de desconexión de red de Gemini activa exitosamente la respuesta mediante el fallback de Claude en menos de 1 segundo.