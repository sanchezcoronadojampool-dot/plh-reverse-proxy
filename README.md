# Proxy compartido

Nginx publica el puerto 80 y enruta aplicaciones por subruta en la red Docker externa `web`. La raíz (`/`) devuelve 404 intencionalmente hasta que se conecte la landing. Los servicios de cada proyecto mantienen sus redes privadas para API y base de datos; solo el frontend se conecta también a `web`. El proxy usa el DNS interno de Docker con resolución periódica para tolerar que se reinicien o recreen contenedores frontend, y limita las cookies de API a la subruta correspondiente para aislar sesiones entre aplicaciones.

## Proyectos configurados

- `/proyecto1/` → `plh-inv-frontend`
- `/proyecto2/` → `plh-serv-frontend`

Cada archivo `projects/*.conf` define las rutas de frontend y API. Nginx quita el prefijo público y entrega `/` o `/api/` al Nginx interno del frontend. El `baseHref` de producción y la URL de API de cada SPA deben conservar su prefijo público para cargar recursos y efectuar llamadas correctas.

## Agregar otro proyecto

1. Conecta su servicio `frontend` a la red `web` además de su red privada; no conectes PostgreSQL ni el backend a `web`.
2. Configura el build de producción con un `baseHref` como `/proyecto3/` y la API pública como `/proyecto3/api`.
3. Crea `projects/nombre-proyecto.conf` con este patrón y cambia el nombre DNS Docker:

```nginx
location = /proyecto3 {
    return 301 /proyecto3/;
}

location /proyecto3/api/ {
    set $app nombre-proyecto-frontend:80;
    rewrite ^/proyecto3/api/(.*)$ /api/$1 break;
    proxy_pass http://$app;
    proxy_cookie_path / /proyecto3/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location /proyecto3/ {
    set $app nombre-proyecto-frontend:80;
    rewrite ^/proyecto3/(.*)$ /$1 break;
    proxy_pass http://$app;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x-forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

4. Inicia/reinicia el proxy y valida/recarga la configuración:

```bash
docker compose run --rm nginx nginx -t
docker compose up -d
docker compose exec nginx nginx -t
docker compose exec nginx nginx -s reload
```

No reutilices una ruta ya asignada. Las aplicaciones con rutas absolutas, WebSockets, OAuth callbacks o generación de enlaces deben configurarse y probarse expresamente para su subruta. Para una instalación comercial, coloca TLS/HTTPS delante del proxy antes de habilitar cookies seguras.
