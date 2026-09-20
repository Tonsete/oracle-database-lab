1. Working directory se refiere a la carpeta local, Staging Area se refiere al área intermedia al que le pasas una captura puntual del archivo, con `git add` especificas exactamente qué archivos quieres que entren, Local Repository es donde acaba finalmente el archivo haciendo `git commit` al archivo y así guardarlo como una captura permanente en el historial de git.
Por ejemplo, `docs/customer-schema.md`: lo modifiqué en el Working Directory con echo, lo pasé a la Staging Area con git add docs/customer-schema.md, y quedó guardado en el Local Repository al hacer git commit.


1. Si modificas el archivo y no haces `git add` los cambios no se te guardarán en el próximo commit, porque `git commit` solo confirma lo que está en la Staging Area, no lo que hay en el disco. Además `git add` hace una captura puntual de como está el archivo en ese momento. Por eso, si haces add y luego vuelves a modificar el archivo antes de comitear, la Staging Area se queda con la versión antigua tendrías que repetir git add para actualizarla.


3. El comando `git status` no muestra las carpetas porque git solo trackea archivos no carpetas. El truco que usábamos para que aparecieran las carpetas era meter un archivo .gitkeep vacío dentro de cada carpeta para que así git tenga algo que trackear dentro de cada carpeta.


4. HEAD es un puntero que indica en qué branch (y por tanto en qué commit) estás ahora mismo. Es lo que determina qué archivos ves en tu carpeta de trabajo, porque Git se coloca exactamente en el contenido del commit al que apunta HEAD en cada momento.


5. Una branch no es una carpeta física. Es un puntero que apunta a una línea de commits. `mkdir`crea una carpeta real y visible en el sistema de archivos.
En la parte G lo comprobamos ejecutando `ls -la` antes y después de `git switch -c feature/customer-search`: el listado de archivos y carpetas fue exactamente el mismo, ninguna carpeta nueva apareció.


6. Cuando Git no puede decidir automáticamente qué versión de una línea conservar, mete las dos versiones en el archivo, separadas por marcadores. Entre `<<<<<<< HEAD` y `=======` está el contenido de la branch que tienes activa en el momento de ejecutar `git merge` (en mi caso, main, con "Training Edition"). Entre `=======` y `>>>>>>> fix/readme-subtitle` está el contenido de la branch que se está fusionando ("Academic Version"). Y digo que se está y no que se ha fusionado ya, porque el merge se queda en un estado de pausa hasta que se abra el archivo y se decida qué versión dejar y se borren los marcadores.


7. `--amend`crea un nuevo commit con distinto hash y este sustituye al commit anterior. Si el commit anterior ya se subió con `git push`, el remoto y cualquiera que lo haya descargado sigue teniendo la versión vieja; al hacer `--amend`y volver a hacer push tu historial y el del remoto ya no coinciden, lo que provoca un rechazo.


8. Si borras por accidente `.git`se borran todos los commits, branches, remotos y los tags, porque `.git` es la base de datos interna donde Git guarda absolutamente todo el historial de versiones. Lo único que sobrevive son los archivos que tienes en local, en la carpeta del proyecto, porque estos viven fuera del `.git`.


9. Git es un programa que se ejecuta en el ordenador y se gestiona de forma local. GitHub es una empresa que aloja una copia de tu repositorio en sus servidores y añade herramientas adicionales para colaborar con Git.


10. Es peligroso subir un `.env` con contraseñas reales porque cuando haces un commit este se vuelve permanente. Aunque el repositorio sea privado esto solo significa quién puede verlo en este momento, pero eso puede cambiar si añades un colaborador más adelante y en ese momento aunque hayas borrado el `.env`, él podrá acceder al historial de ese archivo con `git log -p -- .env` y así ver el contenido de ese `.env` "borrado".


11. Lo que probablemente haya pasado es que alguien más subió commits a la misma rama entre el primer y segundo push. La rama local se quedó atrás y el remoto avanzó, por lo que el push ya no es una continuación de lo que hay en GitHub. Para solucionarlo, lo primero que ejecutaría es `git pull`, para traer esos commits que me faltan y fusionarlos con los míos (resolviendo el conflicto si aparece alguno). Una vez que mi copia local tiene todo lo que tiene el remoto, ya puedo volver a hacer `git push` sin que lo rechace.


12. 
- Para añadir un índice de rendimiento a una tabla usaría `perf`
  - Este es una mejora de rendimiento, no una funcionalidad nueva.
- Para corregir una restricción mal definida usaría `fix`
  - Este sirve para arreglar algo que ya existía y estaba mal en la BD.
- Para actualizar el README usaría `docs`
  - Este solo cambia la documentación, no afecta al código.