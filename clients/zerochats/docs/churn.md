# Churn y cancelaciones — Zerochats

Como Zerochats es un SaaS mensual, **churn es la métrica más crítica**. Cada
mes que un cliente no paga es caja perdida que no se recupera con un upsell
(como sí pasa con coaches).

---

## Cómo detectamos churn

El sync diario (`sync-ghl-zerochats`) reconstruye la verdad cada noche:

1. Trae todos los contactos activos desde GHL (con tag de plan).
2. Para cada cuota en `S.cuotas`:
   - Si el `ghl_contact_id` **ya no aparece** en GHL con tag de plan → el
     contacto ha churneado. La cuota se mantiene pero no se le añade pago en
     el mes en curso.
   - Si aparece pero **sin pago confirmado en Stripe** este mes → pago pendiente
     (puede ser problema de tarjeta, no necesariamente churn).
3. Para cuentas que vuelven (winback): el tag reaparece y al mes siguiente se
   reanudan los pagos.

> **Decisión consciente**: NO borramos cuotas churneadas. El histórico se
> mantiene para poder calcular LTV y reactivación.

---

## Estados de una cuota

| Estado | Significado | Cómo se detecta |
|---|---|---|
| `activo` | Pagando este mes | Tag de plan + pago confirmado |
| `pausa` | Pago pendiente, no canceló | Tag de plan presente, sin pago Stripe |
| `cancelado` | Sin tag de plan en GHL | Tag de plan ausente |

El campo `estado` se actualiza automáticamente en el cron diario.

---

## Cálculo de métricas de churn

> **Pendiente de implementar en panel.** Las fórmulas que vamos a usar:

### Churn rate mensual (logos)

```
churn_rate_mensual = clientes_perdidos_en_mes / clientes_activos_inicio_mes
```

donde:
- `clientes_activos_inicio_mes` = cuotas con `estado === 'activo'` el día 1.
- `clientes_perdidos_en_mes` = cuotas que pasaron de `activo` a `cancelado` durante el mes.

### Churn rate por revenue (MRR)

```
churn_mrr = caja_perdida_en_mes / mrr_inicio_mes
```

Más relevante que el de logos porque BUSINESS (374 €) cuenta más que PRO
(247 €).

### LTV (Lifetime Value)

```
LTV = ticket_medio × meses_promedio_activo
```

donde:
- `ticket_medio` = facturación media por plan ponderada por mix.
- `meses_promedio_activo` = `Σ mesesServicio / count(cuotas)` (excluyendo
  activos para evitar truncamiento).

---

## Señales de alerta

Patrones que deberían disparar revisión manual:

1. **Churn rate mensual > 8%**: alto para SaaS B2B, investigar onboarding o
   problemas de producto.
2. **Más de 3 cuotas en estado `pausa` al cierre de mes**: probablemente fallos
   de tarjeta acumulados, hay que avisar a Zerochats para que limpien.
3. **Caja por cobrar > 2× ticket medio**: significa que hay clientes que no
   están pagando y no estamos detectándolo.

---

## Reactivación (winback)

Si un cliente cancela y luego vuelve:
- La cuota original mantiene su `ghl_contact_id` y sus pagos históricos.
- El cron crea o reactiva la cuota cuando vuelve el tag.
- Importante: el `mesesServicio` debe **acumular** los meses históricos + los
  nuevos, no resetear. Esto está implementado en `sync-ghl-zerochats` al
  procesar cuotas existentes.

---

## TODO pendientes

- [ ] Añadir gráfico de cohort retention (mes de alta → % activos a N meses).
- [ ] Añadir métrica "Net New MRR" (alta nueva − churn − downgrade).
- [ ] Webhook desde Stripe para detectar fallos de tarjeta en tiempo real (hoy
  solo lo vemos al día siguiente vía sync).
