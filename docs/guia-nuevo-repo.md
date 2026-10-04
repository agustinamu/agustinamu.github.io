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

### 4. Workflows y Dependabot canónicos

Copiar literal de **flagmaps** (referencia viva; geojuegos y orbitas son idénticos en lo común):

- `.github/workflows/deploy.yml` — build + deploy a Pages, `dependency-review` en PRs.
- `.github/workflows/dependabot-automerge.yml` — auto-merge de minor/patch.
- `.github/dependabot.yml` — npm semanal (grupos `dependencias` y `seguridad`) + github-actions, con cooldown.
- `.npmrc` — `min-release-age=7`.

No se copia aquí el YAML a propósito: las acciones van fijadas por SHA y Dependabot las actualiza en los repos, así que cualquier copia en esta guía queda desfasada. El del portal es distinto porque no tiene build.

Decisiones que el `deploy.yml` debe conservar (cada una corrige un fallo real, octubre 2026):

- **`concurrency` solo en el job `deploy`, con `cancel-in-progress: false`.** Un grupo único a nivel de workflow hacía que la comprobación de un PR cancelara la de otro, o un despliegue de main en curso.
- **Permisos mínimos:** arriba solo `contents: read`; `pages: write` e `id-token: write` en el job `deploy` (los permisos de un job sustituyen a los del workflow, no se suman). Así los PRs no reciben escritura en Pages.
- **`npm audit --omit=dev`** solo si el repo tiene dependencias de producción; sin ellas no audita nada (orbitas audita todo).

Y el auto-merge:

- **En PRs agrupados `update-type` llega `null`**: la condición debe incluir `steps.meta.outputs.dependency-group == 'dependencias'` (el grupo limitado a minor/patch). Nunca `seguridad`, que admite majors.
- **Lo que fusiona Dependabot con `GITHUB_TOKEN` no lanza `deploy.yml`**: se despliega a mano cuando interese (`gh workflow run deploy.yml --ref main`). Decisión del dueño: sin cron ni PAT.

Notas:

- `npm ci` exige `package-lock.json` commiteado.
- El CI solo ejecuta `npm run build`: **nunca** debe necesitar los scripts de preprocesado ni sus dependencias (ver Datos).

### 5. Activar GitHub Pages

En github.com → repo → Settings → Pages → **Source: GitHub Actions**. Nada más: ni rama `gh-pages`, ni custom domain. Por CLI: `gh api -X POST repos/agustinamu/<herramienta>/pages -f build_type=workflow`.

Ajustes del repo que necesita el auto-merge (sin ellos Dependabot fusiona aunque el build falle o `dependency-review` falla siempre):

```bash
R=agustinamu/<herramienta>
gh api -X PATCH repos/$R -F allow_auto_merge=true
gh api -X PUT repos/$R/vulnerability-alerts          # activa también el grafo de dependencias
gh api -X PUT repos/$R/automated-security-fixes
echo '{"required_status_checks":{"strict":false,"contexts":["build"]},"enforce_admins":false,"required_pull_request_reviews":null,"restrictions":null}' \
  | gh api -X PUT repos/$R/branches/main/protection --input -
``` Tras el primer push a main, verificar que el workflow acaba en verde y que `https://ekain.amutxastegi.com/<herramienta>/` responde.

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
- **Táctil**: si hay pan/zoom propio, no secuestrar el scroll de página (`touch-action: pan-y` + panear solo con dos dedos en táctil; rueda directa para zoom en escritorio — decisión del dueño tras probar Ctrl+rueda) y ofrecer pinza de dos punteros.
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

