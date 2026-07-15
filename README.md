# Tu app en el servidor del ITSP: estado actual y qué te queda arreglar

Buenas noticias: **tu sitio ya está online y funcionando** en
`https://itspvm.duckdns.org/proyectos/elyra/` — carga tu home real, conectada a tu base de datos. Tu proyecto está muy bien estructurado (front controller, router con soporte de subcarpeta, `.env`, autoload) y no hubo que tocar tu código: todos los problemas eran de **configuración de despliegue**. Acá va qué se arregló, por qué, y lo único que te queda a vos.

---

## El concepto clave (ahora simplificado)

Tu app vive bajo una **subcarpeta** del dominio, no en la raíz:

```
https://itspvm.duckdns.org/proyectos/elyra/
                          └────────┬─────┘
                   este prefijo existe en TODOS lados:
                   en el navegador y en tu PHP (REQUEST_URI)
```

Desde el 2026-07-15 el servidor está configurado para que tu PHP vea exactamente el mismo path que el navegador. Regla única: **tu app vive en `/proyectos/elyra/` y ninguna ruta tuya debe empezar con `/` a secas.**

---

## Lo que ya quedó arreglado (leelo — hay dos errores tuyos para no repetir)

### 1. El `.htaccess` — lo rompiste dos veces, de dos formas distintas

**Primera:** el archivo contenía el comando de terminal completo (`cat > /var/www/elyra/.htaccess << 'EOF' ...`). Eso se **ejecuta en la consola SSH**; el contenido del archivo es solo lo de adentro. Apache no pudo parsearlo → error 500 en todo tu espacio.

**Segunda:** lo reemplazaste por el `.htaccess` clásico de la documentación de frameworks:

```apache
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ public/index.php [QSA,L]
```

Ese patrón está pensado para vivir **dentro de `public/`**. En la raíz de tu carpeta rompe dos cosas: la raíz del sitio da 403 (la condición `!-d` excluye los directorios, y en tu raíz no hay `index.*`) y los assets (`css/`, `js/`) se mandan al front controller en vez de servirse como archivos.

**El que está puesto ahora — no lo cambies** — es este, que sirve todo desde `public/` sin loops:

```apache
RewriteEngine On
RewriteRule ^public/ - [L]        # lo que ya apunta a public/ no se toca (corta el loop)
RewriteRule ^(.*)$ public/$1 [L]  # todo lo demás se busca dentro de public/
```

### 2. Tu `.env` — dos valores corregidos

```bash
# ANTES                                                    # AHORA
APP_URL=https://itspvm.duckdns.org/proyectos/elyra/public  APP_URL=https://itspvm.duckdns.org/proyectos/elyra
DB_USERNAME=elyra@localhost                                DB_USERNAME=elyra
```

- **`APP_URL` sin `/public`**: el `.htaccess` ya oculta esa carpeta — públicamente no existe. Tu router descuenta el path de `APP_URL` del `REQUEST_URI` antes de matchear rutas (bien pensado eso); con `/public` en el valor, el prefijo nunca coincidía y toda URL caía en tu 404.
- **`DB_USERNAME` sin `@localhost`**: el `@localhost` es parte de cómo MySQL *define* la cuenta (`'elyra'@'localhost'`), no del nombre de usuario para conectarse. Con él incluido, MySQL buscaba un usuario llamado literalmente `elyra@localhost` → "Access denied", y tu home moría con tu error 500.

Con esos dos valores, tu código funcionó **sin modificaciones**: la detección de subcarpeta que ya habías programado hace todo el trabajo.

---

## Lo único que te queda: los assets con ruta absoluta

Abrí tu sitio con F12 → Network: vas a ver 404 en `manifest.json`, `js/elyra.js`, `js/components/ui.js`, y el sitio carga sin tus estilos. Es porque tus vistas tienen:

```html
<link href="/css/web20.css" rel="stylesheet">
<script src="/js/elyra.js?v=6" defer></script>
```

Una ruta que empieza con `/` es absoluta **al dominio**: el navegador pide `https://itspvm.duckdns.org/css/web20.css` (la raíz, donde vive la plataforma del instituto) en vez de `https://itspvm.duckdns.org/proyectos/elyra/css/web20.css`.

**La solución (2 pasos):**

**a)** En el `<head>` de tu layout (`views/layout/base.php`) y de las vistas que tienen su propio `<html>` completo (`publico/home.php`, `errors/500.php`, `offline.php`, `publico/mis-documentos.php`), agregá un `<base>` calculado desde tu `APP_URL`:

```php
<base href="<?= rtrim((string)(parse_url($_ENV['APP_URL'] ?? '', PHP_URL_PATH) ?: ''), '/') ?>/">
```

Esto le dice al navegador desde dónde resolver toda ruta relativa, estés en la URL que estés (sin esto, `css/x.css` visto desde `/proyectos/elyra/pacientes/5` se buscaría en `.../pacientes/css/`). Como sale de `APP_URL`, en tu máquina local sigue funcionando igual.

**b)** Sacá la `/` inicial de todos los assets, links y `fetch`:

| Antes (roto) | Después |
|---|---|
| `href="/css/web20.css"` | `href="css/web20.css"` |
| `href="/manifest.json"` | `href="manifest.json"` |
| `src="/js/elyra.js?v=6"` | `src="js/elyra.js?v=6"` |
| `fetch('/api/...')` | `fetch('api/...')` |
| `<a href="/pacientes">` | `<a href="pacientes">` |
| `header('Location: /login')` | usar un helper que anteponga el base path |

Archivos donde hay rutas absolutas: `views/layout/base.php` (líneas 8–9, 149–150), `views/publico/home.php`, `views/errors/500.php`, `views/offline.php`, `views/publico/mis-documentos.php`, `views/traslados/tracking.php`, `views/traslados/nuevo.php`, `views/traslados/mapa.php`. Hacé un buscar-en-proyecto de `"/css/`, `"/js/`, `"/manifest` y `'/api` para no dejar ninguno.

---

## Cómo verificar que terminaste

1. `https://itspvm.duckdns.org/proyectos/elyra/` carga **con estilos**.
2. F12 → Network: ningún 404, todas las requests empiezan con `/proyectos/elyra/`.
3. Navegá a una ruta interna y apretá F5 ahí mismo — tiene que seguir funcionando.
