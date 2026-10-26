**Alumnos** Carlos Velasco Sala  y Anass Belghiti

## Punto de partida


1. en el proyecto pistas de mi compañero carlos velasco sala tiene como nombre de ususario  root y  la contraseña vacia
2. una vez reiniciado el equipo habria que inicar el xampp y encender el  servidor apache y el servidor mysql de forma manual
3. al provocar un error en el codigo los registros de dicho error quedan registrados en la ruta logs de XAMPP
4. mi entorno (XAMPP) usa PHP PHP 8.2.12 (cli) (built: Oct 24 2023 21:15:15) (ZTS Visual C++ 2019 x64) y el de ANASS usa la misma versión, PHP 8.2.12 (cli) (built: Oct 24 2023 21:15:15) (ZTS Visual C++ 2019 x64)



# Entrega de Proxmox


**Práctica:** Actividad 2: Las dos máquinas (Proxmox, IPs fijas y validación por ping)

### Enlace al Repositorio de GitHub
El inventario completo de la infraestructura se encuentra documentado en el fichero `infra/maquinas.md` dentro del siguiente repositorio:

* **URL del Repositorio:** [https://github.com/carlosvelascosala27-sys/practica-proxmox](https://github.com/carlosvelascosala27-sys/practica-proxmox)

### Resumen de la Configuración
* **Máquina `dev.grupo3` (VM 101):** IP estática `192.168.50.11/24` (2 vCPU, 1 GB RAM, 15 GB Disco).
* **Máquina `pro.grupo3` (VM 102):** IP estática `192.168.50.12/24` (2 vCPU, 1 GB RAM, 15 GB Disco).
* **Verificación:** Conectividad bidireccional comprobada mediante `ping` con un 0% de pérdida de paquetes.
