# Recomendación · ¿Qué pedidos hay que frenar o revisar antes de despachar?

**Vidriera (tienda online) · Grupo 7 · Trabajo final Data Scientist**
*Datos sintéticos de una empresa ficticia. Población: pedidos con cobro aprobado, 01/01 al 30/09/2025; con etiqueta madura hasta el 13/08/2025.*

## Evidencia: qué muestran los datos

- El fraude es raro: **0,67 % de los pedidos** (IC 95 % 0,62 %–0,72 %), unos **3 por día entre 480**. El contracargo llega entre 10 y 61 días después del pedido.
- **El 80 % del fraude llega en ráfagas**: el mismo cliente compra varias veces en menos de una hora. En esos pedidos el fraude es del 15 %, casi 100 veces más que en el resto. Las cuentas nuevas y los correos con usuario genérico también tienen más fraude.
- **Tres ideas intuitivas no se sostienen**: el fraude no es más caro que el resto, verificar el teléfono no se asocia con menos fraude y el envío al exterior no marca una diferencia significativa.
- En la prueba (julio y agosto, simulando que el sistema se puso en marcha el 1/7), **revisar los 40 pedidos de mayor riesgo de cada día captura el 86 % de los fraudes y el 83 % del monto**. Revisar los 40 más caros (la referencia de hoy) captura el 10 %.
- El modelo (XGBoost) y una **regla simple de cinco señales capturan lo mismo** (86,8 % ambos; diferencia 0,0 % ± ~4 puntos) y cuestan lo mismo. Una red neuronal captura unos 30 puntos menos.

## Interpretación: qué creemos que significa

- **El valor está en revisar los pedidos correctos, no en el algoritmo.** Con la misma gente se pasa de capturar 1 de cada 10 fraudes a casi 9 de cada 10.
- La ráfaga es la firma del fraude **en esta tienda y en este período**. Es una asociación: comprar seguido no es sospechoso en sí, y muchos clientes buenos lo hacen. Por eso los pedidos van a una persona y no se bloquean solos.
- **Con 40 lugares por día, bloquear automáticamente casi nunca conviene**: hay lugar para revisar a todos los pedidos de riesgo alto, e incluso en el tramo más riesgoso hay entre 3 y 7 clientes buenos por cada fraude.

## Recomendación

**Poner en marcha la retención y revisión por prioridad** con el modelo `1.1.0`:

- **Retener** para revisión los pedidos con probabilidad ≥ 0,94 % (entre 36 y 45 por día, según el período); el equipo revisa los 40 de mayor riesgo.
- **Bloquear** solo si la probabilidad es ≥ 41,6 %, o si un retenido no entra en la cola del día y supera el 9,7 %.
- **Aprobar** el resto.
- Mantener la **regla de puntos como respaldo**: si en dos meses con etiqueta madura el modelo captura menos que la regla, se vuelve a la regla.

Costo estimado de los fraudes que pasan más la fricción: de **~$82.700 a ~$32.800 por día**, un ahorro de unos **$1,5 millones por mes**. Con otros supuestos razonables de costo, el ahorro se mantiene entre el 45 % y el 70 %. El precio es demorar unas horas a ~36 clientes buenos por día (7,7 % de los pedidos buenos).

## Una limitación

Con unos 140 fraudes en la prueba **no podemos afirmar que el modelo sea mejor que la regla**, solo que es igual de bueno. Además, la etiqueta solo ve el fraude que se reclama y llega con dos meses de atraso, y los costos son supuestos. El más incierto, y el que más cambia la política, es **cuántos clientes buenos abandonan si se los frena**.

## Un siguiente paso

Un **piloto de 8 semanas**: el modelo decide, la regla corre en paralelo, cada revisión registra si el analista confirmó fraude (señal en horas, no en meses) y se mide el abandono real de los clientes frenados. Con eso se confirma o se corrige el umbral de bloqueo y se decide entre modelo y regla con datos propios.
