# Contrato API REST y Webhooks (Extracto OpenAPI v3.0.3)

> **Especificación Técnica de Endpoints, Modelos de Datos y Protocolos de Integración.**  
> *Versión 2.0 — ValVic Backend Engine.*

---

## Especificación OpenAPI (YAML)

A continuación se presenta el contrato formal que define la comunicación entre clientes externos (Meta Cloud API, CRM Next.js 15, Portal Paciente, Client Factory) y el servidor backend en Oracle Cloud.

```yaml
openapi: 3.0.3
info:
  title: ValVic Multi-Tenant API
  description: >-
    API del backend unificado de ValVic para la gestión conversacional, 
    agendamiento médico/veterinario inteligente impulsado por Google ADK (Val),
    pipeline comercial de oportunidades y administración SaaS multi-tenant.
  version: 2.0.0
servers:
  - url: https://api.valvic.cl
    description: Servidor de Producción (OCI Ampere A1 VM)
  - url: http://localhost:8000
    description: Entorno de Desarrollo Local
paths:
  # ==========================================
  # WEBHOOKS META (WhatsApp / Instagram)
  # ==========================================
  /webhook/whatsapp:
    get:
      summary: Verificación del Webhook de Meta
      description: Valida el token de verificación emitido por Meta for Developers.
      parameters:
        - name: hub.mode
          in: query
          required: true
          schema:
            type: string
            example: subscribe
        - name: hub.verify_token
          in: query
          required: true
          schema:
            type: string
            example: MI_TOKEN_SECRETO_META
        - name: hub.challenge
          in: query
          required: true
          schema:
            type: string
            example: "1158201484"
      responses:
        '200':
          description: Verificación exitosa. Retorna el valor exacto de hub.challenge.
          content:
            text/plain:
              schema:
                type: string
                example: "1158201484"
        '403':
          description: Token inválido.

    post:
      summary: Recepción de Mensajes Entrantes de WhatsApp
      description: Recibe payloads JSON de WhatsApp. Valida firma HMAC-SHA256 y encola en sys_webhook_queue.
      parameters:
        - name: X-Hub-Signature-256
          in: header
          required: true
          schema:
            type: string
            example: "sha256=d3b07384d1134956a5dbe1c482597750..."
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MetaWebhookPayload'
      responses:
        '200':
          description: Payload encolado exitosamente. Retorna de inmediato en <50ms.
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok
        '403':
          description: Firma digital ausente o no coincide con META_APP_SECRET.

  # ==========================================
  # BANDERAS DE SUSCRIPCIÓN (BOTÓN DORADO)
  # ==========================================
  /api/v1/tenants/{tenant_id}/subscription-flags:
    get:
      summary: Consultar Banderas de Suscripción del Tenant
      description: Obtiene los flags de activación de IA (prospección, ventas, agendamiento).
      security:
        - BearerAuth: []
      parameters:
        - name: tenant_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Banderas de suscripción actuales.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubscriptionFlagsResponse'

    patch:
      summary: Actualizar Banderas de Suscripción (Botón Dorado)
      description: Modifica en caliente los módulos de IA activos para el tenant.
      security:
        - BearerAuth: []
      parameters:
        - name: tenant_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SubscriptionFlagsUpdate'
      responses:
        '200':
          description: Banderas actualizadas correctamente.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubscriptionFlagsResponse'

  # ==========================================
  # GESTIÓN DE CHATS & CO-PILOTAJE HUMANO
  # ==========================================
  /api/v1/chats/{conversacion_id}/messages:
    post:
      summary: Enviar Mensaje de Operador (CRM Co-pilot)
      description: Permite al operador humano intervenir en la conversación y desactivar la IA autónoma.
      security:
        - BearerAuth: []
      parameters:
        - name: conversacion_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - contenido
              properties:
                contenido:
                  type: string
                  example: "Hola, soy el Dr. Juan. He tomado el control del chat."
      responses:
        '200':
          description: Mensaje encolado y enviado al cliente.

  /api/v1/chats/{conversacion_id}/pause:
    post:
      summary: Pausar o Reanudar IA en una Conversación
      description: Cambia el flag `escalated` para transferir el control entre el humano y el agente Val.
      security:
        - BearerAuth: []
      parameters:
        - name: conversacion_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - paused
              properties:
                paused:
                  type: boolean
                  example: true
      responses:
        '200':
          description: Estado de pausa actualizado.

  # ==========================================
  # AGENDA Y CITAS MÉDICAS
  # ==========================================
  /api/v1/agenda/slots:
    get:
      summary: Consultar Disponibilidad de Horarios
      description: Retorna los bloques horarios disponibles aplicando lógica anti-double-booking.
      security:
        - BearerAuth: []
      parameters:
        - name: fecha
          in: query
          required: true
          schema:
            type: string
            format: date
            example: "2026-06-15"
        - name: service_id
          in: query
          required: false
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Lista de slots horarios disponibles.
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/AgendaSlot'

  # ==========================================
  # COMPLIANCE (GDPR & META DATA DELETION)
  # ==========================================
  /api/v1/compliance/data-deletion:
    post:
      summary: Solicitud de Borrado de Datos de Usuario (Meta GDPR)
      description: Endpoint oficial requerido por Meta para la eliminación de datos de usuario.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                user_id:
                  type: string
      responses:
        '200':
          description: Código de confirmación de eliminación y URL de seguimiento.
          content:
            application/json:
              schema:
                type: object
                properties:
                  url:
                    type: string
                    example: "https://api.valvic.cl/compliance/data-deletion/status/DEL-12345"
                  confirmation_code:
                    type: string
                    example: "DEL-12345"

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: Autenticación mediante JSON Web Token (JWT) de Clerk.

  schemas:
    SubscriptionFlagsResponse:
      type: object
      properties:
        tenant_id:
          type: string
          format: uuid
        prospector_active:
          type: boolean
        sales_active:
          type: boolean
        booking_active:
          type: boolean
        updated_at:
          type: string
          format: date-time

    SubscriptionFlagsUpdate:
      type: object
      properties:
        prospector_active:
          type: boolean
        sales_active:
          type: boolean
        booking_active:
          type: boolean

    AgendaSlot:
      type: object
      properties:
        start_time:
          type: string
          example: "2026-06-15T10:00:00Z"
        end_time:
          type: string
          example: "2026-06-15T10:45:00Z"
        available:
          type: boolean
          example: true

    MetaWebhookPayload:
      type: object
      required:
        - object
        - entry
      properties:
        object:
          type: string
          example: whatsapp_business_account
        entry:
          type: array
          items:
            type: object
```
