# Proxy compartido

Caddy publica los puertos 80 y 443, obtiene y renueva automáticamente certificados HTTPS, y envía el tráfico a Nginx. Nginx enruta por hostname en la red Docker externa `web`; el DNS interno de Docker se resuelve periódicamente para tolerar reinicios de contenedores frontend. Los servicios de cada proyecto mantienen sus redes privadas para API y base de datos; solo los frontends se conectan también a `web`.

## Hostnames gratis con HTTPS

DuckDNS administra los nombres gratuitos y actualiza sus direcciones IPv4; Caddy emite los certificados TLS automáticamente. DuckDNS registra nombres individuales `nombre.duckdns.org`, no subdominios anidados como `inventario.plh-condominio.duckdns.org`. Para usar el mismo nombre descriptivo como prefijo, registra cada hostname por separado, por ejemplo:

- `inventario-plh-condominio.duckdns.org`
- `servicios-plh-condominio.duckdns.org`

1. Conserva `plh-condominio.duckdns.org` y registra también ambos nombres anteriores (si están disponibles) en [DuckDNS](https://www.duckdns.org/). Copia el token de la cuenta.
2. Si todavía no tienes `.env`, copia `.env.example` a `.env`. Si ya existe, no lo sobrescribas: conserva `DUCKDNS_TOKEN` y actualiza/agrega `BASE_DOMAIN=plh-condominio.duckdns.org`, `INVENTORY_DOMAIN=inventario-plh-condominio.duckdns.org`, `SERVICES_DOMAIN=servicios-plh-condominio.duckdns.org` y `DUCKDNS_SUBDOMAINS=plh-condominio,inventario-plh-condominio,servicios-plh-condominio` (reemplaza la variable singular anterior, si existe). Guarda el token de forma privada; no compartas ni subas `.env`.
3. Permite TCP 80 y 443 desde Internet en el firewall del proveedor. La regla de firewall de Terraform del proyecto de infraestructura permite esos puertos. No debe haber otro servicio ocupándolos en el servidor.
4. Crea la red Docker compartida si aún no existe, y arranca el proxy:

   ```sh
   docker network create web
   docker compose config --quiet
   docker compose up -d
   ```

   Si `web` ya existe, omite `docker network create web`.
5. Cuando DuckDNS publique ambos nombres hacia la IP del servidor, Caddy solicitará los certificados automáticamente. La primera emisión puede tardar unos minutos. Si no funciona, revisa `docker compose logs caddy duckdns` y confirma que los DNS resuelvan a la IP pública y que el puerto 80 sea accesible.
6. Reconstruye los frontends para eliminar sus antiguas subrutas: ejecuta `docker compose up -d --build` desde cada proyecto. En el `.env` de Servicios, configura `FRONTEND_ORIGIN=https://servicios-plh-condominio.duckdns.org` y `PASSWORD_RESET_BASE_URL=https://servicios-plh-condominio.duckdns.org/login/restablecer`. En Inventario, conserva `COOKIE_SECURE=true`.
7. Abre `https://inventario-plh-condominio.duckdns.org` y `https://servicios-plh-condominio.duckdns.org`.

El dominio base conserva HTTPS y sigue mostrando la respuesta 404 temporal; las aplicaciones se abren en sus hostnames dedicados. Si cambias un hostname, actualiza el `.env` del proxy y el `server_name` del archivo correspondiente en `projects/`. Los datos TLS de Caddy se conservan en volúmenes Docker.

## Proyectos configurados

- `inventario-plh-condominio.duckdns.org` → `plh-inv-frontend`
- `servicios-plh-condominio.duckdns.org` → `plh-serv-frontend`

Cada configuración Nginx enruta todas las rutas del hostname a su frontend, que entrega `/api/` a su backend privado. Las SPAs usan `baseHref: "/"` y una URL API relativa a la raíz (`/api`).

## Agregar otro proyecto

1. Registra otro hostname en DuckDNS, por ejemplo `portal-plh-condominio.duckdns.org`, y añádelo a `DUCKDNS_SUBDOMAINS` en `.env`.
2. Añade un bloque de sitio para el hostname en `Caddyfile` y expón las variables de entorno que utilice desde el servicio `caddy` en `docker-compose.yml`.
3. Conecta su servicio `frontend` a la red Docker `web` además de su red privada; no conectes PostgreSQL ni el backend a `web`.
4. Crea `projects/nombre-proyecto.conf` con este patrón:

   ```nginx
   server {
       listen 80;
       server_name portal-plh-condominio.duckdns.org;
       resolver 127.0.0.11 valid=30s ipv6=off;

       location / {
           set $app nombre-proyecto-frontend:80;
           proxy_pass http://$app;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
       }
   }
   ```

5. Configura el build de producción de la SPA con `baseHref: "/"` y la API pública como `/api`. Reconstruye la aplicación y actualiza el proxy:

   ```bash
   docker compose up -d --build
   docker compose exec nginx nginx -t
   ```

No publiques los puertos del backend o PostgreSQL. No compartas la salida de `docker compose config`, ya que puede contener el token resuelto.
