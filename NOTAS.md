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
- Prioridad de espacio: **la pantalla manda**. El bisel y los márgenes se
  mantienen delgados (bisel 5-6u, carcasa 5u) para que en un teléfono la pantalla
  sea lo más grande posible; los botones grandes y separados (dpad 62u, A/B 28u)
  porque se juega con el pulgar.

## La pantalla y la escala

- Canvas fijo de **160×144** (Game Boy real). Nunca dibujar fuera de eso.
- `GROUND = 92` es la línea de piso: los personajes (16×30) se paran en `y = GROUND - 30 = 62`.
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

## Controles

- `keys[k]` = está presionado (para caminar y para acelerar texto manteniendo A);
  `consumed(k)` = flanco de un toque (para confirmar). No mezclarlos.
- Mantener A/B acelera el texto ×3; un toque lo completa; otro toque pasa de página.

## El selector de escenas (herramienta de desarrollo)

- `SALTOS` salta a cualquier punto; cada entrada debe dejar los flags como estarían
  llegando ahí jugando (cangurera, maleta, cuartoNoche, mesaElegida...).
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
4. **Capturas**: `grab.sh nombre "código de dibujo"` congela un cuadro y lo exporta
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
- Los textos de los **meses 2 a 6** siguen siendo relleno; faltan los recuerdos reales.
- El `TEXTOS.md` se regenera a mano tras cada cambio de texto — mantenerlo al día.
- Ideas guardadas: ghosting sutil del LCD, variaciones de la música por acto.
- Etiqueta `antes-pulido-visual`: la versión previa a la pasada visual grande,
  por si hay que comparar o volver.
