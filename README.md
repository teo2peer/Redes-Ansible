# Ansible — Laboratorio ASR (UJI)

Automatiza Labs 1, 2 y 4 (red, gateway/NAT, web, FTP, DNS master+slave) más:
clave SSH, apt upgrade, Samba (server vbox2 + clientes resto), monitor CPU/RAM/disco con orquestador en gateway.

## Topología

| Host         | IP interna     | Rol |
|--------------|----------------|-----|
| vbox-gateway | 192.168.0.1    | gateway, NAT, control node, monitor master, samba client |
| vbox1        | 192.168.0.100  | Apache, samba client |
| vbox2        | 192.168.0.101  | vsftpd, **samba server** |
| vbox-DNS     | 192.168.0.102  | BIND9 master |
| vbox-DNS2    | 192.168.0.103  | BIND9 slave |

Dominio `asr.es`. Gateway externo `150.128.48.1`. **Editar `ip_ext` en `inventory/hosts.yml`** con la IP que dé el profesor.

## Requisitos en control node

```
sudo apt install ansible
ansible-galaxy collection install community.crypto ansible.posix
```

## Uso

```bash
# 1. Bootstrap clave SSH (pide password una vez)
ansible-playbook playbooks/00-bootstrap-ssh.yml --ask-pass --ask-become-pass

# 2. Despliegue completo (fase 1: DNS upstream UJI)
ansible-playbook playbooks/site.yml

# 3. Reaplica fase 2 con DNS interno ya operativo
ansible-playbook playbooks/01-common.yml -e dns_phase=final
```

Playbooks individuales `01..09` se pueden lanzar sueltos.

## Verificación

- `ansible all -m ping` — todos OK
- vbox1 → `ping 8.8.8.8` (NAT gateway)
- móvil 4G → `curl http://<ip_ext>` y `sftp <ip_ext>`
- cualquier host → `dig vbox1.asr.es` resuelve
- vbox1 → `ls /mnt/compartido` y crear archivo, ver en vbox2 `/srv/samba/compartido`
- gateway → `tail /var/log/monitor/vbox1.log` tras 1-2 min
