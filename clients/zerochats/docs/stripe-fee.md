# Comisión Stripe en Zerochats

## Estructura de la comisión

Stripe cobra (en España, plan estándar 2026):
- **1.5%** + **0.25 €** sobre tarjetas del Espacio Económico Europeo.
- **2.5%** + **0.25 €** sobre tarjetas internacionales.

Para una venta BUSINESS de 397 € con tarjeta EEE:
```
fee = 397 × 0.015 + 0.25 = 5.955 + 0.25 = 6.205 €
neto = 397 − 6.205 = 390.795 €
```

> Pero en Zerochats la comisión real observada es **~23 €** (caja 374 €).
> Esa diferencia incluye otras retenciones que aplican al setup actual
> (impuestos sobre la fee, fee adicional por suscripción recurrente, posibles
> chargebacks promediados, etc.).

**Lo importante**: no tratamos de calcular la fee, **la asumimos como
constante** por plan. Si Stripe cambia su estructura o Zerochats migra de
configuración, hay que actualizar las constantes `PLAN_CAJAS` en dos sitios
(`index.html` y `sync-ghl-zerochats`).

---

## Por qué separar facturación y caja

Tres motivos:

1. **Contabilidad**: el cliente factura 397 € al usuario, eso es lo que va al
   IRPF / declaración. La caja (374 €) es lo que entra al banco — relevante
   para flujo de caja pero no para facturación fiscal.
2. **KPI de marketing**: el ticket medio que mostramos en el panel es el de
   **facturación**, porque es el que importa al cliente para tomar decisiones
   de pricing.
3. **KPI de tesorería**: la caja por cobrar y la caja recibida son lo que se
   muestra como métrica financiera real.

---

## Cómo se traduce en el panel

En la tabla de cuotas hay dos columnas (cuando `HAS_TICKET_MENSUAL`):

| Columna | Origen | Ejemplo BUSINESS |
|---|---|---|
| Facturación | `row.facturacion` (autorrelleno por plan) | 397 € |
| Caja | `row.pagos[mes]` (autorrelleno por plan) | 374 € |

El "Bola de nieve" / "Caja por cobrar" usa **caja**.
El KPI "Facturación total" del mes usa **facturación**.
El "Ticket medio" mostrado al cliente usa **facturación**.

---

## Qué hacer si Stripe cambia comisiones

1. Calcular la nueva caja neta para PRO y BUSINESS (Diego pasa el dato real
   desde Stripe Dashboard → Settings → Pricing).
2. Actualizar `PLAN_CAJAS` en `index.html` Y en `sync-ghl-zerochats/index.ts`.
3. Actualizar `CAJA_TO_FACTURACION` en `index.html`.
4. **Backfill opcional** de las cuotas existentes si la diferencia es
   significativa (ver `migrations/` para el patrón).
5. Documentar el cambio en `pricing.md` con fecha y motivo.
