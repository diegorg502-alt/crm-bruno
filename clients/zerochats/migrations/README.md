# Migraciones manuales — Zerochats

Carpeta para guardar el SQL puntual ejecutado sobre `crm_data` del registro
`zerochats_2026`. **NO** son migraciones del schema (eso vive en Supabase
directamente); son operaciones de datos.

---

## Convención de nombres

```
YYYY-MM-DD_<slug>.sql
```

Ejemplos:
- `2026-05-18_business-caja-374.sql` — corregir caja BUSINESS de 397 a 374.
- `2026-04-30_backfill-abril.sql` — backfill de cobros abril desde CSV.

---

## Checklist obligatorio para cada migración

1. **Backup previo manual** (anotar ID resultante en el comentario del SQL):
   ```sql
   INSERT INTO crm_backups (client_id, data)
   SELECT 'zerochats_2026', data
   FROM crm_data
   WHERE id = 'zerochats_2026'
   RETURNING id;
   ```
2. **SELECT previo** con la misma lógica de filtrado que el UPDATE, para
   verificar el conteo de filas afectadas.
3. **UPDATE** scoped a `WHERE id = 'zerochats_2026'` (regla R5: scope por
   cliente).
4. **SELECT posterior** que confirme el conteo coincide.
5. **Anotar en CHANGELOG.md** la fecha, motivo y ID del backup.

---

## Patrón: actualizar valores dentro del JSONB `data`

Como `data` es JSONB, los UPDATE típicos usan `jsonb_set` o reemplazo completo
de subárboles. Patrón seguro **sin cross-join**:

```sql
-- ❌ MAL: cross-join implícito
UPDATE crm_data SET data = jsonb_set(data, '{cuotas}', (
  SELECT jsonb_agg(...) FROM crm_data, jsonb_array_elements(data->'cuotas')
))
WHERE id = 'zerochats_2026';

-- ✅ BIEN: subquery scoped al mismo registro
WITH cuotas_actualizadas AS (
  SELECT jsonb_agg(
    CASE
      WHEN c->>'plan' = 'BUSINESS'
        THEN jsonb_set(c, '{caja}', '374'::jsonb)
      ELSE c
    END
  ) AS new_cuotas
  FROM crm_data,
       jsonb_array_elements(data->'cuotas') AS c
  WHERE id = 'zerochats_2026'
)
UPDATE crm_data
SET data = jsonb_set(data, '{cuotas}', (SELECT new_cuotas FROM cuotas_actualizadas))
WHERE id = 'zerochats_2026';
```

Ver `DATA_PROTECTION_RULES.md` (raíz del repo) para la justificación de por
qué la primera forma rompe.

---

## Historial reciente

| Fecha | Archivo | Motivo | Backup ID |
|---|---|---|---|
| 2026-05-18 | `2026-05-18_business-caja-374.sql` (pendiente exportar) | Caja BUSINESS 397→374 | 113 |
| 2026-05-15 | `2026-05-15_backfill-mayo.sql` (pendiente exportar) | Backfill cobros mayo | 110 |

> Pendiente: exportar el SQL ejecutado en producción durante mayo a esta
> carpeta. Hoy vive solo en el historial de Supabase SQL Editor.
