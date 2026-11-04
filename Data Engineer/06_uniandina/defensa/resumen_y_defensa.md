# Trabajo final Data Engineer · Grupo 6 · UniAndina — Resumen y defensa

3/11/2026 · Mati

> Versión editable: el documento original está en Claude (Docs). Este archivo es una copia para el repo.
> La presentación de la defensa está en `defensa_grupo6_uniandina.pdf`, en esta misma carpeta.

## Qué hicimos

La plataforma de UniAndina decide **a qué 200 alumnos llama Tutorías al empezar cada cuatrimestre**, y lo hace con datos en los que se puede confiar. Los diez módulos están en el notebook `Data Engineer/06_uniandina/trabajo_final_uniandina.ipynb`. Se ejecuta de punta a punta sin errores y reconstruye todo desde las fuentes en cada corrida.

| Módulo | Qué hicimos | El número para recordar |
| --- | --- | --- |
| Diagnóstico | Medimos con código los problemas de las 7 fuentes y decidimos qué hacer con cada uno | 27 problemas, 15 decisiones |
| 1 · Arquitectura | Lakehouse liviano: los archivos crudos se guardan tal cual y las zonas curada, variables y consumo son tablas SQL | Unos 2,5 MB en total: no justifica un clúster |
| 2 · Modelado | Modelo conceptual, lógico y físico. Estrella con un hecho por persona × materia × cuatrimestre y la dimensión alumno con historia (SCD 2). Los datos personales van en una tabla aparte | 12 de 12 restricciones rechazan los datos malos del diagnóstico |
| 3 · Carga (PL/SQL) | Carga incremental de la dimensión alumno: une los dos legajos, guarda la historia del plan, controla antes y después y revierte todo si algo falla | 2.000 personas, 299 versiones nuevas. Optimizada: de 1,5 s a 0,6 s con 100.000 personas |
| 4 · Ingestión | Flujo NiFi diagramado y script equivalente. Contratos versionados, cuarentena por archivo y por fila, balance y semáforo | Todos los lotes cierran el balance; becas en amarillo (6,2 %) |
| 5 · Orquestación | DAG con dependencias, calendario, reintentos solo para fallas transitorias y regla para los datos que llegan tarde | 158 corridas en 2025; una rectificación espera la reapertura |
| 6 · Gobierno | Roles por dato, 10 reglas de calidad medidas en el destino y en el origen, y un contrato de datos ejecutable para las notas | 6 de 10 cumplen; 42 alumnos figuran aprobados y ausentes a la vez |
| 7 · MLOps | Modelo de riesgo de abandono sin información posterior al corte. Dos candidatos contra la regla existente, ficha, promoción, monitoreo y marcha atrás | v2 encuentra 21 % de los abandonos contra 17 % de la regla; la fuga daría 0,95 de AUC |
| 8 · Plataformas | Cuatro zonas con reglas de paso verificadas y arquitectura de ML por lotes | 9 de 9 reglas de paso se cumplen |
| 9 · Seguridad y metadatos | Clasificación de todas las columnas, seudónimo con clave, enmascaramiento por rol, vista filtrada para Tutorías, auditoría, catálogo y linaje | 133 columnas; el linaje coincide en 80 de 80 valores |
| 10 · Proyecto | Matriz de riesgos, backlog priorizado, guion de la defensa y checklist | 10 riesgos, 9 tareas en el backlog |

## Decisiones

Cada decisión arranca por un hecho del caso, descarta al menos una alternativa y dice qué control avisa si sale mal. Casi todas las fallas las decide la **responsable de datos académicos**, una persona designada por la Secretaría Académica, no por el equipo de datos.

| # | Decisión | Por qué (hecho del caso) | Qué descartamos y por qué | Control |
| --- | --- | --- | --- | --- |
| 1 | Lakehouse liviano: lo crudo como archivo intacto, lo demás en tablas | Los archivos cambian de formato sin aviso y hay que poder reprocesar | Warehouse clásico con ETL: pierde el original. Spark + Delta: un clúster para 2,5 MB | Balance de filas por lote |
| 2 | La persona es el DNI: un hecho por persona × materia × cuatrimestre | 60 personas tienen dos legajos | Grano por legajo: cuenta doble a 60 personas | Restricción única en el motor |
| 3 | Dimensión alumno con historia (SCD 2), carga incremental por conjuntos | La base pisa el plan: 300 alumnos ya figuran con el nuevo | SCD 1 (pisar): pierde en qué plan cursó cada uno. Reconstruir siempre desde `cambios_plan`: si un cambio no se registra ahí, se pierde sin aviso | Una sola versión vigente, sin fechas superpuestas; si no, se revierte todo |
| 4 | Datos personales en una tabla aparte (`persona_pii`) y seudónimo con clave | El caso marca DNI, nombre, fecha, correo y legajo como sensibles | Enmascarar dentro de la dimensión: queda a un permiso de distancia. El primer seudónimo repetía el legajo y lo corregimos | Regla de paso: sin datos personales en consumo |
| 5 | Contrato versionado: la escala se declara, no se adivina | Las notas de 2025-1c cambiaron de columnas y de escala (0–100) | Inferir la escala mirando los datos: un cuatrimestre con notas bajas se cargaría mal en silencio | Archivo con columnas desconocidas → cuarentena entero |
| 6 | Cuarentena en dos niveles: archivo entero si se rompe la forma, fila si falla el dato | 26 notas malas sobre 7.956 | Rechazar el archivo por cualquier fila: frena 7.930 notas buenas | Semáforo: rojo si se rompe el contrato, amarillo si más del 5 % va a cuarentena |
| 7 | El acta del docente prevalece sobre el CSV de notas | Son dos copias de la misma nota; difieren en 45 ausentes | Tomar la última que llega: el resultado dependería del orden | Diferencias registradas en el lote |
| 8 | El DAG corre todos los días y la primera tarea decide si sigue | Diario en cierre de actas, semanal el resto (consigna) | Dos DAGs: pueden pisarse. Un *timetable* a medida: difícil de mantener | 158 corridas en 2025 (123 + 35) |
| 9 | Reintentos solo para fallas transitorias | Reintentar un contrato roto no sirve y demora el aviso | Reintentar todo por igual | Aviso de falla por fuera del DAG |
| 10 | Un cuatrimestre queda abierto 60 días; después, cambiarlo requiere reapertura | Las actas se rectifican después del cierre | Aplicar siempre lo último: cambia un indicador ya informado sin aviso | Huella por cuatrimestre |
| 11 | Reglas de calidad en el destino **y** en el origen, con umbral y dueño | La carga arregla sola problemas del origen, que sin regla quedan invisibles | Solo controles de carga. «Cero errores»: una alerta que suena siempre deja de mirarse | Resultados por lote en `resultado_calidad` |
| 12 | Predecir al inicio del cuatrimestre, sin las cursadas «abandonó» | 209 de 247 que abandonan ya tenían materias abandonadas: el abandono había empezado | Usarlas: AUC 0,95, pero «detecta» a quienes ya se fueron | Controles *point-in-time* y criterio AUC < 0,90 |
| 13 | Promover un modelo solo si le gana a la regla existente en promedio y en todas las repeticiones | Una sola partición de los datos daba resultados distintos | Elegir el de mejor AUC en una partición: v1 casi empataba con la regla | 7 criterios de promoción; v1 rechazado, v2 promovido |
| 14 | Marcha atrás automática a la regla si una variable deriva (PSI > 0,25) | La biblioteca no informó julio | Seguir con el modelo hasta conocer la etiqueta: meses de una lista mal ordenada | PSI por variable en cada corte |
| 15 | Acceso por nivel de clasificación: un nivel arriba se enmascara, dos se quita; Tutorías por una vista filtrada | Ley 25.326 y datos de estudiantes | Una matriz rol × columna a mano: con 133 columnas, la nueva queda abierta | Auditoría de accesos, también los denegados |
| 16 | Puntuación por lotes y registro de modelos propio | Una lista por cuatrimestre y un equipo chico | Puntuación en línea; MLflow + *feature store* para un solo modelo | Regla de paso: el modelo en producción se puede reproducir |

## Supuestos

Asumimos 17 cosas que el caso no declara. Cada una está escrita en el notebook con su justificación. Si la profe pregunta «¿cómo saben que…?», la respuesta casi siempre está en esta tabla.

| # | Supuesto | Por qué | Módulo |
| --- | --- | --- | --- |
| S1 | La persona es el DNI; el legajo vigente es el que tiene cursadas | Los 60 legajos L3xxxx no tienen cursadas y comparten DNI, nombre y correo con un L2xxxx | Diagnóstico |
| S2 | La nota oficial de 2025-1c es la del acta del docente | Es el documento firmado; coincide con el CSV en el 97 % y el resto son ausentes | Diagnóstico |
| S3 | En 2025-1c la escala es 0–100 y se divide por 10; se aprueba con 4 | El 99,8 % de esas notas supera 10, y la del acta × 10 coincide con el CSV | Diagnóstico |
| S4 | La encuesta puntúa de 1 a 5 | El 98 % de las respuestas cae ahí | Diagnóstico |
| S5 | Las hojas de las actas no son carreras | Cada hoja mezcla alumnos de las cinco carreras | Diagnóstico |
| S6 | Hay materias compartidas entre carreras | El 80 % de las cursadas es de una materia asignada a otra carrera | Diagnóstico |
| S7 | La primera versión de cada alumno rige desde el 1/8/2024 | No hay fecha de alta en el plan y ningún dato es anterior | Diagnóstico |
| S8 | 1er cuatrimestre: 1/3 al 31/7. 2do: 1/8 al 31/12 | Préstamos, becas y cambios de plan caen en esas ventanas | 2 |
| S9 | Una cursada usa la versión del alumno vigente al inicio del cuatrimestre | El alumno se inscribe con el plan que tiene ese día | 2 |
| S10 | Legajo vigente = el de más cursadas; si empatan, el más antiguo | El desempate tiene que ser fijo para que la carga sea idempotente | 3 |
| S11 | Un DNI con otro nombre o fecha de nacimiento no se fusiona | Unir a dos personas distintas es peor que dejar un duplicado | 3 |
| S12 | Un cambio de plan repetido en los dos legajos vale el primero | Una persona lo tenía cargado dos veces, con fechas distintas | 3 |
| S13 | Cierre de actas: 1/7 al 15/8 y 1/12 al 15/2 | El caso no da fechas; incluye las mesas de febrero | 5 |
| S14 | Un cuatrimestre queda abierto hasta 60 días después de su fin | Da tiempo a las rectificaciones normales | 5 |
| S15 | La corrida semanal es los lunes | Los tableros quedan al día al empezar la semana | 5 |
| S16 | Tutorías puede contactar a 200 alumnos por cuatrimestre (10 %) | El caso no da la capacidad; las métricas se miden sobre esa lista | 7 |
| S17 | El abandono se observa al cierre del cuatrimestre siguiente al corte | Es la única etiqueta del caso | 7 |

## La defensa

Son 8 minutos repartidos entre cuatro personas, uno por parte de la presentación. **Mati** abre con el problema y presenta el modelo; **Gabi** cuenta la arquitectura y cierra con los futuros pasos; **JJ** explica el diseño (modelo de datos, ingestión y orquestación) y las fallas; **Ale** presenta la calidad.

| Tiempo | Quién | Parte | Diapositiva |
| --- | --- | --- | --- |
| 0:00–1:00 | Mati | El problema | 1 y 2 |
| 1:00–2:00 | Gabi | Arquitectura | 3 |
| 2:00–3:30 | JJ | Diseño | 4 y 5 |
| 3:30–4:45 | Ale | Calidad | 6 |
| 4:45–5:45 | Mati | Modelo | 7 |
| 5:45–6:45 | JJ | Fallas | 8 |
| 6:45–7:45 | Gabi | Futuros pasos | 9 |

### Qué dice cada uno

Es un guion para leer en voz alta y después decirlo con palabras propias. Cada bloque entra en su tiempo a ritmo normal.

**0:00–1:00 · Mati · El problema.** «UniAndina tiene 2.000 alumnos y quiere detectar a tiempo a los que van a abandonar. Lo formulamos como una decisión: a qué 200 alumnos llama Tutorías al empezar el cuatrimestre. Para eso los datos tienen que ser confiables, y no lo eran. Medimos 27 problemas en las siete fuentes. Los tres que más pesan: 60 personas con dos legajos, notas que cambian de columnas y de escala entre cuatrimestres, y datos personales de estudiantes.»

**1:00–2:00 · Gabi · Arquitectura.** «Elegimos un lakehouse liviano. Todo lo que llega se guarda primero tal cual, sin tocar, en la zona cruda, y después se limpia en tablas: curada, variables y consumo. Descartamos el warehouse clásico: cuando un archivo cambia de formato, el warehouse pierde el original y no hay reproceso. Descartamos Spark: son 2,5 MB, no justifican un clúster.»

**2:00–3:30 · JJ · Diseño.** «En el modelo, la persona es el DNI, no el legajo: si no, 60 personas cuentan doble. La dimensión alumno guarda una fila por cada plan de carrera. Si una persona cambia de plan, no pisamos su fila: la cerramos y abrimos otra, y cada cursada usa el plan que la persona tenía al empezar el cuatrimestre. Para llevar los datos a la plataforma usamos dos herramientas. NiFi carga la ingesta a la zona cruda: trae cada archivo tal cual llegó y, con su huella, no ingiere dos veces lo mismo. Airflow orquesta las demás capas: curada, variables y consumo, a diario en cierre de actas y los lunes el resto del año, y solo reintenta las fallas transitorias. La cuarentena tiene dos niveles: si se rompe la forma, va el archivo entero; si falla un dato, solo esa fila. Así 26 notas malas no frenan 7.930 buenas.»

**3:30–4:45 · Ale · Calidad.** «Cada dato tiene un dueño, que es un área de la universidad, y un responsable que decide cuando falla un control. Definimos 10 reglas de calidad, medidas en el destino y también en el origen. Son: R1, una sola versión vigente por persona; R2, personas con dos legajos; R3, cursadas de alumnos que no existen; R4, estado de la cursada que no contradice la nota; R5, cursadas terminadas con nota o ausente; R6, acta y CSV que coinciden en las notas; R7, becas cuyo DNI es de un alumno; R8, respuestas de la encuesta válidas; R9, actas cargadas en menos de 24 horas; y R10, lotes cuyo balance cierra. Cumplen 6. Las 4 que no se cumplen, R2, R3, R4 y R7, son problemas del origen que ahora tienen dueño. El más fuerte es R4: 42 alumnos figuran aprobados en la base y ausentes en el acta. Además, los datos personales viven aparte, cada rol ve según su nivel y cada acceso queda auditado.»

**4:45–5:45 · Mati · Modelo.** «El caso tenía una trampa. El estado «abandonó» de una materia aparece solo en quienes abandonan. Si el modelo lo usa, da 0,95 de AUC, pero detecta a gente que ya se fue. Lo sacamos, y el modelo honesto da 0,70. Lo medimos con el recall de la lista de 200: si llamáramos a 200 alumnos al azar, esperaríamos encontrar el 10 % de los abandonos. La regresión logística apenas mejoraba eso y se rechazó. El de árboles encuentra el 21 %, más del doble que el azar, y le gana en las cinco repeticiones, así que se promovió.»

**5:45–6:45 · JJ · Fallas.** «Probamos tres fallas. Llegó un archivo de notas con columnas nuevas: fue entero a cuarentena y el tablero siguió con lo último bueno. Llegó una rectificación de un acta de un cuatrimestre cerrado: entró en la zona curada, pero el tablero no cambió hasta que la responsable reabrió el cuatrimestre. Y una que no planeamos: el monitoreo detectó que la biblioteca no informó julio. El modelo volvió solo a la regla de respaldo.»

**6:45–7:45 · Gabi · Futuros pasos.** «El modelo es modesto: encuentra el 21 % de los abandonos. Sirve para ordenar llamados, no para afirmar que alguien va a abandonar. Lo primero es confirmar con Biblioteca lo de julio, y después reentrenar con otra cohorte y sumar más casos para entrenar, incluso de otras universidades. Los 42 alumnos aprobados y ausentes a la vez los revisamos con el Departamento de Alumnos. Y SQLite es un único almacén: el mayor radio de impacto. Se puede reconstruir entero desde la zona cruda, pero falta un respaldo diario probado. Todo eso está priorizado en el backlog.»

## Preguntas probables

Las tres primeras las anticipa la consigna; las demás son las que más probablemente haga la profe sobre este caso. La regla para responder: **restricción, decisión, evidencia**, y si no lo sabemos, decirlo sin inventar.

| Pregunta | Responde | Respuesta |
| --- | --- | --- |
| ¿Qué cambiaría si la latencia requerida bajara diez veces? | JJ | Hoy el dato más fresco tiene un día; diez veces menos son unas dos horas y media. Alcanza con correr el DAG cada dos horas: todo es idempotente y la huella omite lo que no cambió. Lo que no aguanta es el modelo: sus variables son por cuatrimestre, y alertas en horas pedirían señales diarias, o sea otro modelo. |
| ¿Qué componente concentra el mayor radio de impacto? | Gabi | El almacén (`dwh.db`): curada, consumo, variables, control y modelos viven ahí. Se puede reconstruir entero desde la zona cruda, que es inmutable y tiene huellas; el notebook lo hace en cada corrida. Falta un respaldo diario probado (backlog). |
| ¿Qué control podría dar falsos positivos? | Ale | Tres: el PSI de julio (puede ser el receso), el amarillo de becas (pueden ser ingresantes) y la regla R4 (el error puede estar en el acta). Ninguno bloquea la publicación: avisan y decide una persona. |
| ¿Por qué no usaron las materias abandonadas si es información del pasado? | Mati | Porque en estos datos es la etiqueta escrita de otra forma: aparece en el 100 % de los casos en quienes abandonan, y 209 de 247 ya habían abandonado materias antes del corte. El modelo «acertaría» con gente que ya se fue. Esos alumnos no se predicen: se listan como un hecho (backlog). |
| El modelo da 0,70 de AUC. ¿No es muy bajo? | Mati | Sí, es modesto, y lo decimos en la ficha. La pregunta es si le gana a lo que ya había: le gana a la regla en las cinco repeticiones y encuentra un 23 % más de abandonos. Un modelo con 0,95 sería sospechoso de fuga, y lo medimos. |
| ¿Por qué árboles y no la regresión logística, que es más explicable? | Mati | Probamos las dos. La logística no cumplió los criterios: casi empataba con la regla y en Diseño encontraba a 2 de 22. Para explicar los árboles, la ficha muestra la importancia de cada variable. |
| ¿Cómo saben que la marcha atrás no fue un falso positivo? | Mati | No lo sabemos todavía: falta que Biblioteca diga si fue el receso. Por eso la marcha atrás es a la regla, que es segura, y no a «sin lista». Es la primera tarea del backlog. |
| ¿Qué pasa si un DNI está mal cargado? | JJ | Si el mismo DNI viene con otro nombre o fecha de nacimiento, no se fusiona: va a cuarentena (S11). Unir a dos personas distintas es peor que dejar un duplicado. |
| ¿Qué hicieron con los 14 pares con distinto estado final? | Gabi | Vale el del legajo vigente, el que tiene cursadas (S10). Está en las decisiones del diagnóstico y en la carga del módulo 3. |
| ¿Por qué el acta prevalece sobre el CSV? | JJ | Es el documento del docente (S2). Coinciden en el 97 %; el resto son 45 ausentes donde el CSV traería una nota. La diferencia queda registrada. |
| ¿Cómo reprocesan un día cualquiera? | JJ | Se corre el DAG con esa fecha. Cada paso es idempotente: la huella omite los archivos iguales, las cargas son *upsert* y el hecho se republica por cuatrimestre. Lo probamos: el reproceso forzado no duplica ninguna fila. |
| ¿Por qué cuarentena y no descartar las filas malas? | JJ | Porque descartar es perder el dato sin que nadie se entere. En cuarentena queda con su motivo, y alguien decide si se corrige y se reprocesa. |
| ¿Cómo cumplen la Ley 25.326? | Ale | Datos personales aparte y seudonimizados, acceso por rol según el nivel, Tutorías solo ve a su lista y cada acceso se audita. La ficha del modelo dice para qué no se usa: becas, sanciones ni admisión. |
| ¿Por qué SQLite y no Oracle? | Gabi | El entorno del trabajo es SQLite. Las decisiones no dependen del motor: dejamos el equivalente en Oracle de la carga (`MERGE`), de la seguridad (`DBMS_REDACT`, VPD) y de la auditoría. Pasar a un motor con roles reales está en el backlog. |
| ¿Cómo ingieren y orquestan? | JJ | NiFi lleva los archivos de cada origen a la zona cruda, sin repetir los ya ingeridos (huella). Airflow orquesta curada, variables y consumo. En el notebook dejamos el flujo de NiFi diagramado y un script equivalente que hace lo mismo. |
| ¿Cuánto cuesta esto? | Gabi | Casi nada en infraestructura: 2,5 MB y unos minutos por corrida. El costo real es humano: alguien tiene que atender las alertas. Por eso hay pocas reglas, con umbral razonable y un dueño cada una. |

## Cómo ensayar

Con dos ensayos cronometrados alcanza. En el segundo, alguien que no presenta hace de profe con las preguntas de arriba.

- [ ] Cada uno lee su bloque en voz alta y lo cronometra: si se pasa, recorta un ejemplo, nunca la decisión ni la alternativa descartada
- [ ] Ensayo completo 1: los 8 minutos de corrido, sin cortar
- [ ] Ensayo completo 2, con preguntas: cada uno responde las de su rol en menos de 30 segundos
- [ ] Tener abierto el notebook en GitHub (pull request #1) por si piden ver una salida: los módulos 4, 5 y 7 tienen las pruebas de fallas

**Checklist de la revisión cruzada** (la consigna la usa para revisar a otros equipos, y a nosotros):

- [x] El problema está formulado como decisión y no como deseo: a qué 200 alumnos llama Tutorías
- [x] Las fuentes, transformaciones y consumidores son identificables: diagramas de arquitectura, DAG y linaje
- [x] Los límites de seguridad y responsabilidad están visibles: roles por dato y accesos por rol
- [x] Existe un criterio de calidad medible: 10 reglas con umbral
- [x] Hay una estrategia de reintento, reproceso o rollback: los tres, probados
- [x] Se registra evidencia suficiente para explicar qué ocurrió: `control_lote`, cuarentena, auditoría y registro de modelos
- [x] El costo o la complejidad aceptados están declarados: en cada tabla de decisión

**Si un integrante falta:** Mati toma su parte, porque el guion de cada bloque está escrito arriba.
