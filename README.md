# Agenda en vivo

Agente que revisa periódicamente las agendas de venues, productoras y ticketeras, arma un registro de recitales sin duplicados, detecta cambios (nuevo, agotado, cancelado, cambio de precio u hora) y lo muestra en un tablero web con descarga a planilla.

Corre entero en Cloudflare: un Worker (el agente y el tablero), una base D1 y Workers AI.

## Cómo está armado

| Archivo | Qué hace |
|---|---|
| `src/sources.js` | La lista de páginas a vigilar. Es lo que más vas a editar. |
| `src/parsers.js` | Lectores dedicados para Livepass, Enigma, TuEntrada y Venti: leen el formato de cada sitio sin usar IA. |
| `src/extract.js` | Prepara las páginas. Para fuentes sin lector dedicado usa los datos estructurados (schema.org) o la IA. |
| `src/index.js` | El agente: rota entre fuentes, deduplica, detecta cambios y expone la API. |
| `public/index.html` | El tablero. |
| `schema.sql` | Las tablas de la base. |
| `wrangler.toml` | La configuración para Cloudflare. |

Cada 30 minutos el agente toma las fuentes que hace más tiempo que no revisa (2 por corrida, configurable). Si una página no cambió desde la última vez, no vuelve a gastar IA.

## Instalación (sin terminal)

Los nombres de los botones del panel de Cloudflare pueden variar un poco, pero el camino es este.

### 1. Subir el proyecto a GitHub
1. En GitHub, creá un repositorio nuevo (puede ser privado), por ejemplo `agenda-recitales`.
2. Elegí **Add file → Upload files** y arrastrá el contenido de esta carpeta (las carpetas `src` y `public` y los archivos sueltos). Confirmá con **Commit changes**.

### 2. Crear la base de datos
1. En el panel de Cloudflare andá a **Storage & Databases → D1** y creá una base llamada `agenda-recitales`.
2. Copiá el **Database ID** que te muestra.
3. Entrá a la pestaña **Console** de la base, pegá todo el contenido de `schema.sql` y ejecutalo. Si da error por ejecutar varias sentencias juntas, pegá cada `CREATE` por separado.
4. En GitHub, abrí `wrangler.toml`, tocá el lápiz para editarlo y reemplazá `REEMPLAZAR_CON_EL_ID_DE_TU_BASE` por el ID que copiaste. Guardá.

### 3. Conectar GitHub con Cloudflare
1. En Cloudflare andá a **Workers & Pages → Create** y elegí la opción de **importar un repositorio** de GitHub.
2. Autorizá el acceso y elegí `agenda-recitales`. Dejá la configuración que detecta por defecto y desplegá.
3. Al terminar vas a tener una dirección tipo `https://agenda-recitales.TU-USUARIO.workers.dev`. Esa es tu tablero.

Desde ahora, cada vez que edites un archivo en GitHub, Cloudflare vuelve a publicar solo.

### 4. Configurar la clave para pruebas manuales
En el Worker, **Settings → Variables and Secrets**, agregá un secreto llamado `ADMIN_KEY` con una clave que inventes.

Con eso podés forzar una revisión sin esperar al cron:

```
https://agenda-recitales.TU-USUARIO.workers.dev/api/run?key=TU_CLAVE
https://agenda-recitales.TU-USUARIO.workers.dev/api/run?key=TU_CLAVE&fuente=id-de-la-fuente
```

La respuesta te dice cuántos eventos encontró en cada fuente, cuántos son nuevos y si hubo errores.

### 5. Cargar tus fuentes
Editá `src/sources.js` en GitHub y reemplazá los ejemplos por las páginas reales. Conviene usar la URL de la sección de agenda o "próximos shows", no la página de inicio. Después probá cada fuente nueva con `/api/run?...&fuente=su-id`.

## Correrlo en tu computadora

Necesitás Node.js 20 o más nuevo (nodejs.org). Después, en la carpeta del proyecto:

```
npm install
cp .dev.vars.example .dev.vars
npm run db:init
npm run db:seed
npm run dev
```

(En Windows, en lugar de `cp` usá `copy .dev.vars.example .dev.vars`.)

El tablero queda en `http://localhost:8787`. Con otra terminal podés forzar revisiones:

```
curl "http://localhost:8787/api/run?key=clave-local&fuente=livepass"
curl "http://localhost:8787/cdn-cgi/handler/scheduled?cron=*/30+*+*+*+*"
```

La primera prueba una fuente puntual; la segunda simula el cron. La base local vive en la carpeta `.wrangler` y no toca la de Cloudflare.

Diferencias con la versión publicada: el cron no se dispara solo (lo activás vos), los límites de CPU del plan gratuito no se aplican igual, y la IA de Cloudflare (que hoy solo usa Alpogo) se consulta en tu cuenta real, así que te va a pedir `npx wrangler login`.

## Actualizar una instalación anterior
Si ya tenías el proyecto andando, volvé a ejecutar `schema.sql` en la consola de D1: agrega la tabla `seen_urls` sin tocar tus datos. Opcionalmente, ejecutá también `seed.sql` para cargar los eventos de la prueba inicial.

## Cómo lee cada ticketera

| Ticketera | Método | Notas |
|---|---|---|
| Livepass | Lector dedicado | Toda la agenda en la portada. |
| Enigma | Lector dedicado | Portada completa. No publica precios sin iniciar sesión. |
| TuEntrada | Lector dedicado con paginación | 6 eventos por página. Ajustá `maxPages` según tu plan. |
| Alpogo | IA | Muestra sobre todo los eventos del día. Pendiente: lector dedicado. |
| Venti | Sitemap (a configurar) | Ver abajo. |
| Ticketek | Pendiente | Ver abajo. |
| AllAccess, Passline | Excluidas | Bloquean o prohíben el acceso automatizado. |

El agente revisa el `robots.txt` de cada sitio antes de leer una página, y si el sitio prohíbe esa ruta no la consulta. En el tablero vas a ver el motivo en la sección Fuentes.

**Paginación y plan gratuito.** Cada página adicional suma tiempo de procesamiento. Con el plan gratuito, TuEntrada queda en 3 páginas (18 eventos, los más próximos). Con el plan pago podés subir `maxPages` a 30 o más y cubrir toda su agenda de música.

### Venti: activar el modo sitemap
La portada de Venti está bloqueada para bots, pero cada página de evento se puede leer y trae nombre, fecha y lugar. Para descubrir los eventos nuevos hace falta su sitemap:

1. Abrí `https://venti.live/robots.txt` en el navegador.
2. Buscá una línea que empiece con `Sitemap:` y copiá esa dirección.
3. Abrila: si ves una lista de direcciones que contienen `/evento/`, sirve. Si es un índice de otros sitemaps, mirá cuál tiene los eventos y usá su nombre como `childMatch`.
4. En `src/sources.js`, reemplazá la URL de la fuente `venti` por esa dirección.
5. Revisá también que el robots.txt no prohíba `/evento/`. Si lo prohíbe, el agente no va a entrar (y está bien que así sea).

El modo sitemap detecta eventos nuevos, pero no cambios posteriores como un agotado.

### Ticketek: cómo buscar su fuente de datos
Ticketek Argentina está hecho con una tecnología vieja (AngularJS) que declara soporte para un esquema antiguo de "versiones para buscadores". Hay dos caminos para probar, los dos desde tu navegador:

- **Versión para buscadores.** Abrí `https://www.ticketek.com.ar/?_escaped_fragment_=` y mirá el código fuente (clic derecho, "Ver código fuente"). Si aparecen nombres de shows, esa versión se puede leer con un lector dedicado.
- **Datos internos.** Abrí la portada con las herramientas de desarrollador (F12), pestaña Red, filtro "Fetch/XHR", y recargá. Buscá pedidos cuya respuesta sea una lista de eventos en JSON y copiá su dirección.

Con cualquiera de las dos cosas se puede armar el lector.

## Usar la planilla en Google Sheets
En una celda de Google Sheets escribí:

```
=IMPORTDATA("https://agenda-recitales.TU-USUARIO.workers.dev/api/events.csv")
```

La planilla se actualiza sola cada tanto. También podés descargar el CSV desde el botón del tablero.

## Cosas a tener en cuenta

**El tablero es público por defecto.** Cualquiera con la dirección puede verlo. Si querés ponerle login, usá Cloudflare Access (en Zero Trust, gratis para pocos usuarios) sobre la dirección del Worker. La clave `ADMIN_KEY` solo protege la ejecución manual.

**Plan gratuito.** Cada ejecución tiene un límite muy chico de tiempo de procesamiento. Esperar a que respondan las páginas o la IA no cuenta, pero limpiar páginas muy pesadas sí. Si en **Metrics** del Worker ves errores de CPU excedida, bajá `FUENTES_POR_CORRIDA` a `1` o pasate al plan pago (5 USD por mes).

**Páginas que se arman con JavaScript.** Algunos sitios cargan la agenda después de abrir la página, y el agente ve una página vacía. En el tablero aparecen con el error "no tiene texto legible". Opciones: buscar si el sitio tiene una versión de la agenda en otra URL, un sitemap o el pedido de datos que hace la página (herramientas de desarrollador del navegador, pestaña Red).

**Cuando un lector dedicado deja de funcionar.** Si una ticketera cambia el diseño, su lector devuelve 0 eventos y el tablero lo marca en Fuentes. Hay que ajustar la expresión regular en `src/parsers.js`; cada lector tiene un comentario con el formato que espera.

**Fuentes que "se rompen".** Si una fuente que antes traía eventos de golpe devuelve cero, el tablero la marca con un aviso: casi siempre significa que el sitio cambió de diseño.

**Instagram no está soportado.** Bloquea este tipo de lectura automática y va contra sus términos. Para venues que solo anuncian ahí, lo más práctico es buscar si también publican en alguna ticketera.

**Usar Claude en vez de Workers AI (opcional).** Si agregás un secreto `ANTHROPIC_API_KEY`, la extracción pasa a usar Claude Haiku, que suele ser más preciso con páginas desordenadas. Tiene costo por uso según la API de Anthropic.

**Sé respetuoso con los sitios.** El agente consulta cada página como mucho unas pocas veces por hora. No bajes mucho el intervalo del cron y revisá los términos de uso de cada sitio.
