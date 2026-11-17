# BDO Chile — Guía Maestra de Defensa Técnica: Project Leader TI

> **Candidato:** Víctor Venegas Lindzay  
> **Cargo:** Project Leader TI (Área TI Exclusiva, BDO Chile — Las Condes)  
> **Fecha de Entrevista:** Miércoles  
> **Objetivo:** Demostrar liderazgo de proyectos, rigor de ingeniería, gobernanza de proveedores y dominio conceptual del ecosistema corporativo.

---

## 1. El Posicionamiento Estratégico (El "Elevator Pitch" de 90 Segundos)

Cuando te digan: *"Víctor, cuéntanos sobre ti, tu trayectoria y por qué te interesa BDO"*, tu respuesta debe sonar así:

> *"Soy profesional de TI con una combinación que considero estratégica para este cargo: una sólida base de ciencias de la computación cursada hasta cuarto año de Ingeniería Civil Informática en la PUCV, complementada con la formación práctica y de ejecución de Duoc UC, sumado a certificaciones internacionales en Scrum Master, Product Owner, DevOps, Ciberseguridad y Gestión de Riesgo en IA.*
>
> *En los últimos años he liderado el ciclo de vida completo de productos digitales en ValVic, donde gestioné la arquitectura, la integración de APIs críticas de mensajería (Meta) y servicios en la nube. Esa experiencia como Founder y Technical Product Manager me enseñó de primera mano los dolores reales de la deuda técnica, la importancia de definir contratos de software antes de tirar código y cómo gobernar la entrega de valor.*
>
> *Además, mi paso como Consejero de Carrera en Duoc y Delegado en la PUCV me entrenó en el manejo de stakeholders complejos, negociación de requerimientos y resolución de conflictos ante autoridades. Me entusiasma BDO porque el rol de Project Leader TI requiere exactamente eso: alguien con la solvencia técnica para que un proveedor externo no le venda humo, pero con la empatía y habilidades blandas para acompañar a las áreas de negocio en la gestión del cambio y la adopción tecnológica."*

---

## 2. La Historia de ValVic: Cómo Convertir el "Monolito" en tu Mayor Fortaleza

Uno de los mayores temores al ser evaluado técnicamente es hablar de los proyectos que fallaron o se volvieron difíciles. **En una entrevista para Project Leader, hablar de una falla técnica con madurez analítica te posiciona inmediatamente como Senior.**

### La Narrativa STAR del Monolito:
* **Situación:** En ValVic nos propusimos crear una solución integral para clínicas (captación de prospectos, CRM, atención por WhatsApp con IA y agendamiento).
* **Tarea:** El proyecto creció aceleradamente hasta alcanzar 28 routers REST, 44 tablas en base de datos y múltiples tareas de fondo en memoria.
* **El Quiebre (Confesión Técnica):** *"Al priorizar la velocidad funcional inicial sin haber congelado previamente los contratos de interfaz (OpenAPI), los límites de contexto (Bounded Contexts) y los invariantes de negocio, el sistema se convirtió en un monolito acoplado. Las tareas de fondo compartían el mismo proceso de la API web, y cada cambio generaba deuda técnica acumulada."*
* **La Decisión de Liderazgo:** *"En lugar de seguir parchando el código a ciegas, tomé la decisión ejecutiva de frenar el desarrollo desordenado. Realicé una auditoría de deuda técnica formal, diseñé un Documento de Arquitectura de Software (SAD), redacté Registros de Decisiones de Arquitectura (ADRs) y apliqué el estándar 'Documentation-First' que hoy también rige mis proyectos de ingeniería (como StockPilot). Desacoplamos la arquitectura conceptualmente hacia un Ingress Stateless con broker Redis y workers independientes, erradicando la persistencia dual SQLite/Oracle por paridad estricta con Docker."*
* **El Aprendizaje para BDO:** *"Esta experiencia me grabó a fuego una lección que aplico directamente al gobernar proveedores: **si tú no le exiges a un proveedor contratos de datos estrictos, criterios de aceptación Gherkin y documentación previa, el proveedor te va a entregar un monolito frágil que colapsará en mantenimiento**."*

### 2.1 Cómo Explicar el Estado Actual: As-Is (Código) vs. To-Be (Documentación)

Si un evaluador técnico te pregunta: *"¿Esta arquitectura v2.1 ya está completamente implementada en el código fuente de ValVic?"*, respondes con total aplomo y autoridad profesional:

> *"El repositorio de código fuente refleja actualmente el estado **As-Is** de la plataforma: el prototipo funcional inicial construido en modo Code-First (el backend FastAPI monolítico con la cola SQL y la persistencia dual). Fue precisamente ese código funcionando el que me permitió validar el modelo de negocio con clínicas reales y descubrir en la práctica los dolores del acoplamiento y la deuda técnica.*
>
> *La documentación formal que ven hoy (el SAD v2.1 y los 6 ADRs) representa la **Arquitectura Objetivo (To-Be)**. Detuve deliberadamente el desarrollo a ciegas para aplicar la disciplina **Documentation-First**: primero se formaliza la arquitectura, los límites de contexto y las decisiones técnicas, y sobre ese plano se ejecuta la refactorización. Así es exactamente como opero: no permito que se tire una línea de código sin una especificación arquitectónica aprobada."*

---

## 3. El Puente Conceptual hacia el Ecosistema Microsoft de BDO

En la oferta, BDO menciona: **Microsoft 365, Azure, Power BI, Dataverse y automatizaciones**. No necesitas fingir que tienes 10 años programando en C#; necesitas demostrar que **entiendes la arquitectura subyacente y sabes gobernar estas herramientas**:

| Concepto en tu Experiencia (OCI / Python / SQL) | Equivalente en el Ecosistema Microsoft de BDO | Cómo Defenderlo en la Entrevista |
| :--- | :--- | :--- |
| **Bases de Datos Relacionales & SQLModel** | **Dataverse (Common Data Model - CDM)** | *"Dataverse es esencialmente una base de datos relacional orientada a entidades empresariales con control de acceso basado en roles (RBAC) y lógica de negocio integrada. Modelar entidades, relaciones 1:N, N:M y reglas de validación en Dataverse sigue exactamente los mismos principios de integridad referencial que aplico en SQL."* |
| **Modelado de Datos & Pipelines ETL** | **Power BI (Modelos Semánticos & DAX)** | *"Un buen reporte en Power BI depende en un 80% de cómo está estructurado el modelo de datos subyacente (esquema en estrella con tablas de hechos y dimensiones). Mi experiencia diseñando tablas analíticas y métricas de conversión me permite estructurar modelos de Power BI limpios para los socios de BDO, traduciendo datos operacionales en KPIs estratégicos."* |
| **Infraestructura Cloud (OCI / Linux / Docker)** | **Microsoft Azure (VNets, App Services, VMs)** | *"Los fundamentos de computación en la nube son universales: subredes virtuales, seguridad perimetral, gestión de identidades y secretos. En OCI gestiono VMs ARM, proxies SSL y almacenamiento seguro; en Azure los conceptos equivalentes son Azure Virtual Machines, App Services, Key Vault y Azure SQL."* |
| **Webhooks y Workers Asíncronos** | **Power Automate / Azure Logic Apps** | *"Para automatizar rutinas operacionales y reportes de gestión entre Microsoft 365 (Teams, Outlook, SharePoint) y Dataverse, Power Automate y Logic Apps permiten orquestar flujos basados en eventos con conectores estándar, reduciendo el trabajo manual de los equipos internos."* |

---

## 4. Banco de Respuestas a Preguntas de Fuego (STAR Framework)

### Pregunta 1: *"¿Cómo gestionas a un proveedor tecnológico que está mostrando retrasos en los plazos comprometidos?"*
> **Respuesta:**
> *"Aplico una gobernanza proactiva basada en alertas tempranas, no reactiva a fin de mes. En los comités operativos semanales con el proveedor revisamos el Burndown Chart del Sprint y los ítems bloqueados. 
> 
> Si detecto que la velocidad del proveedor cayó y pone en riesgo el hito contractual:
> 1. Primero analizo si el bloqueo se debe a una dependencia interna nuestra (ej. falta de definición de un requerimiento o demora en accesos de red) para despejarlo de inmediato.
> 2. Si la causa es del proveedor, reviso los Criterios de Aceptación y el alcance: **jamás negocio hacia abajo la seguridad ni la calidad arquitectónica**, pero podemos despriorizar funcionalidades accesorias mediante técnicas MoSCoW para proteger la fecha del hito crítico.
> 3. Si el proveedor reitera en desviaciones, formalizo la alerta según la matriz de SLAs del contrato, solicitando un plan de mitigación formal y la asignación de recursos Senior adicionales sin costo para BDO."*

---

### Pregunta 2: *"¿Cómo levantas y documentas requerimientos cuando los usuarios internos de BDO no saben exactamente lo que necesitan?"*
> **Respuesta:**
> *"Es el escenario más común y donde más valor aporta un Project Leader. Los usuarios de negocio suelen expresar soluciones apresuradas ('necesito un botón aquí') en lugar de explicar su dolor operacional real.
>
> Mi proceso consiste en:
> 1. **Entrevistas de Descubrimiento Operacional:** Les pido que me muestren su día a día: dónde pierden tiempo, qué planillas de Excel usan como parche y qué decisiones no pueden tomar por falta de información.
> 2. **Traducción a Historias de Usuario con Criterios Gherkin:** Transformo esa necesidad en una historia formal: 'Como [Rol], Quiero [Acción] Para [Beneficio de Negocio]', y agrego escenarios `Dado-Cuando-Entonces`. Esto elimina las interpretaciones subjetivas.
> 3. **Prototipado Rápido y Validación Visual:** Antes de mandar a construir la solución al proveedor, valido con el usuario mediante un mockup o un flujo visual en herramientas como Figma o Draw.io. Cuando el usuario ve el flujo, refina su requerimiento antes de que hayamos gastado horas de desarrollo."*

---

### Pregunta 3: *"¿Cómo manejas la gestión del cambio cuando implementas un nuevo sistema y el personal interno se resiste a usarlo?"*
> **Respuesta:**
> *"La resistencia al cambio casi nunca es por terquedad; es por miedo a perder eficiencia en su rutina diaria o por no entender el beneficio personal del nuevo sistema. Mi experiencia como Consejero de Carrera en Duoc UC me enseñó a liderar procesos de cambio con personas escépticas.
>
> Mi estrategia se basa en tres pilares:
> 1. **Identificar y empoderar a 'Champions' o Usuarios Clave:** Involucro desde el día uno a los líderes informales de cada área en el diseño y las pruebas UAT. Si ellos sienten que el sistema es de ellos, se convierten en los principales promotores entre sus pares.
> 2. **Capacitación por Rol y Material de Soporte Autogestionable:** No hago capacitaciones masivas de 3 horas donde nadie retiene nada; diseño cápsulas cortas de 5 minutos orientadas a tareas específicas y guías visuales paso a paso.
> 3. **Medición de Adopción Temprana:** Monitoreo métricas de uso durante las primeras 4 semanas post-lanzamiento para identificar qué departamentos están estancados y ofrecerles apoyo operativo personalizado."*

---

### Pregunta 4: *"Tienes muchas certificaciones: Scrum, DevOps, Ciberseguridad, IA... ¿Cómo se articulan entre sí en el día a día de un proyecto?"*
> **Respuesta:**
> *"Las veo como una caja de herramientas integrada que cubre las cuatro esquinas de un proyecto tecnológico:
> 
> * **Scrum (SPOPC / SMPC)** me da el marco para gestionar las expectativas de los stakeholders, ordenar las prioridades del negocio en el Backlog y asegurar que el equipo trabaje en iteraciones ágiles de entrega de valor.
> * **DevOps (DAPC)** me permite supervisar la calidad de la entrega técnica: exigir que los proveedores tengan pipelines de CI/CD, pruebas automatizadas y ambientes de staging para que los pases a producción no sean traumáticos.
> * **Ciberseguridad (LCSPC)** me aporta la mirada defensiva: asegurar que cualquier nueva herramienta cumpla con la protección de datos corporativos, gestión de identidades y privilegios mínimos.
> * **Gestión de Riesgo en IA (AIRMPC)** me permite evaluar qué procesos de BDO son candidatos viables para automatización o IA sin exponer información confidencial de clientes ni caer en riesgos de cumplimiento.
>
> En resumen: no son diplomas aislados, son los estándares con los que tomo decisiones todos los días."*

---

### Pregunta 5: *"Cuéntanos sobre tu formación universitaria: PUCV y Duoc UC."*
> **Respuesta:**
> *"Tengo una trayectoria dual que considero muy complementaria:
> 
> En la **PUCV cursé hasta cuarto año de Ingeniería Civil Informática**, lo que me entregó los cimientos duros de la ingeniería de software: matemáticas discretas, algoritmos, sistemas operativos, redes y la teoría profunda de concurrencia y bases de datos. 
> 
> Posteriormente en **Duoc UC (Analista Programador)** enfoqué mi formación hacia la ejecución práctica, el desarrollo moderno de aplicaciones, arquitecturas ágiles y la disciplina de culminación académica formal. 
> 
> Esta combinación me permite tener el rigor analítico de un ingeniero para discutir de igual a igual con los arquitectos técnicos de los proveedores, y al mismo tiempo el sentido práctico de entrega y resolución expedita de problemas que exige la operación diaria."*

---

## 5. Preguntas Clave que TÚ Debes Hacer al Cierre de la Entrevista

Cuando te digan: *"Víctor, ¿tienes alguna pregunta para nosotros?"*, hacer preguntas estratégicas demuestra que ya te estás visualizando en el cargo:

1. *"En los primeros 90 días, ¿cuál sería el proyecto o implementación tecnológica prioritaria donde esperan que el Project Leader genere el mayor impacto?"*
2. *"Actualmente, ¿cómo está compuesta la relación entre desarrollo interno y proveedores externos en BDO Chile? ¿Cuáles son los principales desafíos que han experimentado con los proveedores tecnológicos?"*
3. *"En cuanto a la reportería de gestión y analítica de datos, ¿tienen ya modelos de Power BI centralizados o uno de los objetivos del cargo será diseñar y estandarizar estos cuadros de mando para los socios?"*

---
*Esta guía sintetiza tu soberanía técnica, tu madurez frente a los errores del pasado y tu idoneidad exacta para el cargo en BDO Chile.*