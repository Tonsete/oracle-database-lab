He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E027

## Entrada 1
- **Herramienta:** Claude (Anthropic)
- **Prompt (literal, copiado tal cual):**
ahora tengo que hacer esta pracitca
pero tengo que hacer la 6 para esta tarde, no me da tiempo si tengo que responderlas y hacer tambien la 6
pero que pongo aqui
explicame que tengo que poner en cada punto que has dejado en blanco para que yo complete
los prompts almenos si me los puedes hacer tu, me haces un resumen y yo lo pego, ya lo demás si que es más opinion mia. lo de los prompts es objetivo porque es lo que esta en el chat
pero que tendra que ver la entrada 2 con el lab 5
que pongo en el que hace de cada entrada
no me lo pongas tan esquematico
hazme un md con todo lo que me has hecho de comandos para el lab 6
esto como se hace? o ya lo hacen los comandos que me has pasado
explicame todo lo necesario para hacer los postmortems
hazme un md con todos los comandos que me has pasado para saber todo. lo que llevamos hecho
explicame mas detalladamente y con mas contexto el postmortem, porque la vd es que no me he enterado de nada de los comandos que me has puesto y yo he ejecutado
vamos a hacer una cosa, ponme el lab6-guardia.md en un archivo modificable y yo lo voy resolviendo y tu vas mirando si lo voy haciendo bien y luego lo subes
explicame esto (junto con el texto que pegué)
actualizame el md que que me has creado anteriormente con todos los comandos hasta ahora que me has pasado
como lo ves? (junto con mi borrador de lab6-guardia.md, para que lo revisara)
leelo otra vez
metelo tu (junto con los campos de las entradas 2 a 5 que había escrito yo)
si
y el prompt metelo tu que eres la ia y puedes ver el chat entero, hay tanto chat que ns donde esta y metelo ya al md todo
- **Código recibido (el fragmento, o la ruta del archivo donde está):** no aplica
- **Dónde lo usé (parte, script o pregunta):** Todo el laboratorio (Partes A a J)
- **Propósito (para qué lo necesitaba):** Para hacer más ameno el laboratorio ya que es demasiado denso.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** Me explicaba paso a paso que significaban las salidas y los errores antes de que ejecutara los comandos. Tambien adaptaba estos comandos a mi mac. Los comandos los ejecutaba yo y los comits los hacía yo.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica

## Entrada 2
- **Herramienta:** Claude (Anthropic)
- **Prompt (literal, copiado tal cual):**
ahora tengo que hacer esta pracitca
pero tengo que hacer la 6 para esta tarde, no me da tiempo si tengo que responderlas y hacer tambien la 6
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```bash
grep -q EVID_LAB6 scripts/deployment/env/00-config.sh || cat >> scripts/deployment/env/00-config.sh <<'FIN'
export EVID_LAB6="docs/bitacora/lab6/evidencia"
export SQLPATH="$(pwd)"
FIN
sed -i '' 's#^source scripts/deployment/env/00-config.sh$#&; [ -f scripts/ops/cabecera.sh ] \&\& { source scripts/ops/cabecera.sh "backup-estado-${1:-}" || exit 2; }#' scripts/ops/backup-estado.sh
sed -i '' 's#docker exec -i "$CONT_NAME" sqlplus#docker exec -i -e NLS_LANG=AMERICAN_AMERICA.AL32UTF8 "$CONT_NAME" sqlplus#g' scripts/ops/backup-estado.sh
sed -i '' 's#$(date +%s%N | sha256sum | cut -c1-12)#$(od -An -N6 -tx1 /dev/urandom | tr -d " \\n")#g' scripts/lab6/06-cascada.sh
sed -i '' 's#^source scripts/deployment/env/00-config.sh$#&; command -v sha256sum > /dev/null 2>\&1 || sha256sum() { shasum -a 256 "$@"; }#' scripts/lab6/09-verificacion-final.sh
```
- **Dónde lo usé (parte, script o pregunta):** Parte A.2: scripts/deployment/env/00-config.sh, scripts/ops/backup-estado.sh, scripts/lab6/06-cascada.sh y scripts/lab6/09-verificacion-final.sh
- **Propósito (para qué lo necesitaba):** Necesitaba adaptar los scripts del kit para poder ejecutarlos en mi Mac. Algunos comandos del kit, como `sha256sum` y `date +%s%N`, no están disponibles ahí o funcionan de otra manera.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** Añade a la configuración la ruta donde guardaré las evidencias del Lab 6 y configura SQLPATH para que SQLcl encuentre login.sql en la raíz del repositorio. Adapta backup-estado.sh para que genere la cabecera de evidencia y use UTF-8 al ejecutar SQL*Plus. En 06-cascada.sh, cambia la generación de contraseñas temporales para que funcione en macOS, que no incluye date +%s%N ni sha256sum. En 09-verificacion-final.sh, añade shasum -a 256 como alternativa cuando no está disponible sha256sum. También se da permiso de ejecución a los scripts y se crean las carpetas para guardar las evidencias.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
```
20261123T192257Z_00-backup-lab6-inicio.script.log
20261124T171250Z_06-cascada.script.log
```

## Entrada 3
- **Herramienta:** Claude (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
(pegué la salida de la terminal, que terminaba en:)
scripts/lab6/03-sesiones.sh: line 40: syntax error near unexpected token `APP_PID=$!'
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):** parche aplicado con git apply a scripts/lab6/03-sesiones.sh (ver el commit «fix(ops): run 03-sesiones.sh on macOS bash 3.2»)
- **Dónde lo usé (parte, script o pregunta):** Parte D.3, scripts/lab6/03-sesiones.sh
- **Propósito (para qué lo necesitaba):** Necesitaba corregir el script para poder ejecutar la Parte D en mi Mac. Sin el arreglo, se paraba en el paso 2 con un error de sintaxis.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** Bash 3.2 de macOS daba un error con el heredoc que estaba dentro de un bloque `{ ...; }` enviado a un proceso en segundo plano. El arreglo escribe primero el SQL en un archivo temporal y luego se lo pasa al comando, evitando ese problema.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
```
20261123T194139Z_03-sesiones.script.log
```

## Entrada 4
- **Herramienta:** Claude (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
(no hubo pregunta mía: al revisar el kit tras el fallo de 03-sesiones.sh, la IA detectó el mismo problema en estos dos scripts y me propuso el arreglo antes de ejecutarlos)
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):** parche aplicado con git apply a scripts/lab6/05-resumable.sh y scripts/lab6/07-adr-trazas.sh (ver el commit «fix(ops): apply the bash 3.2 fix to 05-resumable.sh and 07-adr-trazas.sh»)
- **Dónde lo usé (parte, script o pregunta):** Partes F.2 y H.4
- **Propósito (para qué lo necesitaba):** Quería arreglar estos scripts antes de ejecutarlos para evitar que fallaran por el mismo problema de compatibilidad de Bash en macOS.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** Aplica a los scripts de espacio reanudable y de ADR/trazas el mismo cambio que a sesiones: primero escribe el SQL en un archivo temporal y después lo envía al comando, en vez de usar el heredoc dentro del bloque que Bash 3.2 no aceptaba.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
```
20261123T194519Z_05-resumable.script.log
20261124T171356Z_07-adr-trazas.script.log
```

## Entrada 5
- **Herramienta:** Claude (Anthropic)
- **Prompt (literal, copiado tal cual):**
```text
(no hubo pregunta mía: al revisar los scripts de la Parte I antes de ejecutarlos, la IA vio que 08-guardia.sh usa setsid, que no existe en macOS, y me propuso el arreglo)
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```bash
sed -i '' 's#^  setsid nohup bash -c#  $(command -v setsid || true) nohup bash -c#' scripts/lab6/08-guardia.sh
```
- **Dónde lo usé (parte, script o pregunta):** Parte I, scripts/lab6/08-guardia.sh
- **Propósito (para qué lo necesitaba):** Necesitaba que el script de guardia pudiera crear el incidente 1 en macOS. Sin el arreglo, `setsid` no se encontraba y la sesión bloqueadora no llegaba a crearse.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** En Linux, `command -v setsid` encuentra la ruta de `setsid` y el script lo usa para ejecutar el comando en una sesión nueva. En macOS, no lo encuentra; el `|| true` evita que eso detenga el script y se ejecuta `nohup` directamente. Así el incidente puede continuar también en el Mac.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):**
```
20261124T171958Z_08-diagnostico-inc1.script.log
```
