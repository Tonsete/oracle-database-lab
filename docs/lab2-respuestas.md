# Laboratorio 2 — Preguntas de comprobación

Administración de Bases de Datos — Práctica 1, Laboratorio 2
Autor: Antonio García Gallego (Tonsete) · Revisor: Marcos García Arroyo
Repositorio: Tonsete/oracle-database-lab · Issue #1 · Pull Request #2

---

## 1. ¿Por qué un Issue sin criterios de aceptación es un problema?

Porque "está claro" es una opinión, no una comprobación. Sin criterios, yo puedo pensar que ya está terminado y el revisor que falta la mitad, y no hay forma de saber quién tiene razón. La checklist del Issue #1 me decía exactamente cuándo parar, y luego le sirvió a Marcos para revisar.

## 2. Diferencia entre "Refs #N" y "Closes #N"

"Refs" solo deja un enlace visible desde el Issue, que sigue abierto. "Closes" lo cierra automáticamente cuando el cambio llega a main. Usé "Refs #1" en el commit y "Closes #1" en el PR. Poner "Closes" en un commit de una rama sin fusionar no cierra nada todavía.

## 3. ¿Qué ocurre si haces push directo sobre main protegida?

GitHub rechaza el push. A mí me salió `GH013: Changes must be made through a pull request` y el commit se quedó en mi Mac. No es un fallo, es la protección haciendo su trabajo. Lo que hice fue `git reset --hard HEAD~1` y seguir por el Pull Request, que es el camino correcto.

## 4. "He aprobado el PR sin mirar los archivos, total ya me fío"

Entonces la aprobación no vale nada y encima es peor que no revisar, porque el equipo cree que hay un control que en realidad no existe. Fiarse no detecta errores: cualquiera puede dejarse un caso raro o un permiso de más sin querer. Se revisa para ver lo que al otro se le ha pasado, no por desconfianza.

## 5. Si el reviewer pide un cambio, ¿hay que abrir un PR nuevo?

No. El PR está atado a la rama, no a un commit: sigues haciendo commits ahí y el PR se actualiza solo. Me pasó tal cual, añadí `security`, hice push y el PR #2 pasó a 2 commits sin tocar nada en la web. Abrir otro PR rompería el hilo de la revisión.

## 6. Merge commit, Squash and merge y Rebase and merge

Merge commit guarda todos los commits más uno de fusión. Squash los junta en uno solo. Rebase los pega en main en línea recta, sin commit de fusión. Para una rama con "wip", "fix", "fix2", "ok ya" usaría Squash: esos mensajes no le sirven a nadie. Es lo que hice, mis dos commits quedaron en uno (`6d24a0d`).

## 7. ¿Por qué borrar una rama no elimina el trabajo?

Porque una rama es solo un puntero a un commit, no una carpeta donde vive el código. Al fusionar, el contenido ya está en main, así que borrar el puntero no borra nada. Lo vi en el grafo: borré la rama y `6d24a0d` sigue ahí con todo.

## 8. ¿Qué debe tener siempre la descripción de un PR?

Qué hace el cambio, qué archivos toca, cómo se ha probado y por qué se hace, con el Issue enlazado. Si no lo pones, el revisor te lo va a preguntar igual, y en un equipo repartido por zonas horarias esa pregunta puede costar un día entero de espera.

## 9. Un reviewer escribe solo "esto está mal"

Le falta decir qué ve, por qué es un problema y qué propone. Así solo consigue que el otro se ponga a la defensiva. Tampoco deja claro si hay que corregirlo o es su gusto personal. Yo lo escribiría así:

> issue (blocking): esto asume que la consulta siempre devuelve alguna fila, pero si la tabla está vacía el acceso al índice [0] peta. Propongo comprobar el tamaño antes y devolver Optional.empty().

## 10. main protegida frente a "nos ponemos de acuerdo"

Un acuerdo depende de que nadie se despiste; la protección no depende de nadie. GitHub rechaza el push siempre, tenga el día que tenga. Es pasar de confiar en que el compañero lo hizo bien a poder comprobarlo.

## 11. issue: (blocking) frente a nitpick: (if-minor)

El primero hay que arreglarlo antes de aprobar; el segundo es un detalle que no debería frenar nada.

> issue (blocking): la migración no tiene rollback. Si falla en producción no hay forma de volver atrás.

> nitpick (if-minor): la tabla se llama CLIENTES y el resto del esquema está en inglés, si te viene bien ponla como CUSTOMERS.

Lo bueno de las etiquetas es que la gravedad la dices tú, no se deduce del tono.

## 12. `feat!:` cambiando la firma de la función principal

Se dispara una MAJOR, de 2.1.0 a 3.0.0. `feat` solo sería MINOR, pero la exclamación marca un cambio que rompe compatibilidad: quien ya usaba esa función deja de poder hacerlo. Por eso el tipo de commit no es decorativo, las herramientas de release calculan la versión leyéndolo.

## 13. ¿Por qué un Draft PR ahorra tiempo?

Porque lo caro no es programar, es programar lo que no era. Si lo enseño al segundo día, alguien me dice que ese enfoque no encaja y he perdido dos horas en vez de dos semanas. Y evita que acabe defendiendo un mal diseño solo porque ya le he metido mucho tiempo.
