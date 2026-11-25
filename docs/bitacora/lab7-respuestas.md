# Lab 7: respuestas

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

Dominó `resmgr:cpu quantum`, con 247 muestras de 324 (76,2 %). Es el tiempo que las sesiones esperan su turno de CPU por la planificación del gestor de recursos; en esta instancia Oracle Free la capacidad está limitada a 2 CPU. La evidencia es [C01 diagnosticar](lab7/evidencia/casos/20261125T175122Z_C01_diagnosticar.txt).

### 3. Ley de Little

Para comprobar N = X · R uso el throughput X en llamadas/s y la **latencia media** R en segundos, no el p95:

| Sesiones | Cálculo con valores mostrados | N aproximado |
| ---: | --- | ---: |
| 1 | 5,2 × 0,1880 | 0,98 ≈ 1 |
| 2 | 10,6 × 0,1866 | 1,98 ≈ 2 |
| 8 | 7,5 × 1,0456 | 7,84 ≈ 8 |

Las pequeñas diferencias se deben al redondeo y a la medición de las tandas. La salida calcula 1,0, 2,0 y 7,9, respectivamente, usando los valores sin redondear. Fuente: [C01 observar](lab7/evidencia/casos/20261125T175553Z_C01_observar.txt).
