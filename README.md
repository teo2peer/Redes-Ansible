# Ansible — Laboratorio ASR (UJI)

Automatiza Labs 1, 2, 4 y 6 (red, gateway/NAT, web, FTP, DNS master+slave, Samba, monitor casero y monitorización Nagios) sobre el dominio `asr.es`.

## Topología

| Host          | IP interna     | Rol |
|---------------|----------------|-----|
| vbox-gateway  | 192.168.0.1    | gateway, NAT, monitor master, samba client, nagios client |
| vbox1         | 192.168.0.100  | Apache2, samba client, nagios client |
| vbox2         | 192.168.0.101  | vsftpd, **samba server**, nagios client |
| vbox-DNS      | 192.168.0.102  | BIND9 master, samba client, nagios client |
| vbox-DNS2     | 192.168.0.103  | BIND9 slave, samba client, nagios client |
| vbox-nagios   | 192.168.0.150  | **servidor Nagios** (config en `/usr/local/nagios/etc`) |

Dominio `asr.es`. Gateway externo `150.128.48.1`. **Editar `ip_ext` en `inventory/hosts.yml`** con la IP que dé el profesor. Usuario por defecto en todas las VMs: `osboxes` (necesita contraseña real: `sudo passwd osboxes` en cada VM antes del bootstrap).

## Requisitos en el nodo de control

```bash
sudo apt install ansible sshpass
ansible-galaxy collection install community.crypto ansible.posix
```

## Despliegue rápido

```bash
# 1. Bootstrap: instala clave SSH y NOPASSWD sudo en osboxes (pide pass 1 vez)
ansible-playbook playbooks/00-bootstrap-ssh.yml --ask-pass --ask-become-pass

# 2. Despliegue completo (fase bootstrap: DNS upstream UJI)
ansible-playbook playbooks/site.yml

# 3. Cambia a DNS internos una vez BIND9 está operativo
ansible-playbook playbooks/01-common.yml -e dns_phase=final
```

## Playbooks disponibles

Cada playbook se puede lanzar suelto con `ansible-playbook playbooks/<archivo>`:

| Playbook | Hosts | Qué hace |
|---|---|---|
| `00-bootstrap-ssh.yml` | `all` (como `osboxes` con pass) | Genera keypair en control node, instala pubkey en `osboxes`, NOPASSWD sudo, intercambia claves root→gateway para el monitor |
| `01-common.yml` | `all:!nagios_server` | hostname, `/etc/hosts`, `apt dist-upgrade`, paquetes base, `netplan` (gateway dual-NIC, resto single-NIC) |
| `02-gateway.yml` | `gateway` | `ip_forward=1`, `iptables` (MASQUERADE + DNAT 80→vbox1, 21→vbox2), systemd `script_firewall.service` persistente |
| `03-web.yml` | `web` | Instala Apache2 + `index.html` mínimo |
| `04-ftp.yml` | `ftp` | Instala vsftpd con auth local, chroot, sin anónimo |
| `05-dns-master.yml` | `dns_master` | BIND9 autoritativo de `asr.es`, zonas directa+inversa generadas desde el inventario |
| `06-dns-slave.yml` | `dns_slave` | BIND9 secundario, AXFR desde el master |
| `07-samba-server.yml` | `samba_server` | Samba en `vbox2`, share `/srv/samba/compartido` (guest, escritura) |
| `08-samba-clients.yml` | `samba_clients` | Instala `cifs-utils`, monta `/mnt/compartido` vía fstab en los 4 clientes |
| `09-monitor.yml` | `monitor_master:monitor_agents` | Script monitor CPU/RAM/disco, cron 1 min, logs centralizados en gateway |
| `10-nagios-clients.yml` | `nagios_clients` | Instala NRPE + plugins, configura `nrpe.cfg` + `nrpe_local.cfg`, servicio activo |
| `11-nagios-server.yml` | `nagios_server` | En `vbox-nagios`: genera `/usr/local/nagios/etc/servers/<host>.cfg` por cada cliente, asegura `check_nrpe` en `commands.cfg`, valida config y recarga `nagios` |
| `site.yml` | (orquestador) | Importa todos en orden: 01 → 02 → 05 → 06 → 03 → 04 → 07 → 08 → 09 → 10 → 11 |

## Comandos ad-hoc útiles

```bash
# Conectividad: todos los hosts deben responder pong
ansible all -m ping

# Instalar un paquete en un host o grupo
ansible vbox1 -b -m ansible.builtin.apt -a "name=htop state=present update_cache=true"

# Actualizar todo el ecosistema bajo demanda
ansible all -b -m ansible.builtin.apt -a "update_cache=true upgrade=dist"

# Reiniciar un servicio en su grupo
ansible web -b -m ansible.builtin.service -a "name=apache2 state=restarted"

# Reiniciar las máquinas (espera a que vuelva el SSH)
ansible all -b -m ansible.builtin.reboot -a "reboot_timeout=300"

# Comando arbitrario
ansible all -m ansible.builtin.shell -a "uptime"
```

## Verificación

```bash
# Conectividad básica
ansible all -m ping

# NAT funciona desde un nodo interno
ansible vbox1 -a "ping -c 2 8.8.8.8"

# DNS resuelve
ansible all -a "dig +short vbox1.asr.es"

# Web por DNAT externo
curl http://<ip_ext_gateway>

# Samba: escribir en un cliente, leer en otro
ansible vbox-gateway -b -a "touch /mnt/compartido/test.txt"
ansible vbox-DNS     -a "ls /mnt/compartido/"

# Monitor casero
ansible vbox-gateway -b -a "tail -n 5 /var/log/monitor/vbox1.log"

# NRPE activo en clientes
ansible nagios_clients -b -a "systemctl is-active nagios-nrpe-server"

# Desde el servidor Nagios (192.168.0.150), abrir http://192.168.0.150/nagios
# (user nagiosadmin) y comprobar que aparecen vbox-gateway, vbox1, vbox2,
# vbox-DNS y vbox-DNS2 con sus servicios PING/SSH/Disk/APT (+HTTP en vbox1).
```

Documentación completa: [`DOCUMENTACION.md`](DOCUMENTACION.md).
