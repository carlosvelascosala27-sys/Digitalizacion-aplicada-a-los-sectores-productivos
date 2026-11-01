# A1.4: Endurecer la máquina de producción

**Alumnos:** Carlos Velasco Sala y Anass Belghiti

**Máquina:** pro.grupo3 (VM 102, nodo Proxmox `hilario`), IP 192.168.50.12

## Entorno

La práctica se ha hecho en local, sobre un Proxmox propio instalado en VirtualBox en el PC de Carlos. `pro.grupo3` es la VM 102 de ese Proxmox, no una máquina del aula. La aplicación no se ha instalado en esta máquina.

## Preparación

Antes de cambiar la VM se hizo una instantánea desde el nodo:

```
qm snapshot 102 antes-a14 --description "Antes de A1.4"
```

Con el disco montado desde el nodo y la VM apagada, se hicieron estos cambios:

- `/etc/network/interfaces`: se añadió `gateway 192.168.50.1` a `ens18`.
- `/etc/resolv.conf`: `nameserver 1.1.1.1` y `nameserver 8.8.8.8`.
- `/etc/hostname` y `/etc/hosts`: nombre `pro.grupo3`.
- `/home/c27/.ssh/authorized_keys`: claves de Carlos y Anass.
- `/root/.ssh/authorized_keys`: clave temporal de Carlos para trabajar como root. Se borró al terminar y root ya no entra por SSH.

Paquetes instalados como root:

```
apt-get update
apt-get install -y --no-install-recommends sudo curl ufw nginx
usermod -aG sudo c27
```

## 1. Usuario de servicio

```
useradd --system --no-create-home --shell /usr/sbin/nologin tablon
getent passwd tablon
ls -d /home/tablon
passwd -S tablon
su - tablon
```

Salida:

```
tablon:x:988:988::/home/tablon:/usr/sbin/nologin
ls: no se puede acceder a '/home/tablon': No existe el fichero o el directorio
tablon L 2026-11-01 -1 -1 -1 -1
su: atención: no se puede cambiar el directorio a /home/tablon: No existe el fichero o el directorio
Actualmente esta cuenta no está disponible.
```

Resultado: no se puede iniciar sesión con `tablon`. No tiene intérprete de órdenes, su contraseña está bloqueada (`L`) y su carpeta personal no existe, aunque `/etc/passwd` muestre la ruta.

## 2. Directorio de la aplicación

```
mkdir -p /opt/tablon
chown tablon:tablon /opt/tablon
chmod 750 /opt/tablon
ls -ld /opt/tablon
```

Salida:

```
drwxr-x--- 2 tablon tablon 4096 nov  1 23:38 /opt/tablon
```

## 3. Cortafuegos

```
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
ufw status verbose
```

Salida de `ufw status verbose`:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
443/tcp (v6)               ALLOW IN    Anywhere (v6)
```

Después de activarlo, se comprobó que `c27` sigue entrando por SSH con su clave.

## 4. Nginx

```
echo '<h1>pro.grupo3 - pagina de prueba</h1>' > /var/www/html/index.html
rm -f /var/www/html/index.nginx-debian.html
systemctl enable --now nginx
systemctl is-active nginx
curl -s -m 5 http://localhost/
```

Salida:

```
active
<h1>pro.grupo3 - pagina de prueba</h1>
```

## 5. Comprobación desde dev.grupo3

Desde `dev.grupo3`, como `c27`:

```
curl -sS -m 5 http://192.168.50.12:3000
curl -sS -m 5 http://192.168.50.12/
```

Salida:

```
curl: (28) Connection timed out after 5177 milliseconds
<h1>pro.grupo3 - pagina de prueba</h1>
```

Resultado: el puerto 3000 de producción **no responde**, y el 80 sí. El timeout indica que el cortafuegos descarta la conexión, porque un puerto permitido sin servicio la rechazaría de inmediato. Además, nada escucha en el 3000, ya que la aplicación no está instalada.

El puerto 443 queda permitido en el cortafuegos, pero todavía no hay ningún servicio escuchando en él.
