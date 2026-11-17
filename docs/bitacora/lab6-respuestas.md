# Lab 6: respuestas

## Partes A-C

1. En la evidencia 20261117T210714Z_01-turno.script.log la consulta de 24 h dio 22 errores ORA-. Al desglosar el alert log de esa ventana salen 16 ORA-00800 (Set Priority Failed de LGWR/VKTM, ruido conocido del contenedor), 2 ORA-65019 (abrir una PDB ya abierta), 1 ORA-01536 (prueba resumable), 1 ORA-19815 (FRA llena de una prueba anterior), 1 ORA-00031 (kill de sesión) y 1 ORA-00000 (normal). Ese 22 no es una alarma por sí solo porque mezcla ruido esperado, errores provocados y errores de pruebas anteriores; hay que compararlo con la línea base y clasificarlo.

2. El miércoles la ventana empieza a las 22:00 y dura 4 h (WEDNESDAY_WINDOW, enabled). En ella corren las 3 AutoTask: auto optimizer stats collection, auto space advisor y sql tuning advisor. Evidencia 20261117T210736Z_01-automatizacion.spool.log.

3. Después de SCOPE=SPFILE, V$SPPARAMETER mostraba 250 con ISSPECIFIED=TRUE y V$PARAMETER seguía mostrando 200 en uso. Sin el RESET, el siguiente reinicio habría arrancado con db_files=250 y se habría perdido el 200 original. Evidencia 20261117T210808Z_02-parametros.script.log.

4. RESET fue correcto para db_files porque no estaba fijado en el PFILE: no aparece ninguna línea db_files. Al borrar el 250 temporal volvió al valor por defecto, 200. Para processes no habría valido: el PFILE sí lo tiene fijado en la línea 31, *.processes=200 (20261117T210819Z_02-init_desde_spfile.ora). Un RESET habría borrado ese 200 y el arranque habría usado el valor por defecto de la imagen, no el mío.

5. C##OPS_RO recibió ORA-01109: database not open al entrar por freepdb1. La instancia decía Database opened porque eso es el CDB; la PDB FREEPDB1 quedó MOUNTED, sin estado guardado, y por eso no se podía entrar en ella. Evidencia 20261117T210828Z_02-estados-pdb.script.log.

## Partes D-E

6. SID 62, SPID 1896, evento PL/SQL lock timer. MODULE=matriculas, ACTION=cierre-periodo y CLIENT_IDENTIFIER=ana.garcia@ucam. USERNAME solo dice APP_ACADEMIA, que es una cuenta compartida; esos tres campos dicen qué proceso de negocio y qué persona o cliente hay detrás. Evidencia 20261117T210959Z_03-sesiones.script.log.

7. Tras CANCEL SQL quedaba 1 transacción abierta; tras KILL SESSION quedaron 0 sesiones de la app y 0 transacciones, y la fila 101 quedó ACTIVO. El KILL terminó la sesión y PMON deshizo el UPDATE no confirmado. Evidencia 20261117T210959Z_03-sesiones.script.log.

8. common_user_prefix = C## (3 caracteres). Por eso una cuenta común tiene que empezar por C##; APP_ACADEMIA no lo lleva y quedó local, con COMMON=NO. Evidencia 20261117T210950Z_03-cuenta-app.script.log.

9. Estas son mis cifras:

| Vista | C##OPS_RO en raíz | C##OPS_RO en FREEPDB1 | APP_ACADEMIA |
|---|---|---|---|
| ALL_TABLES de ADMIN_ACADEMIA | 0 | 0 | 1 |
| DBA_USERS | 37 | 46 | error ORA-00942 |
| CDB_USERS | 37 (con_id 1) + 46 (con_id 3) | 46 (con_id 3) | error ORA-00942 |
| V$DATAFILE | 18 | 11 | error ORA-00942 |

La diferencia principal: DBA_USERS solo cuenta el contenedor actual (37 en raíz y 46 en FREEPDB1), mientras CDB_USERS suma los contenedores visibles (37 + 46). V$DATAFILE muestra 18 en raíz y 11 dentro de la PDB. APP_ACADEMIA no ve el catálogo y solo ve 1 tabla en ALL_TABLES, la que tiene concedida. Evidencia 20261117T211036Z_04-tres-miradas.script.log.

10. 8.452 lecturas lógicas por recorrido; DB time 1,26 s y DB CPU 1,11 s. La relación indica que casi todo el DB time fue CPU (1,11 de 1,26), así que la carga fue más de CPU que de espera. Evidencia 20261117T211105Z_04-deltas.script.log.

## Partes F-G

11. Antes de la carga: asignado 40 MB, libre 31,6 MB, máximo 40 MB; bloques usados 1.080 de 5.120, 21,09 % sobre el máximo. Para alertar usaría el porcentaje sobre el máximo (umbral 80/95), no solo el libre, porque el máximo es el techo real. Evidencia 20261117T211123Z_05-almacenamiento.script.log.

12. El INSERT lo lanzó la sesión del DBA (el script), pero la cuota que se agotó era la de CARGA_HIST: ORA-01950 decía user CARGA_HIST has insufficient quota. Luego, con 2 MB de cuota, el mismo dueño pagó el ORA-01536; el tablespace tenía 31,6 MB libres, así que ampliarlo no habría resuelto el límite de cuota del usuario. Evidencia 20261117T211123Z_05-almacenamiento.script.log.

13. Estuvo suspendida 14,0 s: el alert log la marca suspendida a las 21:11:34.917 por ORA-01536 y reanudada a las 21:11:48.936. La evidencia 05-resumable también muestra STATUS=SUSPENDED, TIMEOUT=180 y la carga terminó a las 21:11:49.

14. Antes: 20.000 filas, 2.944 bloques, 23 MB. Tras el DELETE: 2.000 filas, 2.944 bloques, 23 MB. Tras SHRINK: 2.000 filas, 304 bloques, 2,4 MB, y la cuota consumida quedó en 2,4 MB. El DELETE no devolvió espacio porque no baja la HWM; SHRINK mueve las filas y sí la baja. Evidencia 20261117T211158Z_05-hwm-shrink.script.log.

15. DEFAULT: FAILED_LOGIN_ATTEMPTS 10, INACTIVE_ACCOUNT_TIME 365, PASSWORD_GRACE_TIME 7, PASSWORD_LIFE_TIME 180, PASSWORD_LOCK_TIME 1, PASSWORD_REUSE_MAX UNLIMITED, PASSWORD_REUSE_TIME UNLIMITED, PASSWORD_ROLLOVER_TIME 0 y PASSWORD_VERIFY_FUNCTION NULL. PRF_PERSONAS: 3 intentos, bloqueo 1/1440 (1 min, solo laboratorio), vida 90 días, gracia 7, inactividad 90 y verify function. Justifico 3 intentos porque reduce la fuerza bruta, y 90 días de vida porque obliga a rotar la contraseña sin cambiar cada mes. Evidencia 20261117T211235Z_06-gobierno-acceso.script.log.

16. Ana pudo leer 20.000 estudiantes y 60.000 matrículas porque ROL_LECTURA_ACADEMIA tiene SELECT ANY TABLE ON SCHEMA admin_academia, y ella lo hereda de ROL_SECRETARIA_ACADEMIA; nadie le dio un grant directo sobre ACA_MATRICULAS. Al cambiar nombres recibió ORA-41900, porque solo tiene UPDATE de la columna ESTADO (1 columna), no de nombres. Evidencia 20261117T211243Z_06-probar-acceso.script.log.

## Partes H-I

17. Con la columna nueva telefono no se invalidó nada (0 objetos). Con email_institucional se invalidó 1: V_CONTACTO_LAB6. Con nombres se invalidaron 3: V_ACTIVOS_LAB6, V_CONTACTO_LAB6 y F_TOTAL_ACTIVOS_LAB6 (la función cae por la vista). La columna nueva no invalida porque ningún objeto la referencia; Oracle usa dependencias de grano fino. Evidencia 20261117T211328Z_07-objetos.script.log.

18. No: duplicar la SGA de 1.536 MB a 3.072 MB deja ESTD_DB_TIME_FACTOR=1 y las mismas 25.960 lecturas físicas, así que el advisor no espera mejora. La PGA tendría sobreasignaciones por debajo de 384 MB: con 128 MB y 256 MB el OVERALLOC vale 1; con 384 MB y 512 MB vale 0. Evidencia 20261117T211339Z_07-memoria.script.log.

19. 247 lecturas lógicas. Lo dice la línea FETCH de la traza con cr=247; el informe tkprof lo resume en query=247. Evidencia 20261117T211409Z_07-adr-trazas.script.log.

20. Una línea por incidente:

- INC-601: evidencia 20261117T211447Z: SID 66 esperando por el 65, TX lmode 6 y 1 bloque de undo; intervención KILL SESSION '65,41059'; validación 20261117T212117Z: 0 bloqueadas y fila 101 ACTIVO.
- INC-602: evidencia 20261117T211504Z: academia_app NO; intervención START_SERVICE('academia_app'); validación 20261117T212311Z: conexión por academia_app/FREEPDB1 con 20.000 filas.
- INC-603: evidencia 20261117T211547Z: APP_ACADEMIA LOCKED desde 21:15; intervención ALTER USER app_academia ACCOUNT UNLOCK; validación 20261117T212059Z: APP_ACADEMIA OPEN, 17.600 activos y fila 101 ACTIVO.
