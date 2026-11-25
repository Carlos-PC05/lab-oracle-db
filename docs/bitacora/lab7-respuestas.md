# Laboratorio 7 · Respuestas · E052

<!--
CÓMO SE RELLENA
- Copia este archivo como docs/bitacora/lab7-respuestas.md y pon tu código en el título.
- Responde con TUS números: cada respuesta cita el archivo de evidencia de donde sale (docs/bitacora/lab7/evidencia/...).
  Una cifra sin evidencia no puntúa. Las cifras del PPTX son de referencia: las tuyas serán parecidas, no iguales.
- 3 a 6 líneas por pregunta. No pegues salidas enteras: el archivo de evidencia ya las tiene.
- No borres los títulos: scripts/lab7/09-verificacion-final.sh comprueba que están todos.
-->

## Parte B · C01 · Más sesiones no es más rendimiento
1. ¿Cuántas llamadas por segundo y qué p95 tuviste con 1, 2 y 8 sesiones? ¿Qué número de sesiones recomiendas y por qué?
2. ¿Qué evento domina con 8 sesiones y qué significa en Oracle Free?
3. Comprueba la ley de Little (N = X · R) con tus tres medidas.

## C01: pool de conexiones

### 1. Rendimiento y tamaño del pool

| Sesiones | Llamadas/s | Latencia p95 |
| ---: | ---: | ---: |
| 1 | 5,2 | 291,9 ms |
| 2 | 10,6 | 275 ms |
| 8 | 7,5 | 1427,8 ms |

Los datos de 1 y 8 sesiones proceden de [C01 entender](lab7/evidencia/casos/20261125T175001Z_C01_entender.txt); los de 2 sesiones, de [C01 validar](lab7/evidencia/casos/20261125T175540Z_C01_validar.txt), que muestra el p95 redondeado a milisegundos.

Recomiendo **2 sesiones**: en esta prueba atendieron más llamadas por segundo que 8 y tuvieron un p95 mucho menor. Además, el `CPU_COUNT` observado es 2, por lo que abrir más sesiones genera cola cuando la CPU se satura. Véase [C01 diagnosticar](lab7/evidencia/casos/20261125T175122Z_C01_diagnosticar.txt).

### 2. Espera dominante con 8 sesiones

Dominó `resmgr:cpu quantum`, con 247 muestras de 324 (76,2 %). Es el tiempo que las sesiones esperan su turno de CPU por la planificación del gestor de recursos; en esta instancia Oracle Free la capacidad está limitada a 2 CPU. 

### 3. Ley de Little

Para comprobar N = X · R uso el throughput X en llamadas/s y la **latencia media** R en segundos, no el p95:

| Sesiones | Cálculo con valores mostrados | N aproximado |
| ---: | --- | ---: |
| 1 | 5,2 × 0,1880 | 0,98 ≈ 1 |
| 2 | 10,6 × 0,1866 | 1,98 ≈ 2 |
| 8 | 7,5 × 1,0456 | 7,84 ≈ 8 |

Las pequeñas diferencias se deben al redondeo y a la medición de las tandas. La salida calcula 1,0, 2,0 y 7,9, respectivamente, usando los valores sin redondear.

## Parte C · C03 · Literales y análisis completos
1. Análisis completos y tiempo con literales, con variables de enlace y con CURSOR_SHARING = FORCE (tus cifras).
2. ¿Por qué FORCE es un puente y no la solución? ¿Qué ve el DBA en V$SQL que lo delata (FORCE_MATCHING_SIGNATURE)?

1. Para 5.000 llamadas, con **literales** hubo 5.065 análisis completos y 5.700 ms de tiempo total; con **variables de enlace**, 28 y 390 ms; y con **FORCE**, 34 y 960 ms. Son tiempos totales de la prueba, no solo del análisis. Fuente: [C03 observar](lab7/evidencia/casos/20261125T180315Z_C03_observar.txt).

2. `FORCE` es un puente porque sustituye literales automáticamente y reduce los análisis, pero puede ocultar el defecto de la aplicación y afectar a la elección de planes al sustituir también valores relevantes para la selectividad. La solución es usar variables de enlace en la aplicación. En `V$SQL`, varios SQL_ID con la misma `FORCE_MATCHING_SIGNATURE` delatan sentencias que difieren en sus literales: mi muestra mostró 16 SQL_ID y 16 ejecuciones para la misma firma ([C03 diagnosticar](lab7/evidencia/casos/20261125T175839Z_C03_diagnosticar.txt)). Con FORCE, Oracle mostró un cursor con la variable generada `:"SYS_B_0"` y 5.000 ejecuciones ([C03 observar](lab7/evidencia/casos/20261125T180315Z_C03_observar.txt)).

## Parte C · C04 · El informe que se desborda a TEMP
1. Bloques de TEMP, tipo de pasada y tiempo del informe antes y después. ¿Qué hacía el disparador de logon?
2. ¿Por qué el TOP 100 cabe en memoria y el informe completo no?

1. Antes, el informe completo escribió y leyó **5.468 bloques** de TEMP, hizo **1 PASS** y tardó **4.220 ms**. Tras eliminar el disparador y usar la PGA automática, el mismo informe escribió y leyó **0 bloques**, fue **OPTIMAL** y tardó **1.250 ms**. El TOP 100 también fue OPTIMAL, con 0 bloques y 260 ms ([C04 solucionar](lab7/evidencia/casos/20261125T180425Z_C04_solucionar.txt), [C04 observar](lab7/evidencia/casos/20261125T180444Z_C04_observar.txt)). El disparador de logon imponía `WORKAREA_SIZE_POLICY=MANUAL` y limitaba `SORT_AREA_SIZE` a 1 MB ([C04 diagnosticar](lab7/evidencia/casos/20261125T180404Z_C04_diagnosticar.txt)).

2. El TOP 100 solo necesita mantener los mejores 100 candidatos durante la ordenación, mientras que el informe completo debe ordenar un millón de filas. Con el límite manual de 1 MB, la ordenación completa no cabía y recurrió a TEMP; con la PGA automática sí cupo. Las mediciones y los tipos de ejecución están en [C04 diagnosticar](lab7/evidencia/casos/20261125T180404Z_C04_diagnosticar.txt) y [C04 observar](lab7/evidencia/casos/20261125T180444Z_C04_observar.txt).

## Parte D · C07 · EXPLAIN PLAN frente al plan real
1. Copia el predicado que delata el problema en el plan real y explica de dónde sale.
2. Lecturas y p95 de V1 y V2. ¿Por qué EXPLAIN PLAN no lo vio?

1. El plan real muestra `filter(TO_NUMBER("CODIGO_POSTAL")=:B1)`. La aplicación envió `:B1` como `NUMBER`, aunque `CODIGO_POSTAL` es texto; Oracle convirtió implícitamente la columna y realizó un recorrido completo de `PEDIDO` en vez de usar el índice normal sobre esa columna ([C07 diagnosticar](lab7/evidencia/casos/20261125T180536Z_C07_diagnosticar.txt)).

2. En V1 medí **5.117 lecturas lógicas por ejecución** y un **p95 de 118,3 ms** ([C07 diagnosticar](lab7/evidencia/casos/20261125T180536Z_C07_diagnosticar.txt), [C07 entender](lab7/evidencia/casos/20261125T180529Z_C07_entender.txt)). En V2 el **p95 fue 0,7 ms**, pero la validación dejó vacía la cifra de lecturas y marcó dos fallos, por lo que no puedo dar ese valor con esta evidencia ([C07 validar](lab7/evidencia/casos/20261125T180729Z_C07_validar.txt)). El plan real capturado al solucionar sí muestra `INDEX RANGE SCAN` en V2 ([C07 solucionar](lab7/evidencia/casos/20261125T180713Z_C07_solucionar.txt)). `EXPLAIN PLAN` predijo el índice con una variable tratada como texto, sin ejecutar la consulta ni reflejar el tipo numérico usado por la aplicación; el cursor real reveló la conversión implícita.

## Parte D · C08 · E-Rows frente a A-Rows
1. Estimación y filas reales antes y después del grupo de columnas. ¿En qué línea del plan se separan por primera vez?
2. ¿Por qué el optimizador multiplicaba mal las dos selectividades?

1. En mis planes, tanto antes como después del grupo, el acceso a `CLIENTE` en la **línea 3** estima 1 fila y devuelve 0; ya en la línea 4, el acceso al índice de ciudad estima 1 y devuelve 0. La consulta encontró 0 clientes para la combinación elegida, así que esta ejecución no reproduce la diferencia esperada ni demuestra una mejora tras el grupo ([C08 entender](lab7/evidencia/casos/20261125T180817Z_C08_entender.txt), [C08 diagnosticar](lab7/evidencia/casos/20261125T180827Z_C08_diagnosticar.txt), [C08 solucionar](lab7/evidencia/casos/20261125T180852Z_C08_solucionar.txt)). La validación confirma que el grupo existe, pero falla en la estimación posterior y ya no encuentra la estimación anterior en la caché ([C08 validar](lab7/evidencia/casos/20261125T181008Z_C08_validar.txt)).

2. Sin estadísticas conjuntas, el optimizador puede tratar `CIUDAD` y `COMUNIDAD` como independientes y multiplicar sus selectividades. Sin embargo, en los datos observados cada ciudad pertenece como máximo a una comunidad; conocer la ciudad aporta información sobre la comunidad. El grupo de columnas permite recoger esa relación, aunque mis datos no permiten demostrar aquí la mejora esperada ([C08 entender](lab7/evidencia/casos/20261125T180817Z_C08_entender.txt), [C08 solucionar](lab7/evidencia/casos/20261125T180852Z_C08_solucionar.txt)).

## Parte E · C12 · Estadísticas obsoletas
1. Filas según las estadísticas y reales; método de unión y lecturas antes y después.
2. ¿Por qué la tarea automática no lo habría arreglado a tiempo? ¿Quién debe recoger y cuándo?

1. Antes de recoger, las estadísticas indicaban **100 filas** y había **200.100 reales**; el plan usaba **NESTED LOOPS** y el informe hizo **212.396 lecturas lógicas** ([C12 entender](lab7/evidencia/casos/20261125T181809Z_C12_entender.txt), [C12 diagnosticar](lab7/evidencia/casos/20261125T181816Z_C12_diagnosticar.txt), [C12 provocar](lab7/evidencia/casos/20261125T181803Z_C12_provocar.txt)). Después, las estadísticas reflejaron **200.100 filas**, el plan usó **HASH JOIN** y mostró **5.797 buffers** ([C12 solucionar](lab7/evidencia/casos/20261125T181949Z_C12_solucionar.txt), [C12 validar](lab7/evidencia/casos/20261125T182221Z_C12_validar.txt)). La comparación automática de lecturas falló porque no encontró el cursor anterior en caché; la evidencia de provocar conserva su cifra.

2. La tarea automática está habilitada, pero depende de ventanas de mantenimiento: no garantiza una recogida inmediatamente después de una carga masiva y antes del primer informe. En mi salida la siguiente ventana figuraba a las 22:00 ([C12 observar](lab7/evidencia/casos/20261125T182306Z_C12_observar.txt)). El proceso o job responsable de la carga debe recoger las estadísticas de `T6.CARRITO` al terminar la carga y antes de ejecutar el informe; en la solución se usó `DBMS_STATS.GATHER_TABLE_STATS` ([C12 solucionar](lab7/evidencia/casos/20261125T181949Z_C12_solucionar.txt)).

## Parte E · C13 · El ciclo de estadísticas con red
1. Describe el ciclo que seguiste (pendientes, probar, publicar, restaurar, bloquear) con la vista que comprueba cada paso.
2. ¿Qué error obtuviste al recoger sobre estadísticas bloqueadas y cómo se recoge si de verdad hace falta?

1. Configuré `PUBLISH=FALSE` y recogí estadísticas pendientes: `DBA_TAB_PENDING_STATS` y `DBA_TAB_STATISTICS` mostraron ambas versiones con **1.000.000 filas**. Activé `OPTIMIZER_USE_PENDING_STATISTICS` solo en la sesión y consulté `PLAN_TABLE`: la estimación fue **980.000**, frente a 980.000 reales. Publiqué con `PUBLISH_PENDING_STATS`, devolví `PUBLISH` a `TRUE` y comprobé que no quedaban pendientes y que el histograma era `FREQUENCY` ([C13 solucionar](lab7/evidencia/casos/20261125T182543Z_C13_solucionar.txt), [C13 validar](lab7/evidencia/casos/20261125T182848Z_C13_validar.txt)). Después consulté `DBA_TAB_STATS_HISTORY` (retención: **31 días**), ejecuté `RESTORE_TABLE_STATS` y comprobé `DBA_TAB_COL_STATISTICS`: el histograma seguía siendo `FREQUENCY`, así que esta ejecución no demuestra una vuelta al estado sin histograma. Finalmente bloqueé las estadísticas; `DBA_TAB_STATISTICS.STATTYPE_LOCKED` mostró `ALL`, y después las desbloqueé ([C13 observar](lab7/evidencia/casos/20261125T182900Z_C13_observar.txt)).

2. Al intentar recoger con las estadísticas bloqueadas obtuve **ORA-20005: object statistics are locked (stattype = ALL)**. Si hace falta recogerlas, puede hacerse conscientemente con `DBMS_STATS.GATHER_TABLE_STATS('T6', 'PEDIDO', force => TRUE)` o desbloquearlas primero y recoger normalmente. En la práctica se desbloquearon antes de la recogida final; no se ejecutó `force => TRUE` ([C13 observar](lab7/evidencia/casos/20261125T182900Z_C13_observar.txt)).

**Propuesta antes de ejecutar la solución:** recogería estadísticas pendientes con un histograma de `ESTADO`, probaría en una sesión que las use y compararía la estimación con las **980.000 filas reales** y las **500.000 estimadas** inicialmente. Si la prueba mejora la estimación, publicaría las estadísticas; conservaría la opción de restaurar y bloquearía las estadísticas representativas cuando corresponda. Esto evita cambiar los planes de todas las sesiones antes de probar ([C13 entender](lab7/evidencia/casos/20261125T182407Z_C13_entender.txt), [C13 diagnosticar](lab7/evidencia/casos/20261125T182413Z_C13_diagnosticar.txt)). En mi provocación se actualizaron 0 filas y `STALE` aparece como `NO`, por lo que no doy por reproducido el cambio de datos esperado.

**Comparación:** la solución recogió estadísticas pendientes, las probó en una sesión y las publicó, como había propuesto. La estimación pasó de **500.000 a 980.000 filas**, igual a las reales de la medición previa. La solución también restableció `PUBLISH=TRUE`; no restauró ni bloqueó estadísticas en esta fase. Mi propuesta de bloquearlas solo sería apropiada cuando unas estadísticas representativas deban mantenerse estables: bloquearlas sin ese motivo podría dejar estimaciones obsoletas. La prueba usa `EXPLAIN PLAN`, de modo que el ajuste de la estimación no demuestra por sí solo una mejora del tiempo de ejecución ([C13 solucionar](lab7/evidencia/casos/20261125T182543Z_C13_solucionar.txt)).

## Parte E · C14 · El índice invisible
1. Lecturas por apertura del panel antes y después. ¿Cómo probaste el índice sin que lo viera nadie más?
2. Redo con el índice INVISIBLE y VISIBLE: ¿qué demuestra?

1. El panel pasó de **5.153 a 4 lecturas lógicas por apertura**. Creé el índice como `INVISIBLE` y activé `OPTIMIZER_USE_INVISIBLE_INDEXES=TRUE` solo en la sesión de prueba; su plan real utilizó `INDEX RANGE SCAN` con **3 buffers**. Después lo hice visible y validé su uso ([C14 solucionar](lab7/evidencia/casos/20261125T183001Z_C14_solucionar.txt), [C14 validar](lab7/evidencia/casos/20261125T183110Z_C14_validar.txt)). El panel devolvió 0 pedidos en esta ejecución ([C14 entender](lab7/evidencia/casos/20261125T182930Z_C14_entender.txt)).

2. Al cambiar el canal de **20.000 pedidos**, medí **8.997 KB** de redo sin mantener el índice, **10.665 KB** con el índice invisible y **10.679 KB** visible. Las cifras casi iguales para invisible y visible muestran que la invisibilidad evita su uso normal por el optimizador, pero no el coste de mantenerlo en las escrituras ([C14 observar](lab7/evidencia/casos/20261125T183124Z_C14_observar.txt)).

**Reflexión retrospectiva:** antes de conocer la solución habría considerado un índice compuesto sobre `(estado, canal, fecha_pedido)`, con las igualdades primero y el rango al final, y lo habría probado invisible en una sesión. Esperaría menos lecturas por apertura y comprobaría el plan y el coste de mantenimiento. La solución ya se había ejecutado antes de escribir esta propuesta, así que no constituye una predicción previa ([C14 diagnosticar](lab7/evidencia/casos/20261125T182938Z_C14_diagnosticar.txt)).

**Comparación:** la solución creó ese índice invisible, lo probó con `OPTIMIZER_USE_INVISIBLE_INDEXES=TRUE` y después lo hizo visible. El panel pasó de **5.153 a 4 lecturas lógicas por apertura**; la prueba aislada usó `INDEX RANGE SCAN` con 3 buffers. Coincide con la propuesta retrospectiva, pero el panel devolvió **0 pedidos**, por lo que esta prueba no demuestra el comportamiento con los pedidos esperados ni el coste de mantenimiento en escritura ([C14 entender](lab7/evidencia/casos/20261125T182930Z_C14_entender.txt), [C14 solucionar](lab7/evidencia/casos/20261125T183001Z_C14_solucionar.txt)).

## Parte F · C17 · INC-604 con AWR, ASH y ADDM
1. SLI y SLO; p95 antes y después; instantáneas AWR que usaste.
2. ¿Qué dijeron ASH, AWR y ADDM, y qué no podían decirte? ¿Cuál fue el ÚNICO cambio?

**Propuesta antes de ejecutar la solución:** cambiaría el tipo de la variable de enlace del código postal de numérico a texto en la aplicación, para evitar la conversión de la columna y permitir el uso del índice existente. Repetiría la misma carga y compararía el p95 con los **81,2 ms** iniciales y el SLO de **menos de 20 ms**, además del plan y las lecturas por ejecución. ASH y AWR señalan la consulta V1, pero el plan no se recuperó en el diagnóstico de C17; el predicado `TO_NUMBER` y el tipo numérico sí quedaron capturados en [C07 diagnosticar](lab7/evidencia/casos/20261125T180536Z_C07_diagnosticar.txt) ([C17 entender](lab7/evidencia/casos/20261125T184329Z_C17_entender.txt), [C17 diagnosticar](lab7/evidencia/casos/20261125T184337Z_C17_diagnosticar.txt)).

**Comparación:** la solución coincidió con la propuesta: usar `BUSCAR_CP_V2` con un argumento `VARCHAR2`; la prueba mostró `INDEX RANGE SCAN` ([C17 solucionar](lab7/evidencia/casos/20261125T184458Z_C17_solucionar.txt)). En la misma carga, el p95 bajó de **81,2 a 0,66 ms** y las lecturas por ejecución de **5.056 a 96**. La validación cumplió el SLO, pero falló la comprobación de las líneas base ANTES y DESPUÉS ([C17 validar](lab7/evidencia/casos/20261125T184713Z_C17_validar.txt)).

## Parte F · C18 · SQL Plan Management

**Propuesta antes de ejecutar la solución:** recuperaría el plan del lunes desde AWR como línea base SPM aceptada, para que Oracle pueda reutilizar el acceso por índice aunque falte el histograma. Compararía el plan, las lecturas por ejecución y el tiempo con los del martes, y comprobaría que se utiliza la línea base sin modificar las estadísticas. Espero recuperar el plan 1767728868, pero no presupongo una mejora: las medidas actuales son 5.213 lecturas/ejecución el lunes frente a 5.117 el martes. Como alternativa, corregiría la preferencia de estadísticas y recuperaría el histograma; puede mejorar la estimación, aunque no garantiza que el optimizador conserve el mismo plan.
**Comparación:** la solución de referencia coincidió con la propuesta: cargó desde AWR el plan 1767728868 como línea base aceptada y activa, y el informe la utilizó sin recuperar el histograma ([C18 solucionar](lab7/evidencia/casos/20261125T201315Z_C18_solucionar.txt)). Con SPM se midieron **5.233 lecturas y 35,1 ms por ejecución**, frente a **5.117 y 17,8 ms** el martes; el lunes se habían medido **5.213 y 17,2 ms**. Por tanto, se recuperó el plan, pero esta ejecución no demostró una mejora; la comprobación de lecturas falló ([C18 validar](lab7/evidencia/casos/20261125T201319Z_C18_validar.txt)). El tiempo observado en una ejecución no basta para atribuir la diferencia únicamente al plan. Corregir la preferencia y recuperar el histograma es una alternativa que podría corregir la estimación, pero no se ha probado aquí ni garantiza conservar el plan.

1. **Planes y lecturas por ejecución:** el lunes, `1767728868` y **5.213**; el martes, `2571571058` y **5.117** ([C18 entender](lab7/evidencia/casos/20261125T200913Z_C18_entender.txt)). Con la línea base se recuperó `1767728868` y se midieron **5.150** en la repetición posterior ([C18 solucionar](lab7/evidencia/casos/20261125T201402Z_C18_solucionar.txt)). La validación de esa repetición también falló la condición de reducir las lecturas a menos de un tercio de las del martes ([C18 validar](lab7/evidencia/casos/20261125T201439Z_C18_validar.txt)); SPM recuperó el plan, pero aquí no se demostró la mejora esperada.

2. **Evolución y prevención:** la tarea comparó el plan nuevo autocapturado `SQL_PLAN_0a27u7nan424b95a18d93` con la línea base y lo dejó **no aceptado** porque no superó el criterio de beneficio: el informe indica que su rendimiento verificado fue peor. El plan cargado desde AWR permaneció aceptado ([C18 observar](lab7/evidencia/casos/20261125T201450Z_C18_observar.txt)). Queda corregir la preferencia `METHOD_OPT` de `T6.PEDIDO`, que fuerza `SIZE 1`, y recoger estadísticas que vuelvan a generar un histograma adecuado para `ESTADO`; después hay que comprobar las estimaciones y repetir la medición. SPM conserva un plan probado, pero no corrige por sí solo la causa de la pérdida del histograma ([C18 provocar](lab7/evidencia/casos/20261125T200748Z_C18_provocar.txt), [C18 diagnosticar](lab7/evidencia/casos/20261125T200956Z_C18_diagnosticar.txt)).

## Parte G · Incidente 1 · INC-6NN
- **Síntoma (con números):**
- **Playbook y pasos seguidos (herramientas y vistas):**
- **Causa raíz:**
- **Arreglo aplicado (órdenes exactas) y por qué ese:**
- **Verificación (archivo de evidencia con todo OK):**
- **Prevención (qué evitaría que se repita):**

## Parte G · Incidente 2 · INC-6NN
- **Síntoma (con números):**
- **Playbook y pasos seguidos (herramientas y vistas):**
- **Causa raíz:**
- **Arreglo aplicado (órdenes exactas) y por qué ese:**
- **Verificación (archivo de evidencia con todo OK):**
- **Prevención (qué evitaría que se repita):**

## Parte G · Incidente 3 · INC-6NN
- **Síntoma (con números):**
- **Playbook y pasos seguidos (herramientas y vistas):**
- **Causa raíz:**
- **Arreglo aplicado (órdenes exactas) y por qué ese:**
- **Verificación (archivo de evidencia con todo OK):**
- **Prevención (qué evitaría que se repita):**

## Parte H · Reflexión
1. De todo el laboratorio, ¿qué cambio dio más mejora por menos riesgo? Justifícalo con tus números.
2. Un caso no evaluable (sin ★) que hayas hecho y qué aprendiste de él (opcional, cuenta para la certificación).
