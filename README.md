# Laboratorio Académico con Ansible y AWX

Este proyecto automatiza un laboratorio con un **Gateway Debian** y **PCs Ubuntu** en la subred `10.10.10.0/29`, incluyendo NAT, DHCP, DNS, red estática en clientes, seguridad básica, tareas programadas y almacenamiento/particionado.

## Playbook general
- **`site.yml`**: Playbook principal que ejecuta toda la infraestructura:
  - Play 1: "Configurar Gateway Debian" (roles: `security`, `debian_router`, `storage`).
  - Play 2: "Configurar PCs Ubuntu" (roles: `security`, `ubuntu_client`, `ufw`, `proc_services`, `users`, `automation`).

Ejecutar todo:
```bash
ansible-playbook -i inventories/hosts.ini site.yml
```

## Inventario y variables
- **`inventories/hosts.ini`**: Define grupos `debian_gateway` (10.10.10.1) y `ubuntu_pcs` (10.10.10.2–.4).
- **`group_vars/all.yml`**: Variables globales de ejemplo.
- **`group_vars/debian_gateway.yml`**: Interfaces, DHCP/DNS y particionado para el gateway.
- **`group_vars/ubuntu_pcs.yml`**: Interfaz, gateway, DNS y UFW para clientes.
- **`host_vars/ubuntu-1.yml`**, **`ubuntu-2.yml`**, **`ubuntu-3.yml`**: IP estática por equipo.

## Roles y propósito (títulos en español)
- **`roles/debian_router/` – Router Debian (NAT, DHCP, DNS)**
  - `tasks/main.yml`: Activa IP forwarding, despliega `nftables` (NAT) y `dnsmasq`.
  - `templates/nftables.conf.j2`: Reglas de firewall/NAT.
  - `templates/dnsmasq.conf.j2`: Configuración DHCP/DNS.
  - `handlers/main.yml`: Reinicios/recargas de servicios.
- **`roles/ubuntu_client/` – Clientes Ubuntu (Netplan estático)**
  - `tasks/main.yml`: Instala/aplica Netplan.
  - `templates/netplan.yaml.j2`: Plantilla de red estática.
  - `handlers/main.yml`: `netplan apply`.
- **`roles/security/` – Seguridad (sysctl y permisos críticos)**
  - `defaults/main.yml`: Parámetros `sysctl` y archivos críticos.
  - `tasks/main.yml`: Persiste `sysctl` y fija permisos.
  - `handlers/main.yml`: `sysctl --system`.
- **`roles/ufw/` – Firewall sencillo en clientes**
  - `defaults/main.yml`: Habilitación, políticas y reglas.
  - `tasks/main.yml`: Configura UFW (requiere `community.general`).
- **`roles/proc_services/` – Procesos y servicios (utilidades y estado)**
  - `tasks/main.yml`: Paquetes, servicios, diagnóstico `ps`.
  - `handlers/main.yml`: Handlers.
- **`roles/users/` – Usuarios, grupos y sudoers**
  - `tasks/main.yml`: Cuentas, grupos, claves SSH, sudoers.
  - `templates/sudoers.j2`: Regla `sudo` administrada.
- **`roles/automation/` – Automatización (cron, updates, backups)**
  - `defaults/main.yml`: Schedules para updates/cleanup/backup.
  - `tasks/main.yml`: Cron jobs y script de backup.
  - `templates/backup.sh.j2`: Script de copias.
- **`roles/storage/` – Almacenamiento (particiones, FS, montajes)**
  - `tasks/main.yml`: `community.general.parted`, `filesystem`, `mount`.
  - `defaults/main.yml`: Listas `storage_partitions`, `storage_filesystems`, `storage_mounts`.

## Requisitos
- **Colecciones**: `requirements.yml` incluye `community.general` (para `parted` y `ufw`).
```bash
ansible-galaxy collection install -r requirements.yml
```

## Ejecuciones selectivas (tags útiles)
```bash
# Gateway
ansible-playbook -i inventories/hosts.ini site.yml --limit debian_gateway --tags "router,nat,dhcp,dns,security,storage"

# Clientes Ubuntu
ansible-playbook -i inventories/hosts.ini site.yml --limit ubuntu_pcs --tags "client,net,ufw,users,proc,automation,security"
```

## Ajustes mínimos antes de ejecutar
- En `group_vars/debian_gateway.yml`: `router_external_interface`/`router_internal_interface`.
- En `group_vars/ubuntu_pcs.yml`: `client_interface`.
- (Opcional) Definir `storage_partitions` y montajes.
