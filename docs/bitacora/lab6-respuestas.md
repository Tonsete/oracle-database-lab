# Lab 6 · Oracle DBA Operations: respuestas

| | |
|---|---|
| Estudiante | Antonio García Gallego (E027) |
| Usuario de GitHub | Tonsete |
| Issue / rama | #11 · `chore/11-administracion-operativa` |
| Entorno | macOS (Apple Silicon) + OrbStack · Oracle AI Database 26ai Free · contenedor `oralab-26ai` |

Todas las rutas de evidencia son relativas a `docs/bitacora/lab6/evidencia/`. Los postmortems de la guardia están en `docs/bitacora/lab6-guardia.md`.

## 1. Preguntas de comprobación

### Partes A-C · Rol operativo, parámetros y estados

**1. ¿Cuántos errores ORA- tenía tu alert log en 24 h en la Parte B y a qué se debían? ¿Por qué ese número no es una alarma por sí solo?**

En el log de la Parte B conté 5 errores `ORA-` en las últimas 24 horas, pero ese log no muestra su desglose. En la comprobación posterior de la Parte C, el error repetido era `ORA-00800: Set Priority Failed` para `LGWR`. Ocurre porque Oracle no puede subir la prioridad de sus procesos dentro del contenedor y vuelve a aparecer en cada arranque. El `ORA-65019` no forma parte de los cinco de la Parte B: lo provoqué después al intentar abrir una PDB que ya estaba abierta.

Evidencia: `script/20261123T193417Z_01-turno.script.log` y `script/20261123T193533Z_02-estados-pdb.script.log`.

**2. ¿A qué hora empieza y cuánto dura la ventana de mantenimiento de un miércoles en tu base? ¿Qué tareas automáticas corren en ella?**

La ventana del miércoles empieza a las 22:00 y dura 4 horas. En ella están habilitadas la recogida automática de estadísticas, el asesor de espacio y el asesor de SQL. AWR conserva una captura cada hora durante 8 días.

Evidencia: `spool/20261123T193321Z_01-automatizacion.spool.log`.

**3. Explica con tu salida de la Parte C la diferencia entre lo que mostraban V$SPPARAMETER y V$PARAMETER tras el SCOPE = SPFILE. ¿Qué habría pasado en el siguiente reinicio sin el RESET?**

Después de cambiar `db_files` con `SCOPE=SPFILE`, `V$SPPARAMETER` mostraba 250, pero `V$PARAMETER` seguía mostrando 200: el cambio solo estaba guardado para el próximo arranque. Si no hubiera hecho `RESET`, la instancia habría arrancado con 250 tras reiniciarse.

Evidencia: `script/20261123T193532Z_02-parametros.script.log`.

**4. ¿Por qué RESET fue la vuelta atrás correcta para db_files y no lo habría sido para processes? Cita la línea de tu PFILE que lo demuestra.**

`db_files` no estaba fijado en el SPFILE; el PFILE, que es una copia de ese SPFILE, tampoco mostraba una línea para ese parámetro. Por eso `RESET` quitaba el valor que acababa de fijar y Oracle volvía al valor por defecto. En cambio, `processes` sí estaba fijado como `*.processes=200`. Si hubiera usado `RESET`, Oracle habría pasado a usar su valor por defecto, no necesariamente 200. Para recuperar el valor anterior, lo correcto era volver a fijarlo en 200.

Evidencias: `script/20261123T193532Z_02-parametros.script.log` y `config/20261123T193533Z_02-init_desde_spfile.ora`.

**5. Tras el reinicio sin estado guardado, ¿qué error obtuvo C##OPS_RO y por qué la instancia decía «Database opened»?**

Tras reiniciar sin guardar el estado de la PDB, `C##OPS_RO` recibió `ORA-01109: database not open` al intentar entrar por `freepdb1`. «Database opened» se refería a la base contenedora; `FREEPDB1` seguía en estado `MOUNTED`.

Evidencia: `script/20261123T193533Z_02-estados-pdb.script.log`.

### Partes D-E · Servicios, sesiones y diccionario

**6. ¿Qué SID, SPID y evento tenía la sesión de la aplicación en tu ejecución? ¿Qué te dicen MODULE, ACTION y CLIENT_IDENTIFIER que no te diría USERNAME?**

La sesión de la aplicación tenía SID 221, SPID 2396 y esperaba en `PL/SQL lock timer`. `MODULE=matriculas`, `ACTION=cierre-periodo` y `CLIENT_IDENTIFIER=ana.garcia@ucam` permiten saber qué estaba haciendo la sesión y para quién; `USERNAME` por sí solo solo identifica la cuenta.

Evidencia: `script/20261123T194139Z_03-sesiones.script.log`.

**7. Tras CANCEL SQL, ¿cuántas transacciones abiertas tenía la aplicación? ¿Y tras KILL SESSION? ¿En qué estado quedó la fila 101 y por qué?**

Después de `CANCEL SQL`, la aplicación todavía tenía 1 transacción abierta: cancelar la llamada no terminó la sesión ni deshizo el `UPDATE`. Después de `KILL SESSION`, ya no quedaban sesiones de la aplicación vivas y la fila 101 estaba `ACTIVO`, porque Oracle revirtió el cambio pendiente.

Evidencia: `script/20261123T194139Z_03-sesiones.script.log`.

**8. ¿Qué valor tiene common_user_prefix en tu imagen? ¿Qué implica para el nombre de una cuenta común?**

`common_user_prefix` vale `C##`. Por eso las cuentas comunes deben empezar con ese prefijo; `APP_ACADEMIA` es local a `FREEPDB1` y no lo necesita.

Evidencia: `script/20261123T193949Z_03-cuenta-app.script.log`.

**9. Completa con tus cifras la tabla de las tres miradas (ALL_TABLES, DBA_USERS, CDB_USERS, V$DATAFILE) y explica una diferencia.**

| Vista | Raíz (CDB$ROOT) | PDB (FREEPDB1) | Aplicación |
|---|---|---|---|
| ALL_TABLES del esquema indicado | 0 | 0 | 1 |
| DBA_USERS | 37 | 47 | No tiene permiso; devuelve ORA-00942 |
| CDB_USERS | 37 en raíz y 47 en la PDB | 47 en la PDB | No tiene permiso; devuelve ORA-00942 |
| V$DATAFILE | 17 | 10 | No tiene permiso; devuelve ORA-00942 |

La diferencia se debe a que cada vista muestra información según el contenedor actual y los permisos de quien consulta. La cuenta de aplicación puede ver la tabla que tiene concedida, pero no consultar las vistas administrativas.

Evidencia: `script/20261123T194336Z_04-tres-miradas.script.log`.

**10. Con tu salida de 04-deltas: ¿cuántas lecturas lógicas por recorrido y cuánto DB time costó la carga? ¿Qué indica la relación entre DB time y DB CPU?**

La carga hizo 53.670 lecturas lógicas en 10 recorridos, es decir, 5.367 por recorrido. Costó 0,23 segundos de DB time y 0,23 de DB CPU; en esta ejecución casi todo ese tiempo fue CPU y no hubo una diferencia apreciable por esperas.

Evidencia: `script/20261123T194346Z_04-deltas.script.log`.

### Partes F-G · Espacio y acceso

**11. Da las tres medidas (asignado, libre, máximo) de TBS_ACADEMIA_HIST antes de la carga y explica cuál usarías para alertar.**

Antes de la carga, `TBS_ACADEMIA_HIST` tenía 10 MB asignados, 3,9 MB libres dentro de lo asignado y un máximo de 40 MB. Para alertar miraría el espacio libre frente al máximo y los umbrales, porque el espacio asignado no muestra por sí solo cuánto puede crecer.

Evidencia: `script/20261123T194517Z_05-almacenamiento.script.log`.

**12. ¿Quién insertó y quién «pagó» en el ORA-01950? ¿Por qué ampliar el tablespace no habría resuelto el ORA-01536?**

El `INSERT` lo ejecutó `SYS` mediante `sysdba.sh`, pero la cuota la necesitaba `CARGA_HIST`, que es el dueño de la tabla y del segmento. Ampliar el tablespace no habría arreglado el `ORA-01536`: había espacio en el tablespace, pero `CARGA_HIST` había superado su cuota.

Evidencia: `script/20261123T194517Z_05-almacenamiento.script.log`.

**13. ¿Cuánto tiempo estuvo suspendida la carga reanudable y qué evidencia de ello dejó Oracle en el alert log?**

La carga quedó suspendida desde las 19:45:19 hasta que se reanudó tras ampliar la cuota; la ejecución terminó a las 19:45:33, así que transcurrieron unos 14 segundos. Oracle dejó en el alert log los mensajes de suspensión por `ORA-01536` y de reanudación.

Evidencia: `script/20261123T194519Z_05-resumable.script.log`.

**14. Antes y después del SHRINK: filas, bloques y cuota consumida. ¿Por qué un DELETE no devolvió el espacio?**

Antes del `DELETE` había 20.000 filas, 2.944 bloques y 23 MB. Después del `DELETE` quedaban 2.000 filas, pero seguían siendo 2.944 bloques y 23 MB. Después de `SHRINK SPACE` quedaron 2.000 filas, 304 bloques y 2,4 MB; la cuota usada también bajó a 2,4 MB. `DELETE` quitó filas, pero no compactó el segmento ni bajó su marca de agua alta.

Evidencia: `script/20261123T194533Z_05-hwm-shrink.script.log`.

**15. ¿Qué valores tenía tu perfil DEFAULT? Justifica dos de los límites de PRF_PERSONAS para una persona de secretaría.**

En el perfil `DEFAULT`, `FAILED_LOGIN_ATTEMPTS` era 10 y `PASSWORD_LOCK_TIME` era 1 día. Para `PRF_PERSONAS` elegí limitar los intentos fallidos a 3 para reducir el riesgo de que alguien adivine una contraseña, y dejar el bloqueo en 1 minuto para que un error de escritura no impida trabajar demasiado tiempo. En la práctica, después de tres contraseñas incorrectas, `ANA_GARCIA` pasó a `LOCKED(TIMED)` y luego volvió a `OPEN`. La comprobación de contraseña débil también generó `ORA-28003`.

Evidencias: `script/20261124T171241Z_06-gobierno-acceso.script.log` y `script/20261124T171242Z_06-probar-acceso.script.log`.

**16. ¿Por qué Ana pudo leer ACA_MATRICULAS si nadie se la concedió por nombre? ¿Qué error obtuvo al cambiar el nombre de un alumno?**

Ana pudo leer `ACA_MATRICULAS` porque tenía el rol `ROL_SECRETARIA_ACADEMIA`, que hereda `ROL_LECTURA_ACADEMIA`, con el privilegio de lectura sobre el esquema. Al intentar cambiar el nombre recibió `ORA-41900`, porque el rol solo le daba permiso de `UPDATE` para la columna `ESTADO`.

Evidencia: `script/20261124T171242Z_06-probar-acceso.script.log`.

### Partes H-I · Objetos, diagnóstico y guardia

**17. ¿Qué objetos se invalidaron en cada uno de los tres cambios de columna y por qué no se invalidó nada con la columna nueva?**

Añadir una columna nueva no invalidó objetos: Oracle detectó que las dependencias existentes no la usaban. Al ampliar `email_institucional` se invalidó `V_CONTACTO_LAB6`. Al ampliar `nombres` se invalidaron `V_ACTIVOS_LAB6` y `F_TOTAL_ACTIVOS_LAB6`. La función depende de `V_ACTIVOS_LAB6`, que depende de la tabla; por eso la invalidación llegó hasta la función en cascada.

Evidencia: `script/20261124T171348Z_07-objetos.script.log`.

**18. Según tus advisors, ¿mejoraría el DB time duplicar la SGA? ¿Y por debajo de qué tamaño la PGA tendría sobreasignaciones?**

Duplicar la SGA no mejoraría el DB time según el advisor: con 1.536 MB el factor estimado es 1 y con 3.072 MB sigue siendo 1. Reducirla a la mitad lo multiplicaría por 4,1578. Para la PGA, el advisor muestra 7 sobreasignaciones a 128 y 256 MB; a partir de 384 MB muestra 0.

Evidencia: `script/20261124T171351Z_07-memoria.script.log`.

**19. ¿Cuántas lecturas lógicas necesitó la consulta de la aplicación según la traza (o tkprof)? ¿Qué línea de la traza lo dice?**

La consulta necesitó 247 lecturas lógicas. La línea de la traza es `FETCH ... cr=247`; en el informe de tkprof aparece también como `Fetch ... query 247`.

Evidencia: `script/20261124T171356Z_07-adr-trazas.script.log`.

**20. Para cada incidente de la guardia: evidencia clave, intervención y validación (una línea por cada uno).**

- INC-601: Ana estaba bloqueada por la sesión 57 de `APP_ACADEMIA`. La terminé a las 17:21:07; luego la aplicación confirmó que la fila 101 estaba `ACTIVO`.
- INC-602: `academia_app` tenía `ARRANCADO = NO`. Arranqué el servicio a las 17:23:02 y, a las 17:23:19, la aplicación ya podía conectar.
- INC-603: `APP_ACADEMIA` aparecía como `LOCKED`. Desbloqueé la cuenta a las 18:18:02 y, a las 18:18:12, la aplicación pudo entrar.

## 2. Simulacro de examen 1Z0-182

| Pregunta | Respuesta | Parte donde lo comprobé |
|---|---|---|
| S1. Proceso que genera las alertas de umbral de espacio | B. MMON | B |
| S2. ISSYS_MODIFIABLE = DEFERRED con SCOPE = BOTH | C. Se aplica a las sesiones nuevas | C |
| S3. Copia de texto de los parámetros en uso, incluidos los calculados | B. CREATE PFILE FROM MEMORY | C |
| S4. Dejar una PDB READ ONLY en READ WRITE | B y C | C |
| S5. Vista con el estado guardado de cada PDB | C. DBA_PDB_SAVED_STATES | C |
| S6. Usuario sin RESTRICTED SESSION contra una PDB restringida | B. ORA-01035 | C |
| S7. Paquete que crea y arranca un servicio sin Grid Infrastructure | A. DBMS_SERVICE | D |
| S8. Columnas que unen V$SESSION con V$PROCESS | B. PADDR = ADDR | D |
| S9. Transacción pendiente tras ALTER SYSTEM CANCEL SQL | C. Sigue abierta | D |
| S10. SHOW USER conectado como SYSOPER | C. PUBLIC | D |
| S11. Métricas del último minuto de la PDB | B. V$CON_SYSMETRIC | E |
| S12. Componentes que suma DB time | A y C. CPU de sesiones foreground y esperas no Idle | E |
| S13. SYS inserta en HR.T y recibe ORA-01950: quién necesita cuota | B. HR | F |
| S14. Tablespaces sin cuota posible | A y C. Undo y temporal | F |
| S15. RESUMABLE sin cuota | A y C. La sentencia se suspende y se ve en DBA_RESUMABLE | F |
| S16. Requisito previo de SHRINK SPACE | B. ENABLE ROW MOVEMENT y ASSM | F |
| S17. Estado tras 3 fallos con PASSWORD_LOCK_TIME = 1 | B. LOCKED(TIMED) | G |
| S18. Revocar a A un privilegio de sistema que concedió a B con ADMIN OPTION | B. B lo conserva | G |
| S19. Acciones que dejan un índice UNUSABLE | A y B. MOVE sin UPDATE INDEXES y marcarlo UNUSABLE | H |
| S20. Trazar otra sesión por SID y SERIAL# | B. DBMS_MONITOR.SESSION_TRACE_ENABLE | H |
| S21. Parámetro que fija la raíz del ADR | C. DIAGNOSTIC_DEST | H |
| S22. Primer paso ante enq: TX - row lock contention | C. Identificar al bloqueador y revisar qué está haciendo | I |
