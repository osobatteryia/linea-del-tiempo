# Línea del Tiempo Musical — contexto del proyecto

Archivo de contexto para retomar el trabajo en un chat nuevo. Léelo completo antes de cambiar algo.

## Regla de mantenimiento de este archivo
- **Actualiza este archivo cada 5 cambios** publicados en la app (commits que cambian `index.html` o `prueba.html`).
- Lleva la cuenta en la sección **Registro de cambios** (al final): agrega cada cambio a la lista "Pendientes de consolidar". Cuando haya 5, actualiza las secciones de arriba que correspondan, pasa esos 5 al historial y deja la lista vacía.
- Si un cambio contradice algo escrito aquí, corrige el texto en ese mismo momento, sin esperar a los 5.

## Qué es
Juego musical estilo **Hitster** para jugar en grupo, cada uno desde su celular. El dueño es **Oso** (GitHub `osobatteryia`), en Chile; habla español y prueba todo desde un **iPhone** con Safari. Explicar en español simple, paso a paso, sin jerga.

- Juego: https://osobatteryia.github.io/linea-del-tiempo/
- Página de pruebas (4 celulares en una pestaña): https://osobatteryia.github.io/linea-del-tiempo/prueba.html
- Repo: https://github.com/osobatteryia/linea-del-tiempo (público, GitHub Pages desde `main` / raíz)

## Archivos
- `index.html` — todo el juego en un solo archivo (HTML + CSS + JS, sin build). ~1200 líneas.
- `prueba.html` — banco de pruebas: carga `index.html` 4 veces en iframes (Oso anfitrión, Ana, Pipe, Coni), cada uno con almacenamiento propio (`?perfil=N&soy=N`). Detecta la sala que crea Oso y conecta a los otros 3 solos. Vista por pestañas o "Ver los 4".
- `README.md`, `CLAUDE.md` (este archivo).
- **No subir** carpetas de herramientas (`.claude/skills`, `__pycache__`). Oso lo pidió explícitamente. Si otro chat trabaja en una rama con herramientas, traer solo `index.html` (`git checkout <rama> -- index.html`).

## Arquitectura
- **Sincronización:** MQTT público por WebSocket (`wss://broker.emqx.io:8084/mqtt`, respaldo `broker.hivemq.com`). Tópico `lineadeltiempo-oso-v1/salas/<SALA>/estado`, mensaje **retenido** con todo el estado `st` + `seq` (marca de tiempo). Cada celular publica el estado completo al cambiar algo (`commit()`); se ignoran estados con `seq` menor. Último en escribir gana. Si no hay conexión, el cambio queda pendiente (`dirty`) y se envía al reconectar. El estado también se guarda en `localStorage` (`st:<sala>`).
- **Sin cuentas:** los amigos entran con QR o `?sala=CODIGO`. El que crea la sala es el **anfitrión** (`host:<sala>` en localStorage).
- **Spotify solo en el anfitrión:** OAuth PKCE en el navegador. Client ID `f5875de832bf474398389d12b283967d` (app "Línea del Tiempo" en developer.spotify.com, cuenta de Oso, Premium). Redirect URI registrada: `https://osobatteryia.github.io/linea-del-tiempo/` (exacta, sin `index.html`; por eso existe `SP_REDIRECT`). Scopes incluyen `playlist-read-private` (versión de token `SCOPE_V = 2`; si el token es viejo se pide "Reconectar Spotify"). Al conectar desde `prueba.html` vuelve a esa página (`returnTo`).
- **Pedidos al anfitrión:** los demás celulares escriben `st.req = {type, id}` (`play`, `reveal`, `buy`); el anfitrión los ejecuta en `hostCheck()` y limpia `st.req`. Si el anfitrión no responde en ~9 s, se muestra aviso con "Cancelar y volver a intentar".
- Playlist por defecto: `https://open.spotify.com/playlist/4BX1DjvzqAT3GePXKGzyTn` ("Línea del Tiempo Musical: Adivina el Año", privada, de Oso). Se puede cambiar al crear la partida.
- Lista interna `S` de 150 canciones con año (latinas/chilenas + internacionales), usada para corregir años y como respaldo del canje.

## Cómo suena cada turno (Spotify)
1. El jugador de turno toca **▶ Empezar mi turno** (antes no puede elegir posición).
2. El anfitrión elige una canción de la playlist no usada (`loadPlaylist`, `hostPlayTurn`): pausa la anterior, la **carga en pausa** (cola + siguiente), la mueve a un **punto al azar** y le da play (`loadPausedAt`). Esto es para que en iPhone no suene el comienzo; si falla, usa `play` con `uris` + `position_ms`, y como último recurso el método antiguo (siguiente + seek).
3. Corre la cuenta regresiva (10 s por defecto) en todos los celulares. Al llegar a 0 se entrega lo que haya elegido el jugador.
4. Al terminar el tiempo, el volumen baja a la mitad (`duck`) y vuelve en el próximo turno. **En iPhone Spotify no permite cambiar el volumen** → se avisa al anfitrión ("Entendido" desactiva). **No se pausa** por defecto: en iPhone, Spotify en pausa y en segundo plano se desconecta ("No encuentro ningún Spotify abierto").
5. Al revelar, el anfitrión usa la canción que guardó al empezar (`clipTrack:<sala>`), no lo que diga Spotify.

## Reglas implementadas
- Cada jugador parte con 1 carta a la vista y N fichas (2 por defecto, máx. 5). Gana quien llega a la meta (10 por defecto).
- Turno: elegir posición (antes / entre / después); mismo año vale a cualquier lado.
- Casillas opcionales **canción** y **artista** (con micrófono para dictar, español `es-CL`). Si acierta las dos (comparación tolerante a tildes y errores chicos) → +1 ficha automática.
- **Votación de ficha:** si completó canción y artista pero no acertó las dos, los **demás** votan 👍/👎; se entrega con mayoría de los demás (empate o no votar = no). El jugador de turno no puede darse la ficha. En celular compartido, un solo botón "El grupo decidió darle la ficha". No hay votación si le robaron la carta.
- **Saltar canción:** 1 ficha.
- **Robar:** 1 ficha, **un solo robo por turno**; el que roba elige otra posición en la línea del jugador. Si el jugador falla y el que roba acierta, la carta es del que roba. La ficha se gasta siempre.
- **Revelar:** solo el jugador de turno, tras **5 s** de espera para que los demás roben (respaldo: a los 30 s aparece para todos). Si alguien roba, se revela solo al poner su posición.
- **Siguiente turno:** solo el jugador que terminó su turno (respaldo: 60 s para todos).
- **Canjear 3 fichas por una carta:** en su turno, antes de empezar; la carta entra directo a su línea; sigue jugando el turno.
- Al fallar se muestra **dónde iba la carta** en su línea (✗ Aquí la puso / ✓ Aquí iba).
- "Corregir año" si el año viene de un álbum de Spotify (reediciones).
- Orden de turnos: el de la lista de nombres al crear la partida.
- Opciones al crear: tiempo para responder (10/15/20/30/sin límite), parte de la canción (al azar / desde el comienzo), al terminar el tiempo (bajar volumen / seguir / pausar), cartas para ganar, fichas iniciales, playlist.

## Diseño
Estilo "neón de club" (vino de la rama `estilo-ui-ux-pro-max` de otro chat): fondo `#0b0b10`, acentos rosa `#ff3fa4`, lima `#c6ff3d`, cian `#3de8ff`; fuentes Unbounded / Space Grotesk / Space Mono; cartas con brillo de color por década (`color(y)`, variable CSS `--c`).

## Limitaciones conocidas
- iPhone: Spotify ignora el punto de inicio al cargar canción nueva (se mitigó con carga en pausa; Oso aún debe confirmar si desapareció el "comienzo"). No permite volumen remoto. En pausa + segundo plano se desconecta.
- El anfitrión debe tener la página del juego abierta y la pantalla encendida; si sale de Safari, los pedidos esperan hasta que vuelva.
- Probar con 2 pestañas en un iPhone no sirve (Safari congela la de fondo): usar `prueba.html`.
- MQTT público: sin seguridad; cualquiera con el código de sala puede entrar.

## Cómo probar cambios (en el contenedor)
- Servidor local con broker MQTT propio (aedes + ws) y Playwright/Chromium; reemplazar en una copia de `index.html` la URL de mqtt y `BROKERS` por `ws://localhost:8888`, y simular Spotify con `context.route("https://api.spotify.com/**", ...)`.
- En Playwright sync, usar `page.wait_for_timeout()` (no `time.sleep`) para que respondan las rutas simuladas.
- Siempre `node --check` del script antes de publicar.

## Pendientes / ideas abiertas
- Confirmar con Oso si la carga en pausa eliminó el "comienzo" de la canción en iPhone.
- Propuesto, sin respuesta: bloquear "Siguiente turno" mientras haya votación de ficha abierta.
- Propuesto, sin respuesta: si un robo exitoso debe anular también la ficha automática (hoy no la anula).
- Propuesto, sin respuesta: cerrar el robo a los 5 s (hoy se puede robar hasta que se revela).

## Registro de cambios

### Pendientes de consolidar (2/5)
- Marcador: las fichas de todos los jugadores se ven (las etiquetas pasan a una segunda fila en vez de quedar ocultas a la derecha).
- Cada jugador ve **su propia línea de tiempo** abajo cuando no es su turno (antes todos veían solo la del jugador de turno y parecía que la primera carta era igual para todos; las cartas iniciales siempre fueron distintas).

### Historial
- Juego base, sala por QR con MQTT, Spotify del anfitrión, banco de pruebas de 4 celulares.
- Botón Empezar mi turno, fragmento al azar con cuenta regresiva, micrófono, adivinar canción/artista.
- Arreglos de Spotify en iPhone (502, dispositivo dormido, revelar en pausa, carga en pausa, volumen a la mitad).
- Un robo por turno, canje de 3 fichas, dónde iba la carta, diseño neón.
- Votación de ficha, Revelar solo jugador de turno con 5 s, Siguiente turno solo jugador de turno.
