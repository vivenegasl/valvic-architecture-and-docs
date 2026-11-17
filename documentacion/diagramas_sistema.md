# ValVic — Colección de Diagramas de Arquitectura y Flujos del Sistema

> **Guía Visual y Técnica para Presentaciones, Evaluaciones de Arquitectura y Onboarding.**  
> *Versión 2.0 — Ecosistema Multi-Tenant de Automatización Conversacional e IA.*

---

## Índice de Diagramas

1. [Arquitectura Global de Componentes (C4 Nivel 2)](#1-arquitectura-global-de-componentes-c4-nivel-2)
2. [Flujo End-to-End de Mensajería WhatsApp & Orquestación IA](#2-flujo-end-to-end-de-mensajería-whatsapp--orquestación-ia)
3. [Embudo Comercial Autónomo (Outbound + Inbound + Cierre)](#3-embudo-comercial-autónomo-outbound--inbound--cierre)
4. [Dual Database Engine y Aislamiento Multi-Tenant](#4-dual-database-engine-y-aislamiento-multi-tenant)
5. [Client Factory — Pipeline de Generación de Sitios Web](#5-client-factory--pipeline-de-generación-de-sitios-web)
6. [CRM Multitenant, Co-pilotaje Humano y Botón Dorado](#6-crm-multitenant-co-pilotaje-humano-y-botón-dorado)
7. [Ciclo de Vida de Workers Asíncronos (FastAPI Lifespan)](#7-ciclo-de-vida-de-workers-asíncronos-fastapi-lifespan)

---

## 1. Arquitectura Global de Componentes (C4 Nivel 2)

Este diagrama representa cómo interactúan las aplicaciones del frontend, la API backend, los servicios externos de IA y mensajería, y el motor de persistencia en Oracle Cloud Infrastructure (OCI).

```mermaid
graph TB
    subgraph Canales ["📱 Canales de Entrada"]
        WA["WhatsApp Business<br/>(Meta Cloud API)"]
        IG["Instagram / Messenger<br/>(Meta Cloud API)"]
        WebClients["Pacientes / Clientes Web"]
    end

    subgraph Frontends ["💻 Frontends (Next.js 15 & Astro)"]
        CRM["CRM Dashboard<br/>(Next.js 15 + Clerk Auth)"]
        Landing["Landing Marketing<br/>(Astro + Demo CRM)"]
        Portal["Portal Paciente<br/>(Astro + OTP Auth)"]
        ClientFactory["Client Factory<br/>(Astro Multi-Tenant)"]
    end

    subgraph OCI ["☁️ Oracle Cloud Infrastructure (Always Free - ARM Ampere A1)"]
        Proxy["🔒 Proxy Reverso & SSL<br/>(Caddy / Nginx TLS 1.3)"]
        
        subgraph FastAPI ["⚡ FastAPI Backend Engine (Puerto 8000)"]
            Router["API Routers (/api/v1/*)<br/>(28 Módulos de Negocio)"]
            Lifespan["FastAPI Lifespan Workers"]
            WebhookQ["sys_webhook_queue Worker<br/>(SELECT FOR UPDATE SKIP LOCKED)"]
            ProspectorSched["Prospector Scheduler Loop"]
            ADK["Google ADK Agent Framework<br/>(ValRouter + Tools + Crews)"]
            LLMManager["LLM Manager<br/>(Failover Inteligente)"]
        end

        subgraph DB ["💾 Persistencia de Datos"]
            OracleDB[("Oracle Database 23ai<br/>44 Tablas / 24 DDLs<br/>AI Vector Search")]
            SQLiteDev[("SQLite In-Memory<br/>Pytest Test Suite")]
        end
    end

    subgraph AIServices ["🧠 Proveedores LLM Externos"]
        Gemini["Google Gemini 2.5<br/>(Flash / Pro - Primario)"]
        Claude["Anthropic Claude<br/>(Sonnet / Haiku - Fallback)"]
    end

    %% Relaciones de Entrada hacia Frontends y Proxy
    WebClients -->|Navegación Web| Frontends
    Frontends -->|REST API / HTTPS| Proxy
    WA -->|Webhook HMAC-SHA256| Proxy
    IG -->|Webhook HMAC-SHA256| Proxy

    %% Despacho Interno
    Proxy -->|HTTP Local :8000| Router
    Router -->|Inicia tareas| Lifespan
    Lifespan --> WebhookQ
    Lifespan --> ProspectorSched
    
    WebhookQ -->|Procesa eventos| ADK
    Router -->|Inferencia interactiva| ADK
    ADK --> LLMManager
    
    %% Llamadas externas a IA
    LLMManager -->|Primario API| Gemini
    LLMManager -.->|Fallback API| Claude

    %% Persistencia
    Router -->|Transacciones| OracleDB
    ADK -->|Búsqueda Vectorial & Citas| OracleDB
    ProspectorSched -->|Almacena leads| OracleDB
    FastAPI -.->|Entorno Tests| SQLiteDev
```

---

## 2. Flujo End-to-End de Mensajería WhatsApp & Orquestación IA

El patrón de desacoplamiento asíncrono garantiza que el webhook de Meta reciba una respuesta HTTP `200 OK` en menos de 100 ms, mientras la IA orquesta la respuesta en segundo plano sin riesgo de timeout.

```mermaid
sequenceDiagram
    autonumber
    actor Cliente as 👤 Paciente / Cliente (WhatsApp)
    participant Meta as 🌐 Meta Cloud API
    participant Proxy as 🔒 Caddy / Nginx (SSL)
    participant API as ⚡ FastAPI Router
    participant Queue as 📬 Sys Webhook Queue
    participant ADK as 🤖 Google ADK (ValRouter)
    participant LLM as 🧠 LLM Manager (Gemini / Claude)
    participant DB as 💾 Oracle 23ai / SQLite

    Cliente->>Meta: Envía mensaje ("Hola, necesito agendar una cita")
    Meta->>Proxy: POST /webhook/whatsapp (Payload + Firma HMAC)
    Proxy->>API: Forward payload
    
    rect rgb(235, 245, 255)
        Note over API: 1. Validación de firma HMAC SHA-256<br/>2. Resolución de tenant_id<br/>3. Inserción en sys_webhook_queue
        API->>DB: Guarda evento en cola con estado PENDING
        API-->>Proxy: HTTP 200 OK (inmediato, <50ms)
    end
    
    Proxy-->>Meta: HTTP 200 OK (Meta no reintenta)

    rect rgb(245, 235, 255)
        Note over Queue, ADK: Worker Lifespan procesa en background (SKIP LOCKED)
        Queue->>DB: SELECT FOR UPDATE SKIP LOCKED
        Queue->>ADK: Despacha evento al agente ValRouter
        ADK->>DB: Carga historial, flags del tenant y contexto EAV
        DB-->>ADK: Estado cargado (Flags: booking_active=true)
    end

    rect rgb(255, 245, 235)
        Note over ADK, LLM: Clasificación de intención & Ejecución de Tools
        ADK->>LLM: Prompt con Tool Definitions (check_availability, create_booking)
        LLM-->>ADK: ToolCall: check_availability(service_id, date)
        ADK->>DB: Consulta disponibilidad de agenda (Anti-double-booking)
        DB-->>ADK: Horarios disponibles [10:00, 15:30, 17:00]
        ADK->>LLM: ToolResult -> Genera respuesta conversacional
        LLM-->>ADK: "¡Hola! Tenemos horas disponibles hoy a las 15:30 y 17:00..."
    end

    rect rgb(235, 255, 245)
        Note over ADK, Meta: Persistencia y Envío de Respuesta
        ADK->>DB: Registra mensaje en historial y actualiza estado de chat
        ADK->>Meta: POST /v25.0/{phone_number_id}/messages
        Meta->>Cliente: Entrega mensaje en WhatsApp
    end
```

---

## 3. Embudo Comercial Autónomo (Outbound + Inbound + Cierre)

ValVic automatiza el ciclo comercial B2B completo, permitiendo captar clínicas en frío mediante prospección geolocalizada y cerrar ventas o agendamientos mediante el agente Val.

```mermaid
flowchart TD
    subgraph Outbound["🎯 Fase 1: Prospección Outbound"]
        A[Google Places API v1 / Scraping Local] --> B[Extracción: Nombre, Rating, Reseñas, Web, Teléfono]
        B --> C{Auditoría Digital}
        C -->|Sin Web / Web Básica + Buenas Reseñas| D[Calificación Lead Score ≥ 5]
        C -->|Web Moderna / Sin Teléfono| E[Descartado / Baja Prioridad]
        D --> F[Copywriting Persuasivo B2B con IA]
        F --> G1[Modo Meta API Directo]
        F --> G2[Modo wa.me Click-to-Chat / CLI Revisión]
    end

    subgraph Inbound["💬 Fase 2: Conversión y Negociación (Val)"]
        H[Cliente Responde por WhatsApp] --> I[ValRouter: Clasificación de Intención]
        I --> J{¿Tipo de Consulta?}
        J -->|Información / Servicios| K[RAG Semántico: AI Vector Search]
        J -->|Objeción de Precio| L[Escalera de Concesiones Controlada]
        J -->|Agendamiento Directo| M[Motor de Agenda Anti-Double-Booking]
        
        subgraph Concesiones["🪜 Escalera de Concesiones"]
            L1[1. Precio de Lista] --> L2[2. Descuento en Onboarding]
            L2 --> L3[3. Pago Dividido en Hitos]
            L3 --> L4[4. Descuento Temporal Mensual]
            L4 --> L5[Piso Mínimo Infranqueable / Escalamiento Humano]
        end
        L --> Concesiones
    end

    subgraph Cierre["💳 Fase 3: Cierre & Sincronización CRM"]
        N[Señal de Inicio de Pago / Reserva] --> O[Envío de Link de Pago / Datos Transferencia]
        O --> P[Actualización de Pipeline en CRM]
        P --> Q[Notificación al Operador Humano]
    end

    G1 --> H
    G2 --> H
    K --> N
    Concesiones --> N
    M --> N
```

---

## 4. Dual Database Engine y Aislamiento Multi-Tenant

Arquitectura de persistencia basada en **SQLModel (100% ORM, 0 SQL crudo)** con detección automática de motor en `engine.py`.

```mermaid
flowchart LR
    subgraph AppLayer["Capas de Aplicación"]
        FastAPI_App["FastAPI Routers & Services"]
        ADK_Agents["Google ADK Agents & Tools"]
        Cron_Workers["Lifespan / Scheduled Workers"]
    end

    subgraph ORMLayer["Capa ORM Unificada (SQLModel)"]
        SQLModel_Core["44 Modelos SQLModel<br/>(13 Archivos de Dominio)"]
        Singleton["_build_engine() Singleton"]
    end

    subgraph EngineResolution["Resolución de Motor en Tiempo de Ejecución"]
        CheckEnv{"¿Entorno?"}
        CheckEnv -->|Pytest o VALVIC_ENV=local| SQLiteEngine["SQLite Engine<br/>(valvic_dev.db / Memory)"]
        CheckEnv -->|ORACLE_DSN + PASSWORD| OracleEngine["Oracle 23ai Engine<br/>(Pool Size: 4, Max Overflow: 8)"]
    end

    subgraph MultiTenantSecurity["Aislamiento Multi-Tenant Obligatorio"]
        TenantFilter["Cláusula Determinista:<br/><code>WHERE tenant_id = :current_tenant_id</code>"]
        Checkpoints["Google ADK Checkpoints & Escalations"]
        Vectors["Oracle AI Vector Search Embeddings"]
    end

    AppLayer --> SQLModel_Core
    SQLModel_Core --> Singleton
    Singleton --> CheckEnv
    SQLiteEngine --> MultiTenantSecurity
    OracleEngine --> MultiTenantSecurity
```

---

## 5. Client Factory — Pipeline de Generación de Sitios Web

El módulo **Client Factory** automatiza la creación de páginas web para clínicas veterinarias y pymes asociadas, integrando scraping inteligente y mirrors de Instagram.

```mermaid
sequenceDiagram
    autonumber
    actor Operador as 👨‍💼 Operador ValVic
    participant Script as 🛠️ Scripts (fetch_ig_posts / test_mirrors)
    participant IG as 📸 Instagram Graph / Web Scraper
    participant Config as 📄 Data JSON (slug.json)
    participant Factory as 🚀 Astro Client Factory
    participant Build as 📦 Build Multi-Sitio
    participant Host as 🌐 Hosting (clientes.valvic.cl / spotvet.cl)

    Operador->>Script: Ejecuta script de ingesta (ej. SpotVet)
    Script->>IG: Extrae publicaciones recientes, testimonios y fotos
    IG-->>Script: Retorna media estructurada y textos
    Script->>Config: Genera/Actualiza `src/data/{slug}.json`
    Note over Config: Define identidad visual, colores,<br/>servicios, precios y externalUrl
    
    Operador->>Factory: Inicia compilación (`npm run build` o `build:spotvet`)
    
    alt Cliente en Subdominio General
        Factory->>Build: Genera rutas dinámicas para `clientes.valvic.cl/{slug}`
        Build->>Host: Despliega en `clientes.valvic.cl`
    else Cliente con Dominio Propio (ej. SpotVet)
        Factory->>Build: `BUILD_CLIENT=spotvet SITE_URL=https://spotvet.cl`
        Build->>Host: Genera bundle raíz independiente para `spotvet.cl`
    end
```

---

## 6. CRM Multitenant, Co-pilotaje Humano y Botón Dorado

El CRM en **Next.js 15** permite supervisar todas las conversaciones en vivo, pausar la IA en caliente (cojín humano) y controlar los flags de automatización mediante el **Botón Dorado**.

```mermaid
stateDiagram-v2
    [*] --> IA_Autonoma: Mensaje entrante de cliente

    state IA_Autonoma {
        [*] --> ClasificarIntencion
        ClasificarIntencion --> ResponderConocimiento: FAQ / Info
        ClasificarIntencion --> NegociarVenta: Intención Comercial
        ClasificarIntencion --> AgendarCita: Intención de Cita
    }

    IA_Autonoma --> CoPilotajeHumano: Operador escribe en chat (/messages)
    IA_Autonoma --> Escalado: Intención peligrosa / Límite de concesión superado

    state CoPilotajeHumano {
        [*] --> ChatPausado: Flag escalated = true
        ChatPausado --> MensajesManuales: Humano responde directamente
        MensajesManuales --> ReactivarIA: Operador presiona 'Reanudar IA'
    }

    ReactivarIA --> IA_Autonoma: Flag escalated = false
    Escalado --> CoPilotajeHumano: Notificación en tiempo real al CRM

    state BotonDorado {
        prospector_active: On / Off (Scraping y salida)
        sales_active: On / Off (Módulo de ventas Val)
        booking_active: On / Off (Agendamiento autónomo)
    }

    BotonDorado --> IA_Autonoma: Regula capacidades de los agentes
```

---

## 7. Ciclo de Vida de Workers Asíncronos (FastAPI Lifespan)

A diferencia de arquitecturas tradicionales que requieren crontabs externos fragmentados, ValVic implementa workers residentes en el ciclo de vida de FastAPI (`lifespan`), simplificando el despliegue a un solo contenedor o proceso.

```mermaid
flowchart TD
    Start([🚀 FastAPI Startup - Lifespan]) --> InitDB[Inicializar Engine & Crear Tablas]
    InitDB --> Worker1[Spawn Worker: webhook_queue]
    InitDB --> Worker2[Spawn Worker: prospector_scheduler]

    subgraph W1["📬 Webhook Queue Worker (Loop Permanente)"]
        W1_Fetch[SELECT * FROM sys_webhook_queue WHERE status='PENDING' FOR UPDATE SKIP LOCKED]
        W1_Fetch --> W1_Check{¿Hay eventos?}
        W1_Check -->|Sí| W1_Exec[Procesar con Google ADK ValRouter]
        W1_Exec --> W1_Update[Marcar status='COMPLETED' / 'FAILED']
        W1_Update --> W1_Fetch
        W1_Check -->|No| W1_Sleep[Sleep 100ms] --> W1_Fetch
    end

    subgraph W2["🎯 Prospector Scheduler Worker (Intervalos Programados)"]
        W2_CheckTenant[Consultar tenants con prospector_active=true]
        W2_CheckTenant --> W2_Exec[Ejecutar prospección y scoring por lotes]
        W2_Exec --> W2_Sleep[Sleep Intervalo Configurado] --> W2_CheckTenant
    end

    Worker1 --> W1
    Worker2 --> W2

    Shutdown([🛑 FastAPI Shutdown - Lifespan]) --> StopWorkers[Cancelar tareas asíncronas de fondo]
    StopWorkers --> CloseDB[Cerrar Pool de Conexiones DB]
    CloseDB --> End([Apagado Seguro])
```
