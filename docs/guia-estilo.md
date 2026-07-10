# Guía de estilo del portal ekain

Convenciones visuales y de código compartidas por los tres repos que sirven bajo `ekain.amutxastegi.com`:

| Repo | Ruta publicada | Papel |
|---|---|---|
| `ekain` | `/` | Portada de enlaces (HTML puro, sin build) |
| `geojuegos` | `/geojuegos/` | Juegos de geografía (Vite multipágina) |
| `flagmaps` | `/flagmaps/` | Atlas de Banderas (Vite) |

**Principio rector:** los tres sitios deben leerse como uno solo — misma tinta, mismo latón, misma tipografía — sin compartir código en runtime. La consistencia se mantiene copiando estos valores, no extrayendo paquetes comunes (YAGNI: dos copias con nota de origen valen más que una dependencia).

La estética se llama internamente **«sala de cartografía: tinta nocturna, latón y papel»** (comentario de cabecera de `flagmaps/src/style.css`).

---

## 1. Tipografía

Dos familias, cargadas desde Google Fonts:

| Familia | Variable CSS (flagmaps) | Uso | Fallbacks |
|---|---|---|---|
| **Fraunces** (variable: `ital`, `opsz 9..144`, `wght 300..700`) | `--display` | Titulares (h1, h2), nombres de tarjeta (`.name`), contador, veredictos | `'Iowan Old Style', Georgia, serif` (mínimo `Georgia, serif`) |
| **IBM Plex Mono** 400/500/600 | `--mono` | Todo lo demás (cuerpo, botones, datos) | `'Courier New', monospace` (mínimo `monospace`) |

Pesos realmente en uso: Fraunces **300** (itálicas decorativas, contador), **400–500** (titulares); Plex Mono **400** base, **500/600** para énfasis de datos (`#statusbar b`, métrica resaltada del tooltip).

**URL canónica** — debe ser **byte-idéntica** en todas las páginas de los tres repos, porque así el navegador reutiliza CSS y woff2 al navegar portada → juego (verificado: hoy lo es):

```
https://fonts.googleapis.com/css2?family=Fraunces:ital,opsz,wght@0,9..144,300..700;1,9..144,300..700&family=IBM+Plex+Mono:wght@400;500;600&display=swap
```

Con sus dos `preconnect` (`fonts.googleapis.com` y `fonts.gstatic.com` con `crossorigin`) y `display=swap`.

Reglas:

- **Nunca cambiar la URL de fuentes en un solo repo.** Cualquier recorte (quitar la itálica de Fraunces, ~80 KB usados solo en 2-4 palabras por sitio; quitar Plex Mono 500/600 donde no se usen) o el autoalojamiento de los woff2 se hace en los tres repos a la vez o no se hace.
- Si algún día se autoaloja: 5 woff2 subset latin (~194 KB), declarando los rangos variables (`font-weight: 300 700` y eje `opsz` de Fraunces).
- Datos numéricos siempre con `font-variant-numeric: tabular-nums`.
- La itálica de Fraunces es un recurso de marca (ver wordmark, §6), no de énfasis.

## 2. Paleta canónica

Variables en `:root`, mismos nombres y valores en los tres repos:

```css
:root {
  --ink: #0b131c;        /* fondo de página */
  --ink-2: #101b27;      /* superficies: tarjetas, inputs, barras */
  --brass: #d2a14c;      /* acento latón: títulos, hovers, valores */
  --brass-soft: #8a6f3d; /* bordes en reposo — NO para texto (§8) */
  --paper: #ece5d3;      /* texto principal */
  --paper-dim: #9a937f;  /* texto secundario */
  --ok: #7fbf7f;         /* acierto / origen / confirmación */
  --bad: #d47d6a;        /* fallo / error */
}
```

`--ok` y `--bad` los define geojuegos y son **los canónicos para semántica de estado en todo el portal**. Divergencia resuelta: el toast de error de flagmaps (`#b4553f`/`#f0c8bd` en `src/style.css`) debe migrar a `border-color: var(--bad)` con texto `var(--paper)`. No introducir más rojos ni verdes.

**Fondo de página canónico** (portada y hubs):

```css
background: radial-gradient(ellipse at 50% -20%, rgba(210, 161, 76, 0.08), transparent 60%), var(--ink);
```

**Latón translúcido:** los estados intermedios se construyen con `rgba(210, 161, 76, α)` sobre fondos oscuros. Escala de alfas en uso: `0.08` (hover de tarjeta/botón, gradiente de fondo), `0.15–0.16` (sugerencia activa, fila seleccionada), `0.22–0.25` (separadores, bordes internos). Bordes atenuados: `rgba(138, 111, 61, 0.5)` (brass-soft al 50%).

**Colores locales de mapa** (permitidos, no forman parte de la paleta compartida): en flagmaps `--ocean #0e1822`, `--land #2b3a47`, `--land-hover #3d5266`, `--border #0a1118`, esfera `#101c28`, rampa choropleth de 7 latones (`#4f4026 → #fccc79`) y paleta clara solo para export impreso; en geojuegos océano `#0c1620` y país base `#1b2a37`. Si se añade un mapa nuevo, partir de estos.

Contrastes medidos sobre `--ink` / `--ink-2` (referencia rápida): `--paper` 14.9/13.8 · `--ok` 8.6/8.0 · `--brass` 8.0/7.4 · `--bad` 6.2/5.8 · `--paper-dim` 6.1/5.7 · `--brass-soft` **3.9/3.7 (insuficiente para texto)**.

## 3. Superficie, espaciado, radios y sombras

- **Esquinas vivas: sin `border-radius`.** Excepciones deliberadas: `.pip` (círculo, `50%`) y el confeti (`1px`). Nada más lleva radio.
- **Bordes de 1px sólidos**, `--brass-soft` en reposo → `--brass` en hover/activo.
- **Transiciones de 0.15s** en `border-color`, `background` y `color`. Animaciones de entrada de listas: `li-in` 180ms (opacidad + `translateY(-4px)`).
- **Sin sombras en superficies planas.** Solo los overlays flotantes las usan: fondo `rgba(11, 19, 28, 0.88–0.96)` + `backdrop-filter: blur(3px)`, y `box-shadow: 0 6px 22px rgba(0, 0, 0, 0.45)` únicamente en el menú contextual.
- **Espaciado en `rem`.** Referencias: padding de tarjeta `1.1rem 1.3rem`, gap entre tarjetas `1rem`, columna de contenido `max-width: 34rem` (portadas) o `26–30rem` (juegos; Viaje amplía a `min(60rem, 96vw)` porque el mapa es el protagonista).
- **Reset mínimo:** `* { box-sizing: border-box }`, `body { margin: 0 }`. En apps con `hidden` dinámico, incluir `[hidden] { display: none !important }` para que gane a displays de autor.

## 4. Escala tipográfica y rótulos

- **Suelo de texto funcional: 0.68rem (~11px).** Nada que el usuario necesite leer para usar la app baja de ahí. (Estado deseado: los rótulos de 0.58rem de flagmaps — tira de comparación, leyenda, `dt` del tooltip — suben a 0.68–0.7rem.)
- **Descripciones de tarjeta (`.desc`): 0.85rem.** (Estado deseado; el 0.74rem original era incómodo en móvil para el único texto que explica cada herramienta.)
- **Micro-etiquetas decorativas** (el dominio `.sub`, statusbar): pueden ser menores porque son redundantes, siempre `text-transform: uppercase` + `letter-spacing` amplio (`0.34em` en `.sub`, `0.12–0.14em` en barras). Las mayúsculas se aplican **solo por CSS**, nunca en el contenido.
- Titulares: h1 de portada 2.6rem, h1 de juego 1.7rem, h1 de app densa (flagmaps) 1.45rem; h2 de sección 1.05rem Fraunces itálica.

## 5. Componentes comunes

### Tarjeta-enlace (`.card`)

Nombre canónico de clase: **`.card`** (geojuegos; en ekain aún se llama `a.tool` — al tocarla, renombrar). El hover se replica en `:focus-visible`:

```html
<a class="card" href="./siluetas/">
  <span class="name">Siluetas</span>
  <span class="desc">Adivina el país por su forma. Seis intentos…</span>
</a>
```

```css
.card {
  display: block;
  padding: 1.1rem 1.3rem;
  background: var(--ink-2);
  border: 1px solid var(--brass-soft);
  color: var(--paper);
  text-decoration: none;
  transition: border-color 0.15s, background 0.15s;
}
a.card:hover,
a.card:focus-visible {
  border-color: var(--brass);
  background: rgba(210, 161, 76, 0.08);
}
.card .name { font-family: 'Fraunces', Georgia, serif; font-size: 1.25rem; color: var(--brass); }
.card .desc { display: block; margin-top: 0.35rem; font-size: 0.85rem; letter-spacing: 0.04em; color: var(--paper-dim); }
.card.soon { opacity: 0.45; border-style: dashed; } /* «próximamente» */
```

### Botón

Base común: transparente o `--ink-2`, borde `--brass-soft`, hover borde `--brass` + latón translúcido. Dos registros según densidad:

```css
/* Juegos (geojuegos): texto en latón, tamaño heredado */
button {
  font: inherit;
  padding: 0.65rem 1rem;
  background: var(--ink-2);
  border: 1px solid var(--brass-soft);
  color: var(--brass);
  cursor: pointer;
  transition: border-color 0.15s, background 0.15s;
}
button:hover:not(:disabled) { border-color: var(--brass); background: rgba(210, 161, 76, 0.08); }
```

En apps de barra densa (flagmaps): fondo transparente, `0.72rem` uppercase con `letter-spacing: 0.08em`, texto `--paper`; variante `.accent` (borde y texto `--brass`, hover invertido `background: var(--brass); color: var(--ink)`) reservada a las acciones destacadas (exportar). Barras segmentadas (`#mode-bar`): botones sin borde propio unidos por `border-right`, `.active` con fondo `--brass` y texto `--ink`, **siempre con `aria-pressed`** (§8).

### Combobox de países

Componente canónico: **`createCombobox` en `geojuegos/src/ui.ts`** (autocontenido, ~80 líneas). Comportamiento definido: matching sin acentos vía `normalize()` (NFD + quitar diacríticos), prefijos antes que `includes`, máximo 8 sugerencias, flechas/Escape, `pointerdown` con `preventDefault` en las sugerencias y `blur` diferido 150ms para que el clic gane.

```html
<div id="combo">
  <input id="guess" placeholder="Escribe un país…" spellcheck="false" autocomplete="off" aria-autocomplete="list" />
  <ul id="suggestions" hidden></ul>
</div>
```

Lista: fondo `--ink-2`, borde `--brass` sin borde superior, ítem activo/hover `rgba(210, 161, 76, 0.15)` + texto `--brass`.

Estado deseado (pendiente en el componente): semántica ARIA completa (`role="combobox"` + `aria-expanded` en el input, `aria-controls`, `role="listbox"`/`role="option"` con ids, `aria-activedescendant`) y resaltar `items[0]` como `.active` cuando no hay selección explícita, porque Enter lo envía.

Divergencia conocida: flagmaps usa `<datalist>` nativo (popup con chrome del navegador, fuera de tema, igual que su `<select>`). Mitigación mínima si no se porta el combobox: `color-scheme: dark` en `:root`. Portarlo requiere adaptar Enter/form y el tipo `Country`.

### Mensajes de estado

Dos patrones, sin inventar terceros:

**Toast efímero** (apps tipo herramienta — `flagmaps/src/menu.ts`): elemento único `.toast` fijo abajo-centro, reutilizado entre llamadas, visible 2200ms con transición de opacidad+transform. `kind: 'ok' | 'bad'`; el `bad` colorea el borde con `var(--bad)`. Debe llevar `role="status"` y, para que el primer aviso se anuncie, crearse vacío al inicio en vez de perezosamente.

**Veredicto persistente** (juegos): `<p id="verdict" aria-live="polite">` **siempre fuera de cualquier contenedor `hidden`** (aria-live dentro de hidden no anuncia — comentario canónico en `banderas/index.html`), con:

```css
#verdict:empty { position: absolute; } /* fuera del flujo SIN display:none,
   que sacaría la región del árbol de accesibilidad */
```

Clases semánticas `.ok` / `.bad` sobre las variables. Limpiar con `replaceChildren()` al empezar ronda.

**Errores de carga:** texto canónico `'No se pudieron cargar los datos. Recarga la página.'` (`loadError` en `geojuegos/src/ui.ts`); variante para mapas: `'No se pudo cargar el mapa. Recarga la página.'`. **Ninguna promesa de carga se deja sin `.catch`**: todo fallo de red produce uno de estos mensajes, nunca pantalla vacía ni reversión silenciosa.

### Otros patrones establecidos

Chips de cadena (`.chip`, borde semántico `--ok`/`--brass`/`--paper`), pips de fallo (`.pip` / `.pip.used` en `--bad`), listas con filas cebra `rgba(255,255,255,0.02)`, tooltip fijo con blur. Reutilizarlos antes de crear variantes.

## 6. Cabecera de documento: título, favicon, meta

Esqueleto obligatorio en toda página:

```html
<!doctype html>
<html lang="es">
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

Sin `user-scalable=no`: el zoom de página del navegador es la red de seguridad táctil.

**Títulos.** Dos niveles:

- Página raíz de cada sitio: `Ekain · <Nombre visible>` → `Ekain · Juegos y utilidades`, `Ekain · Geojuegos`, `Ekain · Atlas de Banderas` (estado deseado: el `Flagmaps · …` actual filtra el nombre interno del repo, que no aparece en ningún sitio visible).
- Subpáginas: `<Página> · <Sitio>` → `Siluetas · Geojuegos`, `Banderas · Geojuegos`, `Viaje · Geojuegos`.

**Favicon:** emoji como data URI SVG, plantilla exacta:

```html
<link rel="icon"
  href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>🛠️</text></svg>" />
```

Asignados: 🛠️ portal · 🎯 hub Geojuegos · 🗺️ Siluetas · 🧭 Viaje · 🚩 Banderas · 🌍 Atlas de Banderas.

**Wordmark.** El h1 corta el nombre con itálica latón: `Geo<em>juegos</em>`, `Band<em>eras</em>`, `Atlas <em>de</em> Banderas`. Regla de higiene (estado deseado): como el uso es decorativo, el marcado canónico es `<span class="accent">` con `font-style: italic; font-weight: 300; color: var(--brass)`, no `<em>`. Debajo, `.sub` en uppercase espaciado con guiones largos: `— ekain.amutxastegi.com —`, `— geografía a ciegas —`.

**Navegación entre sitios.** Toda herramienta enlaza de vuelta: el hub con `<p class="back"><a href="https://ekain.amutxastegi.com/">← herramientas</a></p>`, las páginas de juego con `<a class="home" href="../">←</a>`. Estilo discreto: `--paper-dim`, hover `--brass`. Flagmaps debe incorporarlo (hoy es el único sin vuelta al portal). Contenido colgando de un landmark `<main>` (moviendo a él el layout flex si hace falta, no con `display:contents`).

## 7. Tono de los textos

- **Español siempre** (`lang="es"`, nombres de países en español — `NAME_ES` de Natural Earth —, números con `Intl` es-ES).
- Frases cortas, directas, sin exclamaciones ni relleno. Descripciones de tarjeta: una frase que dice qué hace la herramienta (`«Adivina países por su silueta, su bandera o viajando de país en país.»`) — y **actualizarla cuando la herramienta crezca**: no infravender contenido publicado.
- Errores: qué pasó + qué hacer, en ≤10 palabras (`«No se pudo copiar la bandera»`, `«Recarga la página»`).
- Instrucciones de uso con `·` como separador y el gesto en negrita: `<b>clic</b> bandera · <b>rueda</b> zoom`. Toda función debe aparecer en la ayuda visible; nada solo descubrible por accidente (el clic derecho de flagmaps debe listarse).
- Sin emojis en el contenido (solo favicons). Sin mayúsculas en el texto fuente: el uppercase es cosa del CSS.
- Comentarios de código: en español, documentando el **porqué** y los gotchas medidos (es la convención dominante en los tres repos).

## 8. Accesibilidad mínima (no negociable en código nuevo)

**Foco.**
- Prohibido `outline: none` sin sustituto visible. Indicador canónico para inputs y controles: `outline: 2px solid var(--brass); outline-offset: -1px;` (un cambio de color de borde de 1px no basta: ~2:1 entre estados).
- Todo `:hover` con significado se replica en `:focus-visible`.
- Todo elemento clicable es enfocable y operable con Enter/Espacio: si es acción, que sea `<button>`; si no puede serlo (fila de lista, figura), `tabindex="0"` + keydown.
- Gestión de foco al ocultar el control enfocado: mover el foco explícitamente (patrón `againBtn.focus()` con su comentario en banderas/viaje).

**Contraste.**
- Texto normal ≥ 4.5:1 sobre `--ink`/`--ink-2`. Aptos: `--paper`, `--paper-dim`, `--brass`, `--ok`, `--bad`. **`--brass-soft` es solo para bordes**; si un texto necesita ese tono (h2 de sección de la portada), usar `#9d804a` (5.0:1) como mínimo aclarado. Ojo: el gradiente radial aclara la zona alta de la página y come ~0.4 puntos de ratio.

**Semántica y anuncios.**
- Estados de toggle con `aria-pressed`, tanto en el HTML inicial como en cada cambio de clase `.active`.
- Regiones vivas según §5: toasts `role="status"`, veredictos `aria-live="polite"` fuera de `hidden`.
- `aria-label` solo sobre elementos con rol: `role="region"` para viewports de mapa (no `role="img"` en contenedores con controles dentro: aplana el subárbol).
- `<em>`/`<strong>` solo con intención semántica; decoración con clases.

**Táctil y movimiento.**
- Mapas con pan/zoom propio: `touch-action: pan-y` (no `none`, no `pinch-zoom`: un dedo hace scroll de página, la pinza llega a la app), pan de un dedo solo con ratón, zoom de rueda solo con Ctrl/Cmd cuando el mapa convive con una página que hace scroll.
- Objetivos táctiles de acciones frecuentes ≥ ~44px de lado (los paddings de botón actuales lo cumplen; no reducirlos).
- Toda animación decorativa (destapes, confeti, ondeo) dentro de `@media (prefers-reduced-motion: no-preference)` o anulada en `reduce`, dejando el estado final correcto.

**Layout estable.**
- Los contenedores que reciben contenido asíncrono reservan su hueco (`aspect-ratio` — `1` en siluetas, `960/620` en el mapa de Viaje): la página no salta cuando llegan los datos, y los controles se deshabilitan hasta tenerlos.

## 9. Checklist para una herramienta o página nueva

1. `lang="es"`, viewport estándar, título según patrón (§6), favicon emoji propio.
2. La URL de Google Fonts, copiada byte a byte (§1).
3. `:root` con la paleta de §2; `--ok`/`--bad` se añaden en el primer uso semántico de estado (YAGNI), el resto desde el inicio.
4. Fondo canónico, tarjetas/botones/combobox de §5 antes que componentes nuevos.
5. Enlace de vuelta al portal y entrada en la portada de ekain (hay plantilla en comentario HTML de `ekain/index.html`), con `.desc` fiel a lo publicado.
6. Repasar §8 completo: foco, contraste, aria-pressed, live regions, táctil, reduced-motion, `.catch` en toda carga.

