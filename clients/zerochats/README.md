# Zerochats — Documentación interna

Zerochats es el único cliente **SaaS de Agencia Legado**: no es un coach, no
vende infoproductos, no hay lanzamientos. Por eso necesita un modelo de datos,
un flujo de cobros y unas vistas distintas al resto de clientes.

Esta carpeta agrupa **todo lo específico de Zerochats**: documentación,
referencias a las edge functions que lo alimentan y migraciones SQL puntuales.

---

## Acceso al panel

- **URL principal Agencia (admin):** `panel-de-metricas.vercel.app`
  → Diego entra aquí y ve a todos los clientes con el dropdown del admin.
- **URL dedicada Zerochats:** `zerochats.panel-de-metricas.vercel.app`
  → El equipo de Zerochats entra aquí y ve **solo** sus datos. No hay dropdown
  de clientes ni modo admin visible.

La detección se hace en `index.html` por `window.location.host`. Si el host
empieza por `zerochats.` el panel:
- Carga directamente el `record_id = zerochats_2026`.
- Oculta el `<select id="admin-client-selector">`.
- Cambia el título del navegador a `Zerochats — CRM`.

> **Importante**: el subdominio `*.vercel.app` lo gestiona Vercel
> automáticamente — no requiere configuración DNS. Solo añadir el alias en el
> dashboard del proyecto (Settings → Domains → Add → `zerochats.panel-de-metricas.vercel.app`).

---

## Flags activos en `crm_clients`

| Flag | Valor | Efecto |
|---|---|---|
| `auto_renew` | `true` | Activa todo el modo Zerochats |
| `has_planes` | `true` (auto) | Plan column (PRO / BUSINESS / ANUAL) |
| `has_utms` | `true` (auto) | Captura UTMs de leads |
| `show_renovaciones` | `false` (auto) | Oculta tabla de renovaciones |
| `has_lanzamientos` | `false` | Sin lanzamientos |
| `show_alertas` | `false` (auto) | Oculta panel de alertas |
| `show_ia` | `false` (auto) | Oculta chat IA |

Internamente esto se traduce a la constante `HAS_TICKET_MENSUAL=true` en
`index.html`, que distingue Zerochats del resto en docenas de funciones:
`agendoLlamada`, `getCuotaFacturacion`, `syncCuota`, `esRecurrente`, `renderMes`,
`vistaKpisMes`, etc.

---

## Estructura de esta carpeta

```
clients/zerochats/
├── README.md           ← este archivo
├── functions/          ← punteros a las edge functions específicas
│   └── README.md
├── docs/               ← documentación funcional
│   ├── pricing.md
│   ├── stripe-fee.md
│   ├── ghl-integration.md
│   ├── data-model.md
│   ├── churn.md
│   └── deployment.md
└── migrations/         ← SQL puntual ejecutado sobre crm_data
    └── README.md
```

Las edge functions reales **no** viven aquí: viven en `/supabase/functions/`
porque ahí es donde el Supabase CLI las despliega. En esta carpeta hay solo
documentación + referencias para que todo lo de Zerochats se entienda desde un
único sitio.

---

## Reglas de oro al tocar Zerochats

1. **Scope explícito**: si un cambio es solo para Zerochats, guardarlo detrás
   de `HAS_TICKET_MENSUAL` o `auto_renew`. NO extender a otros clientes (regla
   R5 del CHANGELOG).
2. **Backup antes de UPDATE**: cualquier modificación masiva sobre
   `crm_data.data` del registro `zerochats_2026` requiere backup previo en
   `crm_backups` y verificación de conteos posterior.
3. **Idempotencia**: el webhook GHL y el sync diario deben ser idempotentes
   por `ghl_contact_id` y `(plan, mes)` respectivamente.
4. **Separación facturación vs caja**: BUSINESS factura 397€ pero cobra 374€
   en caja (comisión Stripe ~23€). Ver `docs/pricing.md` y `docs/stripe-fee.md`.
