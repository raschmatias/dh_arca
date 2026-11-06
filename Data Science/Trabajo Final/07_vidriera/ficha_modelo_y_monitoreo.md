# Ficha de modelo y plan de monitoreo · Vidriera · Grupo 7

**Modelo:** `vidriera-fraude-precontracargo 1.1.0` · **Fecha:** 15/10/2025 · **Dueño del modelo:** Ciencia de datos (Grupo 7) · **Dueño de la decisión:** Prevención de fraude
*Datos sintéticos de una empresa ficticia, generados en el notebook `07_Vidriera.ipynb`.*

## Qué hace

Antes de despachar cada pedido con cobro aprobado, devuelve la **probabilidad de que termine en un contracargo por fraude** («el titular no reconoce la compra») y una decisión:

| Probabilidad calibrada | Decisión | Qué pasa |
| --- | --- | --- |
| ≥ 41,6 % | **Bloquear** | Se frena y se pide verificación al cliente |
| ≥ 0,94 % | **Retener para revisión** | Entra a la cola por prioridad; el equipo revisa los 40 de mayor riesgo del día |
| < 0,94 % | **Aprobar** | Se despacha |
| Desborde | Retenidos que no entran en los 40 | Se bloquean si p ≥ 9,7 %; si no, se liberan |

Los umbrales salen de los costos (supuestos): un fraude que se despacha cuesta el importe más $3.000 de comisión; el analista detecta el 85 % de lo que revisa; el 40 % de los clientes buenos bloqueados abandona y se pierde el 30 % de margen.

## Por qué este modelo

**XGBoost** (300 árboles, profundidad 3, peso 10 para el fraude) sobre **16 variables**, con calibración de Platt. Se eligió en validación entre 12 configuraciones de XGBoost, una regresión logística, un Random Forest y una red neuronal.

En la prueba (1/7 al 13/8/2025, ~140 fraudes) **empata con una regla de cinco señales** en la cola de 40 (86,8 % de los fraudes capturados ambos; diferencia 0,0 % [−3,4 % ; +3,8 %]) y en costo. Se prefiere porque da una **probabilidad continua** para fijar los umbrales por costo, **se reentrena sin rediseño manual** y **explica cada pedido** con tres motivos. La regla de puntos queda como **referencia permanente y respaldo**.

## Con qué datos

| | |
| --- | --- |
| Población | Pedidos únicos con cobro aprobado (sin reintentos duplicados) |
| Entrenamiento | 01/01 al 13/07/2025 · 93.385 pedidos · 630 fraudes · huella `70f6244bc7b2da17` |
| Calibración | 14/07 al 13/08/2025 (último mes con etiqueta madura) |
| Etiqueta | Contracargo «no reconoce la compra»; madura a los **62 días** (el contracargo llega entre 10 y 61 días después) |
| Variables | Velocidad del cliente (pedidos en 1 h y 24 h, minutos desde el anterior), antigüedad de cuenta y dispositivo, correo genérico o ya usado, envío al exterior, madrugada, importe, ítems, cuotas, medio de pago. Todas usan solo información previa al pedido (verificado con una prueba de «punto en el tiempo») |
| Versión anterior | `1.0.0` (entrenada ene–mar), la que se evaluó en la prueba |

## Contrato de entrada

`modelo/contrato_entrada.yaml`. Un pedido por vez, **17 campos** con tipo, rango y valores permitidos. Campos de más se rechazan (por ejemplo `fraude` o `fecha_contracargo`, que serían una fuga). Si la solicitud no cumple → **422 y el pedido va a revisión manual** (falla segura). Respuesta: probabilidad, decisión, tres motivos y versión. Tiempo de respuesta medido: **p50 5 ms, p95 10 ms** (objetivo < 50 ms). La prueba de humo (11 chequeos) corre antes de cada publicación.

## Riesgos principales

1. **Dependencia de la ráfaga**: ~70 % de la importancia está en «pedidos del cliente en la última hora». Si quienes defraudan dejan de comprar en ráfaga, el modelo pierde la mayor parte de su poder.
2. **Etiqueta demorada**: un cambio en la forma de defraudar se confirma recién 2 meses después.
3. **Variables que se mueven con el calendario**: la antigüedad de las cuentas crece sola y corre el puntaje (PSI del puntaje 0,14 a fines de septiembre con la 1.0.0).
4. **Fricción**: ~36 clientes buenos retenidos por día (7,7 % de los pedidos buenos), con demora de horas.
5. **Lectura causal de los motivos**: explican por qué el pedido entró a la cola, no por qué es fraude.

## Plan de monitoreo

| Señal | Umbral | Acción | Responsable |
| --- | --- | --- | --- |
| Solicitudes rechazadas por el contrato (422) | > 0,5 % en un día | Revisar el sistema que arma las variables | Ingeniería de datos |
| Tiempo de respuesta p95 | > 50 ms durante 15 min | Escalar; si sigue, pasar a la regla de puntos | Plataforma |
| Retenidos por día | > 40 tres días seguidos, o < 20 | Revisar el umbral de revisión o la capacidad | Prevención de fraude |
| PSI semanal del puntaje | > 0,10 vigilar · > 0,25 actuar | Ver qué variable se movió; adelantar el reentrenamiento | Ciencia de datos |
| Precisión confirmada por analistas (señal temprana) | < 3 % semanal (hoy ~6 %) | Muestrear casos; buscar patrones nuevos | Prevención + Ciencia de datos |
| Fraude capturado con etiqueta madura | < 75 %, o por debajo de la regla, mensual | Reentrenar; si no mejora, volver a la regla | Ciencia de datos |
| Fricción a clientes buenos | > 10 % semanal | Subir el umbral de revisión | Prevención de fraude |
| Calibración | Fuera del IC 95 % dos meses seguidos | Recalibrar con el último mes maduro | Ciencia de datos |

**Reentrenamiento:** mensual, con la brecha de 62 días. Se publica una versión nueva solo si supera a la vigente **y a la regla de puntos** en el último mes maduro y pasa la prueba de humo.
