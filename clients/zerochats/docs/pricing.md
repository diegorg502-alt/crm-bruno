# Modelo de precios — Zerochats

## Planes activos

| Plan | Recurrencia | Precio facturado | Caja neta (tras Stripe) |
|---|---|---|---|
| **PRO** | mensual | 247 € | 247 € (no aplica Stripe fee separada) |
| **BUSINESS** | mensual | 397 € | **374 €** |
| **ANUAL** | anual (12 meses) | variable | variable |

> El plan **ANUAL** no autorrellena. Se introduce manualmente porque el precio
> depende del descuento aplicado al cierre.

---

## La distinción crítica: facturación vs caja

Zerochats cobra por Stripe. Stripe se queda una comisión por transacción
(aprox. 23 € en el plan BUSINESS). Por eso el panel separa dos conceptos:

- **Facturación**: lo que se factura al cliente. Es lo que aparece en el
  contrato y en las facturas emitidas. Para BUSINESS = **397 €**.
- **Caja**: lo que entra realmente en la cuenta bancaria tras descontar la
  comisión de Stripe. Para BUSINESS = **374 €**.

### Implementación en código

**Frontend (`index.html`):**
```js
const PLAN_TICKETS = { PRO: 247, BUSINESS: 397 };   // → ticket / facturación
const PLAN_CAJAS   = { PRO: 247, BUSINESS: 374 };   // → caja real

if (plan === 'ANUAL') {
  row.mesesServicio = 12;
} else if (plan && PLAN_TICKETS[plan]) {
  row.facturacion   = PLAN_TICKETS[plan];
  row.caja          = PLAN_CAJAS[plan];
  row.mesesServicio = 1;
}
```

**Edge function (`sync-ghl-zerochats`):**
```ts
const PLAN_TICKETS = { PRO: 247, BUSINESS: 397, ANUAL: 0 };
const PLAN_CAJAS   = { PRO: 247, BUSINESS: 374, ANUAL: 0 };

// Al construir la cuota:
nuevaCuota.facturacion  = PLAN_TICKETS[plan];
nuevaCuota.pagos[mesYM] = PLAN_CAJAS[plan];
```

---

## Reverse-lookup: del cobro mensual a la facturación

Cuando entra un cobro recurrente en `S.cuotas`, lo que tenemos es la caja del
mes (374). Para mostrar la facturación correcta hace falta el mapeo inverso:

```js
const CAJA_TO_FACTURACION = { 247: 247, 374: 397 };

function getCuotaFacturacion(c, year) {
  if (HAS_TICKET_MENSUAL) {
    const caja = c.pagos[mes] || 0;
    return CAJA_TO_FACTURACION[caja] || caja;
  }
  return c.facturacion || 0;
}
```

Esto es lo que permite que el dashboard muestre 397 € "Facturación" aunque la
caja sea 374 €.

---

## Histórico de cambios de precio

| Fecha | Cambio | Motivo |
|---|---|---|
| 2026-05-18 | BUSINESS caja 397 → 374 | Descubrimos que veníamos contando la facturación como caja; la comisión Stripe (~23 €) no se restaba |

Cualquier nuevo cambio de precio debe registrarse aquí + actualizar las
constantes en `index.html` y `sync-ghl-zerochats/index.ts` **en el mismo PR**.
