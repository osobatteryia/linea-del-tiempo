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
- **Pedidos al anfitrión:** los demás celulares escriben `st.req = {type, id}` (`play`, `check`, `reveal`, `buy`); el anfitrión los ejecuta en `hostCheck()` y limpia `st.req`. Si el anfitrión no responde en ~9 s, se muestra aviso con "Cancelar y volver a intentar".
- Playlist por defecto: `https://open.spotify.com/playlist/4BX1DjvzqAT3GePXKGzyTn` ("Línea del Tiempo Musical: Adivina el Año", privada, de Oso). Al crear la partida se pueden pegar **varias playlists** (una por línea, `playlistUris`); se mezclan sin duplicados (hasta 1000 canciones por playlist).
- **Lectura de playlists (cambio de Spotify, feb 2026):** se usa `GET /playlists/{id}/items` (respuesta `items[].item`); `/tracks` quedó eliminado y, al fallar, el juego caía al método antiguo (shuffle de Spotify) y las canciones se repetían. Spotify **solo entrega canciones de playlists propias o colaborativas**; si no puede leer ninguna, se avisa (`plWarn`).
- **Reproducción sin "fugas":** el método directo manda **solo** la canción elegida (`uris:[t.uri]`), no una lista; antes mandaba 60 canciones más que seguían sonando y luego parecían repetidas. Si el método de cola (`loadPausedAt`) hace sonar otra canción (cola de Spotify con restos), se marca `queueBad` y desde ahí se usa solo el directo. Si no se puede leer la playlist se muestra el motivo (`plWarn`).
- **Diagnóstico en el Menú del anfitrión:** "🎵 Playlist: N canciones · sonaron en esta partida · historial de este celular · modo directo".
- **Sin repetir:** `pickFresh` evita canciones ya sonadas en la partida (`st.played`, `st.used`), las que ya son carta de alguien, y las sonadas en partidas anteriores (historial `heardHist` en el anfitrión, `markHeard`). Si ya sonaron todas, elige entre el 25 % más antiguo.
- Los nombres de Spotify se muestran limpios (`cleanName`: quita "- 2019 Remaster", "(Live)", "- En Vivo", etc.).
- Lista interna `S` de 150 canciones con año (latinas/chilenas + internacionales), usada para corregir años y como respaldo del canje.

## Cómo suena cada turno (Spotify)
1. El jugador de turno toca **▶ Empezar mi turno** (antes no puede elegir posición).
2. El anfitrión elige una canción de la playlist no usada (`loadPlaylist`, `hostPlayTurn`): pausa la anterior, la **carga en pausa** (cola + siguiente + **pausa inmediata dos veces**, porque en iPhone "siguiente" da play solo), la mueve a un **punto al azar** y le da play (`loadPausedAt`). Si falla, usa `loadDirectAt` (play solo de esa canción + pausa inmediata + seek + play); último recurso el método antiguo. En iPhone puede quedar un "clic" muy corto del comienzo: Spotify iOS ignora `position_ms` y no deja bajar el volumen. Con parlante Spotify Connect/computador/Android no pasa.
3. Corre la cuenta regresiva (10 s por defecto) en todos los celulares. Al llegar a 0 se entrega lo que haya elegido el jugador.
4. Al terminar el tiempo, el volumen baja a la mitad (`duck`) y vuelve en el próximo turno. **En iPhone Spotify no permite cambiar el volumen** → se avisa al anfitrión ("Entendido" desactiva). **No se pausa** por defecto: en iPhone, Spotify en pausa y en segundo plano se desconecta ("No encuentro ningún Spotify abierto").
5. Al revelar, el anfitrión usa la canción que guardó al empezar (`clipTrack:<sala>`), no lo que diga Spotify.

## Mapa del código (`index.html`)
Para encontrar algo rápido sin leer todo el archivo: busca el nombre con `grep -n "nombre" index.html`. Los números de línea son aproximados (cambian con cada edición); confía en los nombres.

| Tema | Dónde buscar |
|---|---|
| Lista interna de 150 canciones | `const S=[` (~línea 130) |
| Configuración (Client ID, playlist, brokers MQTT, `NS`) | sección `// ===== Configuración =====` |
| Modo de pruebas (`perfil`, `soy`, `nombres`, `STORE`, `selfUrl`) | justo debajo de Configuración |
| Comparar canción/artista escritos (tolerante a errores) | `norm`, `findInDB`, `lev`, `close`, `guessTitle`, `guessArtist` |
| Micrófono / dictado | `// ===== Dictado por voz =====`: `micBtn`, `startMic`, `stopMic` |
| Estado (`st`, `setup`, `me`, `isHost`, `room`) | `// ===== Estado =====` |
| Conexión entre celulares (MQTT) | `connectRoom`, `commit`, `flush`, `saveLocal` |
| Login Spotify | `spotifyLogin`, `spotifyCallback`, `accessToken`; llamadas a la API: `sp()` |
| Elegir y poner la canción del turno | `loadPlaylist`, `pickFresh`, `markHeard`, `hostPlayTurn`, `loadPausedAt`, `pickDevice`; método antiguo: `hostStart`, `hostNext`, `hostClip` |
| Fin del fragmento (bajar volumen / pausar) | `afterClip`, `duck`, `unduck`, `pauseClip` |
| Revisar respuesta (✓/✗ del año, sin mostrar la canción) | `doCheck`, `checkOwner`, `CHECK_FALLBACK`; con Spotify: `request("check")` → `resolveSong` en el anfitrión |
| Revelar con Spotify | `hostReveal` = `resolveSong` + `reveal()` (usa `clipTrack:<sala>` guardado) |
| Pedidos de otros celulares al anfitrión | `request()` → `hostCheck()` (tipos `play`, `check`, `reveal`, `buy`) |
| Canje de 3 fichas | `grantCard`, `buyFromList`, `hostBuy` |
| Reglas del turno | `// ===== Reglas =====`: `startGame`, `nextTurn`, `reveal`, `undoReveal` |
| Cartas y línea de tiempo (HTML) | `slotLabel`, `cardHTML`, `tlHTML`, `missHTML` (dónde iba la carta), `color()` |
| Botón Revelar (solo jugador de turno, sola a los 30 s) | `STEAL_WINDOW`, `revealWait`, `revealOwner`, `revealReadyAt`, `autoRevealLeft`, `doReveal`, `canReveal`, `revealBtnHTML` |
| Botón Siguiente turno (solo jugador de turno) | `NEXT_FALLBACK`, `nextOwner`, `nextBtnHTML` |
| Votación de ficha | `voteInfo`, `voteHTML`, `applyVote`, `myVote` |
| Ver la línea de otro jugador | `viewerHTML`, `viewTl`, `case "viewTl"`; en la pantalla final: `winView` |
| Marcador, barra de sala, avisos, panel de Spotify, QR | `scoreHTML`, `statusHTML`, `msgHTML`, `hostPanel`, `linkHTML`, `drawQR` |
| Pantallas | `render()`: portada (`if (!room)`), crear partida (`if (!st)`), menú (`menuOpen`), fases `turn`, `check`, `steal`, `needyear`, `reveal`, `win`. Línea propia de cada jugador: `myTl` |
| Cuenta regresiva y entrega automática al acabarse el tiempo | `timeLeft` + `setInterval` en `// ===== Cuenta regresiva =====` |
| Botones (todas las acciones) | `document.addEventListener("click"` → `switch (act)` con `case "play"`, `"slot"`, `"confirm"`, `"skip"`, `"buy"`, `"pickStealer"`, `"reveal"`, `"vote"`, `"next"`, etc. |
| Opciones al crear partida | `document.addEventListener("change"` (`clip`, `after`, `part`, `tokens`, `target`, `src`) |
| Arranque de la página | `// ===== Inicio =====` |

Fases de `st.phase`: `turn` → `check` (botón Revisar respuesta) → `steal` (✓/✗ a la vista; si falló, 5 s para robar) → `reveal` → `turn` del siguiente… → `win`. Si falta el año sale `needyear` (en la revisión con `st.fromCheck`, o al revelar).

## Reglas implementadas
- Cada jugador parte con 1 carta a la vista y N fichas (2 por defecto, máx. 5). Gana quien llega a la meta (10 por defecto).
- Turno: elegir posición (antes / entre / después); mismo año vale a cualquier lado.
- Casillas opcionales **canción** y **artista** (con micrófono para dictar, español `es-CL`). Si **acertó el año** y acierta las dos (comparación tolerante a tildes, mayúsculas, signos, artículos iniciales, "(Remaster)"/"- …" y ~1 error cada 4 letras; en la **canción** un pedazo del nombre vale solo si es al menos la mitad, ej. "corazon" ya no vale por "Corazón Espinado"; en el **artista** basta una palabra o uno de los artistas) → +1 ficha automática al revelar. Si falló el año, no hay ficha.
- **Votación de ficha:** solo si **acertó el año** (carta bien ubicada) **y** completó canción y artista, pero no acertó las dos, **y las dos respuestas se parecen** (`near`/`nearArtist`: ≥50 % de parecido letra por letra o una palabra de 4+ letras en común). Si alguna es muy distinta → no hay ficha ni votación (`farOff`), **salvo** que el jugador de turno active el interruptor "Pedir votación al grupo" (`st.voteReq`, `case "voteReq"`; apagado por defecto, no se puede apagar; solo si escribió las dos casillas); los **demás** votan con llave de sol 𝄞 (dar ficha) / llave de fa 𝄢 (no), clase CSS `.clef`; se entrega con mayoría de los demás (empate o no votar = no). **El voto no se puede cambiar**; apenas los "no" hacen imposible la mayoría se muestra "El grupo decidió no darle la ficha" (`lost` en `voteInfo`). El jugador de turno no puede darse la ficha. En celular compartido, un solo botón "El grupo decidió darle la ficha". No hay votación si le robaron la carta.
- **Saltar canción:** 1 ficha.
- **Cada celular ve solo su propia línea** (el jugador de turno ve la suya como "Tu elección"). Las líneas de los demás se ven con los botones de "Ver la línea de otro jugador" (`viewerHTML`, `viewTl`, solo en ese celular). Al revelar, todos ven el resultado (dónde iba / línea del que se quedó la carta).
- **Nadie ve dónde puso la carta el jugador de turno** hasta que se revela (solo él, o el celular compartido).
- **Revisar respuesta:** al terminar el tiempo (o confirmar), el jugador de turno toca "Revisar respuesta" (respaldo: a los 30 s aparece para todos). Todos ven solo **✓ acertó el año** o **✗ falló**; no se ve canción, año ni artista.
- **Robar:** solo si falló. Hay **5 s**; el primero que toca Robar (1 ficha, **un solo robo por turno**) ubica la carta **en su propia línea**. La ficha se paga al tocar Robar y **no se puede cancelar**; tiene **10 s** (`ROB_TIME`, `robLeft`) para ubicarla. Si acierta, la carta es suya; si falla o se le acaba el tiempo, se descarta.
- **Revelar:** el botón lo tiene **solo el jugador de turno** (los demás ven "se revela sola en N s"). Se puede si acertó, de inmediato; si falló, cuando el que roba ya ubicó la carta o pasaron los 5 s sin robo. Si no lo toca en **30 s**, se revela sola (`autoRevealLeft`, `doReveal`; lo hace su celular, y si no está, el anfitrión 5 s después).
- **Siguiente turno:** solo el jugador que terminó su turno (respaldo: 60 s para todos).
- **Canjear 3 fichas por una carta:** en su turno, antes de empezar; la carta entra directo a su línea; sigue jugando el turno.
- Al fallar se muestra **dónde iba la carta** en su línea (✗ Aquí la puso / ✓ Aquí iba).
- Ya no se muestra el aviso "Año según el álbum en Spotify… Corregir año" al revelar (Oso pidió quitarlo). El código de `fixYear` sigue, pero sin botón.
- Orden de turnos: el de la lista de nombres al crear la partida.
- Opciones al crear: tiempo para responder (10/15/20/30/sin límite), parte de la canción (al azar / desde el comienzo), al terminar el tiempo (bajar volumen / seguir / pausar), cartas para ganar, fichas iniciales, playlist.

## Diseño
Estilo "neón de club" (vino de la rama `estilo-ui-ux-pro-max` de otro chat): fondo `#0b0b10`, acentos rosa `#ff3fa4`, lima `#c6ff3d`, cian `#3de8ff`; fuentes Unbounded / Space Grotesk / Space Mono; cartas con brillo de color por década (`color(y)`, variable CSS `--c`).
- **Paleta nueva (en prueba, activa por defecto):** solo rosa `#ff3fa4`, morado `#e252ff`, celeste `#52d4ff`, turquesa `#52ffd4`, menta `#3dffb0`, lima `#c6ff3d` y verde lima `#9aff52`. Clase `html.p2` (`applyPalette`, `palette()`, guardada en `localStorage["ldt-paleta"]`, compartida entre los 4 celulares de prueba.html). Décadas `DEC_NEW`: 50s y 60s lima, 70s menta, 80s rosa, 90s morado, 2000s celeste, 2010s turquesa, 2020s verde lima. Error (`--bad`) pasa a rosa. En el Menú: botón "Colores: nuevos/originales" para comparar.

## Limitaciones conocidas
- iPhone: Spotify ignora el punto de inicio al cargar canción nueva (se mitigó con carga en pausa; Oso aún debe confirmar si desapareció el "comienzo"). No permite volumen remoto. En pausa + segundo plano se desconecta.
- El anfitrión debe tener la página del juego abierta y la pantalla encendida; si sale de Safari, los pedidos esperan hasta que vuelva.
- Probar con 2 pestañas en un iPhone no sirve (Safari congela la de fondo): usar `prueba.html`.
- MQTT público: sin seguridad; cualquiera con el código de sala puede entrar.

## Cómo probar cambios (en el contenedor)
- Subir `VERSION` en `index.html` en cada cambio publicado (se ve abajo en la pantalla; sirve para saber si Safari tiene la última).
- Servidor local con broker MQTT propio (aedes + ws) y Playwright/Chromium; reemplazar en una copia de `index.html` la URL de mqtt y `BROKERS` por `ws://localhost:8888`, y simular Spotify con `context.route("https://api.spotify.com/**", ...)`.
- En Playwright sync, usar `page.wait_for_timeout()` (no `time.sleep`) para que respondan las rutas simuladas.
- Siempre `node --check` del script antes de publicar.

## Pendientes / ideas abiertas
- Confirmar con Oso si la carga en pausa eliminó el "comienzo" de la canción en iPhone.
- Propuesto, sin respuesta: bloquear "Siguiente turno" mientras haya votación de ficha abierta.

## Registro de cambios

### Pendientes de consolidar (3/5)
- Inicio al azar sin "comienzo": pausa inmediata tras cargar la canción (cola o directo) y luego seek + play; `queueBad` solo si suena una canción distinta a la anterior 3 veces.
- Pantalla final: cada jugador toca un nombre del ranking para ver su línea o la de otros (`winView`, `case "winView"`, solo en ese celular; por defecto la ganadora).
- Aleatorio: solo se reproduce la canción elegida (sin lista extra), detección de cola de Spotify con restos (`queueBad`), aviso si la playlist no se puede leer, diagnóstico en el Menú del anfitrión.

### Historial
- Juego base, sala por QR con MQTT, Spotify del anfitrión, banco de pruebas de 4 celulares.
- Botón Empezar mi turno, fragmento al azar con cuenta regresiva, micrófono, adivinar canción/artista.
- Arreglos de Spotify en iPhone (502, dispositivo dormido, revelar en pausa, carga en pausa, volumen a la mitad).
- Un robo por turno, canje de 3 fichas, dónde iba la carta, diseño neón.
- Votación de ficha, Revelar solo jugador de turno con 5 s, Siguiente turno solo jugador de turno.
- Marcador con fichas visibles, línea propia de cada jugador, paso Revisar respuesta (✓/✗) con robo en la línea propia y posición oculta, versión visible (`VERSION`) y sin caché en prueba.html, Revelar solo del jugador de turno y automático a los 30 s.
- Votación y ficha automática solo si acertó el año, robo sin cancelar con 10 s para ubicar, canciones sin repetir (historial entre partidas) y varias playlists, cada celular ve solo su línea con botones para ver las de los demás.
- Quitado aviso "Año según el álbum…", aviso grande de carta ganada al revisar, canción exige al menos la mitad del nombre, votación con voto fijo y resultado cuando gana el no, votación con llave de sol / llave de fa.
- Sin votación si la respuesta no se parece, playlists con `/items` (arreglo de repeticiones), interruptor "Pedir votación al grupo" + nombres limpios, interruptor alineado, paleta nueva de prueba con botón en el Menú.
