# A1.3: Acceso y entorno de desarrollo

**Alumnos:** Carlos Velasco Sala y Anass Belghiti

**Máquina:** dev.grupo3 (VM 101, nodo Proxmox `hilario`), IP 192.168.50.11, usuario `c27`

## Entorno

La práctica se ha hecho en local, sobre un Proxmox propio instalado en VirtualBox en el PC de Carlos. `dev.grupo3` es la VM 101 de ese Proxmox, no una máquina del aula.

Las órdenes del paso 1 se ejecutaron desde el PC con Windows. Las de los pasos siguientes se ejecutaron en el nodo o en la VM, como se indica en cada uno.

## 1. Pares de claves

Carlos (Windows):

```
ssh-keygen -t ed25519
```

Anass (Mac):

```
ssh-keygen -t ed25519 -C "anass@grupo3"
```

Se usaron las rutas por defecto y sin passphrase.

## 2. Problemas de red encontrados y corrección

El primer intento no funcionó por tres motivos:

- El nodo usaba la puerta de enlace `10.0.2.2`, pero la `NatNetwork` de VirtualBox usa `10.0.2.1`.
- Nadie hacía de router en la red `192.168.50.0/24`, y la VM no tenía puerta de enlace.
- A la clave pública de Carlos en `authorized_keys` le faltaba un carácter, por lo que no podía funcionar.

En el nodo `hilario`, `/etc/network/interfaces` quedó así en `vmbr0`:

```
auto vmbr0
iface vmbr0 inet static
	address 10.0.2.15/24
	address 192.168.50.1/24
	gateway 10.0.2.1
	bridge-ports nic0
	bridge-stp off
	bridge-fd 0
	post-up sysctl -w net.ipv4.ip_forward=1
	post-up iptables -t nat -C POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -o vmbr0 -j MASQUERADE 2>/dev/null || iptables -t nat -A POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -o vmbr0 -j MASQUERADE
	post-up iptables -t nat -C PREROUTING -d 10.0.2.15 -p tcp --dport 2211 -j DNAT --to-destination 192.168.50.11:22 2>/dev/null || iptables -t nat -A PREROUTING -d 10.0.2.15 -p tcp --dport 2211 -j DNAT --to-destination 192.168.50.11:22
	post-up iptables -t nat -C PREROUTING -d 10.0.2.15 -p tcp --dport 3000 -j DNAT --to-destination 192.168.50.11:3000 2>/dev/null || iptables -t nat -A PREROUTING -d 10.0.2.15 -p tcp --dport 3000 -j DNAT --to-destination 192.168.50.11:3000
```

```
ifreload -a
```

En el PC, redirección de puertos en VirtualBox (solo accesibles desde el propio PC):

```
VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "pve-ssh:tcp:[127.0.0.1]:2222:[10.0.2.15]:22"
VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "dev-ssh:tcp:[127.0.0.1]:2211:[10.0.2.15]:2211"
VBoxManage natnetwork modify --netname NatNetwork --port-forward-4 "dev-api:tcp:[127.0.0.1]:3000:[10.0.2.15]:3000"
```

Antes de cambiar la VM se hizo una instantánea desde el nodo:

```
qm snapshot 101 antes-a13 --description "Antes de A1.3 local"
```

En la VM, con el disco montado desde el nodo y la VM apagada, se hicieron estos cambios:

- `/etc/network/interfaces`: se añadió `gateway 192.168.50.1` a `ens18`.
- `/etc/resolv.conf`: `nameserver 1.1.1.1` y `nameserver 8.8.8.8`.
- `/etc/hostname` y `/etc/hosts`: nombre `dev.grupo3`.
- `/home/c27/.ssh/authorized_keys`: claves correctas de Carlos y Anass.
- `/root/.ssh/authorized_keys`: clave temporal de Carlos para instalar como root. Se borró al terminar y root ya no entra por SSH.

## 3. Comprobación de las claves

Desde el PC de Carlos:

```
ssh -p 2211 c27@127.0.0.1
```

Resultado: entra sin contraseña.

Clave de Anass, en la VM:

```
ssh-keygen -lf ~/.ssh/authorized_keys
```

Resultado: su huella coincide con la generada en el Mac (`SHA256:/71V1LmlCdHlOmF8S0lDNxlIvQ8WUiuyGkCMFVySBd0`).

**Pendiente:** Anass no ha probado la entrada desde su Mac, porque el acceso solo está abierto desde el PC de Carlos. El grupo tiene dos miembros, así que hay dos claves.

## 4. Desactivar la contraseña en SSH

En la VM, con una sesión de reserva abierta:

```
cat > /etc/ssh/sshd_config.d/00-sin-password.conf <<'C'
PasswordAuthentication no
KbdInteractiveAuthentication no
C
sshd -t && systemctl reload ssh
```

Comprobación desde el PC:

```
ssh -p 2211 c27@127.0.0.1
ssh -p 2211 -o PubkeyAuthentication=no -o PreferredAuthentications=password c27@127.0.0.1
```

Resultado: la primera entra con clave y la segunda devuelve `Permission denied (publickey)`.

## 5. Instalación

En la VM, como root:

```
apt-get update
apt-get install -y --no-install-recommends sudo git curl ca-certificates postgresql nodejs npm
usermod -aG sudo c27
```

Versiones instaladas: Node 20.19.2, npm 9.2.0 y PostgreSQL 17.11.

## 6. Clonado y base de datos

Como `c27`:

```
cd ~
git clone https://github.com/esrora/tablon-avisos.git
```

Como root:

```
cd /tmp
sudo -u postgres psql -v ON_ERROR_STOP=1 -c "CREATE ROLE tablon LOGIN PASSWORD 'tablon';"
sudo -u postgres createdb tablon --owner=tablon
```

Como `c27`:

```
cd ~/tablon-avisos
PGPASSWORD=tablon psql -v ON_ERROR_STOP=1 -h localhost -U tablon -d tablon -f db/init.sql
```

Nota: el primer intento con `sudo -u postgres psql -f` no creó las tablas, porque `postgres` no puede leer dentro de `/home/c27`. La contraseña `tablon` es la de `.env.example`, válida solo para este entorno de desarrollo.

Resultado: tabla `avisos` creada con 3 avisos de ejemplo.

## 7. Arranque de la API

Como `c27`:

```
cd ~/tablon-avisos/api
cp .env.example .env
npm install
nohup node --env-file=.env server.js > ~/api.log 2>&1 &
```

Comprobación desde el navegador del PC:

- `http://localhost:3000/api/version`: `{"version":"0.0.0-dev","commit":"sin-commit","entorno":"desarrollo","minClientVersion":"0.0.0"}`
- `http://localhost:3000/api/avisos`: devuelve los 3 avisos de ejemplo.

## 8. Captura de /api/version

![/api/version respondiendo desde dev.grupo3](api-version.png)
