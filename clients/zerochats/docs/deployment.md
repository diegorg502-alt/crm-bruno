# Deployment — Zerochats

## Stack

| Pieza | Plataforma | URL/ref |
|---|---|---|
| Panel web | Vercel | `panel-de-metricas.vercel.app` (Diego) + `zerochats.panel-de-metricas.vercel.app` (Zerochats) |
| Backend datos | Supabase | proyecto `obeopzavwnquapjdjwrx` |
| Edge functions | Supabase Functions | desde `/supabase/functions/` |
| CRM operativo | GoHighLevel | location `pJyuDyDmqRLuYm63c6Oj` |
| Pagos | Stripe | conectado a GHL |

---

## Subdominio `zerochats.panel-de-metricas.vercel.app`

### Cómo se configuró (paso a paso, una vez)

1. Entrar a [vercel.com](https://vercel.com) → proyecto `panel-de-metricas`.
2. **Settings → Domains → Add Domain**.
3. Escribir: `zerochats.panel-de-metricas.vercel.app`.
4. Vercel detecta que es un subdominio de su propio dominio (`vercel.app`),
   no pide DNS, lo configura automáticamente en segundos.
5. En la lista de dominios, marcar como **alias del branch `main`** (no
   branch-specific).

### Detección en frontend

`index.html` mira `window.location.host` en `showApp()`. Si el host empieza
por `zerochats.`:
- Resuelve el record llamando a `crm_clients` con `auto_renew = true` (o por
  record_id `zerochats_2026`).
- Llama directamente a `loadClientConfig(email_zerochats)`.
- **Oculta** `#admin-client-selector` aunque el usuario sea admin.
- Cambia el título del navegador a `Zerochats — CRM`.

Resultado:
- Diego entra a `panel-de-metricas.vercel.app` → ve admin con dropdown.
- Cualquiera (incluido Diego) entra a `zerochats.panel-de-metricas.vercel.app`
  → ve solo Zerochats, sin dropdown.

### Para volver a admin desde el subdominio

Si Diego está en el subdominio y quiere volver a admin: cambiar la URL a
`panel-de-metricas.vercel.app` manualmente.

---

## Despliegue de edge functions

Todas las edge functions de Zerochats se despliegan con el Supabase CLI desde
la raíz del repo:

```bash
# Sync diario
supabase functions deploy sync-ghl-zerochats --project-ref obeopzavwnquapjdjwrx

# Webhook de registrados (sin JWT)
supabase functions deploy ghl-registrado-webhook --project-ref obeopzavwnquapjdjwrx --no-verify-jwt

# Utilidades manuales
supabase functions deploy dedupe-cuotas-zerochats --project-ref obeopzavwnquapjdjwrx
supabase functions deploy reconcile-zerochats-csv --project-ref obeopzavwnquapjdjwrx
```

> **Importante**: el flag `--no-verify-jwt` es obligatorio para el webhook
> porque GHL no puede mandar un JWT. La URL es el secret.

---

## Variables de entorno necesarias

En el Dashboard de Supabase → Edge Functions → Manage secrets:

| Secret | Quién la necesita |
|---|---|
| `GHL_API_KEY` | `sync-ghl-zerochats`, `ghl-registrado-webhook` |
| `GHL_LOCATION_ID` | `sync-ghl-zerochats` |
| `SUPABASE_SERVICE_ROLE_KEY` | todas (para escribir en `crm_data`) |
| `SUPABASE_URL` | inyectado automáticamente |

---

## Backups antes de operaciones masivas

Antes de ejecutar `dedupe-cuotas-zerochats` o `reconcile-zerochats-csv` con
`dry_run: false`, **siempre** crear backup manual:

```sql
INSERT INTO crm_backups (client_id, data)
SELECT 'zerochats_2026', data
FROM crm_data
WHERE id = 'zerochats_2026';
```

Aparte, hay un cron diario que crea `crm_daily_backups` a las 03:00 UTC. El
ID del backup se anota en `migrations/` con la operación que se vaya a hacer.

---

## Rollback

Si algo se rompe:

```sql
-- 1. Ver backups recientes
SELECT id, created_at FROM crm_daily_backups
WHERE client_id = 'zerochats_2026'
ORDER BY created_at DESC LIMIT 10;

-- 2. Restaurar uno concreto
UPDATE crm_data
SET data = (SELECT data FROM crm_daily_backups WHERE id = <ID_BACKUP>)
WHERE id = 'zerochats_2026';
```

Después de cualquier rollback, **el cliente tiene que cerrar todas las
pestañas del panel** (su localStorage cacheado puede sobreescribir el
rollback al hacer cualquier edición).
