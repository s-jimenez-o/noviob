# NOTAS DE DISEÑO — 6 MESES

Reglas del proyecto para quien lo retome (humano o Claude). El juego completo vive
en `index.html`: un solo archivo, cero dependencias, canvas de 160×144. Léelas
antes de tocar el código; el juego se rompe fácil si se ignoran.

## El tono

- Todo el texto en **MAYÚSCULAS**, seco y con humor. Nada cursi ni explicativo.
  El chiste se muestra, no se cuenta ("(QUÉ FEA CANGURERA.)" y no una charla de tres cajas).
- Frases cortas. Si una línea necesita más de dos renglones, casi siempre sobra algo.
- Los finales malos son absurdos y rápidos (piano, avión, pizza). No se explican.
- Santiago y Roberto nunca se declaran nada solemne: el romance sale por los hechos.

## La consola (CSS)

- Carcasa **teal GBC** con botones A/B frambuesa; el fondo de la página es azul
  petróleo. Si se cambia el color, cambiar juntos: gradiente de `.gb`, fondo de
  `body`, color del logo `.gbtxt` y el radial de los botones `.ab`.
- Si sobra alto (teléfonos largos), la carcasa se **estira** hasta llenarlo: `fit()`
  le pone un `height` y los `div.aire` (flex) reparten el sobrante arriba, en medio
  y abajo. Así no queda hueco negro y los controles bajan a donde están los pulgares.
  El tope es 1.3× el alto natural para que no se deforme.
- Prioridad de espacio: **la pantalla manda**. El bisel y los márgenes se
  mantienen delgados (bisel 5-6u, carcasa 5u) para que en un teléfono la pantalla
  sea lo más grande posible; los botones grandes y separados (dpad 62u, A/B 28u)
  porque se juega con el pulgar.

## La pantalla y la escala

- Canvas de **160×`ALTO`**, con `ALTO = 168`. La Game Boy real son 144, pero se le
  dio aire abajo para aprovechar la pantalla del teléfono. **Nunca escribas 144 ni
  168 a mano: usa `ALTO`.** Los píxeles siguen cuadrados (mismo `--u` en los dos ejes);
  si alguna vez cambia `ALTO`, hay que cambiar también el `height` del `<canvas>` y
  el `height:calc(...*var(--u))` del CSS, o la imagen se deforma.
- Un fondo que termine justo abajo se escribe `vgrad(0, y, 160, ALTO - y, ...)`,
  nunca con el alto calculado a mano: si no, queda una franja negra.
- `GROUND = 92` es la línea de piso: los personajes (16×30) se paran en `y = GROUND - 30 = 62`.
  El piso llega hasta `ALTO`, así que abajo hay bastante suelo — ahí cae la caja de
  diálogo (`BOX_Y = 120`) y los avisos.
- La carcasa se escala con la variable CSS `--s` (la fija `fit()`); TODA la consola
  está medida en `var(--u)`, así que crece y encoge junta. No usar px sueltos ahí.

## Dibujo: cómo se logra la profundidad

- Orden por escena: **cielo/pared (vgrad) → cosas del fondo → piso (vgrad) → líneas
  del piso → personajes → FRENTES** (lo que va delante de la gente).
- Los degradados siempre con `vgrad(...)` (bandas + costura tramada con `dith`);
  nunca gradientes suaves del canvas: rompen el look 8-bit.
- El piso siempre lleva sus líneas horizontales `rgba` para dar perspectiva.
- Interiores de la misma casa comparten el mismo piso de madera (`#8a6038` →
  `#5c3d22`) para que se sienta continuidad (cuarto, cocina, sala).
- Sombras: `sombraPiso(cx, y+30)` bajo cada personaje parado. `persona()` ya la pone.

## Nombres de funciones: cuidado con pisarlas

El archivo es enorme y ya hubo un choque: una función `burbuja` nueva pisó la de
los mensajes de chat (las declaraciones `function` se hoistean, gana la última).
**Antes de nombrar una función nueva, `grep -n "function nombre"`.** Los `const`
son peor: usar uno antes de su línea de declaración mata el juego entero al cargar
(pasó con los sprites de las amigas), y las sondas normales no lo detectan — solo
la página con `window.onerror`.

## Personajes

- Sprites indexados: arrays de strings de 16 de ancho, con `armar(cabeza, torso, piernas)`
  (14 + 7 + 9 = 30 filas). Claves de color: `k` contorno, `1`/`2` pelo, `s`/`S` piel,
  `e` ojo, `b` barba, `m` boca, `c`/`C` ropa, `t` detalle, `p`/`Q` pantalón, `o` zapatos,
  `g` arete, `l` labios.
- **Consistencia ante todo**: Roberto = pelo oscuro, ropa azul marino; Santiago = arete
  dorado, camisa clara, pantalón verde. Cualquier personaje nuevo se hace recolorando
  cabezas/torsos existentes (así se hicieron las amigas, Leche y los perfiles de la app).
- Props del jugador: la cangurera y la maleta se pintan DENTRO de `persona()` según
  los flags globales `cangurera` / `maleta`. Si una escena dibuja sprites a mano
  (con `drawSprite` directo), hay que pintar el prop a mano también — ya nos pasó
  que la cangurera desaparecía en los bailes por esto.
- Personajes "de voz" sin cuerpo (amigas, losDos) existen solo para el color de la
  caja de diálogo.

## Tipografía y texto

- `FONT` es un bitmap 5×7 (14 hex por glifo). Antes de escribir un texto nuevo,
  verifica que todos sus caracteres existan; si falta uno, se dibuja un espacio.
  Para agregar un glifo: 7 filas de 2 hex, bit 4 = columna izquierda.
- Acentos permitidos: Á É Í Ó Ú Ñ ¿ ¡ (las mayúsculas acentuadas llevan la tilde
  en la fila 0 y la letra empieza en la fila 2 — no comprimir la letra).
- Límites duros: **23 caracteres por renglón** en la caja de diálogo (3 renglones,
  se pagina solo), **~16 en chats**, **~21 en opciones de menú**, y los avisos de
  `avisoArriba` no deben pasar de ~150px (`textW(txt)+8`).
- La letra gorda (`drawTextG`, escala 3) solo para las tarjetas tipo Bob Esponja
  (`tarjetaTiki`): máximo ~8 letras por renglón.

## Componentes de UI (usar siempre estos, no inventar variantes)

| Cosa | Función | Cuándo |
|---|---|---|
| Caja de diálogo | `abrirCharla` / pasos con `q`+`t` | Alguien habla. La barra toma el color del personaje y el retrato del hablante se asoma arriba a la izquierda (solo si tiene sprite). |
| Menú de decisión | `abrirMenu(pregunta, ops, quien, fondo, fn)` | Decisiones. Sin nombre arriba, pregunta centrada, corazón como cursor. |
| Cartel de narrador | `cartelEscena([lineas], y)` | Texto sin voz ("SÁBADO.", "* NO SE DEJAN..."). Nunca caja de diálogo para esto. |
| Aviso de juego | `avisoArriba(txt, y)` | Instrucciones jugables ("PÁSAME EL QUESO PORFA"). Abajo (y≈102/118) si arriba tapa la escena. |
| Botón A | `botonA()` | SIEMPRE que una pantalla espera input y no hay otra pista. Parpadea solo. |
| Tarjeta gorda | `tarjetaTiki(t, [lineas])` | Cortes de tiempo estilo Bob Esponja. Siempre con `espera:true`. |
| HUD | `hud()` | Solo en los meses jugables 2+ (`mes > 0`). |

## El motor de guiones

- Un guion = lista de pasos `{ e:'cuadro', a:{...}, q:'quien', t:'línea', dur:N,
  espera:true, hacer:fn }`.
  - `dur` → avanza solo tras N cuadros (los beats de acción duran 40–90; el viaje 106–130).
  - `espera:true` → se queda hasta que piquen A (usa `botonA()` en el cuadro).
  - `t` → caja de diálogo, avanza con A.
  - `hacer` → efectos colaterales al entrar al paso (flags como `cuartoNoche`).
- `correrGuion(lista, fin, sinFade)` — **la regla de oro de las transiciones**:
  el barrido SOLO se usa cuando cambia el lugar físico. Si la escena siguiente
  comparte fondo, pasa `sinFade=true`. El usuario pidió explícitamente quitar
  barridos innecesarios; no los regreses.
- El fade tarda 20 cuadros por lado (8px/cuadro). No hacerlo más lento.

## Audio

- Todo el sonido pasa por `tone()` y está **condicionado a `musicOn`** (blip/chime/buzz
  ya lo checan). Nunca sonar si el usuario no prendió el sonido.
- Voces: cada personaje tiene su frecuencia de bip en `VOCES` (Roberto grave 560,
  Santi agudo 920). Personaje nuevo → agrégale voz.
- `chime()` = recoger/confirmar algo bueno; `blip(660)` = mover cursor; `buzz()` = error
  ("MENTIROSO"). Mantén ese vocabulario.
- La música es un loop de 32 pasos; corre a 148ms durante los bailes, 185ms el resto.
- En la despedida la música se apaga sola (y el botón se pone en OFF) antes del
  apagón: el silencio es parte del cierre. No lo quites.

## El cierre

No hay pantalla de créditos: la última línea de la historia pasa directo a la
despedida. El juego abre con la pantalla verde preguntando **WHAT IS LOVE?** y cierra con la
misma pantalla respondiéndola. `DESPEDIDA` es una lista de pasos que alternan
pantallas verdes (`{t:[líneas]}`) con **momentos** reales de ellos dos
(`{e:funcion, txt, veloz}`). **Corre sola**: cada paso tiene su `dur` y se funde a
negro al entrar y al salir. Aquí no se pica nada — es cine, no diálogo; si agregas
un paso, dale su `dur` (~130 para una frase, ~150-190 para un momento con animación). Los momentos son escenas de una sola
función (`momBarragan`, `momTectonic`, `momYale`, `momCeramica`, `momConcierto`,
`momAlberca`, `momCasa`) que pisan en `MOM_PIE = 116` en vez de `GROUND`, porque
ahí no hay caja de diálogo y conviene usar toda la pantalla. Para agregar un
momento: escribe su `mom*`, mételo en `DESPEDIDA` donde toque, y ya.
De ahí el apagón clásico de Game Boy: la imagen se colapsa a una raya, la raya a un
punto, negro. Es el remate del regalo — si se toca algo, que sea sin romper esa simetría.

## El menú de escenas

Va **compacto a propósito** (26 entradas, una por capítulo). Cada vez que se agrega
una escena, la tentación es meterle una entrada más: no. Sólo capítulos que alguien
querría revisitar, y con nombres de capítulo ("EL SHOW DE DRAGS", no "llegaRoberto").

## Controles

- `keys[k]` = está presionado (para caminar y para acelerar texto manteniendo A);
  `consumed(k)` = flanco de un toque (para confirmar). No mezclarlos.
- **Toda pantalla interactiva que entra desde un guion necesita su contador de
  gracia** (`medT`, `numT`, y ~20 cuadros antes de aceptar A). Si no, el mismo
  toque con el que pasaste el último diálogo se cuenta como respuesta y sale un
  error antes de que el jugador vea nada.
- **Nunca dejes al jugador atorado.** Si algo se puede fallar, a la segunda o
  tercera se explica y se avanza (el medidor pasa solo, el teclado suelta pista).
- El canvas escucha `pointerdown`/`move`/`up` y traduce a coordenadas de 160×144
  (`aLCD` + `toqueEnPantalla`): con eso el teclado del 911 se pica con el dedo y la
  lista de escenas se arrastra. Es el patrón a seguir si otra pantalla necesita toque.
  Ojo con distinguir arrastre de toque: si el dedo se movió menos de 4 px cuenta
  como toque, si no, sólo se scrollea (si no, cualquier arrastre dispararía un salto).
- Mantener A/B acelera el texto ×3; un toque lo completa; otro toque pasa de página.

## El selector de escenas (herramienta de desarrollo)

- `SALTOS` salta a cualquier punto; cada entrada debe dejar los flags como estarían
  llegando ahí jugando (cangurera, maleta, cuartoNoche, mesaElegida...).
- **La entrada 0 es `SEGUIR JUGANDO` con `f:null`**: cierra el menú y devuelve al
  jugador exactamente donde estaba. El cursor SIEMPRE arranca ahí, para que picar
  SELECT sin querer y luego A por reflejo no mande a nadie al arranque. Si agregas
  entradas, que sea después de esa. Tampoco se abre el menú durante un barrido
  (se perdería el callback pendiente).
- **Si insertas pasos en un guion, revisa los `irGuion(GUION, índice, ...)` de SALTOS**:
  los índices son posiciones absolutas y se corren. Imprime el guion numerado para verificar.
- Para el regalo final: borrar el botón `.escenas` del HTML y el bloque de "saltos".

## Verificación (no negociable)

Nada se da por bueno sin renderearlo. Herramientas ya hechas (en `/tmp`, se
regeneran fácil — el patrón está en el historial de la conversación):

1. **Sonda de dibujo**: llama cada `CUADROS`/`ESCENAS`/`FRENTES` con muchos `t` y
   args dentro de try/catch → lista de los que truenan. Ojo: si la página muere al
   cargar, la sonda calla — usar también la página con `window.onerror` que reporta
   la línea (`err.html`); un `const` usado antes de su línea de inicialización mata
   TODO el juego.
2. **Auditoría de texto**: recorre todos los guiones/menús/chats y verifica glifos
   existentes y anchos dentro de la caja.
3. **Bots de partida**: manejan `loop()` directo (headless Chrome no dispara rAF con
   `--dump-dom`; se anula `requestAnimationFrame` y se llama `loop()` en un
   `setInterval(...,0)`). Uno del boot al antro, otro de la casa a los créditos, y
   uno por cada final malo. La ruta de estados impresa debe llegar a `creditos`.
4. **Sonda de huecos**: dibuja cada escena y lee el canvas para ver si quedan
   píxeles sin pintar en las últimas filas. Es la única forma sensata de revisar
   50+ escenas tras un cambio de tamaño; encontró 17 fondos que terminaban en 144.
5. **Capturas**: `grab.sh nombre "código de dibujo"` congela un cuadro y lo exporta
   a PNG (dibuja el frame a mano, escala ×4 con `imageSmoothingEnabled=false` y
   saca un dataURL). SIEMPRE mirar la captura: los bugs de encimado (texto sobre
   caras, props tapados) solo se ven así.
- macOS: no existe `timeout`; los playthroughs largos se parten en mitades para no
  pasar el límite de 2 minutos.

## Publicación

- Repo público `s-jimenez-o/noviob` → GitHub Pages sirve `index.html` en
  `https://s-jimenez-o.github.io/noviob/`.
- Flujo: commit → `git push origin main` → esperar `gh api .../pages/builds/latest`
  = `built` → `diff` del HTML vivo contra el local para confirmar.
- En el teléfono: Safari → compartir → "Añadir a pantalla de inicio".

## Pendientes conocidos

- `icon.png` no existe: el ícono de pantalla de inicio sale como captura. Hacer un
  pixel art (la GBC morada con corazón) de 180×180.
- Los meses 2 a 6 (relleno) se eliminaron: el juego ahora va intro → patio →
  club → drags → casa del amigo → Puerto Escondido → cena → créditos.
- El `TEXTOS.md` se regenera a mano tras cada cambio de texto — mantenerlo al día.
- Ideas guardadas: ghosting sutil del LCD, variaciones de la música por acto.
- Etiqueta `antes-pulido-visual`: la versión previa a la pasada visual grande,
  por si hay que comparar o volver.
