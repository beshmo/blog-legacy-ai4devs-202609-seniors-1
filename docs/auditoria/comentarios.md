# Auditoría de la feature «comentarios»

Alcance: crear, editar, borrar y listar comentarios, en sus tres capas.

- **API:** `POST /posts/:slug/comentarios`, `PATCH /comentarios/:id`, `DELETE /comentarios/:id` y el trozo de comentarios de `GET /posts/:slug` (`backend/start/routes.ts:152`). El recuento `num_comentarios` de `GET /posts` (`:75`) y de `/relacionados` (`:222`) también cuenta, porque es lo que la web enseña de la feature.
- **Tabla:** `comentarios` (`backend/database/migrations/1769000000005_create_comentarios_table.ts`).
- **Web:** `frontend/src/componentes/Comentarios.jsx`, el recuento de `DetallePost.jsx` y la tarjeta de `ListaPosts.jsx`.

No se audita posts, autores, categorías ni etiquetas. Donde un hallazgo de comentarios toca a otra feature, se nombra y no se desarrolla.

Cómo se leyó: cada «dónde está» se ha comprobado abriendo la línea. Los datos de la tabla se han consultado en `backend/tmp/db.sqlite3` en modo solo lectura. Nada se ha ejecutado contra la API en marcha.

## Cómo se ordenó

1. **Nivel = precedencia.**
   - Nivel 0: desbloquea a casi todos los demás.
   - Nivel 1: independiente, se puede hacer ya sin esperar a nadie.
   - Nivel 2: espera al Nivel 0 y a su vez desbloquea a otros.
   - Nivel 3: espera y no desbloquea nada.
2. **Dentro del nivel, daño / radio.**
   - Daño es un juicio mío de 1 a 5: a cuánta gente llega y si se puede deshacer.
   - Radio es la casilla 4, los sitios contados.
3. **Partidos en dos filas** (daño máximo con precedencia baja):
   - Clave de moderación: **#6** contención y **#14** fondo.
   - Identidad por correo: **#3** contención y **#8** fondo.
4. **Orden que desmiente la regla.** #6 cae cuarto por radio (4 archivos) aunque su daño es 5. **#3 y #6 se despliegan juntos**: cada uno cierra uno de los dos caminos que tiene el mismo `DELETE`, y por separado el otro sigue abierto.

| # | Nivel | Daño | Radio | Daño/radio |
|---|---|---|---|---|
| 1 | 0 | desbloqueador | 6 | n/a |
| 2 | 0 | desbloqueador | 13 | n/a |
| 3 | 1 | 5 | 1 | 5,00 |
| 4 | 1 | 4 | 1 | 4,00 |
| 5 | 1 | 2 | 1 | 2,00 |
| 6 | 1 | 5 | 4 | 1,25 |
| 7 | 1 | 2 | 7 | 0,29 |
| 8 | 2 | 5 | 4 | 1,25 |
| 9 | 2 | 4 | 9 | 0,44 |
| 10 | 2 | 2 | 5 | 0,40 |
| 11 | 3 | 3 | 1 | 3,00 |
| 12 | 3 | 3 | 1 | 3,00 |
| 13 | 3 | 2 | 1 | 2,00 |
| 14 | 3 | 5 | 3 | 1,67 |
| 15 | 3 | 2 | 2 | 1,00 |
| 16 | 3 | 2 | 2 | 1,00 |
| 17 | 3 | 1 | 1 | 1,00 |

## Hallazgos

### Nivel 0 — desbloquean a otros

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe, y quién lo nota | 4. Sitios | 5. ¿Revert? | 6. ¿Bloquea / desbloquea? |
|---|---|---|---|---|---|---|
| 1 | La feature no tiene tests de caracterización: el único test cubre el camino feliz de `POST`. | `backend/tests/functional/comentarios.spec.ts:16` (único `test`; la cabecera, líneas 5-10, lo admite). El umbral de cobertura está en 0: `.claude/hooks/guard-tests.mjs:32`. | Nadie todavía. Cualquier cambio de esta lista se haría sin red, y `CLAUDE.md` prohíbe refactorizar sin caracterización. | 6 superficies sin test: `PATCH`, `DELETE`, el trozo de comentarios de `GET /posts/:slug`, `num_comentarios` en `GET /posts` (`:75`) y en `/relacionados` (`:222`), y los caminos no felices de `POST`. Se amplía 1 archivo de test existente. | **Sí.** Solo añade tests. | **Desbloquea** a todos los demás: el #2 lo necesita para refactorizar, y cada arreglo de bug (#3 en adelante) necesita su caracterización. |
| 2 | Toda la lógica de comentarios vive dentro de la ruta: SQL, validación, autorización y forma de la respuesta, sin controlador, validador, servicio, repositorio ni transformer. | `backend/start/routes.ts:295-393` (tres handlers); consultas de comentarios en `:75`, `:152` y `:222`. Prueba del coste: la regla de «quién es el autor» está escrita dos veces y salió distinta, `:375-377`. | Nadie todavía en producción. Cada corrección de abajo hay que hacerla a mano en cada copia. | 13: 3 handlers + 3 consultas en `routes.ts` (6); alias `#repositories/*` en `package.json` (1); 5 archivos nuevos (controlador, validador, servicio, repositorio, transformer); `app/exceptions/handler.ts` para centralizar errores (1). | **Sí**, si se hace por pasos con la suite en verde. No toca datos. | **Bloqueado por #1.** **Desbloquea** #8, #9, #10, #12, #13, #14, #15, #16, #17. Los Niveles 2 y 3 se escriben una sola vez, en la capa correcta. |

### Nivel 1 — independientes, se hacen ya

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe, y quién lo nota | 4. Sitios | 5. ¿Revert? | 6. ¿Bloquea / desbloquea? |
|---|---|---|---|---|---|---|
| 3 | **Contención.** Cualquiera que conozca el correo de quien comentó puede borrar su comentario, porque `DELETE` solo compara el correo, que no es un secreto. | `backend/start/routes.ts:352-353` (`esAutor` compara solo `autor_email`) y `:355` (la rama que deja pasar). El correo es público: ver #4. | Cualquier autor de comentario aprobado pierde su comentario, y el borrado es duro (#12). Lo nota el autor al ver que desapareció. La contención cuesta **a nadie**: la web no llama a `DELETE`. Solo hay dos `fetch` a comentarios, `Comentarios.jsx:19` y `:31`, y ninguno borra. Quitar `esAutor` cambia el contrato de la API, así que es decisión explícita antes de tocar. | 1 línea (`routes.ts:355`) + su test. | **Parcial.** El código sí; los comentarios ya borrados no vuelven. | **Desbloquea** #8 (el fondo sustituye al correo por un token). **Va junto a #6**: sin #6 queda la clave filtrada. No cierra el caso del correo del staff: `routes.ts:168` devuelve `post.autor_email` con `select *`, y el comentario 2 de la semilla es de `marta.ruiz@corriente.es`. Eso es de posts/autores y hay que avisar a quien lo audite. |
| 4 | El correo de quien comenta sale en la respuesta pública del post. | `backend/start/routes.ts:152` (`select * from comentarios`) devuelto en `:171`. Los 8 campos de la fila salen: también `estado` y `post_id`, que la web no usa. | Cualquier visitante que abra un post ve los correos de los comentaristas aprobados (por ejemplo `ana.cortes@ejemplo.es`). Incumple la regla de negocio 6 de `CLAUDE.md`. Además alimenta el #3, que usa ese correo como credencial. | 1 línea para cortarlo (`:152`, listar las columnas que la web usa: `id`, `autor_nombre`, `cuerpo`, `creado_en`, `padre_id`). El transformer definitivo es parte del #2. | **Código sí.** Los correos ya servidos no se pueden recuperar. | **Alimenta** el #3. Cortarlo no cierra el #3 por sí solo (correo del staff, ver #3). Su forma definitiva (transformer) **queda bloqueada por #2**. |
| 5 | `POST` devuelve «el último comentario de la tabla», no el que acaba de insertar. | `backend/start/routes.ts:329-331` (`select * from comentarios order by id desc limit 1`, sin transacción tras el `insert` de `:324`). | Un visitante que comenta a la vez que otro puede recibir el comentario del otro, con su correo (`:333`). Con el tráfico de hoy casi nadie; **no lo he reproducido**. | 1 (`:324-331`, usar el id del `insert`). | **Sí.** | A ninguno. Su sitio definitivo es el repositorio (#2). |
| 6 | **Contención.** La clave de moderación está escrita en el código y vale para borrar cualquier comentario. | `backend/start/routes.ts:25` (`SECRETO_SESION = 'dev-secret-cambiar'`) y `:351` (`clave === SECRETO_SESION` convierte a cualquiera en moderador). Se envía en la cabecera `x-clave-moderacion`. | Cualquiera con lectura del repositorio borra cualquier comentario. Lo nota el moderador al ver desaparecer comentarios sin explicación. | 4 archivos versionados: `routes.ts:25`, `start/env.ts`, `.env.example`, `.env.test`. Más la rotación en cada entorno desplegado, que **no se puede contar desde aquí**. | **No**, por la clave. El código se revierte; el valor ya está en el historial de git desde `e0e0943` y solo la rotación lo invalida. | **Va junto a #3.** El fondo es el #14, que espera al #2. |
| 7 | La web mantiene dos recuentos de comentarios independientes y el de abajo sube con un comentario que no se ve. | `frontend/src/componentes/Comentarios.jsx:11`, `:23`, `:39` (`setTotal(total + 1)`), `:54` (`<h2>`) y `DetallePost.jsx:30`, `:38`, `:71`. | Quien envía un comentario (queda `pendiente`, no se pinta) ve el `<h2>` pasar de N a N+1 comentarios mientras la lista sigue igual y la cabecera del artículo no cambia. Lo nota cada persona que comenta. | 7 líneas en 2 archivos (4 en `Comentarios.jsx`, 3 en `DetallePost.jsx`). | **Sí.** | **Parcial con #9**: este arregla la web, #9 arregla de dónde viene el número. No dependen entre sí para empezar. |

### Nivel 2 — esperan al Nivel 0 y desbloquean a otros

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe, y quién lo nota | 4. Sitios | 5. ¿Revert? | 6. ¿Bloquea / desbloquea? |
|---|---|---|---|---|---|---|
| 8 | **Fondo.** La identidad de quien escribe es su correo, un dato que no es secreto, tanto para borrar como para editar. | `backend/start/routes.ts:352-353` (`DELETE`) y `:378-379` (`PATCH`). | Ver #3: cualquiera que sepa el correo actúa como su dueño. En `PATCH` el daño es menor: solo permite editar comentarios `pendiente`, que no son públicos. Lo nota el autor del comentario. | 4: 2 reglas de autorización (`:352`, `:378`) + 1 migración (token por comentario) + el `POST` que lo devuelve (`:333`). La web no consume editar/borrar, así que no hay sitios en `frontend/`. | **Sí.** `down()` quita la columna; los tokens se regeneran. | **Bloqueado por #1, #2 y #3.** **Absorbe el #16**: si se hace, el #16 deja de existir. |
| 9 | «Aprobado» significa tres cosas distintas: `estado` es texto libre y las consultas no usan el mismo criterio. | `backend/start/routes.ts:75-76` y `:223` (`estado <> 'pendiente'`) contra `:153` (`estado = 'aprobado'`). Sin `CHECK`: migración `:18` (`table.string('estado')`). Datos en `backend/tmp/db.sqlite3`: `aprobado` 4, `Aprobado` 2, `ok` 1, `pendiente` 2. | Todo lector: la tarjeta del post 1 dice **4** comentarios y el detalle enseña **2**. El post 6 dice **1** y enseña **0**. Incumple la regla 5 de `CLAUDE.md` (solo aprobados). Cifras consultadas con las mismas condiciones SQL. | 9: 5 consultas con `estado` (`:75`, `:153`, `:223`, `:326`, `:379`) + 1 migración + 3 filas de datos. | **Código sí. Datos no.** Normalizar `Aprobado` y `ok` a `aprobado` pierde cuál era cuál; un `down()` no lo sabe sin guardar los ids de esas 3 filas. | **Bloqueado por #1** (caracterizar el 4 contra 2 de hoy) **y por una decisión explícita**: qué cuenta como visible. **Desbloquea** el cierre completo del #7 y el #12 (las mismas 3 consultas). |
| 10 | Los errores de `POST` tienen cuatro formas distintas, y un cuerpo corto devuelve `200`. | `backend/start/routes.ts:311` (`400 {error}`), `:314` (`422 {message}`), `:317` (`400 {errores:[]}`), `:320` (`200 {ok:false, motivo}`); `PATCH` en `:385` (`400 {error}`). La web lo esquiva en `Comentarios.jsx:44` (`d.error \|\| d.message \|\| d.motivo`). | Cualquier cliente que no sea la web trata un comentario rechazado como éxito. La web solo funciona porque mira `d.comentario`. | 5: 4 retornos de `POST` + `PATCH :385`. Más 1 línea de la web (`Comentarios.jsx:44`) cuando se unifique. | **Sí** el código. Pasar `200` a `422` cambia el contrato público: es un bug que se corrige con decisión explícita. | **Bloqueado por #1** (fijar `200 {ok:false}` primero) **y #2** (validador VineJS). **Desbloquea** #11, #13, #15 y #17: cuelgan todos de la misma forma de error. |

### Nivel 3 — esperan y no desbloquean nada

| # | 1. Qué es | 2. Dónde está | 3. Qué se rompe, y quién lo nota | 4. Sitios | 5. ¿Revert? | 6. ¿Bloquea / desbloquea? |
|---|---|---|---|---|---|---|
| 11 | Un comentario no tiene tamaño máximo. | `backend/start/routes.ts:319` (solo comprueba el mínimo) y `backend/config/bodyparser.ts:69` (`limit: '20mb'`). | Nadie todavía. Cualquiera puede guardar cuerpos de hasta unos 20 MB sin autenticarse; los vería el moderador y, una vez aprobados, todo lector, porque `GET /posts/:slug` devuelve todos. | 1: una regla en el validador (`routes.ts:319` hoy). | **Sí.** | **Bloqueado por #2 y #10** (se escribe en el validador, con la forma de error ya decidida). |
| 12 | `DELETE` borra la fila y deja huérfanas sus respuestas. | `backend/start/routes.ts:359` (`delete from comentarios where id = ?`); `padre_id` sin clave foránea, migración `:20`. | Al borrar un comentario raíz sus respuestas dejan de pintarse (`Comentarios.jsx:64` solo cuelga de raíces) y siguen contando en el `<h2>`. Lo notan los lectores y el moderador. El borrado no tiene papelera. | **1** con cascada en el servicio (`:359`). **5** con borrado lógico: 1 migración + 1 `delete` + las 3 consultas de #9. La decisión entre ambas está sin tomar; no la tomo yo. | **No.** El commit se revierte pero las filas borradas no vuelven: el borrado es duro. | **Bloqueado por #2** (el servicio es el único que muta). Reducido por #3 y #6, que acortan quién puede borrar. El borrado lógico comparte consultas con #9. |
| 13 | `PATCH` no aplica la longitud mínima de `POST`. | `backend/start/routes.ts:384` (`!cuerpo`) contra `:319` (`length < 5`). | Nadie todavía (la web no edita): un autor puede dejar su comentario pendiente en «a». Lo vería el moderador. | 1 (`:384`) y el validador compartido. | **Sí.** | **Bloqueado por #2 y #10.** |
| 14 | **Fondo.** La moderación se decide con una clave compartida en una cabecera, no con una cuenta autenticada. | `backend/start/routes.ts:351` (`const esModerador = clave === SECRETO_SESION`); el starter kit de autenticación existe y no se usa aquí (`:399-418`). `User` no tiene campo de rol: `backend/app/models/user.ts`, migración de usuarios. | Cualquier persona con la clave actúa como moderador y no queda rastro de quién. Lo nota el equipo si alguna vez quiere saber quién borró algo. | 4: `routes.ts:351`, `middleware.auth()` en el grupo de rutas, 1 migración de rol en `users`, el modelo `User`. | **Sí.** | **Bloqueado por #2 y #6** (la contención va antes). La web no llama a `PATCH` ni a `DELETE`, así que cambiar este contrato **no rompe nada del frontend**. |
| 15 | `padre_id` se guarda sin comprobar que exista ni que sea del mismo post. | `backend/start/routes.ts:308` (leído sin validar) y `:326` (insertado tal cual). | Nadie todavía: la web no envía `padre_id` (`Comentarios.jsx:33`). Por API se puede guardar una respuesta a un comentario inexistente o de otro post. Además la web no pinta las respuestas a una respuesta (`:50` y `:64` solo cuelgan de raíces), pero sí las cuenta. | 2: la validación (`:308`) y el pintado anidado de `Comentarios.jsx:64`, si se quiere. | **Sí.** | **Bloqueado por #2 y #10.** |
| 16 | `PATCH` y `DELETE` no comparan el correo igual. | `backend/start/routes.ts:352-353` (minúsculas, `trim()`, admite moderador) contra `:379` (comparación exacta, sin moderador, y exige `pendiente`). | Nadie todavía: la web no consume ninguna de las dos. Quien guardó `Lucia@x.es` y escribe `lucia@x.es` puede borrar y no editar. El moderador puede borrar pero no editar ni un spam. | 2 (`:352-353` y `:378-379`). | **Sí.** | **Bloqueado por #2. Lo absorbe el #8**: si el #8 se hace, desaparece. |
| 17 | Un correo es válido si contiene `@`. | `backend/start/routes.ts:316` (`indexOf('@') === -1`). | Nadie todavía de forma visible: `a@` entra. El moderador acaba con correos inservibles. | 1 (`:316`). | **Sí.** | **Bloqueado por #2 y #10** (validador VineJS con `isEmail`). |

## Propuestas sin medir (fuera de la tabla)

Ninguna entra en la tabla porque nadie ha medido el problema. Al lado, lo que habría que medir.

| Propuesta | Por qué se plantea | Qué habría que medir |
|---|---|---|
| Endpoint propio `GET /posts/:slug/comentarios` | La web pide el post entero dos veces para sacar los comentarios: `DetallePost.jsx:34` y `Comentarios.jsx:19`. | Bytes y tiempo de `GET /posts/:slug` frente a solo los comentarios; cuántas veces se carga un post por visita; si el doble `fetch` pasa por caché del navegador. |
| Índice en `comentarios.post_id` | La migración no lo tiene (`:6` del comentario de cabecera, `:14` la columna) y `GET /posts` ejecuta una subconsulta correlacionada por post (`routes.ts:75-76`). | `EXPLAIN QUERY PLAN` y tiempo de `GET /posts` con una tabla de al menos 10 000 comentarios. Hoy hay 9 filas y no se ve. |
| Antispam o límite de tasa en `POST` | No hay limitador en `start/` ni en `package.json`, y la semilla contiene un spam pendiente (`promo@plantillas-baratas.example`). | Cuántos comentarios llegan al día y qué fracción es spam en producción. La semilla no es una medida. |
| Paginar los comentarios de un post | `GET /posts/:slug` devuelve todos los aprobados sin límite (`routes.ts:152-155`). | Distribución real de comentarios por post. Hoy el máximo es 4. |
| Interfaz para editar o borrar en la web | La API lo permite y la web no lo ofrece: ninguna llamada `PATCH` ni `DELETE` en `frontend/src`. | Cuántas peticiones de corrección o borrado llegan por otra vía. |

## Revisado y descartado

- Los comentarios se pintan como texto: `Comentarios.jsx:20-21` no usa `dangerouslySetInnerHTML`. El único uso (`DetallePost.jsx:92`) es del cuerpo del post, que es de otra feature.
- CSRF desactivado (`config/shield.ts`): la API no usa cookies de sesión para los comentarios, así que no es un hallazgo de esta feature.
- `key={index}` en las listas (`Comentarios.jsx:57`, `:66`): los comentarios no tienen estado local, así que no produce un fallo observable.
- La API no tiene ninguna ruta que apruebe un comentario. No lo he puesto como hallazgo porque no sé si es una carencia o si la aprobación se hace por fuera: ver Parte B, línea 3.

---

## Parte B: las tres líneas

1. **Hallazgos devueltos y comprobados.** Devolvió **17 filas**. **Cuántos has comprobado tú abriendo el archivo: 1**.
2. **El hallazgo del que no supiste decidir la prioridad: el #6.** Tiene el daño más alto (la clave pública abre `DELETE` a cualquiera), pero por la regla de daño/radio cae cuarto en el Nivel 1 porque toca 4 archivos; lo dejé ahí y lo pegué al #3 por una razón mía (cierran los dos caminos del mismo `DELETE`), no por la fórmula.
3. **Algo que el agente afirmó y no pudiste verificar.** No pude comprobar cómo se aprueban hoy los comentarios. La semilla habla de un «panel viejo de moderación» (`blog_seeder.ts:509`), no está en el repositorio y la API no tiene ruta de aprobación. Tampoco reproduje la condición de carrera del #5 ni sé si el repositorio es público (lo que decide el alcance real del #6).
