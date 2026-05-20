# Modelo de datos — Zerochats

Todo Zerochats vive dentro del JSONB `crm_data.data` del registro
`id = 'zerochats_2026'`. La forma de los datos es la misma que para el resto
de clientes, pero hay campos que **solo** se usan aquí.

---

## Forma global de `S`

```ts
{
  llamadas: { [YYYY-MM]: Llamada[] },
  kpis_diarios: { [YYYY-MM]: KPI[] },     // no se usa apenas (no hay Meta Ads en Zerochats todavía)
  objetivos: { [YYYY]: Objetivo },
  cuotas: Cuota[],                         // ← núcleo de Zerochats
  renovaciones: [],                        // vacío (no hay renovaciones)
  ads: {},                                  // vacío
  contactados: Lead[],                      // ← leads del webhook
  clientes: { [email]: ClienteAlfa },
  balance: { gastos: Gasto[], impuestos_pct: 25 },
  lanzamientos: [],                        // vacío
  escalado: { config: null, evaluaciones: [] },  // vacío
  meta_token_status?: { ... }              // no aplica aún
}
```

---

## `Llamada` (fila del CRM mensual)

```ts
{
  fecha: 'YYYY-MM-DD',
  hora: 'HH:MM',
  nombre: string,
  email: string,
  telefono: string,
  embudo: 'VSL' | 'Quiz' | 'Social' | 'Referido' | 'Directo',
  estado: 'Pendiente' | 'No show' | 'Realizada' | 'Cancelada',
  resultado: 'Cerrada' | 'Perdida' | 'Seguimiento' | '',
  agendoLlamada: 'SI' | 'NO' | '',   // ← clave para Zerochats
  asistencia: 'SI' | 'NO' | '',
  closer: string,
  setter: string,
  utm_source?: string,
  utm_medium?: string,
  utm_campaign?: string,
  utm_content?: string,
  utm_term?: string,
  ghl_contact_id?: string,
  source?: 'ghl' | 'manual',
  recurrente?: boolean,            // ← clave para Zerochats: cobros recurrentes
  plan?: 'PRO' | 'BUSINESS' | 'ANUAL',
  facturacion?: number,
  caja?: number,
  pagos?: { [YYYY-MM]: number }    // pagos persistentes
}
```

### Campos exclusivos / especiales en Zerochats

- **`recurrente`**: marca el cobro como recurrente (mes 2 en adelante). Se usa
  para excluirlo del cálculo de CPA y embudos (no es un lead nuevo).
- **`pagos`**: array de pagos efectivos por mes. Mientras que para otros
  clientes el cobro está implícito en `facturacion`, aquí hay que tracer mes a
  mes porque el plan se factura mensualmente.
- **`agendoLlamada`**: en Zerochats se usa criterio **estricto** (solo cuenta
  si está marcado 'SI' explícito). En el resto de clientes es laxo.

---

## `Cuota` (registro de cliente activo)

```ts
{
  id: string,
  email: string,
  nombre: string,
  plan: 'PRO' | 'BUSINESS' | 'ANUAL',
  fechaAlta: 'YYYY-MM-DD',
  mesesServicio: number,
  facturacion: number,          // PLAN_TICKETS[plan]
  caja?: number,                // PLAN_CAJAS[plan] (solo se usa para autorrelleno)
  pagos: { [YYYY-MM]: number }, // ← caja neta por mes
  source: 'ghl' | 'manual',
  ghl_contact_id?: string,
  recurrente?: boolean,
  estado: 'activo' | 'cancelado' | 'pausa'
}
```

### Por qué `pagos` y no un solo `facturacion`

En clientes coach un infoproducto se cobra **una vez** (pago único o
financiación a X meses fijos). En Zerochats el plan es **mensual sin fecha de
fin**: cada mes hay un cobro nuevo. El array `pagos[YYYY-MM]` permite:
- Mostrar la línea de tiempo de cobros reales.
- Calcular caja mensual sumando solo los pagos del mes en cuestión.
- Detectar churn (mes sin pago).
- Reconstruir el LTV histórico sin perder información si el cliente sube de
  plan.

---

## `Lead` (entrada del webhook)

```ts
{
  ghl_contact_id: string,         // ← clave de idempotencia
  nombre: string,
  email: string,
  telefono: string,
  fecha: 'YYYY-MM-DD',
  embudo: string,
  utm_source: string,
  utm_medium: string,
  utm_campaign: string,
  utm_content: string,
  utm_term: string
}
```

Se guarda en `S.contactados`. Si el lead luego se convierte en cuota (compra
plan), la cuota referencia el mismo `ghl_contact_id` para poder cruzarlos.

---

## Reglas de derivación importantes

### Leads totales (agregado mensual)

- **Zerochats** (`HAS_TICKET_MENSUAL=true`):
  `leads_totales = S.llamadas[mes].filter(r => !r.recurrente).length`
- **Resto de clientes:**
  `leads_totales = Σ S.kpis_diarios[mes].leads` (vienen de Meta Ads)

### `agendoLlamada` (cuenta como llamada agendada)

- **Zerochats**: solo si `agendoLlamada === 'SI'` explícito.
- **Resto**: `agendoLlamada !== 'NO'` (laxo, retrocompatible).

### `esRecurrente(r)`

```js
function esRecurrente(r) {
  return r.recurrente === true;
}
```

Se usa para excluir filas del cálculo de CPA, embudo y leads nuevos.

---

## Concurrencia

El registro `zerochats_2026` tiene activado el trigger
`crm_data_no_rollback_trg` (Postgres) + comprobación `lastLoadedTs` en
frontend. Esto previene que un browser cacheado sobreescriba cambios más
recientes hechos por el cron o por otra pestaña.
