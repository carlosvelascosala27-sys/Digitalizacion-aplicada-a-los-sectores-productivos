# Inventario de Infraestructura

Máquinas del grupo en el Proxmox del instituto (nodo `proxmox`, pool `pool-grupo3`). Las dos son clones de la plantilla 118 (`template-debian`, Debian 13).

| VM | Nombre | IP actual (DHCP) | IP fija | Recursos | Para qué sirve |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 119 | **dev.grupo3** | 192.168.205.114/20 | Pendiente de confirmar por el profesor (prevista 192.168.50.11) | 1 vCPU, 2 GB RAM | Entorno de desarrollo para pruebas y validación de cambios. |
| 120 | **pro.grupo3** | 192.168.205.115/20 | Pendiente de confirmar por el profesor (prevista 192.168.50.12) | 1 vCPU, 2 GB RAM | Entorno de producción principal y estable. |

Puerta de enlace actual, recibida por DHCP: `192.168.200.1`.

## Notas

- Las IP actuales las asigna el DHCP del aula y pueden cambiar. Las IP fijas quedan pendientes hasta que el profesor confirme cuáles hay que usar y su puerta de enlace.
- Al ser clones, las dos VM tenían las mismas claves de servidor SSH. En `pro.grupo3` se regeneraron:

  ```
  sudo rm -f /etc/ssh/ssh_host_*
  sudo dpkg-reconfigure openssh-server
  sudo systemctl restart ssh
  ```

- Hasta el 1 de noviembre se trabajó con unas VM 101 y 102 en un Proxmox local (VirtualBox), con las IP 192.168.50.11 y 192.168.50.12.
