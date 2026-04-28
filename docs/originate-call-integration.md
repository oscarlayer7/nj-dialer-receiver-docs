# NJDialer-Receiver · Integración originate-call ↔ Routing-Service

> Documento técnico del comportamiento **real** (no planeado) del endpoint `POST /api/originate-call` en el servicio **NJDialer-Receiver** y su integración con **Routing-Service**.
>
> - **Servicio:** `nj-dialer-receiver` (Laravel 12, PHP 8.2+)
> - **Branch documentado:** `trunk`
> - **Fecha:** 2026-04-20
> - **Alcance:** lado Receiver. El contrato del Routing-Service se describe solo desde lo que consume/observa este servicio.

---

## 1. Flujo paso a paso

**Endpoint:** `POST /api/originate-call` — definido en `routes/api.php:12` bajo middleware `auth:sanctum`.
**Controller:** `App\Http\Controllers\Call\OriginateCallController` (invocable `__invoke`).

| # | Paso | Ubicación |
|---|------|-----------|
| 1 | Autenticación Sanctum (Bearer token) | `routes/api.php:12` |
| 2 | Validación del payload vía `OriginateCallRequest` | `app/Http/Requests/OriginateCallRequest.php:30-54` |
| 3 | Resolver `request_id` (idempotencia): usa el del cliente o genera `Str::ulid()` | `OriginateCallController.php:83` |
| 4 | Construir `OriginateCallDTO` (enriquecido con `user_id` y `user_name` del usuario autenticado) | `OriginateCallController.php:86-90` |
| 5 | Log `originate_call_received` (`request_id`, `phone`, `source`, `has_flow`) | `OriginateCallController.php:93-98` |
| 6 | `CallIntentService::create($dto, $id)` — aquí ocurre toda la resolución de ruta y persistencia | `OriginateCallController.php:100` |
| 6a | Check idempotencia: busca `CallIntent` por `request_id`; si existe devuelve el mismo (log `intent_replayed`) | `CallIntentService.php:28-42` |
| 6b | Genera `call_id` (ULID) y construye `tenant_id = {user_id}-{normalized_user_name}` | `CallIntentService.php:44-46` |
| 6c | **Llamada a Routing-Service:** `PhoneRoutingService::resolve()` → `RoutingServiceClient::resolve()` → `POST {ROUTING_SERVICE_URL}/internal/resolve-route` con **timeout 2s** | `RoutingServiceClient`, `PhoneRoutingService` |
| 6d | Determina `routing_status`: `no_route` / `fallback` / `resolved` | `CallIntentService.php:57-61` |
| 6e | Log `routing_no_route` \| `routing_fallback` \| `routing_resolved` | `CallIntentService.php:63-78` |
| 6f | Evalúa entitlement VMD (feature flag + kill-switch + entitlement del usuario) | `CallIntentService.php:80-82` |
| 6g | `DB::transaction`: crea `CallIntent`; si `routing_status='no_route'` hace short-circuit (no crea outbox); si no, crea `CallActionOutbox` con payload completo | `CallIntentService.php:84-155` |
| 7 | Si `routing_status === 'no_route'` → responde **HTTP 422** con `reason_code` | `OriginateCallController.php:103-110` |
| 8 | Si hay `flow.steps` → `CallFlowStore::store()` en Redis (TTL 1800s). Los errores aquí se capturan y **no fallan la request** (log `call_flow_store_failed`) | `OriginateCallController.php:112-126` |
| 9 | Incrementa contador OpenTelemetry `njdialer.receiver.call.originated.total` con tag `has_flow` | `OriginateCallController.php:128-132` |
| 10 | Log `originate_call_completed` y responde **HTTP 202** | `OriginateCallController.php:134-147` |

### Comportamiento según respuesta de Routing-Service

Definido en `RoutingServiceClient.php` y envuelto por `PhoneRoutingService.php`.

- **200 OK** → `RoutingResultDTO` con `providerId / providerCode / matchedPrefix / routingVersion` y **`sip_endpoint`**. El destino SIP de marcado (`destination` / outbox `number`) es el **string** `sip_endpoint` tal cual (tras trim). Si `sip_endpoint` es **null o vacío**, no se inventa un URI con `CURRENT_PSTN_DOMAIN`: se trata como `no_route` con `reason_code` **`SIP_ENDPOINT_UNPUBLISHED`**, se persisten el proveedor y la versión de ruta en `call_intents` para trazabilidad, y **no** se crea outbox.
- **422** → `RoutingResultDTO::noRoute($reasonCode)`. Se extrae `reason_code` del body; si viene vacío se usa `'NO_ROUTE'`. Se crea `CallIntent` con `routing_status='no_route'` pero **NO se crea outbox**.
- **ConnectionException (timeout o red)** → se re-lanza y la captura `PhoneRoutingService` (línea 128) → **fallback local**.
- **5xx u otro status inesperado** → `RoutingServiceClient` lanza `RuntimeException` → la captura `PhoneRoutingService` → **fallback local**.
- **Cualquier otro `Throwable`** en `RoutingServiceClient::resolve()` → idem fallback.

El fallback produce: `providerCode='fallback'`, `routingVersion=0`, `reasonCode='ROUTING_UNAVAILABLE_FALLBACK'`, y `destination` construido con `getDestinationRoute()` (lógica local).

---

## 2. Request y Response

### 2.1 Request esperado

Validación en `app/Http/Requests/OriginateCallRequest.php:30-54`.

| Campo | Tipo | Obligatorio | Regla | Notas |
|---|---|---|---|---|
| `request_id` | string | No | `max:64` | Idempotencia. Si el servidor ve el mismo `request_id`, devuelve el `call_id` original sin crear duplicados. Si no se envía se genera un ULID. |
| `phone` | string | **Sí** | regex `^\+?[1-9]\d{6,14}$` | Formato E.164. |
| `source` | string | No | `max:50` | Origen lógico de la llamada; default `'api'`. |
| `webhook_url_status` | string (URL) | No | `url` | Callback opcional para updates de estado. |
| `vmd_enabled` | boolean o string | No | — | Sujeto a feature flag + kill-switch + entitlement del usuario. |
| `flow` | object | No | — | Si se envía debe tener `steps` con al menos 1 paso. |
| `flow.steps[].action` | string | **Sí si hay flow** | `in:say,audio_fork_start` | |
| `flow.steps[].text` | string | Sí si `action=say` | `max:500` | |
| `flow.steps[].voice` | string | No | `max:50` | |
| `flow.steps[].wss_url` | string | Sí si `action=audio_fork_start` | | |
| `flow.steps[].metadata` | object | No | | |

Headers: `Authorization: Bearer <sanctum-token>`, `Content-Type: application/json`.

### 2.2 Response 202 — queued

Para `routing_status` en `resolved` o `fallback` (`OriginateCallController.php:142-147`).

```json
{
  "request_id": "01JRX...",
  "call_id": "01ARYZ...",
  "status": "queued",
  "topic": "call_actions"
}
```

### 2.3 Response 422 — no_route

`OriginateCallController.php:103-110`. `reason_code` es el mismo que devolvió Routing-Service, o `"NO_ROUTE"` si venía vacío.

```json
{
  "request_id": "01JRX...",
  "call_id": "01ARYZ...",
  "status": "no_route",
  "reason_code": "NO_ROUTE"
}
```

### 2.4 Response 422 — validation error

Formato Laravel estándar:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "phone": ["Use international format, e.g. +14155552671"]
  }
}
```

### 2.5 Otros códigos

- **401** — Bearer token Sanctum inválido o ausente.
- **500** — Excepción en `CallIntentService::create()` (ej. error DB). Log `intent_create_failed` con trace. La transacción hace rollback: **no queda ni intent ni outbox**.

---

## 3. Metadata propagada desde Routing-Service

Campos devueltos por `/internal/resolve-route` (encapsulados en `RoutingResultDTO`) y cómo los propaga Receiver:

| Campo | `call_intents` | `call_action_outbox.payload` | Response HTTP | Notas |
|---|:---:|:---:|:---:|---|
| `provider_id` | ✅ `provider_id` | ✅ `provider_id` | ❌ | nullable |
| `provider_code` | ✅ `provider_code` | ✅ `provider_code` | ❌ (sí en log) | `'fallback'` cuando aplica fallback |
| `matched_prefix` | ✅ `matched_prefix` | ✅ `matched_prefix` | ❌ | nullable |
| `routing_version` | ✅ `routing_version` | ✅ `routing_version` | ❌ (sí en log) | `0` cuando aplica fallback |
| `reason_code` | ✅ `routing_reason_code` | ✅ `routing_reason_code` | ✅ (solo en 422 no_route) | `NO_ROUTE` / `SIP_ENDPOINT_UNPUBLISHED` / `ROUTING_UNAVAILABLE_FALLBACK` / null |
| `sip_endpoint` (200) | n/a (equivale a `destination` cuando aplica) | n/a | ❌ | En ruta resuelta, `destination` = `sip_endpoint`. Si es null, ver `SIP_ENDPOINT_UNPUBLISHED` (sin outbox) |
| `destination` (fallback / killswitch) | ✅ `destination` | ✅ `number` | ❌ | Ruta local vía `getDestinationRoute()` cuando no aplica Routing-Service o fallo técnico |
| `routing_status` (derivado local) | ✅ `routing_status` | ✅ `routing_status` | ✅ parcial | enum: `resolved` \| `no_route` \| `fallback` |

### Campos adicionales del outbox payload

Generados en `CallIntentService.php:113-140`:

- `command: 'ORIGINATE_CALL'`
- `call_id`, `request_id`, `phone`, `number`, `submitted_at` (ISO8601 UTC), `source`, `tenant_id`
- `caller_id: '+528135470703'` — **hardcoded** en `CallIntentService.php:120` (TODO pendiente)
- `webhook_url_status` — solo si no es null
- `vmd_enabled` — boolean o string según entitlement del usuario

### `POST /v1/route-plan` (opcional)

`RoutingServiceClient::fetchRoutePlan()` deserializa `primary` y `fallbacks[]` con el mismo campo por fila: `sip_endpoint` (string o null). **No** participa hoy en `originate-call`; sirve consumo futuro o herramientas. Ver `App\Data\RoutePlanDTO` y `RoutePlanEntryDTO`.

---

## 4. Fallback técnico

**Sí existe.** Definido en `PhoneRoutingService::resolve()` líneas 128-135.

### Cuándo se activa

- `ConnectionException` (timeout de 2s o red caída).
- Routing-Service devuelve 5xx o status distinto de 200/422 → `RoutingServiceClient.php:82-84` lanza `RuntimeException`.
- Cualquier otro `Throwable` durante `RoutingServiceClient::resolve()`.

### Qué hace

- Log `PhoneRoutingService: Routing-Service unavailable, applying local fallback` (WARNING).
- Devuelve `RoutingResultDTO::fallback($destination)` con:
  - `resolved=true`, `noRoute=false`
  - `providerCode='fallback'`, `routingVersion=0`
  - `reasonCode='ROUTING_UNAVAILABLE_FALLBACK'`
  - `destination` desde `getDestinationRoute()` (lógica local en `PhoneRoutingService.php:143-168`, con overrides de números de prueba y default `CURRENT_PSTN_DOMAIN`).
- `CallIntent` se crea con `routing_status='fallback'`, `CallActionOutbox` se crea normalmente, la llamada **procede**.

### Cuándo NO hay fallback

- **422 NO_ROUTE** de Routing-Service: resultado de negocio, no error técnico → responde 422 al cliente, no crea outbox.
- **Metadata incompleta** en 200 OK: los campos faltantes quedan `null` (`$data['provider_id'] ?? null`, etc.) y la llamada **procede** con `routing_status='resolved'` y campos nulos. No hay validación de completitud.

---

## 5. Validación operativa end-to-end

Para comprobar que una nueva ruta publicada en Routing-Service está siendo consumida por Receiver:

### 5.1 Logs (canal configurado en `config/logging.php`)

Eventos relevantes emitidos durante el flujo:

| Evento | Nivel | Emisor | Significado |
|---|---|---|---|
| `originate_call_received` | INFO | Controller + Service | Inicio de procesamiento |
| `intent_replayed` | INFO | `CallIntentService` | Hit de idempotencia (mismo `request_id`) |
| `RoutingServiceClient: NO_ROUTE received` | INFO | Client | 422 del servicio |
| `RoutingServiceClient: connection error` | WARNING | Client | Timeout o red caída |
| `RoutingServiceClient: unexpected HTTP status` | WARNING | Client | 5xx u otro |
| `PhoneRoutingService: Routing-Service unavailable, applying local fallback` | WARNING | Wrapper | Fallback activado |
| `routing_resolved` \| `routing_fallback` \| `routing_no_route` | INFO | `CallIntentService` | Resultado final (incluye `provider_code`, `routing_version`, `reason_code`) |
| `intent_created` | INFO | `CallIntentService` | `CallIntent` persistido |
| `outbox_created` | INFO | `CallIntentService` | `CallActionOutbox` persistido |
| `intent_create_failed` | ERROR | `CallIntentService` | Excepción en la transacción |
| `call_flow_store_failed` | ERROR | Controller | Falló Redis (call continúa) |
| `originate_call_completed` | INFO | Controller | Fin exitoso (incluye `routing_status`, `provider_code`, `routing_version`) |

### 5.2 Base de datos — `call_intents`

```sql
SELECT request_id, call_id, phone, routing_status, routing_reason_code,
       provider_id, provider_code, matched_prefix, routing_version, destination
FROM call_intents
WHERE request_id = '<id>'
ORDER BY id DESC
LIMIT 1;
```

Interpretación:

- `routing_status='resolved'` y `routing_version > 0` → Routing-Service respondió 200.
- `routing_status='fallback'` y `routing_version=0` → Routing-Service falló/timeout; aplicó fallback.
- `routing_status='no_route'` → Routing-Service dijo 422.

### 5.3 Base de datos — `call_action_outbox`

```sql
SELECT id, status, kafka_key, payload
FROM call_action_outbox
WHERE kafka_key = '<call_id>'
ORDER BY id DESC
LIMIT 1;
```

- El `payload` (JSON) incluye `routing_version`, `provider_code`, `matched_prefix` — ahí se ve si la ruta nueva realmente aplicó.
- **Si no existe outbox para el `call_id`**, probablemente el intent fue `no_route` (el servicio no crea outbox en ese caso — `CallIntentService.php:108-111`).

### 5.4 Test de humo con curl

```bash
curl -X POST https://<host>/api/originate-call \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"request_id":"smoke-01","phone":"+14155552671"}'
```

Esperar `202 {status:"queued"}`, luego consultar `call_intents` y `call_action_outbox` como en 5.2/5.3.

### 5.5 Tests automatizados

- `tests/Feature/OriginateCallEnhancementsTest.php` — idempotencia, reason_code real, fallback.
- `tests/Feature/CallIntentServiceTest.php` — propagación de campos a intent/outbox.
- `tests/Feature/CallFlowStoreDspTest.php` — flow storage y DSP.
- `tests/Feature/CallsRelayClaimTest.php` — claim/publish desde outbox.

Ejecución: `./vendor/bin/sail artisan test tests/Feature/OriginateCallEnhancementsTest.php` (requerido usar Sail — PHP 8.4 dentro del contenedor).

---

## 6. Troubleshooting

| Síntoma | Causa real |
|---|---|
| **422 en `originate-call`** | (a) **Validation failure** del `OriginateCallRequest` (ej. `phone` no cumple regex E.164, `flow.steps[*].action` no es `say`/`audio_fork_start`, `webhook_url_status` no es URL válida). Respuesta con `errors{}`. (b) **Routing-Service devolvió 422** → respuesta con `status:"no_route"` y `reason_code`. Buscar log `RoutingServiceClient: NO_ROUTE received`. |
| **202 queued con metadata inesperada** | Routing-Service respondió 200 con campos nulos o distintos. `RoutingServiceClient.php:65-74` acepta `null` en todos los campos sin validar. Revisar `call_intents.routing_version` y `provider_code`: si son `null` pero `routing_status='resolved'`, la respuesta 200 vino incompleta. |
| **`routing_version` no coincide con lo esperado** | (a) `routing_version=0` → **fallback activo**; Routing-Service no respondió (ver log warning de fallback). (b) `routing_version` distinto pero no cero → Routing-Service está sirviendo otra versión (cache, deploy pendiente, etc. — investigar lado Routing-Service). (c) **Idempotencia**: `intent_replayed` devuelve el intent original porque el `request_id` coincide con uno previo. Ver log `intent_replayed`; probar con `request_id` nuevo. |
| **Provider correcto no aparece en `call_intent`** | (a) **Fallback activo** (`provider_code='fallback'`, `routing_version=0`). (b) Routing-Service devolvió 200 sin `provider_code` → queda null. (c) Idempotencia: se devolvió un intent antiguo de antes del cambio en Routing-Service. |
| **Fallback inesperado** | (a) **Timeout de 2s** (`RoutingServiceClient.php:34`): red lenta o Routing-Service lento. (b) **5xx** de Routing-Service. (c) **JSON malformado** en response. (d) `ROUTING_SERVICE_URL` o `ROUTING_SERVICE_TOKEN` mal configurados (ver `config/services.php:50-53`). Diagnóstico directo: log `PhoneRoutingService: Routing-Service unavailable, applying local fallback` trae el `error` message. |
| **401 Unauthorized** | Bearer token Sanctum inválido o ausente en el header `Authorization`. |
| **500 Internal Server Error** | Excepción dentro de `CallIntentService::create()` (ej. DB down, constraint violation). Log `intent_create_failed` con `trace`. Rollback total de la transacción: **no queda intent ni outbox**. |

---

## 7. Ejemplos

### 7.1 Éxito (Routing-Service responde 200)

**Request:**
```http
POST /api/originate-call
Authorization: Bearer <token>
Content-Type: application/json

{"phone":"+14155552671","source":"crm","request_id":"req-001"}
```

**Response 202:**
```json
{
  "request_id": "req-001",
  "call_id": "01ARYZ...",
  "status": "queued",
  "topic": "call_actions"
}
```

**Estado en DB:**
- `call_intents.routing_status = 'resolved'`
- `call_intents.provider_code = 'twilio'` (ejemplo)
- `call_intents.routing_version = 42` (ejemplo)
- `call_action_outbox.status = 'pending'` con payload completo.

### 7.2 No route (Routing-Service responde 422)

**Response 422:**
```json
{
  "request_id": "req-002",
  "call_id": "01ARYZ...",
  "status": "no_route",
  "reason_code": "NO_ROUTE"
}
```

**Estado en DB:** solo `call_intents` con `routing_status='no_route'`. **No hay outbox.**

### 7.3 Fallback (Routing-Service timeout)

**Response 202** (idéntico al éxito, pero con diferencias en DB):
- `call_intents.routing_status = 'fallback'`
- `call_intents.provider_code = 'fallback'`
- `call_intents.routing_version = 0`
- `call_intents.routing_reason_code = 'ROUTING_UNAVAILABLE_FALLBACK'`
- Log WARNING: `PhoneRoutingService: Routing-Service unavailable, applying local fallback`.

### 7.4 Idempotencia (mismo `request_id` dos veces)

La segunda llamada devuelve el mismo `call_id` sin crear duplicados. Log `intent_replayed` en el segundo intento. No se llama a Routing-Service la segunda vez.

---

## 8. Configuración

### Variables de entorno relevantes

Definidas en `config/services.php` y `config/kafka.php`:

| Env var | Default | Uso |
|---|---|---|
| `ROUTING_SERVICE_URL` | `http://routing-service:8080` | Base URL del Routing-Service |
| `ROUTING_SERVICE_TOKEN` | (sin default) | Bearer token opcional. Si está presente se envía `Authorization: Bearer ...` |
| `KAFKA_PHONE_TOPIC` | `call_actions` | Topic Kafka al que se publicará desde el outbox |
| `CURRENT_PSTN_DOMAIN` | `@nj-dialer-beta.pstn.twilio.com` | Usado por el fallback local para construir el SIP URI |
| `OUTBOX_BATCH_SIZE`, `OUTBOX_LOCK_TTL_SEC`, `OUTBOX_IDLE_SLEEP_MS` | ver `config/outbox.php` | Relay/claim del outbox |

### Timeout HTTP

**Hardcoded a 2 segundos** en `RoutingServiceClient.php:34` (`Http::timeout(2)`). No es configurable por env hoy.

---

## 9. Dependencias externas y supuestos

- **Contrato observado** del Routing-Service (solo lo que Receiver consume):
  - `POST /internal/resolve-route`
  - Request body: `{ tenant_id: string, to_e164: string, campaign_id?: string }`
  - Response 200: incluye `sip_endpoint: string | null` (además de `provider_id?`, `provider_code?`, etc.). El destino de marcado se toma de `sip_endpoint` cuando no es null; si es null, ver `SIP_ENDPOINT_UNPUBLISHED` arriba.
  - Response 422: `{ reason_code? }` — ausencia de `reason_code` se normaliza a `"NO_ROUTE"`.
- **Plan de ruta completo** (`POST /v1/route-plan`): modelado en `RoutePlanDTO`; no bloquea originate-call hoy.
- **El `caller_id`** enviado en el outbox está **hardcoded** (`+528135470703`, `CallIntentService.php:120`).
- **Test numbers** (8112220000..8112220205) en `PhoneRoutingService.php:143-168` tienen overrides para entornos de desarrollo y están marcados con TODO para eliminar cuando Routing-Service sea autoridad única.

---

## 10. Referencias de código

- `routes/api.php:12`
- `app/Http/Controllers/Call/OriginateCallController.php`
- `app/Http/Requests/OriginateCallRequest.php`
- `app/Services/CallIntentService.php`
- `app/Services/PhoneRoutingService.php`
- `app/Services/RoutingServiceClient.php`
- `app/Services/CallFlowStore.php`
- `app/Data/OriginateCallDTO.php`
- `app/Data/RoutingResultDTO.php`
- `app/Models/CallIntent.php`
- `app/Models/CallActionOutbox.php`
- `database/migrations/2026_02_17_000001_create_call_intents_table.php`
- `database/migrations/2026_02_17_000000_create_call_action_outbox_table.php`
- `database/migrations/2026_04_15_000001_*` (columnas de routing en `call_intents`)
- `database/migrations/2026_04_16_000001_*` (columna `request_id` en `call_intents`)
- `config/services.php:50-53`
- `tests/Feature/OriginateCallEnhancementsTest.php`
- `tests/Feature/CallIntentServiceTest.php`
