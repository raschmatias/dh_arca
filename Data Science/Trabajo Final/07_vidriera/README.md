# Trabajo final Data Scientist · Grupo 7 · Vidriera

**Pregunta oficial:** ¿Qué pedidos hay que frenar o revisar antes de despachar?
**Decisión que habilita:** cuáles se bloquean, cuáles se revisan a mano (capacidad: 40 por día) y cuáles se aprueban.
**Tipo de problema:** clasificación extremadamente desbalanceada con etiquetas demoradas.

*Vidriera es una empresa ficticia y los datos son sintéticos: se generan por código en el notebook.*

## El caso en cinco líneas

Vidriera es una tienda online que paga cada contracargo que le reclaman. El fraude es raro (0,67 % de los pedidos, unos 3 por día entre 480) y **se confirma tarde**: el contracargo llega entre 10 y 61 días después del pedido. Hoy se revisan a mano 40 pedidos por día; si se eligen por importe, se captura el 10 % del fraude. **Ordenando los pedidos por riesgo, con la misma gente, se captura el 86 %.** El modelo (XGBoost) y una regla de cinco señales rinden lo mismo, porque el 80 % del fraude llega en ráfagas; proponemos el modelo con la regla como respaldo y un piloto para decidir con datos propios.

## Los seis entregables

| # | Entregable | Dónde |
| --- | --- | --- |
| 1 | Notebook reproducible | [`07_Vidriera.ipynb`](07_Vidriera.ipynb): de los datos sucios a la recomendación, ejecutado de punta a punta |
| 2 | Ficha de evidencia | En el notebook, sección 9 |
| 3 | Ficha de modelo y plan de monitoreo (una página) | [`ficha_modelo_y_monitoreo.md`](ficha_modelo_y_monitoreo.md) (también en la sección 9 del notebook) |
| 4 | Tablero de resultados (6 gráficos, uno por pregunta) | En el notebook (sección 6) y en [`img/tablero_vidriera.png`](img/tablero_vidriera.png); tablas para Superset en [`superset/`](superset/) con la guía para armarlo |
| 5 | Recomendación (una página) | [`recomendacion.md`](recomendacion.md) (también en la sección 9 del notebook) |
| 6 | Defensa (4 min + 3 de preguntas) | [`defensa/defensa_grupo7_vidriera.pdf`](defensa/defensa_grupo7_vidriera.pdf) y el guion con preguntas probables en [`defensa/guion_defensa.md`](defensa/guion_defensa.md) |

Además, el artefacto del modelo y su contrato: [`modelo/`](modelo/) (`vidriera_fraude_1.1.0.joblib`, metadatos `.json`, `contrato_entrada.yaml`).

## Cómo correrlo

En Codespaces o local, con Python 3.10 o superior: abrir `07_Vidriera.ipynb`, ejecutar la celda de instalación una vez y después **Run All**. Tarda unos 2 minutos. El notebook vuelve a crear `img/`, `modelo/` y `superset/` en esta carpeta. Todo usa semillas fijas, así que los números se reproducen.

## Qué hay en cada sección del notebook

| Sección | Módulo | Lo principal |
| --- | --- | --- |
| 1 | Introducción | Definición del problema: población (pedidos con cobro aprobado), período, granularidad, objetivo (contracargo «no reconoce la compra») y qué se sabe al decidir |
| 2 | Python | 12 problemas de calidad anotados en la bitácora: reintentos duplicados, fechas UTC y locales, importes en 4 formatos, variantes de país, correo y provincia. 22 variables construidas solo con información previa |
| 3 | Estadística | Demora del contracargo → ventana de maduración de 62 días. Tasa de fraude con IC de Wilson. Siete comparaciones con chi cuadrado y Bonferroni: la ráfaga multiplica el fraude casi 100 veces; teléfono verificado y envío al exterior no muestran diferencia; el fraude no es más caro |
| 4 | Machine Learning | Validación temporal simulando un despliegue el 1/7 con brecha de maduración. Referencias, regresión logística, Random Forest, XGBoost con pesos. Controles de fuga. Bootstrap por días. Calibración y dos umbrales por costo |
| 5 | Deep Learning | Red densa (3 semillas): captura ~30 puntos menos que XGBoost en la cola. No se justifica |
| 6 | Superset | Operación simulada del 1/7 al 30/9 y tablero de 6 gráficos; exportación a SQLite |
| 7 | Producción | Reentrenamiento (versión 1.1.0), contrato YAML, servicio con 422 y falla segura, prueba de humo de 11 chequeos, ~5 ms por pedido, deriva (PSI) y plan de monitoreo con responsables |
| 8 | Negocio | Costo de cada error, costo de cinco políticas, sensibilidad a los supuestos, explicación global y ficha del analista con los motivos de cada pedido |
| 9 | Fichas | Ficha de evidencia, ficha de modelo, recomendación |

## Cómo se cubren los criterios de calidad

| Criterio | Dónde se ve |
| --- | --- |
| 1 · Responde la pregunta y la decisión | Dos umbrales + cola por prioridad (4.8), recomendación (9) |
| 2 · Población, período, granularidad y objetivo | Sección 1 y resumen de población al final de la 2 |
| 3 · Ruta reproducible | Notebook de punta a punta con semillas; bitácora de calidad; artefacto versionado con huella de datos |
| 4 · Referencia y métricas acordes al costo | Dos referencias (4.3); recall en la cola de 40, monto capturado, PR-AUC, fricción y costo en pesos |
| 5 · Calidad, fuga, sesgo y generalización | Chequeos de integridad (2.6), prueba de punto en el tiempo y etiquetas inmaduras (4.5), validación temporal, calibración por tramos, deriva |
| 6 · Evidencia, interpretación y recomendación separadas | `recomendacion.md` y sección 9 |
| 7 · Limitación y siguiente paso | Recomendación, ficha de evidencia y defensa |

**Resultados negativos que dejamos a la vista:** el modelo no supera a la regla de cinco señales; la red neuronal no se justifica; bloquear automáticamente casi nunca conviene con la capacidad actual; teléfono verificado, envío al exterior e importe alto no se asocian con más fraude.

## Supuestos a validar

- **Costos:** comisión de contracargo $3.000; margen 30 %; 40 % de los clientes buenos bloqueados abandona; el analista detecta el 85 % del fraude que revisa. La sensibilidad está en 8.2.
- **Cómo revisa hoy Vidriera:** la consigna no lo dice; usamos «los 40 pedidos más caros del día» como referencia de la situación actual.
- **Logística:** retener un pedido unas horas no demora la entrega (despacho al día hábil siguiente).
