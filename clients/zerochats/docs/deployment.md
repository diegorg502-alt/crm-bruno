# Deployment — Zerochats

## Stack

| Pieza | Plataforma | URL/ref |
|---|---|---|
| Panel web | Vercel | `panel-de-metricas.vercel.app` (Diego) + `zerochats-crm.vercel.app` (Zerochats) |
| Backend datos | Supabase | proyecto `obeopzavwnquapjdjwrx` |
| Edge functions | Supabase Functions | desde `/supabase/functions/` |
| CRM operativo | GoHighLevel | location `pJyuDyDmqRLuYm63c6Oj` |
| Pagos | Stripe | conectado a GHL |

---

## URL dedicada `zerochats-crm.vercel.app`

### Cómo se configuró (proyecto Vercel separado, no subdominio)

Decisión: en vez de añadir un subdominio anidado (`zerochats.panel-de-metricas.vercel.app`)
al proyecto existente, creamos un **proyecto Vercel independiente** conectado
al mismo repo + rama `main`. Ventajas: URL más limpia, logs y métricas
separados, sin tocar el dashboard de Vercel.

Setup ejecutado una vez:
```bash
# 1. Crear proyecto vacío en Vercel
npx vercel projects add zerochats-crm

# 2. Linkear el repo local al nuevo proyecto
npx vercel link --project zerochats-crm --yes

# 3. Conectar el proyecto a GitHub (auto-deploys en push a main)
npx vercel git connect https://github.com/diegorg502-alt/panel-de-metricas-legado --yes

# 4. Deploy inicial a producción
npx vercel deploy --prod --yes
```

Resultado: `https://zerochats-crm.vercel.app` operativo y vinculado al mismo
repo. Cada push a `main` desplegará **dos proyectos en paralelo** automáticamente
(`panel-de-metricas` + `zerochats-crm`).

### Detección en frontend

`index.html` mira `window.location.host` en `showApp()`. Si el host empieza
por `zerochats-crm` (o `zerochats.` como alias futuro):
- Resuelve el record vía `crm_clients.record_id = 'zerochats_2026'`.
- Llama directamente a `loadClientConfig(email_zerochats)`.
- **Oculta** `#admin-client-selector` aunque el usuario sea admin.
- Cambia el título del navegador a `Zerochats — CRM`.

Resultado:
- Diego entra a `panel-de-metricas.vercel.app` → ve admin con dropdown.
- Cualquiera (incluido Diego) entra a `zerochats-crm.vercel.app` → ve solo
  Zerochats, sin dropdown.

### Para volver a admin desde la URL de Zerochats

Si Diego está en `zerochats-crm.vercel.app` y quiere volver a admin: cambiar
la URL a `panel-de-metricas.vercel.app` manualmente.

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
