# Lab 4 · Oracle DBA Workflow I: respuestas

| | |
|---|---|
| Estudiante | Antonio García Gallego |
| Usuario de GitHub | Tonsete |
| Profesor | Dr. Richard Avilés López |
| Issue / rama | #7 · `chore/7-oracle-dba-workflow-i` |
| Entorno | macOS (Apple Silicon) + Docker · Oracle AI Database 26ai Free 23.26.3.0.0 · plataforma `Linux OS (AARCH64)` |

Todas las rutas de evidencia son relativas a `docs/bitacora/lab4/evidencia/`.

## 0. Diferencias de mi entorno respecto al enunciado

1. El enunciado da los comandos para Ubuntu y yo trabajo en macOS, así que ajusté cuatro detalles de portabilidad sin cambiar la lógica: `shasum -a 256` cuando no existe `sha256sum` (en `backup-estado.sh`, `restaurar-estado.sh` y `10-verificacion-final.sh`), `paste -sd " " -` en `07-copiar-pfile.sh` (el `paste` de macOS exige el `-`), `wc -l | tr -d '[:space:]'` en la verificación final (el `wc` de BSD rellena con espacios) y la sintaxis BSD de `script` para grabar la Parte E. En la copia, además, `chown` ya no es fatal, porque en macOS el archivo ya pertenece a mi usuario.
2. En mi imagen `$ORACLE_HOME/dbs` es un enlace simbólico al volumen (`/opt/oracle/oradata/dbconfig/FREE/dbs`). Por eso en la Parte B no salió ningún archivo FUERA (ver el punto de control B).
3. Mi base está en `ARCHIVELOG`, mientras que el enunciado supone `NOARCHIVELOG` (`00-backup-lab3`: `log_mode=ARCHIVELOG`; `05-instancia-y-base`: `LOG_MODE ARCHIVELOG`). De ahí los 4 procesos ARCn de la Parte H. No hay FRA (`db_recovery_file_dest` vacío) ni `log_archive_dest_n` en el SPFILE, así que Oracle archiva en su destino por defecto; lo revisaré en el Lab 5 con `ARCHIVE LOG LIST`.

## 1. Puntos de control

### Parte B: ¿cuántos archivos salieron como FUERA?

Ninguno. En `script/20261022T161435Z_00-backup-lab3.script.log` las 26 rutas aparecen como `dentro`, incluidos `academia01.dbf`, `clinica01.dbf`, `retail01.dbf`, `logistica01.dbf` y `fintech01.dbf`, aunque su ruta lógica es `/opt/oracle/product/26ai/dbhomeFree/dbs/…`. El script usa `readlink -f`, que resuelve la ruta real, y el diagnóstico C.1 (`script/20261022T161642Z_01-diagnostico.script.log`) lo confirma: `$ORACLE_HOME/dbs -> /opt/oracle/oradata/dbconfig/FREE/dbs`, y los cinco datafiles salen como `DENTRO del volumen, pero en carpeta no estándar`. Habrían sobrevivido a un `docker rm`, pero dependían de un enlace que una actualización de la imagen puede cambiar y estaban en una carpeta donde ningún DBA los buscaría, así que también se reubicaron a `FREE/FREEPDB1`.

### Parte E: explica cada error con tus palabras

- ORA-01507 en NOMOUNT: la instancia solo ha leído el SPFILE, reservado la SGA (`Total System Global Area 1603392144 bytes`) y arrancado los procesos. `v$database` necesita el control file, que todavía está cerrado. `v$parameter`, en cambio, responde porque los parámetros ya están en memoria (`spfile`, `control_files`, `db_name`).
- ORA-01219 en MOUNT: ya se abrieron los control files (la instancia conoce los 16 datafiles y el `LOG_MODE`), pero los datafiles siguen cerrados. `dba_tables` es una vista del diccionario, que vive en el tablespace SYSTEM; en esta etapa solo funcionan las vistas fijas (V$/X$), que leen memoria y control file.

Evidencia: `script/20261022T161756Z_06-arranque-por-etapas.script.log`.

### Parte G: interpreta el experimento

Evidencia `script/20261022T161812Z_08-experimento-cache.script.log`: `ACA_ESTUDIANTES` tiene 20 000 filas en 244 bloques.

| Ronda | Lecturas lógicas | Lecturas físicas | Tasa de acierto (1 − físicas/lógicas) |
|---|---|---|---|
| 1 (caché fría, tras `FLUSH BUFFER_CACHE`) | 357 | 262 | 26,6 % |
| 2 (caché caliente) | 247 | 0 | 100 % |

Toda lectura física acaba en una lógica: el bloque se trae del disco al Buffer Cache y se lee desde allí. En la ronda 2 las 247 lecturas lógicas recorren la tabla entera sin ir a disco ni una vez. Los 245 bloques de `v$bh` son los 244 de datos más la cabecera del segmento, que el full scan también lee.

En mi ronda 1 las lecturas lógicas no coinciden con las de la ronda 2 (357 frente a 247) y las físicas son 262 en lugar de 245. Esas 110 lógicas y 17 físicas de más son lecturas recursivas del diccionario: el `FLUSH` vació también los bloques de sus tablas, y el primer análisis (*hard parse*) de la consulta tuvo que volver a leerlos de disco. En la ronda 2 el cursor ya estaba analizado y esos bloques seguían en caché.

## 2. Preguntas de comprobación

### Respaldo

#### 1. ¿Por qué la copia de la Parte B es consistente? ¿Qué pasaría si se tomara con la base abierta?

Se tomó después de `SHUTDOWN IMMEDIATE` (`Database closed. / Database dismounted. / ORACLE instance shut down.` en `00-backup-lab3`). Ese cierre hace un checkpoint final: todos los datafiles y el control file quedan con el mismo SCN y sin transacciones a medias, y la copia se restaura y se abre con `STARTUP` sin aplicar redo. Con la base abierta, DBWn seguiría escribiendo bloques mientras `tar` copia y cada archivo quedaría en un instante distinto. Esa copia inconsistente solo serviría con ARCHIVELOG y una recuperación con RMAN que aplicara el redo hasta un punto común.

#### 2. ¿Para qué sirven `manifiesto.txt` y `SHA256SUMS`? ¿Qué comprueba `restaurar-estado.sh` antes de tocar nada?

`manifiesto.txt` describe la copia: etiqueta `lab3-final`, fecha `20261022T161629Z`, contenedor, volumen, imagen, commit `aea44ff`, tipo (en frío, consistente) y `archivos_fuera_del_volumen=0`. `SHA256SUMS` guarda la huella de cada archivo (`volumen.tar.gz`, `inventario.txt`…), y en B.3 las cuatro comprobaciones dieron `OK`. `restaurar-estado.sh` verifica primero esas sumas con `--check` y, si alguna no coincide, se detiene sin tocar el volumen. Antes de borrarlo pide además escribir su nombre como confirmación.

#### 3. Define RPO y RTO con los datos de tu propia copia.

- RPO (cuántos datos puedo perder): mi punto de restauración es el del 22-10-2026 a las 16:14:35 UTC (SCN 3 708 269). Si restaurase esa copia perdería todo lo hecho después, es decir, la reubicación de datafiles, V003, V002, `C##OPS_RO` y los cambios de las Partes C a I.
- RTO (cuánto tardo en volver a dar servicio): no lo he medido restaurando, porque el simulacro es del Lab 5. Como referencia, para copiar el volumen de 7 162 MB (961 MB comprimido) la base estuvo sin servicio unos 2 minutos, desde el `SHUTDOWN IMMEDIATE` hasta `READWRITE/READWRITE`. Restaurar es el camino inverso (SHA-256, descomprimir y arrancar), así que espero un RTO de unos pocos minutos.

### Diagnóstico y correcciones

#### 4. ¿Dónde creó Oracle los datafiles de V000 y por qué? ¿Qué parámetro habría evitado el problema?

En `$ORACLE_HOME/dbs` (`/opt/oracle/product/26ai/dbhomeFree/dbs/academia01.dbf`…, ficheros 16 a 20 en `01-diagnostico`). V000 usó una ruta relativa (`DATAFILE 'academia01.dbf'`) y no había destino OMF (`db_create_file_dest` vacío en el apartado 5 del diagnóstico), así que Oracle resolvió la ruta a partir de `$ORACLE_HOME/dbs`, que en mi imagen es un enlace al volumen (`/opt/oracle/oradata/dbconfig/FREE/dbs`). Lo habría evitado `DB_CREATE_FILE_DEST` (OMF) o, mejor aún, una ruta absoluta dentro del volumen con `MAXSIZE`.

#### 5. ¿Por qué `ALTER TABLESPACE … RENAME DATAFILE` no basta para mover un archivo?

Porque solo cambia la ruta que guarda el control file y no copia ningún byte. Antes hay que dejar el tablespace OFFLINE (o la base en MOUNT si es SYSTEM) y copiar el archivo con el sistema operativo; al ponerlo ONLINE, Oracle valida la cabecera del archivo nuevo y solo entonces se borra el original. `03-reubicar-datafiles` muestra esa secuencia (OFFLINE → `cp -p` → `RENAME` → ONLINE → `rm`) para los cinco `TBS_*`, que acaban en `/opt/oracle/oradata/FREE/FREEPDB1/*.dbf`, `AVAILABLE` y con `MAX_MB 1024`.

#### 6. ¿Por qué no se edita V000 y se crea V003? ¿Por qué invalidar una contraseña es mejor que borrarla del archivo?

V000 ya se ejecutó, y una migración aplicada es historia (*forward-only migrations*). Si la editara, el repositorio dejaría de describir lo que se ejecutó, un entorno nuevo no se reconstruiría igual y la contraseña seguiría en el historial de Git. Invalidarla la deja inútil esté donde esté: V003 pasa las cinco cuentas `ADMIN_*` a `AUTHENTICATION_TYPE = NONE` (`spool/…_04-V003__cuentas_esquema_sin_password.spool.log`), y al intentar conectar con la contraseña antigua se obtiene `ORA-01017`. En la industria se llama *rotate/revoke the secret*.

#### 7. ¿Qué demuestra la prueba de persistencia y por qué se ejecuta después de reubicar?

Que la base (los archivos) vive en el volumen y el contenedor se puede tirar. En `03-prueba-persistencia` aparecen los mismos 5 tablespaces, las mismas 3/3/2/3/3 tablas por esquema y los mismos datafiles antes y después de `docker rm` + `docker run`, aunque el contenedor pasó de `f563b6d819a7` a `24355ec38065`; el nuevo, además, solo publica 1521. Se ejecuta después de reubicar (y del backup) porque cualquier datafile que estuviera en la capa del contenedor se borraría con el `docker rm`.

### Instancia y arranque

#### 8. Tres datos que pertenecen a la instancia y tres que pertenecen a la base de datos.

Evidencias `spool/…_05-instancia-y-base.spool.log` y `06-arranque-por-etapas`.

- Instancia: `HOST_NAME 24355ec38065` (cambió al recrear el contenedor), `STARTUP_TIME 2026-10-22 16:16:54`, una SGA de `1 603 392 144 bytes` y 62 procesos `db_*_FREE`.
- Base de datos: `DBID 1510161217` (no cambió al recrear el contenedor), `CREATED 2026-07-08`, `LOG_MODE ARCHIVELOG`, 16 datafiles (4 364 MB), 2 control files y 3 miembros de redo.

#### 9. ¿Qué lee Oracle en NOMOUNT, en MOUNT y en OPEN? Relaciónalo con ORA-01507 y ORA-01219.

- NOMOUNT: lee el SPFILE (`spfileFREE.ora`), reserva la SGA y arranca los procesos. Sin control file, `v$database` da ORA-01507.
- MOUNT: abre los control files que indica `control_files` (`control01.ctl` y `control02.ctl`) y con ellos conoce datafiles y redo, aunque no los abre. Con SYSTEM cerrado, `dba_tables` da ORA-01219.
- OPEN: abre los datafiles y los redo online y, si hace falta, recupera la instancia. En el alert log: `Thread 1 opened at log sequence 13`, `Completed: ALTER DATABASE OPEN`.

#### 10. ¿Por qué FREEPDB1 se abre sola tras `ALTER DATABASE OPEN`? ¿Qué harías si no lo hiciera?

Porque tiene un estado guardado: `dba_pdb_saved_states` muestra `FREEPDB1  FREE  OPEN` (la imagen ejecutó `SAVE STATE` al crearla) y el script confirmó `FREEPDB1 abierta sola en READ WRITE`. Si se quedara en MOUNTED, las aplicaciones recibirían ORA-01109. La abriría con `ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;` y, para que no se repita, ejecutaría `ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;`.

#### 11. Clasifica ORA-00800 y ORA-20013 del alert log.

Los clasifico como ruido conocido:

- ORA-00800 `[Set Priority Failed] [VKTM]` y `[LGWR]` aparece en cada arranque de mi alert log (`06-alert-log`). Oracle intenta subir la prioridad de esos procesos y el contenedor no lo permite, porque le falta `CAP_SYS_NICE`. Es un error *soft*: justo después la base hace MOUNT y OPEN sin problemas.
- ORA-20013 (DBMS_QOPATCH) no apareció en las 400 últimas líneas que revisé. Cuando sale, indica que el inventario de parches no se puede leer dentro de una imagen de contenedor, y tampoco afecta al funcionamiento.

Los dos se documentan como conocidos y se excluyen de las alertas; cualquier otro ORA- se investiga.

### Parámetros

#### 12. Diferencia entre `v$parameter` y `v$spparameter` con el ejemplo de `db_files`.

`v$parameter` da el valor en memoria de la instancia en marcha y `v$spparameter`, lo que hay escrito en el SPFILE. En `07-parametros-cambios`, después de `ALTER SYSTEM SET db_files = 300 SCOPE = SPFILE`, se ve `db_files  200 (memoria)  300 (SPFILE)`: el cambio queda pendiente hasta el próximo arranque porque `db_files` es estático (con `SCOPE=MEMORY` dio ORA-02095). Tras `ALTER SYSTEM RESET db_files SCOPE = SPFILE` el SPFILE queda `(no especificado)`, y la verificación final lo confirma (`db_files sin línea pendiente en el SPFILE (FALSE)`).

#### 13. ¿Qué hace `SCOPE=BOTH` y cuál es el SCOPE por defecto con SPFILE?

Aplica el cambio en memoria al momento y lo escribe en el SPFILE, así que sobrevive al reinicio. En mi evidencia: `open_cursors 500 (memoria) 500 (SPFILE)` con el comentario `Lab4: prueba SCOPE=BOTH`. Si la instancia arrancó con SPFILE, como la mía (`spfile = …/spfileFREE.ora`), el SCOPE por defecto es BOTH; si arrancó con PFILE, es MEMORY.

#### 14. ¿Por qué PROCESSES rechaza el cambio en memoria aunque `v$parameter` lo marque IMMEDIATE?

`07-parametros-lectura` muestra `processes … SYS_MOD IMMEDIATE`, pero `ALTER SYSTEM SET processes = 250 SCOPE = MEMORY` dio `ORA-02097` + `ORA-02095`. PROCESSES dimensiona estructuras de la SGA (las tablas de procesos y de sesiones) que se reservan al arrancar la instancia. En la práctica es estático: se cambia con `SCOPE = SPFILE` y un reinicio.

#### 15. ¿Para qué sirve versionar el PFILE? Explica "configuration drift".

`config/initFREE_lab4.ora` (32 líneas, sacadas del SPFILE) y `config/initFREE_lab4_memoria.ora` (122 líneas, sacadas de la memoria) son una foto legible de la configuración. Si un SPFILE mal editado impide arrancar, sirven de copia de rescate (`STARTUP PFILE=…` + `CREATE SPFILE FROM PFILE`). Y al exportarlo otra vez, `git diff` saca a la luz cualquier parámetro que alguien cambió sin documentarlo; esa separación entre la configuración real y la documentada es el *configuration drift*. Comparar los dos PFILE también enseña los parámetros que están en memoria pero no en el SPFILE, porque son derivados o por defecto: `connection_brokers`, `job_queue_processes`, `log_buffer`, `optimizer_mode`…

### Memoria y procesos

#### 16. ¿Qué modelo de gestión de memoria usa tu instancia? Justifícalo con tres parámetros.

ASMM, la gestión automática de la memoria compartida (`spool/…_08-memoria.spool.log`):

- `memory_target = 0` y `memory_size = 0`: no es AMM, que gestionaría SGA y PGA juntas.
- `sga_target = 1536M`: Oracle reparte la SGA entre sus componentes. Ahora mismo el buffer cache tiene 880 MB, el shared pool 528 MB, el Shared IO Pool 80 MB y el large pool 16 MB; los 960 MB de `Database Buffers` son 880 + 80.
- `pga_aggregate_target = 512M`: la PGA se gestiona aparte, con `pga_aggregate_limit = 2G` como límite duro.

`db_cache_size = 0` y `shared_pool_size = 0` solo indican que no hay un mínimo fijado para esos componentes, y su tamaño lo decide ASMM.

#### 17. Explica las cifras de tu experimento de caché.

Está desarrollado en el punto de control G. En resumen: la ronda 1 hizo 357 lecturas lógicas y 262 físicas (26,6 % de acierto, por la caché vacía y las lecturas recursivas del diccionario), la ronda 2 hizo 247 lógicas y 0 físicas (100 % de acierto), y en `v$bh` quedaron 245 bloques de la tabla (244 + cabecera del segmento).

#### 18. ¿Por qué no se suma el RSS de los procesos para saber la memoria de la instancia?

Porque el RSS de cada proceso incluye las páginas de la SGA compartida que tiene mapeadas, y esa memoria se contaría una vez por proceso. En `09-procesos-so` los procesos de background marcan entre 69 y 324 MB, así que sumar 61 daría varios GB, mientras que `docker stats` mide 2,035 GiB en total (`05-instancia-so`). Esa cifra sí cuadra con la SGA (≈1 529 MB) más la PGA asignada (424,5 MB, `v$pgastat`).

#### 19. Relaciona el SPID de LGWR de `v$process` con el PID de `/proc`. ¿Por qué `docker top` muestra otro número?

`v$process` da `LGWR  SPID 695` (`09-procesos`) y `/proc` dentro del contenedor da `695 … db_lgwr_FREE` (`09-procesos-so`, apartado 4): es el mismo proceso. `docker top` lo muestra como 3662 porque lee los procesos desde el espacio de nombres de PID del anfitrión (la máquina virtual de Docker en mi Mac), mientras que Oracle ve el PID dentro del espacio de nombres propio del contenedor. Por eso el cruce se hace leyendo `/proc` dentro del contenedor, ya que la imagen no trae `ps`.

#### 20. ¿Qué hace PMON y qué hace SMON? Pon un ejemplo de cada uno.

- PMON (*process cleanup*, SPID 643) limpia los procesos de usuario caídos: deshace su transacción con el undo y libera sus bloqueos. Si el portátil de un usuario se apaga en mitad de un UPDATE, PMON detecta el proceso servidor huérfano, deshace el cambio y desbloquea las filas que esperaba otra sesión.
- SMON (*System Monitor*, SPID 699) recupera la instancia al arrancar y limpia segmentos temporales. Tras un `SHUTDOWN ABORT` o un corte de luz, en el siguiente `STARTUP` aplica el redo (*roll forward*) y deshace lo no confirmado. En mi `06-alert-log` no hizo falta, porque el cierre había sido limpio: `Crash Recovery excluding pdb 2/3 which was cleanly closed`.

### Seguridad y trabajo profesional

#### 21. ¿Por qué C##OPS_RO y no SYSTEM para observar? ¿Qué no puede hacer C##OPS_RO?

Por mínimo privilegio: la cuenta que observa no necesita poder modificar nada. SYSTEM tiene el rol DBA y puede borrar o alterar cualquier objeto, así que un error en una consulta, o una credencial filtrada de la herramienta de monitorización, tendría consecuencias graves. `C##OPS_RO` es un usuario común (`COM=YES` en `CDB$ROOT` y `FREEPDB1`, en `02-cuenta-observabilidad`) con solo `CREATE SESSION`, `SET CONTAINER` y `SELECT_CATALOG_ROLE`, más `CONTAINER_DATA=ALL` para ver todas las PDB desde la raíz. No puede modificar datos ni objetos, ejecutar `ALTER SYSTEM`, parar o arrancar la base, crear usuarios ni conceder privilegios.

#### 22. ¿Qué dos vías de exposición de contraseñas corrige este laboratorio respecto al Lab 3?

1. Las contraseñas literales en el repositorio: las de los cinco propietarios de esquema en V000, que siguen en el historial de Git. V003 las invalida (`NO AUTHENTICATION`).
2. La contraseña de SYS en la línea de comandos: `08-aplicar-migraciones.sh` pasaba `sys/<contraseña>@…`, visible con `ps` y guardado en el historial de la shell. La versión 2 usa `/ as sysdba`, la autenticación del sistema operativo dentro del contenedor.

Además, `05` v2 ya no pasa `ORACLE_PWD` al reutilizar el volumen (no aparece en `docker inspect`), y la contraseña de `C##OPS_RO` viaja por la entrada estándar desde `config/.env`, que no se versiona.

## 3. Preguntas tipo examen

| P | Respuesta | Justificación |
|---|---|---|
| P1 | A, D | Tras SHUTDOWN IMMEDIATE todos los archivos comparten SCN: la copia es consistente y se abre con STARTUP sin recuperación. No exige ARCHIVELOG ni RMAN. |
| P2 | B | Standard Edition no tiene el MOVE online, así que el método es OFFLINE → copiar con el SO → RENAME DATAFILE → ONLINE. |
| P3 | A, C | STARTUP_TIME es de la instancia y cambia en cada arranque; CREATED es de la base y no cambia (mi DBID tampoco cambió al recrear el contenedor). |
| P4 | A, C | En NOMOUNT los parámetros ya están en memoria (`v$parameter`) y es la etapa de `CREATE CONTROLFILE`. ARCHIVELOG y renombrar SYSTEM requieren MOUNT, y `dba_users`, OPEN. |
| P5 | B | `ALTER DATABASE ARCHIVELOG` modifica el control file con los datafiles cerrados, en MOUNT. |
| P6 | B | Con SPFILE, el SCOPE por defecto es BOTH (memoria y SPFILE). |
| P7 | C | PROCESSES es un parámetro de instancia que no se modifica por PDB: ORA-65040, como en mi evidencia 07. |
| P8 | A, C | Se arranca con un PFILE bueno y desde él se regenera el SPFILE. |
| P9 | B | SGA_TARGET > 0 significa ASMM, y un DB_CACHE_SIZE fijado actúa como mínimo y no como tamaño fijo. |
| P10 | A, D | En servidor dedicado, el área de ordenación y la memoria de sesión están en la PGA; shared pool, log buffer y buffer cache son SGA. |
| P11 | C | El COMMIT espera a que LGWR escriba el redo en los redo logs online; DBWn escribe los bloques más tarde. |
| P12 | B, D | PMON y SMON son obligatorios en cualquier instancia; ARCn solo existe en ARCHIVELOG, y RECO y CJQ0 no son imprescindibles. |
