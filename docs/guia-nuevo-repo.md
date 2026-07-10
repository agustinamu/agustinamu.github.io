# Guía: crear una herramienta nueva del portal

Cómo añadir un repo hermano bajo `ekain.amutxastegi.com/<herramienta>/`, desde `git init` hasta la card en la portada. Para todo lo visual (paleta, tipografía, componentes), ver la [guía de estilo](guia-estilo.md).

## Cómo funciona el dominio

El repo del portal (`~/workspace/casa/ekain`) es en realidad el **user site** `agustinamu/agustinamu.github.io`, con `CNAME` = `ekain.amutxastegi.com`. GitHub Pages sirve automáticamente cualquier **project site** de la misma cuenta bajo ese dominio: el repo `agustinamu/<herramienta>` se publica en `https://ekain.amutxastegi.com/<herramienta>/` sin configurar nada de dominio.

Consecuencias:

- **Nunca añadir un `CNAME`** al repo de una herramienta. El dominio vive solo en el user site.
- El nombre del repo **es** el path público. Elegirlo en minúsculas y pensando en la URL (`flagmaps` → `/flagmaps/`).
- Como todo comparte origen, la caché del navegador (fuentes de Google Fonts incluidas) se reutiliza al navegar entre portal y herramientas — siempre que las URLs de recursos compartidos sean byte-idénticas (ver Convenciones).

## Estructura de repo esperada

Vite + TypeScript estricto, vanilla (sin framework, sin router). Referencias vivas: `geojuegos` (multipágina, pipeline de datos) y `flagmaps` (página única).

```
<herramienta>/
├── index.html                 # entrada principal (portada si es multipágina)
├── <subpagina>/index.html     # solo multipágina; cada página en su carpeta
├── src/
│   ├── main.ts (o <pagina>.ts por entrada)
│   └── style.css              # UN solo CSS para todo el sitio
├── public/                    # datos servidos tal cual (SE VERSIONAN)
├── data/cache/                # descargas de fuentes externas (.gitignore)
├── scripts/                   # preprocesado local *.mjs (no va a CI)
│   └── package.json           # deps de tooling pesadas (mapshaper, sharp) aparte
├── vite.config.ts
├── tsconfig.json
├── package.json
├── .gitignore
├── .github/workflows/deploy.yml
└── README.md
```

Dependencias runtime: las mínimas imprescindibles y con imports quirúrgicos (`d3-geo`, no `d3`; `topojson-client` solo si hay TopoJSON). Sin tests, sin linter configurado: `tsc` estricto es la red de seguridad.

## Pasos

### 1. Crear el repo hermano

```bash
mkdir ~/workspace/casa/<herramienta> && cd ~/workspace/casa/<herramienta>
git init -b main
gh repo create agustinamu/<herramienta> --public --source=. --remote=origin
```

`.gitignore` mínimo (el de flagmaps):

```
node_modules/
dist/
data/cache/
.playwright-mcp/
```

### 2. package.json y tsconfig

`package.json` (patrón de geojuegos):

```json
{
  "name": "<herramienta>",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "typescript": "^6.0.3",
    "vite": "^8.1.3"
  }
}
```

El `build` encadena `tsc` (typecheck, `noEmit`) antes de `vite build`: el CI falla si no compila. `tsconfig.json` — copiar literal el de geojuegos:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "isolatedModules": true,
    "noEmit": true,
    "strict": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"]
}
```

### 3. vite.config.ts

`base: './'` es **obligatorio**: el sitio se sirve bajo `/<herramienta>/`, no en la raíz, y sin rutas relativas los assets 404ean en producción.

Página única (flagmaps):

```ts
import { defineConfig } from 'vite';

export default defineConfig({
  base: './',
});
```

Multipágina (geojuegos) — cada HTML debe registrarse en `rollupOptions.input`:

```ts
import { resolve } from 'node:path';
import { defineConfig } from 'vite';

export default defineConfig({
  base: './',
  build: {
    rollupOptions: {
      input: {
        main: resolve(import.meta.dirname, 'index.html'),
        subpagina: resolve(import.meta.dirname, 'subpagina/index.html'),
      },
    },
  },
});
```

**Gotcha**: si se olvida registrar una página, el build la omite **en silencio** — funciona en `npm run dev` y 404ea en producción. Es el primer punto a revisar cuando una página nueva no se despliega.

### 4. deploy.yml canónico

Copiar literal `.github/workflows/deploy.yml` (idéntico hoy en geojuegos y flagmaps; es el canónico para herramientas — el del portal es distinto porque no tiene build):

```yaml
name: Deploy a GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v5
        with:
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

Notas:

- `npm ci` exige `package-lock.json` commiteado.
- El CI solo ejecuta `npm run build`: **nunca** debe necesitar los scripts de preprocesado ni sus dependencias (ver Datos).
- Dependabot es opcional; si se quiere, copiar `.github/dependabot.yml` de geojuegos (npm semanal agrupando minor/patch + github-actions semanal). En el portal se eliminó porque sin `package.json` fallaba cada semana — en un repo con npm sí funciona.

### 5. Activar GitHub Pages

En github.com → repo → Settings → Pages → **Source: GitHub Actions**. Nada más: ni rama `gh-pages`, ni custom domain. Tras el primer push a main, verificar que el workflow acaba en verde y que `https://ekain.amutxastegi.com/<herramienta>/` responde.

### 6. Enlazarla desde el portal

En `ekain/index.html` hay dos secciones (`<h2>Juegos</h2>` y `<h2>Utilidades</h2>`) y un comentario que marca dónde añadir entradas. Patrón real de card:

```html
<li>
  <a class="tool" href="/<herramienta>/">
    <span class="name">Nombre visible</span>
    <span class="desc">Qué hace, en una frase concreta.</span>
  </a>
</li>
```

- `.name` es el nombre público de la herramienta (puede diferir del nombre del repo: el repo `flagmaps` se presenta como «Atlas de Banderas»).
- `.desc` describe lo que la herramienta hace **hoy**. Mantenerla al día cuando gane funciones: una desc que dice «y más en camino» cuando ya hay tres juegos publicados infravende el contenido.
- El push a main del portal redespliega la portada con su propio workflow.

## Datos: versionados vs generados

Regla del portal: **lo que se sirve se versiona; lo que se descarga para generarlo, no.**

- `public/` — datos finales (JSON, SVG, WebP…) commiteados en git. Vite los copia tal cual a `dist/`. Así el CI solo hace `npm run build`, sin red ni tooling pesado.
- `data/cache/` — descargas crudas de fuentes externas (Natural Earth, Banco Mundial…). En `.gitignore`; los scripts la reutilizan para no re-descargar.
- `scripts/*.mjs` — generan `public/` desde `data/cache/`. Se ejecutan a mano, en local, cuando cambian los datos. Convención de nombres: `build:<dato>` en los scripts de npm (`build:shapes`, `build:maps`, `build:stats`…) y `sync:<origen>` para copiar de repos hermanos.
- Si el tooling pesa (mapshaper, sharp), va en un `scripts/package.json` propio (`<herramienta>-tools`, ver geojuegos) para que el `npm ci` del CI no lo instale.
- Las banderas SVG no se duplican a mano: se copian del repo hermano con un script tipo `sync-data.mjs` (geojuegos las toma de `../flagmaps`). Si la herramienta muestra banderas en miniatura, generar thumbs WebP (patrón `build-flag-thumbs.mjs`, ~2,7 KB frente a SVGs de hasta 244 KB).
- Documentar en el README de la herramienta la procedencia de cada dato y el comando que lo regenera (ver `geojuegos/README.md` como modelo, incluido el gotcha del winding `gj2008` para d3-geo).

Gotcha de desarrollo: si se regenera `public/` con `npm run dev` arrancado, la caché de públicos de Vite queda obsoleta — reiniciar el dev server.

## Convenciones transversales

Lo visual (paleta tinta/latón, Fraunces + IBM Plex Mono, componentes) está en la [guía de estilo](guia-estilo.md). Además:

- **`<html lang="es">`** y todo el texto de UI en español.
- **`<title>`**: `Ekain · <Nombre público>` en la página principal de la herramienta (`Ekain · Geojuegos`); las subpáginas invierten el orden (`Siluetas · Geojuegos`). No filtrar el nombre interno del repo en el título.
- **Favicon**: emoji como data-URI SVG, mismo patrón que el portal (una línea, sin fichero).
- **Google Fonts**: copiar la URL **byte-idéntica** de los otros repos (mismos ejes, mismo orden, `display=swap`, con los dos `preconnect`). Cualquier variación rompe la caché compartida entre portal y herramientas. Si algún día se cambia (recortar ejes, autoalojar), hacerlo en los tres repos a la vez.
- **Enlace de vuelta al portal**: toda herramienta enlaza a `https://ekain.amutxastegi.com/` desde su página (patrón `<p class="back"><a href="…">← herramientas</a></p>` del hub de geojuegos). La navegación debe ser simétrica: portal → herramienta → portal.
- **Errores de carga visibles**: todo fetch inicial de datos lleva `catch` con mensaje al usuario («No se pudieron cargar los datos. Recarga la página.»), nunca solo `console.error` — una promesa rechazada sin handler deja la pantalla vacía sin explicación.
- **Reservar el alto** de los contenedores que se rellenan tras un fetch (`aspect-ratio` en CSS, patrón `#shape` de geojuegos) para que la página no salte al llegar los datos.
- **Datos pesados de arranque**: si un JSON grande se pide nada más cargar, anunciarlo en el `<head>` fuente con `<link rel="preload" href="data/x.json" as="fetch" crossorigin>` (el `crossorigin` es imprescindible para que el preload case con `fetch()`), y simplificar la geometría en el pipeline hasta lo que la vista realmente resuelve.
- **Accesibilidad mínima**: `aria-live="polite"` en los mensajes de resultado y **fuera** de contenedores `hidden` (no anuncia desde dentro); `aria-pressed` en botones toggle, incluido el estado inicial en el HTML; `role="status"` en toasts; gestionar el foco al ocultar el formulario (`againBtn.focus()`); nunca `outline: none` sin indicador de foco alternativo visible; `@media (prefers-reduced-motion: reduce)` si hay animaciones.
- **Táctil**: si hay pan/zoom propio, no secuestrar el scroll de página (`touch-action: pan-y` + panear solo con dos dedos en táctil; zoom de rueda solo con Ctrl/Cmd en escritorio) y ofrecer pinza de dos punteros.
- **Debug solo en dev**: exponer estado en `window.__*` únicamente tras `if (import.meta.env.DEV)`.

## Checklist final de publicación

1. `npm ci && npm run build` en limpio pasa (typecheck incluido) y `npm run preview` se ve bien.
2. Multipágina: todas las páginas registradas en `vite.config.ts` y presentes en `dist/`.
3. `public/` versionado; `data/cache/`, `dist/`, `node_modules/` ignorados; `package-lock.json` commiteado; sin capturas ni residuos sueltos en la raíz.
4. `deploy.yml` canónico copiado; Pages en Source «GitHub Actions»; sin `CNAME`.
5. Push a main → workflow verde → `https://ekain.amutxastegi.com/<herramienta>/` responde y los assets cargan (sin 404 por `base`).
6. `<title>` con patrón `Ekain · <Nombre>`, `lang="es"`, favicon emoji, URL de Google Fonts idéntica a la de los otros repos.
7. Enlace de vuelta al portal presente.
8. Card añadida en `ekain/index.html` en su sección, con `.desc` fiel a lo publicado; push del portal y comprobar el enlace en producción.
9. README con: qué hace, procedencia de cada dato y comando de regeneración, y cómo desarrollar (`npm install && npm run dev`).
10. Prueba rápida en móvil (o viewport estrecho): sin scroll horizontal, el scroll de página no queda atrapado, texto legible.

