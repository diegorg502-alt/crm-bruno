# Integración con GoHighLevel (GHL) — Zerochats

## Visión general

Zerochats usa GoHighLevel como CRM operativo: ahí se registran los leads,
se etiquetan por plan y se gestiona el ciclo de cobros con Stripe. Nuestro
panel se alimenta de GHL por **dos vías**:

1. **Tiempo real**: webhook `ghl-registrado-webhook` que GHL llama en cuanto
   entra un lead nuevo.
2. **Diario**: cron `sync-ghl-zerochats` que reconcilia todo el estado a las
   05:00 UTC.

```
GHL ──webhook──► ghl-registrado-webhook ──► crm_data (lead nuevo)
GHL ──API──────► sync-ghl-zerochats (05:00) ──► crm_data (cuotas, planes, pagos)
```

---

## Identidades clave

| Concepto | Valor |
|---|---|
| Location ID GHL | `pJyuDyDmqRLuYm63c6Oj` |
| Record ID en `crm_data` | `zerochats_2026` |
| Email cliente (login al panel) | (config en `crm_clients`) |

---

## Webhook de registrados

### Configuración en GHL

1. **GHL → Automations → Workflows → New Workflow.**
2. **Trigger:** "Contact Created" o "Form Submitted" (según el embudo).
3. **Action: Custom Webhook**:
   - Method: `POST`
   - URL: `https://obeopzavwnquapjdjwrx.supabase.co/functions/v1/ghl-registrado-webhook`
   - Headers: `Content-Type: application/json`
   - Body (JSON):
     ```json
     {
       "ghl_contact_id": "{{contact.id}}",
       "location_id": "{{location.id}}",
       "nombre": "{{contact.first_name}} {{contact.last_name}}",
       "email": "{{contact.email}}",
       "telefono": "{{contact.phone}}",
       "tags": "{{contact.tags}}",
       "utm_source": "{{contact.attributionSource.utmSource}}",
       "utm_medium": "{{contact.attributionSource.utmMedium}}",
       "utm_campaign": "{{contact.attributionSource.utmCampaign}}",
       "utm_content": "{{contact.attributionSource.utmContent}}",
       "utm_term": "{{contact.attributionSource.utmTerm}}",
       "fecha_registro": "{{contact.date_added}}"
     }
     ```

### Detección de embudo

El webhook deriva el embudo de origen así:
1. **Prioridad 1**: tag explícito en GHL (`embudo_vsl`, `embudo_quiz`, etc.).
2. **Prioridad 2**: `utm_source` (mapeo en la propia function).
3. **Fallback**: "Directo".

### Idempotencia

La function descarta cualquier POST cuyo `ghl_contact_id` ya esté en
`S.contactados`. Esto permite reintentos seguros desde GHL si el primer envío
falla.

---

## Sync diario

### Qué hace cada noche

1. Trae **todos los contactos** del location ID vía API GHL.
2. Para cada contacto:
   - Determina plan a partir de tags (`plan_pro`, `plan_business`, `plan_anual`).
   - Determina fecha de alta y meses transcurridos.
   - Genera/actualiza la cuota correspondiente en `S.cuotas`.
   - Añade el cobro del mes en `pagos[YYYY-MM]` con la caja neta.
3. Detecta cobros recurrentes y los marca con `recurrente: true`.
4. Persiste todo con `upsert` sobre `crm_data` (record `zerochats_2026`).

### Manejo de churn

Si un contacto pierde el tag de plan (cancelación):
- La cuota existente **NO se borra** — queda con los pagos hasta el mes
  anterior.
- No se genera pago para el mes en curso.
- Ver `churn.md` para más detalle.

### Manejo de cambios de plan

Si un contacto pasa de PRO a BUSINESS:
- Se crea **una nueva cuota** con el plan BUSINESS desde el mes del cambio.
- La cuota PRO antigua se cierra (sin pagos en el mes en curso).
- Esto puede generar duplicados temporales — `dedupe-cuotas-zerochats` los
  limpia bajo demanda.

---

## Credenciales y secrets

Las edge functions leen estas variables de entorno (Supabase Dashboard →
Edge Functions → Secrets):

| Secret | Uso |
|---|---|
| `GHL_API_KEY` | Token de la location de Zerochats |
| `GHL_LOCATION_ID` | `pJyuDyDmqRLuYm63c6Oj` |
| `SUPABASE_SERVICE_ROLE_KEY` | Para escribir en `crm_data` |

Si el token GHL se invalida (cambio de password, revocación), el cron
fallará silenciosamente (response 401 de GHL). Hay que regenerarlo desde
GHL → Settings → Business Profile → API Key.

> **Nota**: a futuro convendría replicar el patrón de `meta_token_status` que
> implementamos en `sync-meta-ads` para detectar tokens caducados de GHL
> también.
