# Defensa · Vidriera · Grupo 7 · Guion de 4 minutos + preguntas

La presentación está en `defensa_grupo7_vidriera.pdf` (7 láminas + 1 de respaldo). El guion sigue el orden que pide la consigna: **decisión, evidencia, método, riesgo y siguiente paso**. Son unas 520 palabras: a ritmo normal, 4 minutos.

Roles rotativos (completar con los nombres del equipo):

| Bloque | Láminas | Rol | Quién | Tiempo |
| --- | --- | --- | --- | --- |
| 1 · Decisión y problema | 1–2 | Conducción | … | 0:00–0:50 |
| 2 · Método | 3 | Implementación | … | 0:50–1:50 |
| 3 · Evidencia y decisión | 4–5 | Validación | … | 1:50–3:10 |
| 4 · Riesgo y siguiente paso | 6–7 | Presentación | … | 3:10–4:00 |

---

## Bloque 1 · Decisión y problema (Conducción, ~50 s)

**[Lámina 1]** Vidriera paga cada contracargo que le reclaman, y revisa pedidos a mano antes de despachar, pero solo alcanza a mirar 40 por día. La pregunta es **qué pedidos frenar, cuáles revisar y cuáles aprobar** antes de despachar.

**[Lámina 2]** El problema tiene tres dificultades. Uno: el fraude es raro, **0,67 %**, unos 3 pedidos por día entre 480. Dos: **la etiqueta llega tarde**: el contracargo aparece entre 10 y 61 días después del pedido, así que los pedidos de los últimos dos meses todavía no sabemos si eran fraude. Y tres: con la regla que suponemos que se usa hoy, revisar los 40 más caros, se captura apenas el **10 %** del fraude.

## Bloque 2 · Método (Implementación, ~60 s)

**[Lámina 3]** Primero limpiamos los datos sin borrar en silencio: 3.570 pedidos duplicados por reintento, fechas en dos zonas horarias, importes en cuatro formatos. Son 12 decisiones, todas en la bitácora.

Construimos 22 variables usando **solo lo que se sabía en el momento del pedido**, y lo comprobamos: recalculadas con los datos cortados en junio dan exactamente lo mismo.

La validación simula que el sistema se pone en marcha el 1 de julio. Ese día solo hay etiquetas completas hasta fines de abril, así que entrenamos con enero a marzo, elegimos con abril, **dejamos dos meses de brecha** y medimos una sola vez en julio y agosto. Si en lugar de eso entrenábamos con las etiquetas incompletas de mayo y junio, el modelo ordenaba peor.

Comparamos dos referencias simples, regresión logística, Random Forest, XGBoost con pesos y una red neuronal.

## Bloque 3 · Evidencia y decisión (Validación, ~80 s)

**[Lámina 4]** El resultado principal: **revisar por riesgo en vez de por importe captura el 86 % del fraude**, contra el 10 %, con la misma gente.

Y un resultado negativo que no queremos esconder: **el modelo no le gana a una regla de cinco señales**. Capturan exactamente lo mismo, 86,8 %, con un intervalo de más o menos 4 puntos. Por qué: el 80 % del fraude llega en **ráfagas**, el mismo cliente comprando varias veces en menos de una hora, y eso lo ve cualquier regla. La red neuronal captura 30 puntos menos.

**[Lámina 5]** La decisión usa **dos umbrales** sobre una probabilidad calibrada. Retener para revisión desde 0,94 %; el equipo revisa los 40 más riesgosos del día. Bloquear, solo desde 41,6 %. Sale de los costos: un fraude cuesta el pedido entero; un cliente bueno revisado no cuesta nada; uno bloqueado abandona casi la mitad de las veces. Por eso, con 40 lugares, **bloquear casi nunca conviene**.

El costo diario baja de unos 83 mil a 33 mil pesos: **un ahorro de 1,5 millones por mes**. El precio: demorar unas horas a 36 clientes buenos por día.

## Bloque 4 · Riesgo y siguiente paso (Presentación, ~50 s)

**[Lámina 6]** Los riesgos. Quienes defraudan se adaptan: si dejan las ráfagas, el modelo pierde el 70 % de su señal. Y nos enteramos dos meses tarde. Por eso monitoreamos señales rápidas, como el resultado de cada revisión, y lentas, como el fraude capturado con etiqueta madura. El monitoreo ya funcionó: el puntaje se corrió en septiembre y reentrenamos la versión 1.1.

Elegimos XGBoost aunque empata porque da una probabilidad continua para los umbrales y se reentrena solo; pero **la regla queda como respaldo**: si dos meses el modelo captura menos, se vuelve a la regla.

**[Lámina 7]** Siguiente paso: un **piloto de 8 semanas** con la regla en paralelo, registrando el resultado de cada revisión y midiendo cuántos clientes abandonan si se los frena, que es el supuesto que más cambia la decisión. Gracias.

---

## Preguntas probables (3 minutos)

**¿Por qué no usaron un modelo más complejo si el fraude es tan raro?**
Lo probamos: la red neuronal captura 30 puntos menos que XGBoost con la misma validación. Con ~300 fraudes para aprender y una señal tan simple como la ráfaga, más complejidad solo agrega costo y opacidad.

**Si la regla rinde igual, ¿para qué el modelo?**
Por operación, no por desempeño: da una probabilidad continua (la regla tiene escalones: con 2 puntos retiene 39 por día y con 3, 16, sin nada en el medio), se reentrena sin rediseñar a mano y explica cada pedido. Pero lo dejamos condicionado: la regla corre en paralelo y es el respaldo. Si el equipo prefiere arrancar con la regla, la evidencia también lo sostiene.

**¿Por qué 62 días de maduración?**
Es el máximo observado de demora del contracargo (61 días) más uno. Con menos, parte del fraude reciente queda etiquetado como bueno: el 18 % de los fraudes de enero a junio todavía no se veía el 1/7.

**¿La tasa de 0,67 % es la real? La consigna decía 0,8 %.**
Es la tasa de fraude **reclamado**. Si algunos titulares no reclaman, la real es mayor. Lo declaramos como límite: el modelo aprende del fraude que se reclama.

**¿Cómo saben que no hay fuga?**
Dos controles. La prueba de punto en el tiempo (las variables no cambian si se borran los datos posteriores) y el contrato, que rechaza cualquier campo de más, como `fraude` o `fecha_contracargo`. Además, la validación es temporal con la brecha de maduración.

**¿De dónde salen los costos?**
Son supuestos explícitos: comisión de $3.000, margen del 30 %, 40 % de abandono si se frena, 85 % de efectividad del analista. Hicimos sensibilidad: el ahorro va del 45 % al 70 % con cualquier combinación. Lo que sí cambia es el umbral de bloqueo, y por eso el piloto mide el abandono.

**¿Qué le muestran al analista?**
La probabilidad y los tres motivos que más empujaron el riesgo, traducidos («3 pedidos antes de este en la última hora», «su pedido anterior fue hace 2 minutos»), y qué chequear. Con una advertencia: los motivos explican por qué el pedido entró a la cola, no prueban que sea fraude. Incluso arriba de la cola, la mayoría son clientes buenos.

**¿Qué pasa si el servicio falla o llega un pedido mal armado?**
Responde 422 y el pedido va a revisión manual: ante la duda, no se despacha sin mirar. Si la latencia supera los 50 ms, se pasa a la regla de puntos.

**¿Y si quienes defraudan dejan las ráfagas?**
Es el riesgo principal. Lo vemos primero en la precisión de la cola (los analistas dejan de confirmar fraudes) y después en el fraude capturado con etiqueta madura. La respuesta es reentrenar; si las variables actuales no alcanzan, hay que sumar señales nuevas.
