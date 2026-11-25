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

## Parte E · C13 · El ciclo de estadísticas con red
1. Describe el ciclo que seguiste (pendientes, probar, publicar, restaurar, bloquear) con la vista que comprueba cada paso.
2. ¿Qué error obtuviste al recoger sobre estadísticas bloqueadas y cómo se recoge si de verdad hace falta?

## Parte E · C14 · El índice invisible
1. Lecturas por apertura del panel antes y después. ¿Cómo probaste el índice sin que lo viera nadie más?
2. Redo con el índice INVISIBLE y VISIBLE: ¿qué demuestra?

## Parte F · C17 · INC-604 con AWR, ASH y ADDM
1. SLI y SLO; p95 antes y después; instantáneas AWR que usaste.
2. ¿Qué dijeron ASH, AWR y ADDM, y qué no podían decirte? ¿Cuál fue el ÚNICO cambio?

## Parte F · C18 · SQL Plan Management
1. PLAN_HASH_VALUE y lecturas del lunes, del martes y con la línea base.
2. ¿Qué hizo la evolución con el plan nuevo y por qué? ¿Qué queda pendiente para que no vuelva a pasar?

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
