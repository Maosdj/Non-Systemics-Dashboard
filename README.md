# Queries del Panel de Flujo Outbound · COCU02

Documentación técnica de los dos queries BigQuery que alimentan el dashboard de Shipping.
Site: COCU02 (MCO) · Mercado Libre Colombia.

---

## Índice

1. [Resumen](#1-resumen)
2. [Arquitectura](#2-arquitectura)
3. [Query maestro](#3-query-maestro)
4. [Query vejez](#4-query-vejez)
5. [Reglas compartidas](#5-reglas-compartidas)
6. [Costos](#6-costos)
7. [Validación contra WMS](#7-validación-contra-wms)
8. [Limitaciones](#8-limitaciones)
9. [Changelog](#9-changelog)
10. [Mantenimiento](#10-mantenimiento)

---

## 1. Resumen

Dos queries corren en paralelo cada 5 minutos:

| Query | Versión | Output | Filas | Scan |
|---|---|---|---|---|
| Panel (`panel_flujo_cpts_v7_14_10_flat.sql`) | v7.14.10 | Agregado, schema flat 8 columnas | 200-400 | 25-30 GB |
| Vejez (`dashboard_vejez_v2_6_1.sql`) | v2.6.1 | Row-level, schema wide 9 columnas | ≤10.000 | 8-12 GB |

**Por qué dos queries separados:**
- Schemas incompatibles (flat vs wide).
- Granularidad distinta (agregado vs paquete individual).
- Falla independiente (si vejez rompe, el panel sigue).
- Costos optimizables por separado.

**Ambos comparten:**
- Fuentes `BT_SHP_MT_FOS_METRICS` ⨝ `BT_SHP_MT_OUTBOUND`.
- CASE de 5 estados WMS.
- CTE `CPTS_DERIVADOS` (≥50 órdenes en 30d).
- Filtros anti-zombie 48h.
- Timezone `America/Bogota` (−1h sobre timestamps).

---

## 2. Arquitectura

```
BigQuery (meli-bi-data.WHOWNER)
   |
   |-- BT_SHP_MT_FOS_METRICS (estado + ZONE)
   |-- BT_SHP_MT_OUTBOUND (timestamps, status, SLAs)
   |-- BT_FBM_STOCK_3_MOVEMENT (movimientos, operario)
   |-- DM_PRODUCTIVITY_ATTRIBUTES (presencia NS)
   |-- LK_KRAKEN_USERS (operario -> LDAP, solo vejez)
   |
   v
[Query Maestro] -- output flat 8 cols, ~17 secciones
[Query Vejez]   -- output wide 9 cols, row-level
   |
   v
Verdi Flows (n8n) -> push cada 5 min
   |
   v
Grid (doc_id 01KT54NA28F8Y5S92H2MHV2QH1) -> panel_outbound.html
```

---

## 3. Query maestro

Archivo: `panel_flujo_cpts_v7_14_10_flat.sql` · v7.14.10

### 3.1 Diseño

Un solo query produce N "secciones" identificadas por la columna `SECTION`. El JS del HTML filtra por `SECTION` para construir cada tabla.

**Ventajas vs N queries separados:**
- Single scan de `OUTBOUND` (-80% costo).
- Consistencia: todas las secciones miran el mismo snapshot.
- Agregar sección = 1 CTE + 1 UNION ALL.

### 3.2 Schema flat (8 columnas)

| Columna | Tipo | Significado |
|---|---|---|
| `SECTION` | STRING | Discriminador de fila (ej. `listo_para_packear`, `canalizacion`, `forecast_ns_requeridos`) |
| `CPT` | STRING | `"HH:MM"` del CPT, o nombre de banda en forecast NS, o `"all"` para KPIs site-wide |
| `RANK_CPT` | INT64 | 1..5 orden cronológico, o `hora_offset` (0/1) en forecast NS |
| `HORAS_AL_CPT` | FLOAT64 | Horas decimales al CPT, o hora absoluta en forecast NS |
| `INTENSITY_PCT` | FLOAT64 | % participación. Denominador varía por sección |
| `AGING_BUCKET_LABEL` | STRING | Comodín: `"F##"`, `"HH:00"`, `"#RRGGBB"`, nombre de día, fecha |
| `CANALIZACION` | STRING | Código ZONE, banda, `DIMENSION_TYPE` o nombre de día |
| `METRICA` | INT64 | El número (órdenes o unidades según sufijo `_uds`) |

### 3.3 Capa de fundación

| CTE | Función |
|---|---|
| `PARAMS` | Constantes: `NS_TS_SEG_PAQ=16.463`, `NS_UTILIZACION=0.825`, `NS_VENTANA_VOL_MIN=90`, `NS_BACKLOG_MIN_H=0.5` |
| `CPTS_DERIVADOS` | Detecta dinámicamente (CPT, DOW) con ≥50 órdenes en 30d. Reemplaza hardcode |
| `FINGER_LK` | Mapping ZONE → finger 1..43 (hardcoded en STRUCT) |
| `OUTBOUND_ENRICH` | Single scan de OUTBOUND. Materializa timestamps, status, dimensions, units, packing_used |
| `FOS_BASE` | JOIN FOS_METRICS ⨝ OUTBOUND_ENRICH + CASE 5 estados + filtros anti-zombie 48h |

### 3.4 CASE de 5 estados (FOS_BASE)

Validado contra WMS Dispatch SLA 20/06/2026.

```sql
CASE
    WHEN GATE_OUT_DATE IS NOT NULL OR STATUS='shipped'              THEN 'despachado'
    WHEN HU_DATE_FINISH_LOCAL IS NOT NULL
         OR STATUS='handling_unit_closed'                            THEN 'hu_cerrada'
    WHEN HU_DATE_START_LOCAL IS NOT NULL                             THEN 'dentro_de_hu'
    WHEN PACKING_DATE_FINISH_LOCAL IS NOT NULL
         OR STATUS IN ('packed','documented','to_document','split')  THEN 'empacado'
    WHEN STATUS='ready_to_pack'
         OR STATUS IN ('grouping','ready_to_group')
         OR PACKING_DATE_READY_LOCAL IS NOT NULL                     THEN 'listo_para_packear'
    ELSE 'pre_packing'
END
```

**Reglas críticas (no romper):**

| Regla | Razón |
|---|---|
| Orden de evaluación: más avanzado primero | Un paquete con `HU.DATE_START` y `PACKING.DATE_FINISH` simultáneos es `dentro_de_hu`, no `empacado` |
| No usar `WALL.DATE_FINISH` como gate de `dentro_de_hu` | WALL.DATE_FINISH es nivel-wall, no nivel-paquete. Causaba 372 query / 106 WMS (bug v7.14.9) |
| `'packed'` en bucket empacado | Status más confiable que `PACKING.DATE_FINISH` (que tiene lag de sync) |
| `PACKING_DATE_READY_LOCAL` evaluado al final | Paquetes ya empacados pueden tener este campo poblado pero `PACKING.DATE_FINISH` aún no |
| No reintroducir gate de edad >90min | WMS Dispatch SLA cuenta toda HU abierta como "HU in" sin importar antigüedad |

### 3.5 Capa de agregación

| CTE | Función |
|---|---|
| `AGREGADO` | Pivot `(CPT × FECHA_ETD × SECTION) → COUNT, SUM(UNITS)` |
| `AGREGADO_ZONE` | Desagrega todos los estados por `(CPT × ZONE × estado)`. Incluye `pre_packing` con ZONE='SIN_ZONE' |
| `AGREGADO_ZONE_TOTAL_POR_CPT` | Suma `AGREGADO_ZONE` sobre estados. Alimenta SECCION_CANALIZACION |
| `TOTALES_ZONE_POR_CPT_SECTION` | Denominador de INTENSITY_PCT para canalización × estado |
| `CPTS_VALIDOS` | Top 5 CPTs futuros con volumen ≥1, DOW match |
| `CPTS_RANKED` | Asigna RANK_CPT (1..5) y calcula HORAS_AL_CPT |
| `TOTALES_POR_CPT` | TOTAL_CPT (incluye pre_packing) y TOTAL_FLUJO (4 estados pre-despacho) |

### 3.6 Capa de presentación (17 secciones)

| CTE | SECTION | Tabla HTML |
|---|---|---|
| `SECCIONES` ×5 | `listo_para_packear`, `empacado`, `dentro_de_hu`, `hu_cerrada`, `despachado` | Estado por CPT |
| `SECCION_TOTAL` | `total` | Header CPT |
| `SECCIONES_UDS` ×5 | `*_uds` | Estado por CPT (toggle Unidades) |
| `SECCION_TOTAL_UDS` | `total_uds` | Header CPT en uds |
| `SECCION_ETIQUETAS_DIA` | `etiquetas_dia` | Día de la semana |
| `SECCION_CANALIZACION` (+`_UDS`) | `canalizacion`, `canalizacion_uds` | Col Total por finger |
| `SECCION_CANALIZACION_X_ESTADO` (+`_UDS`) | `canalizacion_packear`/`_empacado`/`_dentro_hu`/`_hu_cerrada` | Cols por estado en finger |
| `SECCION_CANALIZACION_FALTANTES_HU` (+`_UDS`) | `canalizacion_faltantes_hu` | Col Faltantes HU |
| `SECCION_FORECAST_NS_VOLUMEN` | `forecast_ns_volumen` | Forecast NS · Vol |
| `SECCION_FORECAST_NS_ASIGNADO` | `forecast_ns_asignado` | (interno) |
| `SECCION_FORECAST_NS_REQUERIDOS` | `forecast_ns_requeridos` | Forecast NS · NS req |
| `SECCION_FLUJO_NS_ACTUAL` (+`_UDS`) | `flujo_ns_actual` | KPIs Flujo NS · ahora |
| `SECCION_FLUJO_NS_DIA` (+`_UDS`) | `flujo_ns_dia` | KPIs Flujo NS · día |
| `SECCION_SNAPSHOT_TIME` | `snapshot_time` | Header "Datos a las HH:MM:SS" |

### 3.7 Bloque Forecast NS

**Fórmula:** `HC = CEIL(demanda_efectiva_h × peso_banda × TS / 3600 / utilización)`

**Pipeline:**

```
PARAMS (TS=16.463, util=0.825, ventana=90min)
   |
   v
NS_HORA_ACTUAL (ventana móvil 90min rolling)
   |
   |-- NS_VOL_PACKIN (paq empacados en mesa, excluye SIOC)
   |-- NS_VOL_RS (picks RS desde MZ-0)
   |       |
   |       v
   |   NS_DEMANDA_ACTUAL (throughput vivo = packin_h + rs_h)
   |
   |-- NS_BANDA_ACTIVIDAD (mesas WT-% con cualquier movimiento)
   |       |
   |       v
   |   NS_VOL_POR_BANDA (peso = mesas_activas_banda / total_activas)
   |
   |-- NS_BACKLOG_DEMANDA (EDF: tasa requerida para vaciar backlog al CPT)
   |       |
   |       v
   |   NS_DEMANDA_EFECTIVA = MAX(throughput, backlog)
   |
   |-- NS_DM_HOY -> NS_ASIGNADO_ACTUAL (presencia desde DM_PRODUCTIVITY)
   |
   v
NS_FORECAST_FINAL (cross join + fórmula + color hex)
   |
   v
SECCION_FORECAST_NS_VOLUMEN / _ASIGNADO / _REQUERIDOS
```

**Decisiones clave:**

| Decisión | Razón |
|---|---|
| `NS_VOL_PACKIN` excluye SIOC | Desde v7.13.1, `PACKING.DATE_FINISH` se popula para SIOC. Sin filtro se cuenta 2× (1 en packin_h, 1 en rs_h). 200 paq contaminados detectados 15:55 |
| `peso_banda` por mesas activas, no por throughput | Señal packing/put_wall capturaba <2% del volumen real. Banda Antigua quedaba en peso 0 estando llena. Ahora: # mesas con cualquier movimiento como proxy |
| `NS_DEMANDA_EFECTIVA = MAX(throughput, backlog)` | En banda colapsada, throughput está topado por gente disponible y subcuenta NS faltantes. Backlog rate (EDF single-server) calcula tasa para no incumplir CPT |
| Horizonte 2h (hora actual + siguiente) | TL planifica con 1h. Horas +2/+3/+4 producían caídas artificiales sin valor operativo |
| TS=16.463 fijo, no auto-calibrable | Los NS no escanean → `DM_PRODUCTIVITY` siempre devuelve UNITS=0. Recalibración manual con nuevo estudio de tiempos |

**Escala de color (NS requeridos):**

| NS req | Hex |
|---|---|
| ≥7 | `#1F5F00` |
| 5-6 | `#3D8B00` |
| 3-4 | `#7FB069` |
| 1-2 | `#C5E1A5` |
| 0 | `#E6F4E6` |

### 3.8 Bloque KPIs Flujo NS (v7.14.10)

Cumple Bench Shipping Standardization (RTD 13). Adaptado a COCU02: en lugar del split pescador/ingresador del Bench original, se desagrega por `DIMENSION_TYPE` (conveyable / non_conveyable / bulky).

4 CTEs, sin scan adicional (leen de `FOS_BASE`):

| CTE | Universo | Métrica |
|---|---|---|
| `SECCION_FLUJO_NS_ACTUAL` | `SECTION IN (LPP, EMP, DHU)` + `DIMENSION_TYPE NOT NULL` | COUNT órdenes |
| `SECCION_FLUJO_NS_ACTUAL_UDS` | idem | SUM(UNITS) |
| `SECCION_FLUJO_NS_DIA` | `FECHA_ETD = CURRENT_DATE` + `DIMENSION_TYPE NOT NULL` | COUNT órdenes |
| `SECCION_FLUJO_NS_DIA_UDS` | idem | SUM(UNITS) |

Output: 3 filas por sección (una por DIMENSION_TYPE).

**Justificación del no-split pescador/ingresador:** TS validado 16.463 s/paq corresponde al ciclo completo (BANDA 7.444 + HU 9.019) hecho por un mismo operario. Volumen COCU02 (2-4k paq/día) vs sites Bench (10k+) no justifica especialización de roles.

### 3.9 UNION final

19 UNION ALL → 17 secciones distintas:
- 5 SECCIONES + 5 SECCIONES_UDS
- 7 secciones de canalización (incluye 3 estados × {órdenes, uds} + 1 total × {órdenes, uds} + 1 faltantes × {órdenes, uds})
- 3 secciones de forecast NS
- 4 KPIs Flujo NS
- `total`, `total_uds`, `etiquetas_dia`, `snapshot_time`

`ORDER BY SECTION, RANK_CPT NULLS LAST`.

---

## 4. Query vejez

Archivo: `dashboard_vejez_v2_6_1.sql` · v2.6.1

### 4.1 Diseño

Espejo del maestro a nivel de paquete individual. Solo alimenta la tab "Paquetes Rezagados".

**Principios:**
- Mismo CASE de 5 estados → consistencia visual con el maestro.
- Mismo `CPTS_DERIVADOS` → match 1:1 con celdas del panel.
- INNER JOIN a `CPTS_VALIDOS_VEJEZ` → -83% universo escaneado.

### 4.2 Schema wide (9 columnas)

| Columna | Tipo | Significado |
|---|---|---|
| `ID` | STRING | Shipment ID |
| `CPT` | STRING | `"HH:MM"` del CPT |
| `FINGER` | STRING | Código ZONE destino |
| `PROCESS_PATH` | STRING | TOT_SINGLE_SKU, NON_TOT_MONO_SIOC, etc. |
| `DIM_TYPE` | STRING | conveyable / non_conveyable / bulky |
| `ESTADO` | STRING | Uno de los 5 estados |
| `AGING_MIN` | INT64 | Minutos desde último movimiento operativo |
| `ULTIMA_ACTUALIZACION` | STRING | `"HH:MM"` si <24h, `"DD/MM HH:MM"` si ≥24h |
| `RESPONSABLE` | STRING | LDAP del operario, o `"sin_responsable"` |

`ORDER BY AGING_MIN DESC LIMIT 10000`.
`WHERE ESTADO IN ('listo_para_packear','empacado','dentro_de_hu')` — el TL no interviene en HU cerrada ni despachado.

### 4.3 CTEs

| CTE | Función |
|---|---|
| `CPTS_DERIVADOS` | Réplica exacta del maestro. Mantener sincronizado |
| `CPTS_VALIDOS_VEJEZ` | Genera ETD candidatas hoy/mañana × DOW match, top 5 futuras |
| `ULTIMO_MOVIMIENTO` | Última fila de `BT_FBM_STOCK_3_MOVEMENT` filtrada por procesos con operario humano (packing, put_wall, outbound_picking_order) |
| `ULTIMO_MOVIMIENTO_AMPLIO` | Fallback: cualquier movimiento, sin filtro de proceso ni `FBM_USER_ID` |
| `RESPONSABLES` | Mapping `operario_id → LDAP` desde `LK_KRAKEN_USERS` |
| `OUTBOUND_LITE` | Slim de `OUTBOUND_ENRICH`: solo campos para el CASE y presentación |
| `FOS_BASE_VEJEZ` | JOIN principal: FOS_METRICS ⨝ OUTBOUND_LITE ⨝ movimientos ⨝ responsables ⨝ CPTS_VALIDOS_VEJEZ |

### 4.4 Cascada de responsable y aging

**Responsable:**
```
1. ULTIMO_MOVIMIENTO.FBM_USER_ID → RESPONSABLES.ldap
2. ULTIMO_MOVIMIENTO_AMPLIO.FBM_USER_ID → RESPONSABLES.ldap
3. 'sin_responsable'
```

**Timestamp (AGING_MIN + ULTIMA_ACTUALIZACION):**
```
1. ULTIMO_MOVIMIENTO.FBM_CREATED_DATE − 1h
2. ULTIMO_MOVIMIENTO_AMPLIO.FBM_CREATED_DATE − 1h
3. F.AUD_UPD_DTTM − 1h
```

Garantiza que siempre haya valor, incluso para paquetes fuera de la ventana de 2 días.

---

## 5. Reglas compartidas

Cambiar en uno requiere cambiar en el otro.

| Regla | Implementación |
|---|---|
| Orden CASE: más avanzado → menos avanzado | despachado > hu_cerrada > dentro_de_hu > empacado > listo_para_packear > pre_packing |
| Timezone Bogotá | `DATETIME_SUB(timestamp, INTERVAL 1 HOUR)` sobre todos los campos `*_LOCAL` |
| Anti-zombie 48h | `DATETIME_DIFF(NOW, AUD_UPD_DTTM, HOUR) <= 48` + idem sobre timestamps operativos |
| Exclusiones | `CANCELLED.DATE IS NULL`, `STATUS != 'cancelled'`, `SLA_IGNORED = FALSE`, `UNSHIPPED_FINAL_STATUS_AT IS NULL` |
| Universo CPTs | `CPTS_DERIVADOS` con ≥50 órdenes en 30d + DOW match |

---

## 6. Costos

### Maestro

| Métrica | Valor |
|---|---|
| Scan | 25-30 GB |
| Slot-ms | ~120k |
| Latencia p50 / p95 | 25 / 45 s |
| Costo/corrida | ~$0.12 USD |
| Costo/día (288 runs) | ~$35 USD |
| Output | 200-400 filas, ~50 KB |

### Vejez

| Métrica | Valor |
|---|---|
| Scan | 8-12 GB |
| Slot-ms | ~60k |
| Latencia p50 / p95 | 15 / 30 s |
| Costo/corrida | ~$0.05 USD |
| Costo/día | ~$15 USD |
| Output | ≤10k filas, ~1.5 MB |

**Total mensual:** ~$1.500 USD.

### Optimizaciones aplicadas

| Optimización | Impacto |
|---|---|
| Single scan `OUTBOUND_ENRICH` | -80% costo OUTBOUND |
| INNER JOIN `CPTS_VALIDOS_VEJEZ` | -83% universo vejez |
| Lookback OUTBOUND limitado a 7d | Evita scan completo |
| Filtro `INHUB_DATE` en FOS_METRICS | Aprovecha partition pruning |
| `QUALIFY` vs subqueries con RANK | Single-pass dedup |

---

## 7. Validación contra WMS

Maestro alineado 1:1 con WMS Dispatch SLA desde v7.14.9.

| Fecha | CPT | Estado | Query | WMS | Match |
|---|---|---|---|---|---|
| 20/06/2026 | 20:00 | hu_cerrada | 1.426 | 1.426 | exacto |
| 20/06/2026 | 20:00 | empacado | 1.201 | 1.201 | exacto |
| 20/06/2026 | 20:00 | dentro_de_hu | 106 | 106 | exacto |
| 23/06/2026 | 19:00 | dentro_de_hu | (sin gate 90min) | match | OK |
| Viernes piso 14:31 | — | NS asignados | 4 | 4 | visual |

**Regla de regresión antes de tocar el CASE:**
1. Snapshot WMS en un CPT.
2. Correr query con cambio.
3. Comparar conteos por estado.
4. Documentar diff en changelog.

---

## 8. Limitaciones

| ID | Limitación | Estado |
|---|---|---|
| L1 | Banda Antigua en send-day | Resuelto v7.14.0 (peso por volumen vivo, no por histórico de operario) |
| L2 | NS no escanean → sin productividad medida | Permanente. TS calibrado manualmente. Margen ±1 NS/banda/hora |
| L3 | Forecast NS solo cubre paquetes de banda (conveyable + SIOC). No incluye RK ni Bulky | Permanente. KPIs Flujo NS sí muestran Bulky/Non-Conv como info |
| L4 | TS calibrado solo en viernes | Pendiente. Recalibrar en sábado y parametrizar `NS_FACTOR_DIA` |
| L5 | `FINGER_LK` hardcoded | Deuda. Migrar a fuente dinámica |
| L6 | Vejez sin drill-down a CPTs vencidos | Trade-off. Ampliar LIMIT en `CPTS_VALIDOS_VEJEZ` si se necesita |
| L7 | RS picks cancelados cuentan en vejez | Ruido <1%. No corregir hasta confirmar impacto |

---

## 9. Changelog

### Maestro

| Versión | Cambio |
|---|---|
| v7.14.10 | KPIs Flujo NS (Conv/Non-Conv/Bulky) para Bench |
| v7.14.9 | Alineación WMS Dispatch SLA: removido gate WALL→DHU, agregado `'packed'` a empacado, reordenado gate PACKING_READY |
| v7.14.8 | INTENSITY_PCT en unidades para canalización × estado |
| v7.14.7 | Fix total_uds: filtro explícito 5 estados visibles |
| v7.14.6 | SECCION_CANALIZACION_UDS (espejo en unidades) |
| v7.14.5 | Cleanup Process Path (tab obsoleta removida) |
| v7.14.4 | SECCION_CANALIZACION_FALTANTES_HU |
| v7.14.3 | NS_VOL_PACKIN excluye SIOC (anti-doble-conteo) |
| v7.14.2 | Estudio de tiempos integrado (n=211, TS=16.463 s/paq) |
| v7.14.1 | Horizonte forecast 5h → 2h |
| v7.14.0 | Reemplazo NS_OP_BANDA_DOMINANTE por NS_VOL_POR_BANDA |
| v7.13.6 | Toggle Órdenes/Unidades (secciones espejo `_UDS`) |
| v7.13.1 | Migración schema WMS (nuevos timestamps OUTBOUND) |
| v7.13.0 | Rename `despachados` → `despachado`, refactor secciones |
| v7.7.5b | Fix RS directo (Plan D) |
| v7.7 | PARAMS centralizado |
| v7.4 | Removidas secciones tiempo baker (sesgo HU.DATE_START) |
| v7.3 | Canalización incluye todos los DIMENSION_TYPE |
| v7.0 | Sección Canalización × Estado |
| v6.3 | Schema flat 8 columnas establecido |

### Vejez

| Versión | Cambio |
|---|---|
| v2.6.1 | LIMIT 5k → 10k (días pico cortaban ~834 paquetes) |
| v2.6 | Scope limitado a 5 CPTs visibles vía CPTS_VALIDOS_VEJEZ |
| v2.5 | Anti-zombie por aging operativo real ≤48h |
| v2.4 | CASE de 5 estados alineado con maestro |
| v2.3 | Cascada de responsable + fallback AMPLIO |
| v2.0 | ULTIMA_ACTUALIZACION con formato dinámico (<24h vs ≥24h) |
| v1.0 | Versión inicial |

---

## 10. Mantenimiento

### Deuda técnica activa

| ID | Tarea | Prioridad | Esfuerzo |
|---|---|---|---|
| DT-001 | Migrar `FINGER_LK` hardcoded a fuente dinámica | Media | 2 días |
| DT-002 | Recalibrar TS en sábado y parametrizar `NS_FACTOR_DIA` | Media | 1 día piso |
| DT-003 | Sincronizar `CPTS_DERIVADOS` entre maestro y vejez (view) | Baja | 4 h |
| DT-004 | Log `LOG_FORECAST_GAP` (forecast vs real) | Baja | 1 día |
| DT-005 | Módulo forecast RK/Bulky separado | Futura | 1 semana + estudio |
| DT-006 | Push al TL si `backlog_h > capacidad_instalada` | Futura | 3 días |

### Checklist

**Por release:**
- Validar conteos query vs WMS Dispatch SLA en un CPT random.
- Revisar latencia y costo en BQ query history.
- Verificar snapshot timestamp sin lag >5 min.

**Mensual:**
- Auditar `CPTS_DERIVADOS` (CPTs nuevos / muertos).
- Revisar mesas en `NS_MAPEO_MESA`.
- Verificar `FINGER_LK` vs layout físico.

**Cuando cambia operación:**
- Recalibrar `NS_TS_SEG_PAQ` con nuevo estudio.
- Actualizar `NS_MAPEO_MESA` si se reasigna mesa.
- Actualizar `FINGER_LK` si se reordenan fingers.

**Cuando cambia WMS:**
- Validar timestamps de OUTBOUND siguen poblados igual.
- Validar status (`packed`, `ready_to_pack`, etc.) no cambiaron.
- Re-validar CASE contra WMS.

---

**Documentos relacionados:**
- `README.md` — Dashboard general.
- `panel_outbound.html` — Frontend.
- `estudio_tiempos_ns_cocu02_master.xlsx` — Estudio jun 2026 (n=211).

**Owner técnico:** Mauricio Sierra · `ext_mausierr@mercadolibre.com.co` · +57 305 4154 616

**Última actualización:** v7.14.10 (maestro) + v2.6.1 (vejez) · Junio 2026
