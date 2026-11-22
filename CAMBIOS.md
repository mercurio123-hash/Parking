# Parking Julius Pro — Cambios para funcionar offline

Versión corregida: 19 de noviembre de 2026.
Repo original: https://github.com/mercurio123-hash/Encuentra-mi-auto

## El problema que se arregló

La app funcionaba en casa con wifi, pero en la calle con datos móviles no
cargaba y daba error de conexión. Además, la alarma de 5 minutos por
notificación nunca se programaba.

Causas encontradas:

1. Leaflet (el mapa) se cargaba desde `https://unpkg.com/leaflet@1.9.4/dist/`.
   El CSS estaba en el `<head>` y era bloqueante: si unpkg no respondía
   con datos, la página se retrasaba y el mapa fallaba.
2. El archivo `sw.js` existía, pero `index.html` **nunca lo registraba**:
   no había `navigator.serviceWorker.register`, ni permiso de
   notificaciones, ni `postMessage('PROGRAMAR_ALERTAS')`, ni
   `<link rel="manifest">`. Las notificaciones de 5 minutos eran código
   muerto. La alarma real era solo el `alert()` y abrir el reloj del sistema.
3. `sw.js` no guardaba nada en caché: no tenía `caches.open`, ni
   `cache.addAll`, ni listener `fetch`. No había forma de abrir la app
   sin internet.

## Qué cambió

### 1. Leaflet ahora es local (`leaflet/`)
- Se descargó Leaflet 1.9.4 y se guardó en el proyecto:
  - `leaflet/leaflet.css` (14.8 KB)
  - `leaflet/leaflet.js` (147.5 KB)
  - `leaflet/images/marker-icon.png`, `marker-icon-2x.png`,
    `marker-shadow.png`, `layers.png`, `layers-2x.png`
- `index.html` ya no referencia `unpkg.com` ni ningún otro CDN para
  CSS/JS/fuentes. Las únicas URLs externas que quedan son las de las
  teselas del mapa (OpenStreetMap, solo cuando se muestra el mapa) y el
  enlace a Google Maps (solo al pulsar "REGRESAR AL AUTO").

### 2. `manifest.json` — ahora sí enlazado
- Nombre: **Parking Julius Pro** (antes solo "Control de Parquímetro"),
  nombre corto: **Parquímetro**, `start_url: ./index.html`,
  `display: standalone`, `theme_color: #007bff`, icono
  `icon-parking.png` (512×512, verificado).
- `index.html` ahora incluye `<link rel="manifest" href="manifest.json">`
  y `<meta name="theme-color" content="#007bff">`, así Chrome en Android
  ofrece instalarla y añadirla a la pantalla de inicio.

### 3. `sw.js` — caché offline real
- En `install`: `cache.addAll` con `./`, `index.html`, `manifest.json`,
  `icon-parking.png` y todos los archivos de Leaflet (11 entradas).
  Después, `self.skipWaiting()`.
- En `activate`: borra cachés viejos que no sean `parking-julius-v1`
  y toma el control con `clients.claim()`.
- En `fetch`: estrategia **cache-first con fallback a red**. Si el
  archivo está en caché se sirve de ahí aunque no haya datos; si no
  está, se pide a la red y se guarda (solo mismo origen); si no hay red
  ni caché y es una navegación, se sirve `index.html` guardado.
- Se mantiene el listener `PROGRAMAR_ALERTAS` con las dos
  notificaciones originales, mismos textos y vibraciones:
  - "⚠️ QUEDAN 5 MINUTOS" a `msTotales - 5 min` (solo si > 0).
  - "🚨 ¡TIEMPO AGOTADO!" al vencerse el tiempo.
- Nuevo: si se programa una alarma nueva, se cancelan los
  temporizadores anteriores, y hay un mensaje `LIMPIAR_ALERTAS` que
  cancela las alarmas al pulsar "LIMPIAR PARQUÍMETRO".
- Nuevo: al tocar una notificación se abre (o se enfoca) la app.

### 4. `index.html` — registro y notificaciones
- Al cargar la página, registra el service worker con
  `navigator.serviceWorker.register('./sw.js')`. Si el registro falla,
  la app sigue funcionando con el respaldo de siempre.
- Al pulsar **"📍 GUARDAR Y CALCULAR ALARMA"**, pide permiso de
  notificaciones (solo en ese momento, no al abrir la app) y envía
  `PROGRAMAR_ALERTAS` con `msTotales` al service worker.
- **Se conservan como respaldo** el `alert()` con la hora de
  vencimiento y la de la alarma (5 min antes) y la apertura del reloj
  del sistema (`intent://com.android.deskclock/...`).

### 5. El mapa ya no rompe la app
- `mostrarMapa()` ahora comprueba `typeof L === 'undefined'` y está
  dentro de un `try/catch`. Si Leaflet no cargó o no hay internet para
  las teselas, en lugar de fallar muestra en texto las coordenadas
  guardadas (`📍 Lat: ..., Lng: ... — Mapa no disponible sin internet,
  pero tu ubicación está guardada.`). El temporizador, el guardado en
  `localStorage` y la alarma siguen funcionando.
- Nuevo: si el GPS falla o está apagado, se avisa con un mensaje en
  lugar de no hacer nada.

### Lo que NO cambió
- Misma interfaz, mismos textos en español, mismos botones y mismo
  flujo: minutos, GPS, guardar ubicación en `localStorage`
  (`p_fin`, `p_lat`, `p_lng`), contador, limpiar, y regresar al auto.

## Qué sigue necesitando internet

- **Las teselas del mapa** (las imágenes de OpenStreetMap): sin datos,
  el mapa se verá vacío o con la última zona guardada, pero las
  coordenadas se muestran en texto y el temporizador y la alarma
  funcionan.
- **El botón "🧭 REGRESAR AL AUTO"**: abre Google Maps en modo peatón,
  necesita internet.
- **La primera carga**: la primera vez hay que abrirla con wifi para
  que el service worker guarde todo en el celular. Después ya no hace
  falta internet ni GitHub.

## Cómo subirlo a GitHub

> No se hizo ningún push. Estos archivos están solo en la copia local.
> Hay que subirlos a mano a la raíz del repositorio.

1. En GitHub, entra al repo `mercurio123-hash/Encuentra-mi-auto`,
   rama `main`.
2. Sube / reemplaza en la **raíz** del repo estos archivos:
   - `index.html` (modificado)
   - `sw.js` (reescrito)
   - `manifest.json` (modificado)
   - `icon-parking.png` (sin cambios)
   - Carpeta completa `leaflet/` con `leaflet.css`, `leaflet.js` y
     la subcarpeta `images/` (5 imágenes).
   - Opcional: este `CAMBIOS.md` (documentación, no lo usa la app).
3. Espera 1–2 minutos a que GitHub Pages publique la nueva versión.
   Si la página vieja sigue apareciendo, en Chrome pulsa los 3 puntos →
   "Actualizar" con la página abierta, o borra los datos del sitio
   una vez.
4. Comprueba que en `https://mercurio123-hash.github.io/Encuentra-mi-auto/`
   se ven los archivos nuevos (por ejemplo, abre
   `.../Encuentra-mi-auto/sw.js` y debe decir "caché offline real").

## Cómo probarlo en el celular

1. Con wifi, abre `https://mercurio123-hash.github.io/Encuentra-mi-auto/`
   en Chrome (Android).
2. Si Chrome lo ofrece, toca **"Instalar"**; si no, menú (⋮) →
   **"Añadir a la pantalla de inicio"** / **"Instalar app"**.
   Debe aparecer el icono de Parking Julius en tu inicio.
3. Abre la app desde ese icono (no desde el navegador) y guarda un
   parquímetro de prueba con pocos minutos, por ejemplo 10 minutos.
   Acepta el permiso de notificaciones cuando te lo pida.
   Verás el aviso de la hora de la alarma; también te abrirá el reloj
   del sistema como respaldo (eso es normal).
4. Activa el **modo avión** (o apaga wifi y datos).
5. Cierra la app por completo y vuelve a abrirla desde el icono:
   **debe abrir sin internet**, con el temporizador corriendo.
6. Si el mapa sale vacío en modo avión, es lo esperado: abajo del mapa
   deben aparecer las coordenadas en texto.
7. A los 5 minutos del vencimiento debe llegar la notificación
   "⚠️ QUEDAN 5 MINUTOS" (en Android, revisa que las notificaciones
   de Chrome / de la app instalada estén permitidas). Al vencerse,
   "🚨 ¡TIEMPO AGOTADO!".
8. Pulsa "LIMPIAR PARQUÍMETRO": las notificaciones pendientes se
   cancelan.

## Pruebas hechas (19 nov 2026)

- Servidor HTTP local con la carpeta corregida: `index.html`, `sw.js`,
  `manifest.json`, `icon-parking.png` y todos los archivos de Leaflet
  responden **200**.
- `grep`: **ninguna** referencia a `unpkg.com` ni a ningún CDN en
  `index.html`, `sw.js` ni `manifest.json`.
- `manifest.json`: JSON válido; `sw.js` y `leaflet.js`: sintaxis JS
  válida (`node --check`).
- La lista de `cache.addAll` del SW coincide archivo por archivo con
  lo que hay en disco (11 entradas, todas existen).
- Verificado en el código: registro del SW, envío de
  `PROGRAMAR_ALERTAS`, permiso de notificaciones al guardar, y
  `typeof L === 'undefined'` con texto de coordenadas.
- **No se pudo hacer la prueba real de modo avión en un navegador**
  (no había navegador en vivo disponible en esta revisión): la prueba
  del paso 4–7 de arriba hay que hacerla en el celular después de
  subir los archivos a GitHub. Si en modo avión no abre, lo primero
  es revisar que el SW se registró (Chrome → ⋮ → Información del
  sitio → Permisos) y que la primera carga fue con wifi.

## Archivos entregados

```
parking-julius/
├── index.html          (modificado: Leaflet local, SW, notificaciones, mapa seguro)
├── sw.js               (reescrito: caché offline + alarmas)
├── manifest.json       (modificado: nombre Parking Julius Pro)
├── icon-parking.png    (sin cambios, 512×512)
├── CAMBIOS.md          (este archivo)
├── README.md           (original, sin cambios)
└── leaflet/
    ├── leaflet.css     (Leaflet 1.9.4, local)
    ├── leaflet.js      (Leaflet 1.9.4, local)
    └── images/
        ├── marker-icon.png
        ├── marker-icon-2x.png
        ├── marker-shadow.png
        ├── layers.png
        └── layers-2x.png
```
