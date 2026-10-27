# CartoonScrapper: diseño

Estado: borrador pendiente de revisión del usuario. No hay código todavía.

## Objetivo

Aplicación web para explorar el catálogo de lacartoons.com, descargar capítulos como archivos de video al dispositivo para verlos sin internet, y administrar los capítulos descargados.

Éxito: con internet, elegir una serie y descargar un capítulo. Sin internet, abrir "Mis descargas" y verlo desde celular o computadora.

## Lo que dijo el usuario

- Aplicación web, sin Python.
- Uso personal y de un círculo cercano (familia y amigos), no público.
- Debe servir en celular, tablet y computadora, así que no puede depender de un servidor local en una PC.
- La app no tiene por qué funcionar offline en general. Sí debe poder abrir "Mis descargas" para ver lo ya descargado.
- El sitio tiene tres pantallas: catálogo con filtro por cadena (Nickelodeon, Jetix, Disney, etc.), serie con temporadas y capítulos, y reproductor del capítulo.

## Supuestos (sin verificar)

- Los videos se pueden obtener como archivo `mp4`. Si resultan ser HLS o iframes de terceros, el diseño se ajusta (ver "Pendiente").
- La red del trabajo del usuario (FortiGate) bloquea o intercepta el sitio. El `403` y el error de certificado vistos desde ahí no prueban nada sobre el sitio.

## Enfoque elegido

Un servidor ligero hace de intermediario (scraping y proxy de video). Los videos se guardan en el almacenamiento de cada dispositivo. El servidor no almacena videos.

Descartados: guardar videos en el servidor (costo y riesgo legal) y una extensión solo cliente (CORS, no sirve en celulares).

## Componentes

1. **Módulo `fuente`**: lo único que depende de lacartoons.com.
   - `listarCadenas()`
   - `listarSeries(cadena?)`
   - `obtenerSerie(id)` devuelve temporadas y capítulos
   - `obtenerVideo(idCapitulo)` devuelve la URL real y el tipo (`mp4` o `hls`)
2. **API del servidor (Node.js)**: `/api/cadenas`, `/api/series?cadena=`, `/api/series/:id`, `/api/capitulos/:id/video`. Incluye proxy de video con soporte de `Range`. Acceso protegido con una clave compartida.
3. **Interfaz web (PWA)**: tres vistas que reflejan el sitio, más "Mis descargas".
4. **Gestor de descargas** (en el dispositivo): descarga por partes, guarda en OPFS o IndexedDB, muestra progreso, pausa y reanuda.
5. **Biblioteca** (en el dispositivo): metadatos de cada capítulo (serie, temporada, tamaño, fecha). Permite listar, reproducir, borrar y ver el espacio usado.

Stack propuesto, aún no aprobado: Node.js con Fastify o Express, `cheerio` (y Playwright solo si hace falta), React con Vite y `vite-plugin-pwa`.

## Flujo de datos

- **Navegar**: interfaz, API, `fuente`, sitio. El servidor cachea el catálogo por unas horas.
- **Descargar**: el dispositivo pide `/api/capitulos/:id/video`. El servidor obtiene la URL real en ese momento (puede caducar) y la reenvía por proxy con `Range`. El gestor escribe los trozos al almacenamiento local y guarda los bytes descargados. Al terminar registra el capítulo en la biblioteca.
- **Ver offline**: se lee el archivo local y se reproduce en un `<video>`. No necesita servidor.

## Manejo de errores

| Situación | Comportamiento |
|---|---|
| Sitio caído o HTML cambió | `fuente` lanza error propio, la API responde `502`, la UI muestra "no se pudo leer el sitio" |
| URL del video caduca a mitad de descarga | Se pide una URL nueva y se continúa desde el byte donde iba |
| Se corta la conexión | La descarga queda en pausa y se puede reanudar |
| No hay espacio | Se compara el tamaño con el espacio disponible antes de empezar y se avisa |
| Clave incorrecta | `401` y la UI pide la clave |

## Pruebas

- `fuente`: contra páginas HTML de ejemplo guardadas (fixtures).
- API: con una `fuente` falsa.
- Gestor de descargas: contra un servidor de prueba que soporte `Range`.

## Fuera de alcance

- Servicio público o multiusuario abierto.
- Almacenar o republicar videos en el servidor.
- Saltar protecciones del sitio (anti-bot, DRM, enlaces firmados). Si existen, se declara inviable y se replantea.
- Soporte HLS en la primera versión, salvo que la verificación muestre que es el único formato.

## Pendiente: verificación desde casa (primer paso)

Desde la red del trabajo no se puede inspeccionar el sitio. Hacerlo desde una red sin FortiGate:

1. `curl` a la página principal, a una página de serie y a una de capítulo. Anotar códigos de respuesta y si el HTML trae los enlaces.
2. Si el HTML no trae el video, abrir el capítulo con Playwright y registrar las peticiones de red del reproductor.
3. Responder: ¿de dónde sale el video (mp4 directo, HLS, iframe de terceros)? ¿La URL caduca? ¿Hay protecciones?
4. Anotar los patrones de URL de las tres pantallas y de los filtros por cadena.
5. Guardar HTML de ejemplo para las pruebas de `fuente`.

REA (https://github.com/morluto/rea) se evalúa solo si el reproductor está ofuscado. Su instalación modifica la configuración de Claude Code y requiere aprobación explícita.

## Decisiones abiertas

- Stack del frontend y backend (propuesto arriba).
- Hosting del servidor.
- Forma de la clave compartida.
