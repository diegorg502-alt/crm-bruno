# Edge Functions — Zerochats

Las edge functions reales viven en `/supabase/functions/<name>/index.ts` porque
así las espera el CLI de Supabase para desplegarlas. Aquí solo se documenta el
**propósito** de cada una, **cómo se invocan** y **cuándo tocarlas**.

---

## 1. `sync-ghl-zerochats`

**Ruta canónica:** `/supabase/functions/sync-ghl-zerochats/index.ts`

**Propósito:** Trae diariamente los contactos de GoHighLevel (location ID
`pJyuDyDmqRLuYm63c6Oj`) y los normaliza al modelo de Zerochats:
- Detecta plan (PRO / BUSINESS / ANUAL) por tag.
- Calcula meses de servicio, ticket de facturación y caja neta (tras comisión
  Stripe).
- Construye las cuotas + el array de pagos persistentes (`pagos[YYYY-MM]`).
- Detecta cobros recurrentes vs primer cobro y los marca con
  `recurrente: true`.

**Cron:** diario a las 05:00 UTC.

**Deploy:**
```bash
supabase functions deploy sync-ghl-zerochats --project-ref obeopzavwnquapjdjwrx
```

**Variables clave dentro del código:**
- `PLAN_TICKETS = { PRO: 247, BUSINESS: 397, ANUAL: 0 }`
- `PLAN_CAJAS   = { PRO: 247, BUSINESS: 374, ANUAL: 0 }`
- `RECORD_ID    = 'zerochats_2026'`

---

## 2. `ghl-registrado-webhook`

**Ruta canónica:** `/supabase/functions/ghl-registrado-webhook/index.ts`

**Propósito:** Endpoint público (sin JWT) al que GHL llama en cuanto entra un
lead nuevo (registrado / opt-in). Captura nombre, email, teléfono, UTMs y el
embudo de origen, y lo añade en tiempo real al CRM de Zerochats.

**Endpoint:**
```
POST https://obeopzavwnquapjdjwrx.supabase.co/functions/v1/ghl-registrado-webhook
Content-Type: application/json
```

**Mapeo location → record:**
```js
LOCATION_TO_RECORD = { 'pJyuDyDmqRLuYm63c6Oj': 'zerochats_2026' }
```

Si en el futuro se añade otro cliente con webhook de registrados, ampliar este
diccionario. **No tiene auth**: la URL es el secret.

**Deploy:**
```bash
supabase functions deploy ghl-registrado-webhook --project-ref obeopzavwnquapjdjwrx --no-verify-jwt
```

**Idempotencia:** por `ghl_contact_id`. Si el contacto ya existe, no se
duplica.

---

## 3. `dedupe-cuotas-zerochats`

**Ruta canónica:** `/supabase/functions/dedupe-cuotas-zerochats/index.ts`

**Propósito:** Script puntual para limpiar duplicados en `S.cuotas` causados
por el sync GHL cuando un contacto migra entre tags. **No tiene cron**:
se ejecuta a mano cuando hace falta.

**Cuándo usarlo:**
- Tras un cambio masivo de tags en GHL.
- Si Diego detecta cuotas con mismo `email + mes` pero distinto `recurrente`.

**Modo dry-run obligatorio:**
```bash
curl -X POST \
  https://obeopzavwnquapjdjwrx.supabase.co/functions/v1/dedupe-cuotas-zerochats \
  -H "Authorization: Bearer $SERVICE_ROLE" \
  -d '{"dry_run": true}'
```

Solo cuando el output sea el esperado, se llama con `dry_run: false`.

---

## 4. `reconcile-zerochats-csv`

**Ruta canónica:** `/supabase/functions/reconcile-zerochats-csv/index.ts`

**Propósito:** Importar un CSV histórico de Zerochats (export desde GHL o
Stripe) y reconciliarlo contra `S.cuotas`. Útil para backfill o migración
inicial.

**Cuándo usarlo:**
- Onboarding de un periodo histórico que GHL no expone vía API.
- Auditoría: verificar que `S.cuotas` coincide con el CSV de Stripe.

---

## Resumen de despliegues

| Función | Cron | Trigger | Modifica `crm_data` |
|---|---|---|---|
| `sync-ghl-zerochats` | 05:00 UTC diario | Cron | Sí |
| `ghl-registrado-webhook` | — | HTTP POST desde GHL | Sí |
| `dedupe-cuotas-zerochats` | — | Manual | Sí (con backup) |
| `reconcile-zerochats-csv` | — | Manual | Sí (con backup) |

Todas requieren `service_role` key salvo el webhook (que es público).
