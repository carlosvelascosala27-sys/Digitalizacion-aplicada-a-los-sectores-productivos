# A1.3: Acceso y entorno de desarrollo

**Alumnos:** Carlos Velasco Sala y Anass Belghiti

**Máquina:** dev.grupo3 (VM 101, nodo Proxmox `hilario`), IP 192.168.50.11, usuario `c27`

## Nota sobre la entrega

Esta práctica no se ha podido completar en su totalidad. Al plantearse la tarea, se nos indicó que podía realizarse en local, por lo que intentamos hacerla desde casa sobre la máquina virtual de Proxmox. 
Sin embargo, la VM está en la red interna del instituto y no fue posible completar los pasos que requieren acceso desde fuera (conexión por SSH desde el PC y descarga de paquetes). 
Lo que se ha realizado está documentado a continuación, y lo pendiente aparece marcado como tal.

## 1. Par de claves (Carlos)

Clave generada en el PC con Windows:

```
ssh-keygen -t ed25519
```

Se usaron las rutas por defecto y sin passphrase. La clave pública se copió a la VM.

## 2. Clave pública en la VM

Comandos ejecutados dentro de la VM como `c27`:

```
mkdir -p ~/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKty+mJqPCzMSiGnsXejYjoaZbkQ9NlkjYL8xAd5u52Q carlos@DESKTOP-SV5O6RJ" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
```

Resultado: `authorized_keys` contiene una línea con la clave pública de Carlos.

## 3. Comprobación de acceso

Intento de conexión por SSH desde el PC:

```
ssh c27@192.168.50.11
```

Resultado: `Connection timed out`. La VM está en la red interna de Proxmox y el PC no llega a esa IP.

Comprobación desde dentro de la VM:

```
ssh c27@localhost
```

Resultado: pidió contraseña. La prueba se hizo sin clave privada en la VM, por lo que no verifica la autenticación por clave.

## 4. Clave de Anass Belghiti

Anass no puede tener una VM propia porque su equipo (Mac) no tiene memoria suficiente. Desde el Mac hizo la parte que no necesita servidor.

Clave generada en el Mac el 29/10/2026:

```
ssh-keygen -t ed25519 -C "anass@grupo3"
```

Se usaron las rutas por defecto (`~/.ssh/id_ed25519` y `~/.ssh/id_ed25519.pub`) y sin passphrase.

Huella: `SHA256:/71V1LmlCdHlOmF8S0lDNxlIvQ8WUiuyGkCMFVySBd0`

Clave pública:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICQFpLhbgn2mG0kblD6UFgfU3Nh1kx7gXUAb9eMTVFkZ anass@grupo3
```

La clave pública se envía a Carlos para añadirla a `~/.ssh/authorized_keys` de `c27` en la VM.

Pendiente:

- Añadir la clave a `authorized_keys` en la VM (Carlos) y comprobar la entrada sin contraseña. Depende de tener acceso a la red del instituto.
- Clonar el tablón y leer el README para saber qué se instala y cómo se arranca la API:

  ```
  git clone https://github.com/esrora/tablon-avisos.git
  ```

- Avisar al profesor de que Anass no puede tener una VM propia y preguntarle cómo quiere que entregue su parte.

## 5. Desactivar la contraseña en SSH

No realizado. No se desactivó `PasswordAuthentication` porque la entrada por clave no se pudo comprobar desde el PC.

## 6. Instalación, base de datos, clonado y API

Pendiente. La VM no tiene salida a internet desde casa (`ping: connect: La red es inaccesible`), por lo que no se pudo ejecutar `apt install` (postgresql, nodejs, npm, git) ni clonar el tablón.

Pasos pendientes:

```
git clone https://github.com/esrora/tablon-avisos.git
sudo -u postgres createuser tablon --pwprompt
sudo -u postgres createdb tablon --owner=tablon
sudo -u postgres psql -d tablon -f ~/tablon-avisos/db/init.sql
cd ~/tablon-avisos/api && cp .env.example .env && npm install
node --env-file=.env server.js
curl http://localhost:3000/api/version
```

## 7. Captura de /api/version

No entregada. Pendiente de completar en el instituto.
