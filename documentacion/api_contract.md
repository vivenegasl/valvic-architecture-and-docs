# Contrato API REST y Webhooks (Extracto Público)

Este documento detalla la especificación pública de la interfaz de programación (API) de ValVic. Los esquemas y endpoints se definen utilizando el estándar **OpenAPI v3.0.0**.

---

## Especificación OpenAPI (YAML)

A continuación se presenta el contrato formal que define la comunicación entre clientes externos (Meta Cloud API, Panel CRM) y el servidor backend en Oracle Cloud.

```yaml
openapi: 3.0.3
info:
  title: ValVic API
  description: >-
    API del backend unificado de ValVic para la gestión conversacional y el 
    agendamiento automático impulsado por IA (Vicky). Implementa soporte 
    multi-tenant con aislamiento estricto y comunicación asíncrona.
  version: 2.0.0
servers:
  - url: https://api.valvic.cl
    description: Servidor de Producción (OCI Ampere A1 VM)
  - url: https://api-staging.valvic.cl
    description: Servidor de Staging (Pruebas)
  - url: http://localhost:8001
    description: Entorno de Desarrollo Local
paths:
  /webhook/whatsapp:
    get:
      summary: Verificación del Webhook de Meta (WhatsApp)
      description: >-
        Punto de entrada utilizado por Meta Cloud API para validar la propiedad 
        del webhook de forma segura durante la configuración. Requiere un token de
        verificación que coincida con el configurado en el servidor backend.
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
          description: Verificación exitosa. Retorna el valor exacto de `hub.challenge`.
          content:
            text/plain:
              schema:
                type: string
                example: "1158201484"
        '403':
          description: Fallo en la verificación del token.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HTTPError'

    post:
      summary: Recepción de Mensajes de Meta (WhatsApp / Inbound)
      description: >-
        Recibe notificaciones de eventos en tiempo real desde la plataforma de Meta.
        Valida que el origen sea legítimo comprobando la firma digital en el header.
        Procesa el mensaje en segundo plano para evitar time-outs.
      headers:
        X-Hub-Signature-256:
          schema:
            type: string
          required: true
          description: Firma HMAC-SHA256 del payload utilizando la clave secreta de la App de Meta.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MetaWebhookPayload'
      responses:
        '200':
          description: Webhook recibido de manera conforme y encolado en segundo plano.
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok
        '400':
          description: Estructura JSON inválida o malformada.
        '403':
          description: Firma digital ausente o incorrecta.
        '500':
          description: Error interno del sistema.

  /api/v1/tenants/{tenant_id}/subscription-flags:
    get:
      summary: Consultar Banderas de Suscripción (Botón Dorado)
      description: Obtiene el estado actual de los módulos de IA habilitados para el tenant.
      security:
        - BearerAuth: []
      parameters:
        - name: tenant_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
            example: d3b07384-d113-4956-a5db-e1c482597750
      responses:
        '200':
          description: Banderas de suscripción recuperadas con éxito.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubscriptionFlagsResponse'
        '401':
          description: Token de autenticación inválido o ausente.
        '403':
          description: El usuario autenticado no pertenece al tenant solicitado.
        '404':
          description: Tenant no encontrado.

    patch:
      summary: Modificar Banderas de Suscripción (Botón Dorado)
      description: Habilita o deshabilita en caliente los módulos de Prospección, Ventas y Reservas del tenant.
      security:
        - BearerAuth: []
      parameters:
        - name: tenant_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
            example: d3b07384-d113-4956-a5db-e1c482597750
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/SubscriptionFlagsUpdate'
      responses:
        '200':
          description: Banderas de suscripción actualizadas.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/SubscriptionFlagsResponse'
        '400':
          description: Petición incorrecta o parámetros malformados.
        '401':
          description: Token inválido.
        '403':
          description: Privilegios insuficientes.

  /api/v1/chats/{conversacion_id}/messages:
    post:
      summary: Enviar Mensaje desde el Operador (CRM Co-pilot)
      description: Permite a un operador humano enviar un mensaje directo al cliente, interrumpiendo el flujo autónomo de la IA.
      security:
        - BearerAuth: []
      parameters:
        - name: conversacion_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
            example: c42f4b48-ec8d-4cb0-a54f-124b1790ee2b
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
                  description: Contenido de texto que se le enviará al cliente.
                  example: "Hola, soy el veterinario de turno. ¿En qué le puedo ayudar?"
      responses:
        '200':
          description: Mensaje encolado e insertado en la base de datos de auditoría.
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: message_queued
                  mensaje_id:
                    type: string
                    format: uuid
                    example: a5607998-d191-45bc-9bf8-917cf7b194d2
        '401':
          description: No autorizado.
        '404':
          description: Conversación no encontrada o no pertenece al tenant del operador.

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: Autenticación mediante token JWT de Clerk asociado al tenant del usuario.

  schemas:
    HTTPError:
      type: object
      properties:
        detail:
          type: string
          example: "Verification failed"

    SubscriptionFlagsResponse:
      type: object
      properties:
        tenant_id:
          type: string
          format: uuid
          example: d3b07384-d113-4956-a5db-e1c482597750
        prospector_active:
          type: boolean
          example: false
          description: Habilita el raspado y envío saliente automático (outbound).
        sales_active:
          type: boolean
          example: true
          description: Habilita el nodo conversacional interactivo de venta de servicios.
        booking_active:
          type: boolean
          example: true
          description: Habilita el agendamiento directo autónomo en la agenda médica.
        flags_updated_at:
          type: string
          format: date-time
          example: "2026-05-21T19:35:00Z"

    SubscriptionFlagsUpdate:
      type: object
      properties:
        prospector_active:
          type: boolean
          example: false
        sales_active:
          type: boolean
          example: true
        booking_active:
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
          enum: [whatsapp_business_account, instagram, page]
          example: whatsapp_business_account
        entry:
          type: array
          items:
            type: object
            properties:
              id:
                type: string
                example: "109283719283719"
              changes:
                type: array
                items:
                  type: object
                  properties:
                    field:
                      type: string
                      example: messages
                    value:
                      type: object
                      description: Datos del mensaje, teléfono de origen y texto.
```

---

## Directrices de Integración y Seguridad

1. **Aislamiento Multi-tenant:** Toda petición que interactúe con `/api/v1/` realiza internamente una resolución en base al JWT de Clerk. El `tenant_id` se extrae del reclamo (`claim`) de la organización de Clerk y se inyecta en la cláusula SQL de base de datos de manera determinista (`WHERE tenant_id = :tenant_id`), asegurando que ningún operador pueda visualizar o interactuar con datos de otros clientes.
2. **Control de Flujo Conversacional (Cojín Humano):** Al invocar el endpoint de envío de mensaje de operador (`POST /messages`), el backend marca la conversación actual como escalada (`escalated=true`) en la sesión de LangGraph. Esto deshabilita inmediatamente las respuestas autónomas del agente para dar el control total al humano, hasta que la conversación se cierre manualmente en el CRM.
