# A1.5: Poner el tablón en producción

**Alumnos:** Carlos Velasco Sala y Anass Belghiti

**Máquinas:** dev.grupo3 (VM 101, 192.168.50.11) y pro.grupo3 (VM 102, 192.168.50.12), nodo Proxmox `hilario`

**Dominio:** `pro.grupo3`

## Entorno

La práctica se ha hecho en local, sobre un Proxmox propio instalado en VirtualBox en el PC de Carlos. En el PC, `pro.grupo3` se resuelve con esta línea en `C:\Windows\System32\drivers\etc\hosts`:

```
127.0.0.1 pro.grupo3
```

El puerto 80 del PC llega a `pro.grupo3` en dos saltos. Primero, una regla de VirtualBox:

```
VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "pro-http:tcp:[127.0.0.1]:80:[10.0.2.15]:80"
```

Después, una regla en `vmbr0` del nodo (`/etc/network/interfaces`):

```
post-up iptables -t nat -C PREROUTING -d 10.0.2.15 -p tcp --dport 80 -j DNAT --to-destination 192.168.50.12:80 2>/dev/null || iptables -t nat -A PREROUTING -d 10.0.2.15 -p tcp --dport 80 -j DNAT --to-destination 192.168.50.12:80
```

Las VM se ejecutan sin KVM, emuladas. Para evitar un *kernel panic* al arrancar (`IO-APIC + timer doesn't work`), se añadió `no_timer_check` a `GRUB_CMDLINE_LINUX_DEFAULT` en `/etc/default/grub` de `pro.grupo3`.

## 1. Preparar pro.grupo3

Como root en `pro.grupo3`:

```
apt-get update
apt-get install -y --no-install-recommends postgresql nodejs
```

Versiones: Node 20.19.2 y PostgreSQL 17.11.

`init.sql` no viaja en el artefacto, así que se copió aparte desde `dev.grupo3` a `/tmp/init.sql` de `pro.grupo3`. La contraseña de la base de datos es aleatoria y solo está en el `.env` de producción:

```
cd /tmp
DBPASS=$(head -c 18 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 20)
sudo -u postgres psql -v ON_ERROR_STOP=1 -c "CREATE ROLE tablon LOGIN PASSWORD '$DBPASS';"
sudo -u postgres createdb tablon --owner=tablon
PGPASSWORD=$DBPASS psql -v ON_ERROR_STOP=1 -h localhost -U tablon -d tablon -f /tmp/init.sql
rm /tmp/init.sql
```

Resultado: tabla `avisos` con 3 avisos de ejemplo.

## 2. Construir el artefacto en dev.grupo3

Como `c27` en `dev.grupo3`:

```
cd ~/tablon-avisos
VERSION=$(node -p "require('./api/package.json').version")
rm -rf ~/build && mkdir -p ~/build/tablon/api
cp api/server.js api/db.js api/package.json api/package-lock.json api/.env.example ~/build/tablon/api/
cp -r web ~/build/tablon/web
sed -i "s#API_URL: 'http://localhost:3000'#API_URL: 'http://pro.grupo3'#" ~/build/tablon/web/config.js
cd ~/build/tablon/api && npm ci --omit=dev
cd ~/build && tar -czf tablon-$VERSION.tgz -C tablon api web
sha256sum tablon-$VERSION.tgz
```

Resultado: `tablon-1.0.0.tgz` (830 KB), con SHA256 `c7a77b86b676ea99849f31124e929f2dc3ce354ec456de061c15b8a006861185`. Contiene `api/`, con sus dependencias de producción, y `web/`. No incluye `.env` ni `init.sql`. El `.tgz` no se sube al repositorio.

## 3. Copiar y desplegar en /opt/tablon

El artefacto se copió a `/tmp` de `pro.grupo3` con `scp`, a través del nodo, y se comprobó que su SHA256 coincidía. Como root en `pro.grupo3`:

```
tar -xzf /tmp/tablon-1.0.0.tgz -C /opt/tablon
cat > /opt/tablon/api/.env <<ENV
PORT=3000
ENTORNO=produccion
DB_HOST=localhost
DB_PORT=5432
DB_NAME=tablon
DB_USER=tablon
DB_PASSWORD=$DBPASS
APP_VERSION=1.0.0
GIT_COMMIT=930b528
MIN_CLIENT_VERSION=0.0.0
ENV
chown -R tablon:tablon /opt/tablon
chmod -R g+rX,o-rwx /opt/tablon
chmod 600 /opt/tablon/api/.env
usermod -aG tablon www-data
```

`www-data` entra en el grupo `tablon` para que Nginx pueda leer `web/`. El `.env` queda con permisos `600`, así que solo lo lee `tablon`.

## 4. Unidad de systemd

Fichero `infra/tablon-api.service`, instalado en `/etc/systemd/system/tablon-api.service`:

```
install -m 644 tablon-api.service /etc/systemd/system/tablon-api.service
systemctl daemon-reload
systemctl enable --now tablon-api.service
systemctl is-enabled tablon-api
```

Resultado: `enabled` y `active`.

## 5. Nginx

Fichero `infra/tablon.nginx.conf`, instalado en `/etc/nginx/sites-available/tablon`. Sirve `/opt/tablon/web` y redirige `/api/` a `http://127.0.0.1:3000`:

```
install -m 644 tablon.nginx.conf /etc/nginx/sites-available/tablon
ln -sf /etc/nginx/sites-available/tablon /etc/nginx/sites-enabled/tablon
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl restart nginx
```

El cortafuegos de A1.4 no cambia: solo están abiertos el 22, el 80 y el 443. El 3000 sigue cerrado desde fuera y la API solo se alcanza a través de Nginx.

## 6. config.js

Durante la construcción del artefacto (paso 2), `web/config.js` se ajustó a:

```
const CONFIG = {
  API_URL: 'http://pro.grupo3',
};
```

## 7. Reinicio completo

```
systemctl reboot
```

Tras el reinicio, sin tocar nada:

```
for s in postgresql nginx tablon-api ufw; do echo "$s: $(systemctl is-enabled $s) $(systemctl is-active $s)"; done
journalctl -u tablon-api -b --no-pager -n 4 -o cat
```

```
postgresql: enabled active
nginx: enabled active
tablon-api: enabled active
ufw: enabled active
Started tablon-api.service - API del tablon de avisos.
Base de datos disponible
API del tablon escuchando en el puerto 3000
Version 1.0.0 (930b528) en entorno produccion
```

Resultado: todo vuelve solo después del reinicio.

## 8. Comprobación

Desde el navegador del PC:

- `http://pro.grupo3/api/version`: `{"version":"1.0.0","commit":"930b528","entorno":"produccion","minClientVersion":"0.0.0"}`
- `http://pro.grupo3/`: el tablón muestra los 3 avisos y la versión abajo.

![Tablón en pro.grupo3](captura-tablon.png)
