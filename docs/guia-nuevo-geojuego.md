# Guía para añadir un geojuego

Complementa el checklist «Añadir un juego» del README de `casa/geojuegos` (léelo primero: esta guía lo desarrolla, no lo sustituye). El repo vive en `/home/agustin/workspace/casa/geojuegos` y se publica en `https://ekain.amutxastegi.com/geojuegos/`.

## Arquitectura

Multi-page app de Vite sin framework: **un directorio con `index.html` por juego** (`siluetas/`, `banderas/`, `viaje/`) más la portada, y **un módulo TypeScript por juego** en `src/` que manipula el DOM directamente. No hay router, ni estado global, ni componentes: cada juego es un script que engancha listeners a los ids de su HTML.

- Stack: Vite `^8.1.3` + TypeScript `^6.0.3` estricto (`npm run build` = `tsc && vite build`). Única dependencia runtime: `d3-geo ^3.1.0`.
- `vite.config.ts`: `base: './'` (el sitio cuelga de `/geojuegos/`, todas las rutas son relativas) y `rollupOptions.input` con las entradas HTML. **Vite no descubre páginas solo: si no registras la tuya, el build la omite sin dar error.**
- Deploy: GitHub Actions → Pages ejecuta solo `npm ci && npm run build`. Los datos de `public/` se versionan; el CI nunca los regenera.

### Módulos compartidos

Regla de dependencias: `data.ts` no importa d3-geo **a propósito** — así los juegos que no proyectan geometría (Banderas) no arrastran el chunk de d3 (~21 KB). Importa de `geo.ts` solo si pintas siluetas o mapas.

**`src/data.ts`** — índice de países y rutas de datos:

| Export | Qué es |
|---|---|
| `Country` | `{ iso, name, continent, centroid: LonLat }` |
| `loadCountries()` | fetch de `../data/countries.json` → `Country[]` |
| `flagUrl(iso)` / `flagThumbUrl(iso)` | `../flags/<iso>.svg` / `../flags/thumb/<iso>.webp` |
| `fetchJson<T>(url)` / `fetchText(url)` | fetch que lanza si `!res.ok` (fallar rápido) |

Las rutas son relativas a la página (`../`) porque cada juego vive un nivel bajo la raíz.

**`src/ui.ts`** — helpers de UI:

| Export | Qué es |
|---|---|
| `qs<T>(selector)` | querySelector tipado que **lanza** si el id no existe (mejor que `!`) |
| `loadError(el, err)` | mensaje común «No se pudieron cargar los datos. Recarga la página.» |
| `normalize(s)` | sin diacríticos, minúsculas, trim — para comparar nombres |
| `span(className, text)` / `shuffle(arr)` | utilidades menudas |
| `createCombobox({ form, input, list, candidates, onPick })` → `{ clear() }` | la entrada de país de todos los juegos |

El combobox filtra sin acentos (starts-with primero, máx. 8), navega con flechas/Escape y resuelve el conflicto pointerdown-vs-blur con un `setTimeout(hide, 150)`. `candidates` es una función: el juego decide las exclusiones en cada llamada (p. ej. `countries.filter((c) => !guessed.has(c.iso))`). Convenciones pendientes que un juego nuevo debe heredar cuando se apliquen en `ui.ts` (un solo cambio cubre a todos): semántica ARIA completa (`role="combobox"`, `aria-expanded`, `role="listbox"/"option"`, `aria-activedescendant`) y resaltar `items[0]` como selección implícita de Enter (`highlighted = 0` al mostrar), para que nunca se envíe una opción que no se ve marcada.

**`src/geo.ts`** — único puente a d3-geo para siluetas:

| Export | Qué es |
|---|---|
| `loadShape(iso)` | fetch de `../shapes/<iso>.json` (GeometryCollection del país) |
| `distanceKm(a, b)` / `bearingDeg(from, to)` | distancia ortodrómica redondeada / rumbo 0–360 |
| `MAX_DISTANCE_KM` | π·6371 (antípodas), para el % de proximidad |
| `silhouettePath(shape, centroid, w, h, margin = 12)` | path SVG con `geoAzimuthalEqualArea` rotada al centroide |
| `silhouetteSvg(shape, centroid, size)` | `<svg role="img">` cuadrado listo para insertar |

La proyección azimutal centrada en el país no es capricho: evita la distorsión en latitudes altas **y los cortes en el antimeridiano** (Rusia, Fiyi) sin código especial.

## Anatomía de un juego (Siluetas como ejemplo)

`siluetas/index.html` + `src/siluetas.ts` (179 líneas) es la plantilla de facto. Estructura del módulo:

1. **Constantes** arriba: `MAX_ATTEMPTS = 6`, tamaños, etc.
2. **Refs al DOM** con `qs<...>('#id')` — fallan ruidoso al cargar si el HTML y el TS divergen.
3. **Estado de ronda** en variables de módulo: `countries`, `target`, `guessed: Set<string>`, `finished`, `round`.
4. **Combobox** creado una vez, con `onPick: submitGuess`.
5. **Ciclo**: `newRound()` (resetea estado, `combo.clear()`, `input.focus()`) → `submitGuess(country)` (guarda contra `finished`/repetidos, pinta el intento en `#attempts`, decide) → `win()` / `lose()` (ocultan `form`, muestran `#result`).
6. **Arranque**: `loadCountries().then(... newRound()).catch(showLoadError)` y `againBtn` reengancha `newRound` con el mismo catch.
7. **Debug solo en dev**: `if (import.meta.env.DEV) Object.defineProperty(window, '__target', { get: () => target?.name })`. Nunca exponer la respuesta fuera de `DEV`.

HTML: copiar el `<head>` entero de `siluetas/index.html` (mismas Google Fonts byte-idénticas en los tres repos del dominio — no tocar la URL solo aquí, se perdería la caché compartida), cambiar `<title>` (patrón `Página · Geojuegos`), favicon emoji como data-URI y `h1`. El body lleva `class="game"` (más una clase propia si necesitas overrides de layout, como `body.viaje`).

Para mecánicas complejas mira `src/viaje.ts`: grafo + BFS (`bfsDist`, `bfsPath`, `buildPairs`), pan/zoom propio sobre un `<g transform>`, revelado progresivo y ayudas.

## Pasos

1. **HTML**: `cp siluetas/index.html <juego>/index.html`; ajustar title, favicon, h1 e ids propios. El script del final apunta a `/src/<juego>.ts`.
2. **Módulo**: `src/<juego>.ts` siguiendo la anatomía anterior. Reutiliza `ui.ts`/`data.ts` siempre; `geo.ts` solo si proyectas.
3. **Registrar en `vite.config.ts`**: añadir `<juego>: resolve(import.meta.dirname, '<juego>/index.html')` a `rollupOptions.input`. Verifica que `dist/<juego>/index.html` existe tras `npm run build` — el olvido es silencioso.
4. **Card en la portada** (`index.html`): nuevo `<li><a class="card" href="./<juego>/">` con `.name` y `.desc`. Si se anuncia antes de estar listo, existe `.card.soon` (dashed, apagada) con `<span>` en vez de `<a>`. Al publicar, **actualizar también la descripción de la card de Geojuegos en el repo `ekain`** para que enumere los juegos reales.
5. **Estilos**: reutilizar lo de `body.game`; lo nuevo al final de `src/style.css` bajo `/* ——— <Juego> ——— */` (así están Banderas en la 422 y Viaje en la 568). Paleta: solo variables de `:root` (`--ink`, `--brass`, `--paper`, `--ok`, `--bad`…). Toda animación nueva dentro del criterio de `@media (prefers-reduced-motion: reduce)`.

## Datos

Todo lo que sirve la web está en `public/` y **versionado**; los scripts de regeneración viven en `scripts/` con su propio `package.json` (`geojuegos-tools`: mapshaper `^0.7.39` + sharp `^0.35.3`, `npm install` aparte dentro de `scripts/`, nunca en CI).

| Dato | Comando | Fuente |
|---|---|---|
| `public/flags/` + `flags/thumb/` (WebP 320px q80) | `npm run sync:data` | copia del repo hermano `../flagmaps` + `build-flag-thumbs.mjs` (sharp) |
| `public/shapes/<iso>.json` + `public/data/countries.json` | `npm run build:shapes` | Natural Earth 10m, ~1200 vértices/país; microterritorios (<40 vértices) desde geoBoundaries/Nominatim |
| `public/data/borders.json` | `npm run build:borders` | mledoze/countries, cca3→alpha-2, simetría forzada; validar con `node scripts/verify-borders.mjs` |
| `public/data/world.json` | `npm run build:world` | unión de los shapes resimplificada hasta <1,5 MB (`MAX_BYTES`), winding gj2008, precisión 0.001 |

Orden de dependencia: `sync:data` → `build:shapes` (excluye países sin bandera para que las rondas de bandera nunca rompan) → `build:world`. Las descargas se cachean en `data/cache/` (gitignored).

Un juego nuevo que necesite otro dato sigue el patrón: script `scripts/build-<dato>.mjs` con comentario de cabecera denso (fuente, formato, decisiones), entrada npm `build:<dato>`, salida versionada en `public/data/` y, si el fichero pesa, un script de verificación tipo `verify-borders.mjs`.

**Presupuesto de peso**: `countries.json` son 20 KB y cada shape ~16 KB; `world.json` (~1,5 MB, ~500 KB gzip) es ya la mayor descarga de todo el portal. No añadir datos de arranque de ese orden; si es imprescindible, la vía conocida de mejora es TopoJSON + `topojson-client` (deduplica arcos; flagmaps ya lo usa) — mapshaper lo emite con `format=topojson`.

## Gotchas conocidos

- **Winding gj2008.** d3-geo no sigue RFC 7946: todo GeoJSON que consuma d3 debe emitirse con `gj2008` en mapshaper. Sin eso el país se pinta como **la esfera entera** y `geoCentroid` devuelve el antípoda. Aplica a cualquier dato nuevo que pase por mapshaper (`build-shapes` y `build-world` ya lo hacen).
- **Caché de `public/` del dev server.** Si regeneras shapes/thumbs/JSON con `npm run dev` arrancado, Vite sigue sirviendo lo viejo: reinicia `npm run dev`.
- **Ultramar / `mainPart`.** Antes de `fitExtent` sobre un país, recorta a su masa principal: Francia con la Guayana, EE. UU. con Alaska o Chile con Pascua estiran el encuadre a varios hemisferios. `viaje.ts` lo resuelve con `mainPart(iso)` (polígono mayor + los que estén a ≤1400 km del centroide oficial, memoizado en `mainPartCache`). Ojo al reutilizarlo como *fuente de siluetas*: la geometría de `world.json` es demasiado basta para microestados (Vaticano ~100 B); para siluetas reconocibles usa siempre `loadShape()`, y si cargas varias, lanza todas las promesas antes de esperar (`const loads = isos.map(loadShape)`) en vez de `await` en serie dentro del bucle.
- **Antimeridiano.** Dos técnicas según el caso: siluetas → proyección azimutal centrada en el centroide (`silhouettePath`, gratis); mapas regionales → recentrar el meridiano con `rotate([-centerLon(fc), 0])` corrigiendo que `geoBounds` devuelve oeste>este cuando la caja cruza el ±180 (ver `centerLon` en `viaje.ts`).
- **SVG de banderas.** El ratio real se parsea del texto del SVG (`flagRatio` en `banderas.ts`): varios (bd, dk, ki, no, uy) solo traen viewBox y `naturalWidth` miente; `width="100%"` debe descartarse. No inyectar esos SVG en el DOM (algunos traen `<style>` con clases genéricas); usar blob-URL + `<img>`. `img.decode()` es irregular con SVG en Safari: usar `onload`/`onerror` clásicos.
- **Animaciones `forwards`.** Quitar clases no revierte una animación con `fill: forwards`; Banderas reconstruye los paneles en cada ronda (`replaceChildren`) en vez de limpiarlos.

## Patrones de juego (convenciones)

**Rondas e intentos.** 6 intentos (`MAX_ATTEMPTS = 6`) en todos los juegos. `newRound()` resetea *todo* el estado visible — incluido `verdictEl` (`textContent = ''`, `className = ''`) — y termina en `input.focus()`. «Jugar otra vez» es siempre `#btn-again` reutilizando `newRound` con el mismo `.catch(showLoadError)` del arranque.

**Anti-carreras.** Todo flujo con `await`/`setTimeout` captura un token al empezar y lo compara antes de tocar el DOM: `round` (siluetas/banderas), `hintToken` (viaje). Sin esto, un «Jugar otra vez» rápido mezcla rondas.

**Carga y errores.** Fallar rápido y visible: `fetchJson` lanza con status, y el único mensaje de error es `loadError()` sobre un elemento visible del juego. Deshabilitar la entrada hasta que los datos estén listos (`input.disabled`, como Banderas). Reservar el alto de los contenedores que se rellenan tras un fetch con `aspect-ratio` (como `#shape` con `aspect-ratio: 1`) para que la página no salte al llegar los datos. Si el juego arranca con un fetch pesado, anunciarlo en su `index.html` *fuente* con `<link rel="preload" href="../data/<fichero>.json" as="fetch" crossorigin>` para solapar la descarga con el JS. Prefetch de la siguiente ronda cuando sea barato (Siluetas adelanta el shape siguiente).

**Veredicto accesible.** `#verdict` va **fuera de cualquier contenedor `hidden`**, con `aria-live="polite"` (aria-live dentro de un contenedor oculto no anuncia — es el patrón de Banderas y Viaje, y `#verdict:empty { position: absolute }` en `style.css` existe para preservarlo). `win()`/`lose()` escriben texto + clase `ok`/`bad`.

**Foco.** Tras cada intento, `input.focus()`. Al terminar la ronda el formulario se oculta: sin `againBtn.focus()` el foco cae a `body` (comentario literal en `banderas.ts` y `viaje.ts`). Si deshabilitas el botón que tiene el foco (rejillas de opciones), muévelo explícitamente.

**Toggles y estado.** Todo botón que conmuta algo lleva `aria-pressed` sincronizado con la variable (nace con `aria-pressed="false"` en el HTML), idealmente centralizado en un setter único. `role="img"` va en el propio `<svg>` con su `aria-label`, nunca en un `<div>` contenedor que además aloje botones (los hijos de `role="img"` son presentacionales). Las opciones cuyo nombre se oculta a propósito (rejilla de banderas) se distinguen con `aria-label="Opción N de 8"`, retirándolo al resolver.

**Mapa con pan/zoom.** Reutilizar el patrón de `viaje.ts`: `view = { k, x, y }` aplicado como `transform` a un `<g>`, `Map` de punteros para la pinza, `zoomAt` alrededor del cursor, dblclick/botón «Centrar» para resetear. Convenciones de convivencia con la página: rueda solo con Ctrl/Cmd (patrón de mapas embebidos), `touch-action: pan-y` en el svg (no `pinch-zoom`: cedería la pinza al navegador) y en `pointermove` no panear cuando `e.pointerType === 'touch'` y hay un solo dedo — un dedo hace scroll de página, dos manejan el mapa.

**Celebración.** Confeti con WAAPI (`confeti()` en `banderas.ts`: 80 piezas, paleta `['#d2a14c','#ece5d3','#7fbf7f','#d47d6a','#8a6f3d']`, se autolimpia). Comprobar `matchMedia('(prefers-reduced-motion: reduce)')` y ofrecer la variante instantánea. El veredicto de derrota es sobrio, sin ceremonia.

