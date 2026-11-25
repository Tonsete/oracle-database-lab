# Guardia nocturna · Lab 6 · E027

## INC-601 · La pantalla de secretaría se queda colgada

- Detección: A las 17:19 llegó el ticket de secretaría. Ninguna alerta detectó el problema antes del aviso.
- Impacto: Ana no pudo guardar la ficha del alumno 101 durante unos 2 minutos, desde las 17:19 hasta que terminé la sesión bloqueadora a las 17:21:07.
- Diagnóstico:
  1. Ana, con el SID 213, esperaba por `enq: TX - row lock contention`.
  2. La columna `BLOQ` mostraba el SID 57. Eso significa que la sesión 57 bloqueaba a Ana.
  3. La sesión 57 era `APP_ACADEMIA`. Estaba en `PL/SQL lock timer` (Idle), pero seguía manteniendo la fila bloqueada con una transacción abierta.
  4. Evidencia: `20261124T171958Z_08-diagnostico-inc1.script.log`.
- Causa raíz: La aplicación hizo un `UPDATE` y dejó la transacción abierta sin hacer `COMMIT` ni `ROLLBACK`. Por eso Ana no podía actualizar la misma fila.
- Intervención: A las 17:21:07 ejecuté `ALTER SYSTEM KILL SESSION '57,4920' IMMEDIATE;`. La sesión estaba ociosa, así que `CANCEL SQL` no habría liberado el bloqueo. Solo había un registro de undo que deshacer. No reinicié la base ni terminé la sesión de Ana.
- Validación: A las 17:21:29 entré como la aplicación con `app.sh` y comprobé que la fila 101 estaba `ACTIVO`. El diagnóstico posterior ya no mostraba sesiones esperando. Evidencias: `20261124T172129Z_08-validacion-inc1.script.log` y `20261124T172131Z_08-diagnostico-inc1-despues.script.log`.
- Acción preventiva: La aplicación debe hacer `COMMIT` o `ROLLBACK` al terminar cada operación. También añadiría al turno un aviso si una sesión lleva más de 30 segundos bloqueando a otra. `turno.sql` ya cuenta las sesiones bloqueadas; se podría ampliar esa comprobación. Otra opción es evaluar `MAX_IDLE_BLOCKER_TIME`.

## INC-602 · La aplicación no conecta: ORA-12514

- Detección: El ticket llegó a las 17:22. Ninguna alerta detectó el problema antes del aviso. Reproduje el error `ORA-12514` a las 17:22:25.
- Impacto: Todos los usuarios de la aplicación se quedaron sin poder conectar durante aproximadamente 1 minuto y 19 segundos, desde las 17:22 hasta que validé la conexión a las 17:23:19.
- Diagnóstico:
  1. El diagnóstico mostraba `academia_app` como existente, pero `ARRANCADO = NO`. Evidencia: `20261124T172227Z_08-diagnostico-inc2.script.log`.
  2. `lsnrctl status` no mostraba el servicio `academia_app`.
  3. `C##OPS_RO` sí podía conectarse, así que la base seguía abierta y disponible.
- Causa raíz: El servicio dependía de que alguien lo arrancara a mano y no se había guardado su estado en la PDB. Por eso, si la base se reiniciaba, `academia_app` podía quedarse parado sin que nadie lo detectase.
- Intervención: A las 17:23:02, dentro de `FREEPDB1`, ejecuté `DBMS_SERVICE.START_SERVICE('academia_app')` y después `ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE`. No reinicié la base porque seguía disponible: `C##OPS_RO` podía conectarse y el problema estaba en el servicio. Guardar el estado después de arrancar el servicio hace que tanto la PDB como sus servicios arrancados se restauren al reiniciar.
- Validación: A las 17:23:19 `app.sh` pudo conectar. También vi con `lsnrctl status` que el listener anunciaba `academia_app`. Evidencia de la conexión: `20261124T172319Z_08-validacion-inc2.script.log`.
- Acción preventiva: Añadir a `validar-arranque.sql` o `turno.sql` una comprobación de que la PDB está abierta y de que `academia_app` está arrancado y anunciado por el listener. Así el problema se detectaría antes de que avisasen los usuarios.

## INC-603 · La aplicación no entra: ORA-28000

- Detección: El ticket llegó a las 18:16. Ninguna alerta detectó el problema antes del aviso. Reproduje el error `ORA-28000` a las 18:16:56.
- Impacto: Los usuarios de la aplicación no pudieron entrar mientras la cuenta `APP_ACADEMIA` estuvo bloqueada: aproximadamente 2 minutos y 12 segundos, desde las 18:16 hasta la validación a las 18:18:12.
- Diagnóstico:
  1. La cuenta `APP_ACADEMIA` aparecía como `LOCKED`, no como `LOCKED(TIMED)`. Evidencia: `20261124T181658Z_08-diagnostico-inc3.script.log`.
  2. `LOCKED(TIMED)` indica que Oracle bloqueó la cuenta al superar los intentos fallidos permitidos por el perfil. `LOCKED` a secas indica que alguien la bloqueó expresamente con `ACCOUNT LOCK`.
  3. La nota del ticket mencionaba el script de bajas. El estado `LOCKED` confirmó que la cuenta había sido bloqueada expresamente, en vez de por intentos fallidos.
- Causa raíz: El script de bajas bloqueó la cuenta de la aplicación porque no la distinguía de las cuentas personales. No había una marca clara, como un perfil propio o una lista de exclusión, para que el script supiera qué cuentas debía dejar intactas.
- Intervención: A las 18:18:02 ejecuté `ALTER USER app_academia ACCOUNT UNLOCK`. No cambié la contraseña porque la evidencia apuntaba a un bloqueo manual, no a intentos fallidos. Si cambiaba la contraseña, la aplicación dejaría de conectar hasta que también se actualizara su configuración.
- Validación: A las 18:18:12 `app.sh` pudo entrar y la sección 7 del diagnóstico posterior quedó vacía. Evidencias: `20261124T181812Z_08-validacion-inc3.script.log` y `20261124T181814Z_08-diagnostico-inc3-despues.script.log`.
- Acción preventiva: Crear un perfil propio para las cuentas de aplicación y hacer que el script de bajas las excluya usando ese perfil como marca. Añadir también al turno un aviso si una cuenta de aplicación aparece como `LOCKED`.