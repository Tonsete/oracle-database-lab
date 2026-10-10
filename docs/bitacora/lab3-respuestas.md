# Laboratorio 3 — Preguntas de comprobación

Administración de Bases de Datos — Práctica 1, Laboratorio 3
Autor: Antonio García Gallego (Tonsete)
Repositorio: Tonsete/oracle-database-lab · Issue #5
Entorno: macOS ARM64 (Apple Silicon) + OrbStack (Docker 29.4.0), Java 21.0.12.1, SQLcl 26.3.0, Oracle 23.26.3.0.0, ORDS 26.3.0

---

## Docker (1-7)

### 1. ¿Qué diferencia hay entre una imagen y un contenedor? Usa como ejemplo lo que hiciste en los ejercicios G2 y G4.

La imagen es la plantilla de solo lectura (la receta), el contenedor es una instancia viva creada a partir de ella (el plato cocinado). En G2 `docker run hello-world` descargó la imagen, creó un contenedor que imprimió su mensaje y terminó (Exited 0); la imagen siguió intacta y pude crear otro contenedor idéntico. En G4 `docker run -it --name prueba alpine:3.20 sh` creó otro contenedor a partir de otra plantilla y me dejó dentro (hostname = ID del contenedor, `/etc/os-release` = Alpine aunque mi equipo sea macOS). Puedo borrar contenedores sin afectar a la imagen.

### 2. En el Ejercicio G5 el archivo nota.txt desapareció y en el G6 no. Explica por qué.

En G5 escribí en el sistema de archivos propio del contenedor (`/tmp/nota.txt`). Al hacer `docker rm prueba` ese sistema se destruyó, y el contenedor nuevo `prueba2` partió de la plantilla limpia: `cat: No such file`. En G6 monté un named volume (`-v datos-prueba:/datos`) y escribí en `/datos/nota.txt`, que vive fuera del contenedor en `/var/lib/docker/volumes/`. Aunque borré el contenedor con `--rm`, el volumen siguió existiendo y el segundo contenedor encontró el dato. Es exactamente por lo que Oracle usa `-v oralab-26ai-data:/opt/oracle/oradata`.

### 3. ¿Qué diferencia hay entre docker ps y docker ps -a, y qué significa STATUS = Exited (0)?

`docker ps` solo lista contenedores en marcha (Up). `docker ps -a` lista todos, incluidos detenidos. `Exited (0)` significa que el proceso principal terminó por sí solo con código 0 (éxito): es lo que hizo `hello-world` tras imprimir. `Exited (1 o más)` sería error. `Up X minutes (healthy)` como `oralab-26ai` significa que el proceso principal (la base) sigue vivo y su healthcheck responde bien.

### 4. En -p 8181:8181, ¿qué número corresponde a tu equipo y cuál al contenedor? ¿Qué pasaría con -p 80:8080 en el ejercicio de nginx?

Formato `anfitrión:contenedor` (primero lo mío, después lo del contenedor). En `-p 8181:8181` mi 8181 redirige al 8181 del contenedor; en Oracle `-p 1521:1521` mi 1521 va al listener del contenedor. En nginx la imagen escucha en 80, por eso es `-p 8080:80` (mi 8080 → su 80, abro `http://localhost:8080`). Con `-p 80:8080` invertiría el mapeo: mi 80 iría al 8080 del contenedor donde nginx no escucha y la página no cargaría.

### 5. ¿Por qué un contenedor de Oracle se queda en marcha y el de hello-world termina solo?

Un contenedor vive exactamente lo que vive su proceso principal. `hello-world` ejecuta `/hello` que imprime y sale: el contenedor pasa a Exited. `alpine sh` vive hasta que escribo `exit`. Oracle ejecuta el motor de base de datos (proceso que no termina nunca salvo `docker stop`): por eso `oralab-26ai` lleva `Up 47 minutes (healthy)`. Si el motor cayese, el contenedor pasaría a Exited o Restarting.

### 6. ¿Qué es el digest de una imagen y por qué lo registramos si ya sabemos que usamos :latest?

El digest (`sha256:f988b0c04c4c386cd306a2a914c0d7a9702d83acc31b064a28ad8eb6278a8fba` en mi evidencia 04) es la huella exacta bit a bit de la imagen. `:latest` es una etiqueta móvil: hoy apunta a 23.26.3.0.0 (9.46 GB en disco), el mes que viene puede apuntar a otra build distinta. Registrar el digest permite a cualquiera saber qué versión exacta instalé aunque `:latest` haya cambiado. En producción se hace pin del digest, nunca se despliega `:latest`.

### 7. ¿Qué comando borraría realmente los datos de Oracle? ¿Por qué docker rm oralab-26ai no lo hace?

Los datos viven en el named volume `oralab-26ai-data` (5.3 GB en mi `docker system df`), no en el contenedor. `docker rm oralab-26ai` solo borra el proceso y su sistema temporal; el volumen sobrevive y un contenedor nuevo lo recupera. Lo que borra de verdad es `docker volume rm oralab-26ai-data` (irreversible) o `docker system prune -a --volumes` sin leer (borra volúmenes no usados). Por eso la regla es: nunca borrar masivo sin mirar `docker volume ls`.

---

## Git, organización y evidencia (8-12)

### 8. ¿Por qué este laboratorio se hace dentro del repositorio oracle-database-lab, con Issue, branch y Pull Request, en vez de en una carpeta aparte?

Porque instalar no es distinto de programar: es un cambio de infraestructura que debe ser reproducible, verificable y revisado. Con Issue #5 → `chore/5-install-oracle-environment` → 14 commits (uno por Parte) → PR → merge, cualquiera puede reconstruir mi entorno con `05-crear-contenedor.sh` + `08-aplicar-migraciones.sh`, mi revisor ve scripts + salida real (evidencias 01-13 con marca UTC), y no hay "snowflake server" que solo funcione en mi máquina. En una carpeta aparte no habría historial ni revisión.

### 9. ¿Qué diferencia hay entre source 00-config.sh y bash 00-config.sh? ¿Por qué usamos source?

`bash 00-config.sh` lo ejecuta en una shell hija que muere al terminar: las `export` se pierden. `source 00-config.sh` lo ejecuta en mi shell actual: `CONT_NAME`, `VOL_NAME`, `EVID`, `ts()` quedan disponibles para los siguientes comandos (`echo "$CONT_NAME"` → `oralab-26ai`). Por eso cada terminal nueva empieza vacía y hay que repetir el `source`; si lo olvido las rutas salen raras (`/script/_01-...` sin `docs/...` ni marca).

### 10. Explica cada parte del nombre 20260915T091230Z_02-docker.script.log.

`20260915T091230Z` = marca ISO 8601 en UTC (2026-09-15 09:12:30 Zulu): ordena cronológicamente sin depender de mi zona horaria. `_02` = número del paso que la produjo (Parte F, Docker, según el mapa 01-13). `-docker` = descripción en kebab-case (minúsculas, guiones, sin tildes/espacios, portable). `.script.log` = sufijo que dice que es salida de terminal (`.spool.log` sería sesión SQL, `.png` captura). Ejemplo real mío: `20261010T214634Z_13-verificacion-final.script.log`.

### 11. ¿Para qué sirve .gitattributes y qué error evita?

Declara una vez cómo normaliza Git los finales de línea para todo el equipo (`*.sh text eol=lf`, `*.sql`, `*.md`). Sin él, un `.sh` guardado en Windows lleva CRLF (`\r\n`) y al ejecutarse en Linux/Contenedor falla con `$'\r': command not found`. Con `.gitattributes` Git guarda siempre LF aunque cada persona use Windows, macOS o Linux, y los diffs no se llenan de ruido invisible. Es higiene cross-platform de equipos mixtos.

### 12. ¿Por qué en este Pull Request elegimos Create a merge commit en lugar de Squash and merge?

En el Lab 2 hice squash porque eran correcciones de un mismo doc. Aquí cada commit tiene valor propio y verificable (requisitos, Docker, tutorial, imagen, contenedor, arranque, migraciones, Java/SQLcl, ORDS, verificación, respuestas): quiero conservar el paso a paso para ver cuándo y cómo se verificó cada herramienta (`git log --oneline --graph`). Squash los aplastaría en uno solo y perdería la trazabilidad del audit trail. Merge commit conserva la historia.

---

## Seguridad (13-15)

### 13. Describe las cuatro capas de la estrategia de contraseñas (Parte D) y qué pasaría si te saltas la primera.

Capa 1: decirle a Git qué ignorar **antes** de crear secretos (`echo "config/.env" >> .gitignore`). Capa 2: plantilla versionada sin valores (`config/.env.example` con `change_me`, más `ORDS_PUBLIC_PWD=change_me_ords`). Capa 3: archivo real solo local (`cp .env.example .env` + contraseñas fuertes, verificado con `git check-ignore -v` y `git status`). Capa 4: leer sin teclear (`set -a; source config/.env; set +a` y usar `"$ORACLE_PWD"`, nunca literal). Si me salto la 1, `config/.env` con `XWECSH...` entraría en `git status` y acabaría en un commit para siempre (aunque lo borre después sigue en historial y GitHub lo escanea como incidente).

### 14. ¿Por qué no escribimos la contraseña directamente en el comando docker run, aunque el script no se suba a Git?

Porque todo lo tecleado queda en `~/.zsh_history` / `~/.bash_history` en texto plano para siempre, y en la grabación `script` de la Parte G o en un `tee` capturado. Con `-e ORACLE_PWD="$ORACLE_PWD"` en el historial solo queda el nombre de la variable; con `-e ORACLE_PWD=MiClave123` la contraseña real queda expuesta aunque el script esté gitignored. Además el patrón plantilla + inyección es el que usan los pipelines (código en Git, secreto desde almacén en ejecución).

### 15. Si descubres tu contraseña en un commit ya publicado, ¿basta con borrarla en un commit nuevo? ¿Qué debes hacer?

No basta: Git nunca olvida, cualquiera puede hacer `git log -p` o navegar al commit antiguo y recuperarla. Se da por comprometida y se rota inmediatamente: cambiar `ORACLE_PWD`/`APP_USER_PWD`/`ORDS_PUBLIC_PWD` en `config/.env`, recrear el contenedor (`docker rm -f` + `docker volume rm` si aún está vacío, o `ALTER USER` si ya tiene datos), avisar al docente/revisor, y limpiar la rama (en empresa: revocar token, auditar accesos). Mi control O.2 (`git grep -F "$ORACLE_PWD"` → `OK: no aparece`, `git log -p | grep -c` → `0`) demuestra que no me pasó.

---

## Oracle y herramientas (16-20)

### 16. ¿Por qué no usamos SPOOL ni @archivo.sql con sqlplus dentro del contenedor, y qué hicimos en su lugar?

Porque `sqlplus` corre **dentro** del contenedor (`docker exec -i`). Un `SPOOL /tmp/x.log` escribiría dentro del contenedor, no en mi repo; y `@archivo.sql` buscaría el archivo dentro del contenedor donde no existe (`SP2-0310`). Lo resolví desde mi equipo: redirección de entrada `< scripts/.../07-primera-conexion.sql` (mi archivo entra por stdin del sqlplus del contenedor) y `| tee docs/.../07-....spool.log` (guardo la salida en mi repo). Con SQLcl instalado en mi Mac (Parte L) sí uso `SPOOL` normal porque ya corre fuera.

### 17. ¿Qué hace WHENEVER SQLERROR EXIT SQL.SQLCODE al inicio de V000 y V001, y qué pasaría sin esa línea?

Hace fail-fast: si cualquier sentencia falla, sqlplus sale inmediatamente con ese código ORA como exit code. Mi `08-aplicar-migraciones.sh` usa `set -euo pipefail` + `if ! ... | tee; then exit 1`, así que V001 nunca se ejecuta sobre un V000 a medias. Sin esa línea, sqlplus seguiría con las siguientes sentencias aunque una falle (p.ej. `ORA-01920: user already exists` en segunda ejecución o `ORA-01543: tablespace exists`), dejaría esquemas a medias y la verificación final (3,3,2,3,3) no cuadraría sin que el script avisase.

### 18. ¿Qué es una migración y por qué V000 y V001 no se deben editar una vez aplicadas?

Una migración es un script SQL versionado y numerado que lleva la base de un estado al siguiente (como Flyway/Liquibase). V000 crea tablespaces+usuarios, V001 crea tablas; `08-aplicar-migraciones.sh` las aplica en orden en un `for`. Una vez aplicadas no se editan porque mis compañeros ya las ejecutaron: si cambio V000 después, su base y la mía divergen sin forma de saberlo. Si hay que corregir, se crea V002 nueva. Así toda la clase tiene idénticos `ADMIN_ACADEMIA...ADMIN_RETAIL` (lo comprobé comparando mi spool 08 con el esperado).

### 19. ¿Por qué en SQL Developer se usa el servicio FREEPDB1 y no FREE ni un SID?

Porque trabajo en la PDB `FREEPDB1`, no en el contenedor raíz CDB `FREE`. `FREE` (SERVICE_CDB) es administración del contenedor; `FREEPDB1` (SERVICE_PDB) es donde están mis 5 esquemas, ORDS y ALUMNO. Además hay que elegir Service name, no SID: SID conecta a instancia, Service name a servicio registrado en el listener (ORA-12514 si está arrancando, ORA-12505 si pongo SID mal). Mi `SHOW CON_NAME` → `FREEPDB1` y `SELECT ... WHERE owner LIKE 'ADMIN\_%'` lo confirman.

### 20. ¿Qué aporta SQLcl frente a SQL*Plus, y por qué un DBA debe dominar ambas?

SQL*Plus (Parte J) está en cualquier servidor Oracle desde 1982: a las 3am con solo ssh es lo que hay, sin instalar nada. SQLcl (Parte L, 26.3.0 en mi evidencia 09) es la herramienta moderna del día a día: `SET SQLFORMAT ansiconsole` (columnas legibles sin PAGESIZE manual), historial y autocompletado, `CONNECT -save oralab26-system` + `CONNMGR list`, salida JSON/CSV, e integración Liquibase. Uso SQL*Plus dentro del contenedor para instalación y SQLcl en mi Mac con `SPOOL` directo al repo para trabajo diario. Un DBA domina ambas porque en producción no siempre puede instalar.

---

## Entorno de trabajo (21-22)

### 21. ¿Por qué el curso pasa de Git Bash a Ubuntu en WSL 2? Da al menos dos problemas concretos de Git Bash que desaparecen en Ubuntu.

Porque Oracle, Docker y los servidores son Linux: quiero mismos comandos, rutas (`/home`, `/opt`, `/var/log`), `apt` y permisos que en producción. Git Bash es emulación y se nota: convierte rutas `/opt/...` a Windows y rompe args de Docker (`-v`), necesita `winpty` para `docker run -it` ("not a TTY"), y no trae `free`, `ss`, `htop`, `tmux` ni maneja bien contraseñas Java. En Ubuntu/WSL 2 (y en mi caso macOS nativo con OrbStack, que es Unix real) todo eso funciona igual que en un servidor, y Docker publica `1521/8080` en `localhost` para SQL Developer/navegador. Yo estoy en macOS ARM64 con OrbStack 2.2.3, equivalente Unix sin emulación.

### 22. ¿Por qué clonamos el repositorio en ~/oracle-database-lab y no trabajamos sobre la carpeta de Windows (/mnt/c/...)? ¿Y por qué recomendamos bash frente a zsh para los scripts del curso?

Trabajar en `/mnt/c/...` cruza dos filesystems en cada operación: Git/Docker van varias veces más lentos, se pierden permisos de ejecución (`+x`) de los `.sh` y reaparecen CRLF. En `~/oracle-database-lab` (`/home/...` en WSL, `/Users/...` en Mac) todo es nativo y `git status` es limpio. Recomiendo bash porque es lo único garantizado en todos los servidores (Ubuntu, RHEL, Oracle Linux, contenedores): lo que escribo a mano y lo que va en `#!/usr/bin/env bash` se comporta igual. zsh (mi interactiva en Mac, `/bin/zsh`) está bien para comodidad (Oh My Zsh, autocompletado), pero tiene diferencias sutiles (arrays, globbing) y normalmente no existe en servidores; por eso ejecuto siempre `bash script.sh` aunque mi terminal sea zsh.
