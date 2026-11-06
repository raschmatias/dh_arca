# Tablero de Vidriera en Apache Superset

El notebook (sección 6) genera `07_Vidriera.db` (SQLite) y un CSV por tabla. Si no están, corré el notebook de punta a punta.

## Conectar la base

1. **Settings → Database Connections → + Database → SQLite**.
2. SQLAlchemy URI: `sqlite:////ruta/completa/a/07_vidriera/superset/07_Vidriera.db` (cuatro barras: ruta absoluta). En algunas versiones hay que habilitar SQLite con `PREVENT_UNSAFE_DB_CONNECTIONS = False` en `superset_config.py`.
3. **Datasets → + Dataset**: crear uno por tabla.

## Las tablas

| Tabla | Una fila es… | Para qué |
| --- | --- | --- |
| `pedidos_puntuados` | un pedido del 01/07 al 30/09 | Probabilidad, decisión (aprobar, revisar, bloquear), decisión con la regla de hoy, tres motivos. `fraude` está vacío si la etiqueta no maduró (después del 13/08) |
| `resumen_diario` | un día | Pedidos, retenidos para revisión, bloqueados, desbordados, fraudes capturados (propuesta y hoy), clientes buenos frenados, montos. Columna `etiqueta`: `madura` o `preliminar` |
| `deriva_semanal` | una semana | PSI del puntaje y de las variables clave |
| `metricas_modelos` | un modelo | PR-AUC, ROC-AUC, recall y precisión en la cola de 40, monto capturado (tramo de prueba) |
| `politicas_costo` | una política | Fraude capturado, revisados y bloqueados por día, costo por día |
| `importancia_global` | una variable | Importancia en XGBoost |
| `bitacora_calidad` | un problema de datos | Qué se encontró y qué se decidió |

## Los seis gráficos (uno por pregunta)

Poné la definición de cada métrica en la descripción del gráfico, para que quede a la vista.

| # | Pregunta | Dataset y filtro | Tipo | Métrica |
| --- | --- | --- | --- | --- |
| 1 | ¿Qué parte del fraude frenamos antes de despachar? | `resumen_diario`, `etiqueta = 'madura'` | Línea, eje X `semana` | `SUM(fraudes_capturados) / SUM(fraudes)` y `SUM(fraudes_capturados_hoy) / SUM(fraudes)` |
| 2 | ¿Cuántos clientes buenos frenamos? | `resumen_diario` | Barras, eje X `semana` | `SUM(buenos_frenados) / COUNT(dia)`; color por `etiqueta` |
| 3 | ¿Alcanza la capacidad de revisión? | `resumen_diario` | Barras apiladas, eje X `dia` | `SUM(a_revisar)` y `SUM(desbordados)`; línea de referencia en 40 |
| 4 | ¿Cuánta plata de fraude frenamos? | `resumen_diario`, `etiqueta = 'madura'` | Barras apiladas, eje X `semana` | `SUM(monto_fraude_capturado)` y `SUM(monto_fraude) - SUM(monto_fraude_capturado)` |
| 5 | ¿Están cambiando los pedidos que llegan? | `deriva_semanal` | Línea, eje X `semana` | `psi_puntaje`, `psi_cli_pedidos_1h`, `psi_cuenta_nueva_7d`; referencias en 0,10 y 0,25 |
| 6 | ¿Por qué entra un pedido a la cola? | `pedidos_puntuados`, `decision = 'revisar'` | Tabla | `dia`, `id_pedido`, `probabilidad`, `motivo_1`, `motivo_2`, `motivo_3`, ordenada por `probabilidad` descendente |

Filtro nativo recomendado para todo el tablero: rango de `dia`.

El mismo tablero, hecho con matplotlib, está en `../img/tablero_vidriera.png` (y cada gráfico por separado en `../img/grafico_N.png`).
