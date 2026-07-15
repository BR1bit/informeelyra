# Tu app en el servidor del ITSP: qué arreglar y por qué

Hola! Revisando tu proyecto (está muy bien estructurado 👏 — el router con front controller, el autoload, el `.env`, todo eso está perfecto y no hay que tocarlo). Los problemas que tenés son de **despliegue bajo subcarpeta**, no de tu arquitectura. Acá va el detalle de qué pasa, qué ya arreglamos nosotros y qué te toca arreglar a vos.

---

## Cómo llega una request a tu app (importante para entender todo lo demás)

Tu sitio público es `https://itspvm.duckdns.org/proyectos/elyra/`, pero esa URL atraviesa varias capas antes de llegar a tu PHP:

```
Navegador                    Proxy (AWS)                  Apache (servidor)
/proyectos/elyra/pacientes → quita "/proyectos" →         /elyra/pacientes
                                                          → tu .htaccess lo manda a public/
                                                          → tu index.php lo recibe
```

La consecuencia clave, que explica tus dos bugs:

> **El navegador ve tu app en `/proyectos/elyra/`, pero tu PHP la ve en `/elyra/`.**
> Son dos prefijos distintos y cada uno importa en un lugar diferente.

---

## Arreglo 1: el `.env` — por qué tu ruta raíz da 404

Tu `.env` tiene:

```
APP_URL=https://itspvm.duckdns.org/proyectos/elyra/public
```

Tu código (en `public/index.php` y en `Router.php`) hace algo muy bien pensado: descuenta el path de `APP_URL` del `REQUEST_URI` antes de matchear rutas. El problema: intenta descontar `/proyectos/elyra/public`, pero como vimos arriba, **tu PHP recibe `/elyra/...`** — ese prefijo nunca matchea, no se descuenta nada, y tu router busca una ruta llamada `/elyra/` que no existe → tu 404.

Como `APP_URL` también se usa para armar los links de los mails de reset de contraseña (que sí necesitan la URL **del navegador**), lo correcto es separar los dos conceptos:

```bash
# .env
APP_URL=https://itspvm.duckdns.org/proyectos/elyra   # URL del navegador (ojo: SIN /public — tu .htaccess ya lo oculta)
APP_BASE_PATH=/elyra                                  # prefijo que ve el servidor
```

Y en los **dos** lugares donde descontás el prefijo, priorizá la variable nueva (es cambiar una línea en cada uno):

```php
// public/index.php (~línea 104) y src/Infrastructure/Web/Router.php (~línea 64)

// ANTES:
$appUrlPath = parse_url($_ENV['APP_URL'] ?? '', PHP_URL_PATH);

// DESPUÉS:
$appUrlPath = $_ENV['APP_BASE_PATH'] ?? parse_url($_ENV['APP_URL'] ?? '', PHP_URL_PATH);
```

En tu máquina local no definís `APP_BASE_PATH` y todo sigue funcionando como hasta ahora (cae al comportamiento anterior).

---

## Arreglo 2: assets con ruta absoluta — los 404 de la consola

Los errores que ves en la consola del navegador:

```
GET https://itspvm.duckdns.org/manifest.json      404
GET https://itspvm.duckdns.org/js/elyra.js?v=6    404
GET https://itspvm.duckdns.org/js/components/ui.js?v=5  404
```

Vienen de tus vistas: `<link href="/css/web20.css">`, `<script src="/js/elyra.js">`, etc. Una ruta que empieza con `/` es **absoluta al dominio**: el navegador la resuelve como `https://itspvm.duckdns.org/css/...` — la raíz del dominio, donde vive otra aplicación — en vez de `https://itspvm.duckdns.org/proyectos/elyra/css/...`.

**La solución: un `<base>` en el `<head>` + rutas relativas.** El `<base>` le dice al navegador desde dónde resolver toda ruta relativa de la página, sin importar en qué URL estés parada (sin él, `css/x.css` visto desde `/proyectos/elyra/pacientes/5` buscaría en `/proyectos/elyra/pacientes/css/` — mal).

En el `<head>` de `views/layout/base.php` (y de las vistas standalone que tienen su propio `<html>` completo: `publico/home.php`, `errors/500.php`, `offline.php`, `publico/mis-documentos.php`):

```php
<base href="<?= rtrim((string)(parse_url($_ENV['APP_URL'] ?? '', PHP_URL_PATH) ?: ''), '/') ?>/">
```

Y después, **sacale la `/` inicial a todos los assets**:

| Antes (roto) | Después |
|---|---|
| `<link href="/css/web20.css">` | `<link href="css/web20.css">` |
| `<link rel="manifest" href="/manifest.json">` | `<link rel="manifest" href="manifest.json">` |
| `<script src="/js/elyra.js?v=6">` | `<script src="js/elyra.js?v=6">` |
| `<script src="/js/components/ui.js?v=5">` | `<script src="js/components/ui.js?v=5">` |
| `fetch('/api/...')` en tu JS | `fetch('api/...')` |

Archivos donde encontramos rutas absolutas para corregir: `views/layout/base.php` (líneas 8–9, 149–150), `views/publico/home.php`, `views/errors/500.php`, `views/offline.php`, `views/publico/mis-documentos.php`, `views/traslados/tracking.php`, `views/traslados/nuevo.php`, `views/traslados/mapa.php`. Conviene que hagas un buscar-en-proyecto de `"/css/`, `"/js/` y `"/manifest` para no dejar ninguno.

El `<base>` sale de `APP_URL`, así que en tu máquina local (con `APP_URL=http://localhost:8000`) queda `href="/"` y todo sigue igual. Ojo también con los **links de navegación** (`<a href="/pacientes">` → `<a href="pacientes">`) y las **redirecciones PHP** (`header('Location: /login')` → conviene un helper que anteponga el base path).

---

## Cómo verificar que quedó bien

1. `https://itspvm.duckdns.org/proyectos/elyra/` → tiene que cargar tu home **con estilos**.
2. Consola del navegador (F12) → pestaña Network: ningún 404, y todos los assets pidiendo URLs que empiecen con `/proyectos/elyra/`.
3. Navegá a una ruta interna y recargá la página ahí (F5) — tiene que seguir funcionando (eso prueba el `<base>` y el rewrite juntos).

Cualquier duda, avisá.
