# Auditoría UX + rendimiento del portal — julio 2026

Instantánea de la revisión multi-agente (2026-07-10) de los tres repos del portal
(`ekain`, `flagmaps`, `geojuegos`). **Lista de trabajo, no convención**: borrar los
hallazgos según se apliquen (o el fichero entero cuando quede vacío). Las
convenciones derivadas viven en [guia-estilo.md](guia-estilo.md),
[guia-nuevo-repo.md](guia-nuevo-repo.md) y [guia-nuevo-geojuego.md](guia-nuevo-geojuego.md).

Método: 8 revisores (UX+accesibilidad y rendimiento por repo, consistencia
cruzada, revisión en vivo con navegador) → dedup → un verificador escéptico por
hallazgo contra el código real. 50 confirmados; 16 de severidad baja quedaron
sin verificar (al final).

Resumen por repo: ekain 9 · flagmaps 26 · geojuegos 15.

**Actualización 2026-07-10: aplicados 45 de 50 confirmados (quedan 5, todos de la familia Google Fonts, descartados por la regla de los 3 repos) + 11 de 16 sin verificar; ver git diff de cada repo (sin commitear). Flecos de la revisión también resueltos: toast espurio de flagmaps (select() devuelve éxito), salto residual de #map (aspect-ratio sobre content-box, medido 0.00 px) y aria-labels muertos de siluetas/fronteras.**

---

## Hallazgos confirmados

### [ekain] [media] [rendimiento] 163 KB de fuentes web para una página de 3,9 KB; la itálica de Fraunces (81,5 KB) se descarga para 4 palabras
- fichero: index.html:14 (fuente: perf:ekain)
- detalle: Medido con curl (UA Chrome): la portada descarga en visita fría 3 woff2 subset latin — Fraunces normal 67.304 B, Fraunces itálica 81.520 B, IBM Plex Mono 400 14.708 B — total 163.532 B, ~42x el peso del HTML (3.880 B). El fichero itálico, el asset más pesado de la página, existe solo para renderizar '·', 'y', 'Juegos' y 'Utilidades' (h1 em y los dos h2). La URL pide además los ejes completos de Fraunces (opsz 9..144, wght 300..700 en redonda e itálica) y los pesos 500/600 de Plex Mono que la portada nunca usa (estos últimos no se descargan, solo engordan la CSS). Atenuante importante verificado: la URL es byte-idéntica en los 6 HTML de los tres repos del mismo sitio, así que el navegador reutiliza CSS y woff2 al pasar a /geojuegos/ y /flagmaps/ — el coste se paga una vez por visita fría al portal.
- sugerencia: No tocar la URL solo en ekain: perdería la caché compartida con los otros dos repos y empeoraría el flujo portada→juego. Si se actúa, hacerlo en los tres repos a la vez: recortar la petición a los ejes/pesos realmente usados (Fraunces wght 300..500 basta en la portada; auditar flagmaps/geojuegos antes) o autoalojar 3 woff2 subseteados. Si no se va a tocar el conjunto, dejarlo como está: es la opción correcta bajo YAGNI.
- corrección del verificador: Dos matices: (1) "ejes completos de Fraunces" solo es exacto para opsz (9..144); el eje wght completo de Fraunces es 100..900 y la URL pide 300..700 — un sub-rango, aunque más ancho que el 300..500 realmente usado en la portada. (2) Con display=swap el coste para el usuario en visita fría es FOUT/cambio de fuente tras la descarga, no bloqueo de render, lo que refuerza que la severidad sea como mucho media.
- **Estado:** no aplicado — descartado deliberadamente (YAGNI/regla de los 3 repos): solo compensa si se recorta la URL de Google Fonts en ekain, flagmaps y geojuegos a la vez, para no perder la caché compartida.

### [ekain] [media] [rendimiento] La CSS de Google Fonts (8,5 KB, tercer origen) es el único recurso render-blocking y domina la ruta crítica
- fichero: index.html:13 (fuente: perf:ekain)
- detalle: El primer render de la portada espera a que fonts.googleapis.com devuelva la hoja de estilos (8.543 B, 21 @font-face): con el HTML propio en ~1,5 KB comprimidos, esa petición a un tercer origen (DNS+TLS+RTT, mitigado por los preconnect de las líneas 11-12) es prácticamente todo el tiempo hasta primer pintado en frío. display=swap ya está, así que tras llegar la CSS el texto pinta con fallback mientras bajan los woff2; el bloqueo es solo la CSS.
- sugerencia: La solución probada es autoalojar las fuentes con @font-face inline en el <style> existente: elimina el recurso bloqueante y las dos conexiones a terceros. Solo compensa si se hace en los tres repos (misma trade-off de caché que el hallazgo anterior); como cambio aislado en ekain no lo recomiendo. Alternativas parciales (media=print+onload, preload) añaden complejidad o FOUC sin ganancia clara: descartadas.
- corrección del verificador: Única imprecisión: los 8.543 B son el tamaño sin comprimir; sobre el cable la CSS viaja con gzip y son ~829 B, menos que el propio HTML. La comparación de tamaños es engañosa, pero el argumento central se sostiene porque el coste dominante es la latencia de la petición bloqueante al tercer origen (conexión + RTT), no los bytes, tal como el propio hallazgo indica.
- **Estado:** no aplicado — descartado deliberadamente (YAGNI/regla de los 3 repos): autoalojar fuentes solo compensa si se hace a la vez en ekain, flagmaps y geojuegos.

### [flagmaps] [media] [rendimiento] Se descargan 67 KB de Fraunces itálica para renderizar solo la palabra «de» del título
- fichero: index.html:14 (fuente: perf:flagmaps)
- detalle: La request a Google Fonts pide Fraunces con ambos ejes ital (0 y 1). Medido: el woff2 latin itálico pesa 67.304 B de los 194.040 B totales de fuentes. El único uso de itálica es `.brand h1 em` (src/style.css:59-63), dos letras en la cabecera; IBM Plex Mono no usa itálica en ningún sitio.
- sugerencia: Quitar el tramo `;1,9..144,300..700` de la URL de Google Fonts (dejar solo la romana). El navegador sintetizará la cursiva del «de» (oblicua, visualmente aceptable a 1.45rem) o se elimina el <em>. Ahorro directo del 35% del peso de fuentes.
- corrección del verificador: El woff2 latin itálico de Fraunces pesa 81.520 B (no 67.304 B, que es el latin romano); sobre los 194.040 B totales de fuentes el ahorro real al quitar `;1,9..144,300..700` es ~42%, no 35%. El resto del hallazgo es exacto.
- **Estado:** no aplicado — no abordado en esta ronda; forma parte de la familia de hallazgos de Google Fonts (ver también los de ekain y geojuegos), pendiente de decidir si se aborda coordinadamente en los 3 repos.

### [flagmaps] [baja] [rendimiento] CSS de Google Fonts render-blocking desde origen de terceros en los tres sitios del dominio
- fichero: index.html:13 (fuente: perf:flagmaps)
- detalle: El <link rel="stylesheet"> a fonts.googleapis.com bloquea el primer render hasta resolver DNS+TLS+CSS de un origen externo (el preconnect mitiga, no elimina). Además ekain, flagmaps y geojuegos repiten la misma request, sin compartir caché entre orígenes de fuentes por el particionado de caché de los navegadores modernos.
- sugerencia: Autoalojar los 5 woff2 latin (194 KB medidos, menos si se aplica el hallazgo de la itálica) en public/fonts con un bloque @font-face en style.css. Elimina el CSS externo bloqueante y la dependencia de terceros; patrón probado y sin build extra. Solo si se hace, hacerlo igual en los tres repos para mantener consistencia.
- corrección del verificador: El <link rel="stylesheet"> a fonts.googleapis.com (flagmaps/index.html:13, idéntico en ekain y en las 4 páginas de geojuegos) bloquea el primer render en visita fría hasta resolver DNS+TLS+CSS del origen externo; el preconnect mitiga pero no elimina. Nota: al estar los tres sitios bajo el mismo host ekain.amutxastegi.com, comparten partición de caché — las fuentes cacheadas en uno se reutilizan en los otros; el coste repetido solo ocurre tras expirar la caché (CSS ~24h). El particionado penaliza únicamente frente a terceros sitios con Google Fonts. Sugerencia válida: autoalojar los 5 woff2 latin (194 KB medidos: Fraunces normal+itálica variable y Plex Mono 400/500/600) con @font-face, cuidando declarar los rangos variables (font-weight: 300 700 y eje opsz de Fraunces). Impacto: solo primera visita / caché fría; severidad baja correcta.
- **Estado:** no aplicado — descartado deliberadamente (YAGNI/regla de los 3 repos): el propio hallazgo condiciona la mejora a aplicarla igual en ekain, flagmaps y geojuegos.


### [geojuegos] [media] [rendimiento] CSS de Google Fonts render-blocking en las 4 páginas y petición de Fraunces sobredimensionada (itálica completa para un <em>)
- fichero: index.html:14 (fuente: perf:geojuegos)
- detalle: Las 4 páginas cargan la hoja de fonts.googleapis.com como stylesheet normal: bloquea el primer render hasta resolver conexión + CSS de un tercer origen (el preconnect mitiga, no elimina). Payload real medido para una página en español (subconjunto latin): 194.040 B en 5 woff2 — Fraunces roman 81,5 KB, Fraunces itálica 67,3 KB, IBM Plex Mono 400/500/600 ~45 KB. La itálica de 67 KB se usa solo para `h1 em` y `.sub`/cabeceras (style.css:38), y se pide el rango completo ital,opsz 9..144, wght 300..700 cuando el CSS solo usa pesos 300-600.
- sugerencia: Autoalojar los woff2 latin (5 ficheros, 194 KB, URLs estables de fonts.gstatic.com) en public/fonts/ con @font-face en style.css y <link rel="preload" as="font"> para Fraunces roman y Plex Mono 400: elimina la petición CSS render-blocking y las 2 conexiones a terceros. Como los tres sitios (ekain, flagmaps, geojuegos) comparten origen ekain.amutxastegi.com y tipografías, alojarlos una vez (p. ej. en el repo portal bajo /fonts/) daría caché compartida entre los tres, a costa de acoplar los repos: valorar según cuánto pese la independencia.
- corrección del verificador: Dos imprecisiones: (1) la itálica se usa SOLO en `h1 em` (style.css:37-39); `.sub` no es itálica (solo uppercase con letter-spacing). (2) El CSS no usa "pesos 300-600" sino únicamente 300 y 500 (más el 400 implícito), confirmado en el CSS compilado de dist/: Plex Mono 500/600 y Fraunces 600-700 se piden y no se usan nunca, así que el recorte posible es mayor que el descrito (los ~30 KB de Plex Mono 500/600 son eliminables íntegros además de poder prescindir de la itálica u obtenerla sintética).
- **Estado:** no aplicado — descartado deliberadamente por ahora (YAGNI/regla de los 3 repos): mismo trade-off de caché compartida que los hallazgos equivalentes de ekain y flagmaps.

---

## Sin verificar (severidad baja; algunos solapan con los confirmados)

- **[geojuegos] [consistencia]** El <head> completo está duplicado en 5 HTML de 3 repos sin snippet canónico — `README.md`. Documentar el snippet canónico del <head> una vez (en el README de geojuegos, que ya tiene la checklist, con nota de que ekain y flagmaps usan el mismo) o al menos listar los 5 ficheros a tocar cuando cambien las fuentes. No compensa plantilla compartida entre repos: son sitios estáticos independientes y el snippet cambia poco. **Estado:** cubierto por docs/guia-estilo.md §1 y §6 (URL de fuentes byte-idéntica y esqueleto de <head> canónico documentados).
- **[geojuegos] [rendimiento]** Los 236 SVG de banderas (3,7 MB) están duplicados con flagmaps bajo el mismo dominio sin caché compartida — `src/data.ts:14`. Mantener la duplicación es defendible (independencia de deploys, ya automatizada con sync-data.mjs); no tocar salvo que moleste. Si se quisiera caché compartida, bastaría apuntar flagUrl/flagThumbUrl a '/flagmaps/flags/…' (2 líneas en data.ts:14-15), pero acopla geojuegos al deploy de flagmaps y rompe `vite dev` sin ese repo servido: solo hacerlo asumiendo ese acoplamiento. No añadir nada más (YAGNI). **Estado:** no aplicado — descartado deliberadamente (YAGNI/regla de los 3 repos): la duplicación es defendible por independencia de deploys, según concluye el propio hallazgo.
