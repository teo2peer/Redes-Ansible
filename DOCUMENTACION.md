# Automatización del dominio `asr.es` con Ansible

## Índice

1. [Introducción](#1-introducción)
   - [¿Qué es y para qué sirve Ansible?](#qué-es-y-para-qué-sirve-ansible)
   - [¿Qué es y cómo funciona Ansible?](#qué-es-y-cómo-funciona-ansible)
2. [Instalación y configuración del nodo de control](#2-instalación-y-configuración-del-nodo-de-control)
   - [Pasos previos](#pasos-previos)
   - [Estructura del proyecto](#estructura-del-proyecto)
   - [Configuración del inventario](#configuración-del-inventario)
   - [Variables globales (`group_vars`)](#variables-globales-group_vars)
   - [Configuración de Ansible (`ansible.cfg`)](#configuración-de-ansible-ansiblecfg)
   - [Bootstrap SSH y sudo sin contraseña](#bootstrap-ssh-y-sudo-sin-contraseña)
3. [Configuración base de todos los nodos (rol `common`)](#3-configuración-base-de-todos-los-nodos-rol-common)
4. [Configuración del gateway: NAT y firewall](#4-configuración-del-gateway-nat-y-firewall)
5. [Configuración del servicio web (Apache2)](#5-configuración-del-servicio-web-apache2)
6. [Configuración del servicio FTP (vsftpd)](#6-configuración-del-servicio-ftp-vsftpd)
7. [Configuración del servicio DNS (BIND9 maestro/esclavo)](#7-configuración-del-servicio-dns-bind9-maestroesclavo)
8. [Configuración del servidor de almacenamiento (Samba)](#8-configuración-del-servidor-de-almacenamiento-samba)
9. [Monitor casero (CPU/RAM/Disco)](#9-monitor-casero-cpuramdisco)
10. [Configuración de clientes Nagios NRPE](#10-configuración-de-clientes-nagios-nrpe)
11. [Caso práctico](#11-caso-práctico)
12. [Bibliografía](#12-bibliografía)

---

## 1. Introducción

### ¿Qué es y para qué sirve Ansible?

Ansible es una herramienta de automatización de la configuración, despliegue y orquestación de sistemas. Permite describir el estado deseado de una infraestructura completa en ficheros de texto plano (YAML) y aplicarlo de forma repetible, ordenada y desatendida sobre cualquier número de máquinas. A diferencia de los scripts tradicionales, Ansible es **idempotente**: ejecutar el mismo playbook varias veces no rompe el sistema ni duplica configuraciones, sino que se limita a aplicar los cambios necesarios para que el estado real coincida con el declarado.

Frente a otras herramientas de automatización, Ansible destaca por no necesitar agente en los nodos gestionados: el nodo de control se conecta por SSH y ejecuta módulos Python remotamente, lo que evita la instalación y mantenimiento de software adicional. Su naturaleza declarativa hace que el playbook describa el "qué" en lugar del "cómo paso a paso", y su inventario permite agrupar hosts para que un mismo playbook se aplique de forma consistente a uno o a cien servidores. Como contrapartida, el rendimiento en flotas muy grandes es inferior al de herramientas con agente como Puppet o Chef, la curva de aprendizaje inicial con YAML y Jinja2 es notable, y depurar plantillas complejas puede llegar a ser tedioso.

| Ventajas | Desventajas |
|---|---|
| Sin agente: SSH es el único requisito | Más lento en flotas muy grandes que herramientas con agente |
| Idempotente y declarativo | Curva inicial con YAML y plantillas Jinja2 |
| Inventario agrupable y reutilizable | Depuración compleja en bucles Jinja anidados |
| Gran ecosistema (Ansible Galaxy) | Requiere Python en los nodos gestionados |

En la práctica, Ansible se utiliza para provisionar servidores desde cero (paquetes, usuarios, claves SSH, red), desplegar servicios coordinados entre varias máquinas (web + DNS + base de datos), aplicar cambios masivos sobre toda la flota en segundos y, lo que resulta especialmente útil en el día a día, lanzar **tareas ad-hoc** como instalar un paquete o actualizar el sistema en cualquier subconjunto de hosts sin escribir un playbook nuevo.

### ¿Qué es y cómo funciona Ansible?

Ansible fue creado en 2012 por Michael DeHaan y actualmente lo mantiene Red Hat. Está escrito en Python y se distribuye bajo licencia GPLv3. Su arquitectura es muy simple: existe un **nodo de control** (la máquina donde se ejecuta `ansible-playbook`, en nuestro caso el equipo del usuario) que solo necesita Python, Ansible y acceso SSH al resto, y varios **nodos gestionados** (las cinco VMs del dominio `asr.es`) que únicamente requieren Python instalado y aceptar conexiones SSH desde el nodo de control.

Toda la configuración se organiza en tres tipos de ficheros. El **inventario** (`inventory/hosts.yml`) enumera los hosts y los agrupa por funciones. Los **playbooks** son ficheros YAML que asocian grupos de hosts con tareas a ejecutar. Los **roles** agrupan tareas, plantillas, ficheros estáticos, variables y handlers reutilizables en una estructura de directorios estándar. Las tareas se traducen en llamadas a **módulos** (`ansible.builtin.apt`, `ansible.builtin.template`, `ansible.posix.mount`…), que son las primitivas reales que actúan sobre el sistema. Cuando una tarea cambia un fichero de configuración, puede notificar a un **handler** que reinicie el servicio asociado, garantizando que el reinicio solo ocurre si realmente hubo cambios.

El flujo de ejecución es directo: al lanzar `ansible-playbook site.yml`, Ansible lee el inventario, calcula a qué hosts aplica cada `play`, abre una sesión SSH por host, copia los módulos Python necesarios a un directorio temporal y los ejecuta, recoge resultados, decide qué handlers notificar y continúa con la siguiente tarea. Comparado con Puppet o Chef, su YAML resulta más legible y no requiere daemon en los nodos, aunque la rigidez de la indentación YAML y la lentitud relativa frente a herramientas con agente son sus principales puntos débiles.

---

## 2. Instalación y configuración del nodo de control

### Pasos previos

En el nodo de control (Linux o WSL) se necesita Ansible y dos colecciones adicionales que usan los roles del proyecto:

```bash
sudo apt update
sudo apt install -y ansible python3-pip
ansible-galaxy collection install community.crypto ansible.posix
```

Se recomiendan `ansible-core ≥ 2.14` y Python ≥ 3.8. Tras la instalación, `ansible --version` debe devolver la versión correctamente.

### Estructura del proyecto

El proyecto sigue la convención habitual de Ansible: un `ansible.cfg` en la raíz, un directorio `inventory/` con el inventario, un `group_vars/` con variables globales, un `playbooks/` numerado para imponer un orden lógico al despliegue, y un `roles/` con un subdirectorio por rol, cada uno con su propia carpeta `tasks/`, `handlers/`, `templates/`, `files/` y `defaults/`.

```
Redes Ansible/
├── ansible.cfg
├── README.md
├── DOCUMENTACION.md          ← este fichero
├── inventory/
│   └── hosts.yml
├── group_vars/
│   └── all.yml
├── playbooks/
│   ├── 00-bootstrap-ssh.yml  ← solo la primera vez
│   ├── 01-common.yml ... 10-nagios-clients.yml
│   └── site.yml              ← orquestador
└── roles/
    ├── common/ gateway/ web/ ftp/
    ├── dns_master/ dns_slave/
    ├── samba_server/ samba_client/
    ├── monitor/
    └── nagios_client/
```

### Configuración del inventario

El fichero `inventory/hosts.yml` define los cinco hosts del dominio `asr.es`, sus IPs internas y su pertenencia a grupos funcionales. Cada host puede formar parte de varios grupos a la vez (por ejemplo, `vbox2` es a la vez `ftp`, `samba_server`, `monitor_agents` y `nagios_clients`), lo que permite aplicarle todos los roles correspondientes sin duplicar información.

| Host | IP interna | Rol principal | Grupos |
|---|---|---|---|
| `vbox-gateway` | 192.168.0.1 | NAT + firewall | gateway, samba_clients, monitor_master, nagios_clients |
| `vbox1` | 192.168.0.100 | Apache2 | web, samba_clients, monitor_agents, nagios_clients |
| `vbox2` | 192.168.0.101 | FTP + Samba server | ftp, samba_server, monitor_agents, nagios_clients |
| `vbox-DNS` | 192.168.0.102 | BIND9 master | dns_master, samba_clients, monitor_agents, nagios_clients |
| `vbox-DNS2` | 192.168.0.103 | BIND9 slave | dns_slave, samba_clients, monitor_agents, nagios_clients |

Las variables globales que aplican a todos los hosts se definen en la sección `all.vars` del inventario:

```yaml
ansible_user: osboxes
ansible_python_interpreter: /usr/bin/python3
domain: asr.es
internal_network: 192.168.0.0/24
dns_servers: [192.168.0.102, 192.168.0.103]
dns_forwarders: [150.128.98.10, 8.8.8.8]
dns_phase: bootstrap   # cambiar a 'final' tras desplegar BIND9
```

La variable `dns_phase` merece una mención especial. Cuando se despliega el ecosistema por primera vez, los DNS internos (`vbox-DNS` y `vbox-DNS2`) todavía no existen, así que los clientes se configuran con los forwarders externos (`bootstrap`). Una vez BIND9 está operativo, se reaplica el rol `common` con `-e dns_phase=final` y los clientes pasan a usar los DNS internos. Este pequeño truco evita el problema clásico del huevo y la gallina al automatizar un ecosistema desde cero.

### Variables globales (`group_vars`)

El fichero `group_vars/all.yml` reúne los valores compartidos por varios roles, manteniendo en un único punto las rutas y nombres que de otro modo se repetirían:

```yaml
admin_user: osboxes
ssh_key_path: "{{ lookup('env','HOME') }}/.ssh/id_rsa_ansible"
samba_share_name: compartido
samba_share_path: /srv/samba/compartido
samba_mount_point: /mnt/compartido
monitor_log_dir: /var/log/monitor
monitor_interval_min: 1
nagios_server_ip: 192.168.0.150
```

### Configuración de Ansible (`ansible.cfg`)

```ini
[defaults]
inventory = inventory/hosts.yml
host_key_checking = False
retry_files_enabled = False
roles_path = roles
stdout_callback = yaml
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
```

La opción `host_key_checking = False` evita el prompt de aceptar la huella SSH la primera vez (aceptable en laboratorio, no recomendado en producción). El callback `yaml` produce una salida más legible que el formato por defecto, y `pipelining = True` reduce el número de conexiones SSH por tarea, acelerando notablemente la ejecución.

### Bootstrap SSH y sudo sin contraseña

Antes de poder lanzar el resto de playbooks de forma desatendida, hay que dejar lista la autenticación. Las VMs de OSBoxes vienen con el usuario `osboxes` ya creado y con sudo, así que el playbook `playbooks/00-bootstrap-ssh.yml` se limita a generar una clave SSH dedicada en el nodo de control, distribuirla a los `authorized_keys` de `osboxes` en todos los nodos, configurar `sudo` sin contraseña para `osboxes`, y (para el rol `monitor`) intercambiar claves de `root` entre los agentes y el gateway. Es la única vez en todo el ciclo de vida del proyecto que hace falta autenticarse con contraseña, así que conviene asegurarse antes de que `osboxes` tiene una contraseña real (las imágenes OSBoxes la traen vacía y SSH rechaza contraseñas vacías por defecto): `sudo passwd osboxes` en cada VM.

```bash
ansible-playbook playbooks/00-bootstrap-ssh.yml --ask-pass --ask-become-pass
```

A partir de ese momento, todos los demás playbooks se ejecutan sin contraseña.

---

## 3. Configuración base de todos los nodos (rol `common`)

El rol `common` se aplica a todos los hosts y deja la máquina lista para que los roles específicos puedan trabajar. Empieza fijando el `hostname` desde `inventory_hostname` y regenerando `/etc/hosts` mediante la plantilla `hosts.j2`, que recorre el inventario y produce una entrada por cada host con su nombre corto y su FQDN (`vbox1.asr.es`). A continuación actualiza los repositorios y aplica `dist-upgrade` con `cache_valid_time: 3600` (para no refrescar si ya se hizo hace menos de una hora) e instala los paquetes base que se usan en operaciones cotidianas: `vim`, `curl`, `dnsutils` y `net-tools`.

La parte más interesante es la configuración de red mediante `netplan`. La plantilla `netplan.yaml.j2` distingue entre el gateway (que tiene dos interfaces, una interna y otra externa con la default route) y el resto de hosts (una sola interfaz, IP estática y default route apuntando a `192.168.0.1`). Los DNS configurados dependen del valor de `dns_phase`: los forwarders externos durante el bootstrap, los internos cuando BIND9 ya está desplegado. Tras escribir el fichero, el handler `Apply netplan` ejecuta `netplan apply` si y solo si hubo cambios.

Un fragmento ilustrativo es el bucle que genera `/etc/hosts`:

```jinja
{% for h in groups['all'] %}
{%   set hv = hostvars[h] %}
{%   set ip = hv.ip | default(hv.ip_int | default(hv.ansible_host)) %}
{{ ip }} {{ h }}.{{ domain }} {{ h }}
{% endfor %}
```

Este mismo patrón (recorrer `groups['all']` y consultar `hostvars[h]`) se reutiliza en BIND9 para generar las zonas dinámicamente desde el inventario, eliminando duplicación: añadir un host nuevo al inventario basta para que aparezca tanto en `/etc/hosts` como en las zonas DNS la próxima vez que se aplique el playbook.

---

## 4. Configuración del gateway: NAT y firewall

El rol `gateway`, aplicado únicamente a `vbox-gateway`, prepara la máquina como router y firewall del ecosistema. Lo primero es activar el reenvío IP de forma persistente, escribiendo `net.ipv4.ip_forward=1` en `/etc/sysctl.conf` mediante el módulo `ansible.posix.sysctl` (que se encarga también del `sysctl -p`). A continuación genera el script `/root/bin/firewall.sh` desde plantilla, donde se concentra toda la lógica de NAT y reenvío de puertos:

```bash
#!/bin/sh
IFACE_EXT={{ iface_ext }}
IFACE_INT={{ iface_int }}
WEB_IP={{ hostvars['vbox1'].ip }}
FTP_IP={{ hostvars['vbox2'].ip }}

iptables --flush
iptables -t nat --flush

# NAT salida (MASQUERADE)
iptables -t nat -A POSTROUTING -o $IFACE_EXT -j MASQUERADE

# DNAT: 80 -> vbox1 (web), 21 -> vbox2 (ftp)
iptables -t nat -A PREROUTING -p tcp -i $IFACE_EXT --dport 80 -j DNAT --to $WEB_IP
iptables -t nat -A PREROUTING -p tcp -i $IFACE_EXT --dport 21 -j DNAT --to $FTP_IP

iptables -A FORWARD -i $IFACE_INT -o $IFACE_EXT -j ACCEPT
iptables -A FORWARD -i $IFACE_EXT -o $IFACE_INT -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -p tcp -d $WEB_IP --dport 80 -j ACCEPT
iptables -A FORWARD -p tcp -d $FTP_IP --dport 21 -j ACCEPT
```

Para que estas reglas sobrevivan a un reinicio, el rol instala un servicio `systemd` (`script_firewall.service`) de tipo `oneshot` con `RemainAfterExit=true` que ejecuta el script al arranque. Esto evita depender de paquetes como `iptables-persistent` y mantiene la lógica del firewall en un único fichero plantilla, fácil de revisar. Los handlers `Daemon reload` y `Restart firewall` reaccionan automáticamente a cambios en el script o en el unit.

---

## 5. Configuración del servicio web (Apache2)

El rol `web`, aplicado solo a `vbox1`, es deliberadamente minimalista. Instala `apache2`, deposita un `index.html` mínimo que demuestra que el servicio responde (`<h1>vbox1 - WEB ASR</h1>`) y habilita el servicio para que arranque automáticamente. El acceso desde dentro del dominio se hace por `http://vbox1.asr.es`, y desde fuera por la IP pública del gateway gracias al DNAT del puerto 80 configurado en el firewall.

---

## 6. Configuración del servicio FTP (vsftpd)

El rol `ftp`, aplicado solo a `vbox2`, instala `vsftpd` y genera `/etc/vsftpd.conf` con las opciones habituales para un servidor con autenticación local: `anonymous_enable=NO` para desactivar el acceso anónimo, `local_enable=YES` y `write_enable=YES` para permitir login y escritura a los usuarios del sistema, y `chroot_local_user=YES` junto con `allow_writeable_chroot=YES` para enjaular a cada usuario en su `$HOME`. El handler `Restart vsftpd` reinicia el servicio si la configuración cambia. El acceso externo al puerto 21 llega gracias al DNAT configurado en el gateway.

---

## 7. Configuración del servicio DNS (BIND9 maestro/esclavo)

El servicio DNS se reparte entre dos roles. El rol `dns_master`, aplicado a `vbox-DNS` (192.168.0.102), instala `bind9` y sus utilidades, configura `/etc/bind/named.conf.options` para que el servidor sea autoritativo del dominio (con los `forwarders` comentados), y declara en `/etc/bind/named.conf.local` la zona directa `asr.es` y la zona inversa `0.168.192.in-addr.arpa`, ambas con `also-notify` y `allow-transfer` apuntando al esclavo para que la replicación AXFR funcione automáticamente.

Lo más reseñable es que las zonas se generan dinámicamente desde el inventario. En lugar de mantener manualmente la lista de registros A y PTR, una plantilla recorre todos los hosts y produce las entradas correspondientes:

```jinja
$TTL 604800
@ IN SOA {{ domain }}. root.{{ domain }}. (
    {{ ansible_date_time.epoch }} ; Serial (epoch -> cambia siempre)
    604800 86400 2419200 604800 )
@ IN NS vbox-DNS.{{ domain }}.
@ IN NS vbox-DNS2.{{ domain }}.
{% for h in groups['all'] %}
{%   set hv = hostvars[h] %}
{%   set ip = hv.ip | default(hv.ip_int | default(hv.ansible_host)) %}
{{ '%-15s' | format(h) }} IN A {{ ip }}
{% endfor %}
```

Esto significa que basta con añadir un host al inventario para que aparezca automáticamente en las dos zonas DNS al volver a aplicar el playbook. El uso del `epoch` como número de serie SOA garantiza que cada despliegue incremente correctamente la versión y dispare la transferencia al esclavo.

El rol `dns_slave`, aplicado a `vbox-DNS2` (192.168.0.103), es mucho más simple: instala BIND, configura `named.conf.local` con las mismas zonas pero como `type slave` apuntando al master para AXFR, y ajusta los permisos de `/var/cache/bind` (que es donde BIND escribirá los ficheros de zona recibidos).

Como se mencionó al hablar de `dns_phase`, una vez ambos servidores están operativos se reaplica el rol `common` con `-e dns_phase=final` para que todos los clientes pasen a usar los DNS internos.

---

## 8. Configuración del servidor de almacenamiento (Samba)

El proyecto utiliza Samba como servidor de almacenamiento centralizado: una sola máquina (`vbox2`) expone un directorio compartido y todas las demás lo montan como si fuese local, ofreciendo así una zona común donde compartir ficheros entre los nodos del ecosistema.

El rol `samba_server` instala el paquete `samba`, crea el directorio `/srv/samba/compartido` con `owner=nobody`, `group=nogroup` y modo `0777`, y genera `/etc/samba/smb.conf` desde plantilla:

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba ASR vbox2
   netbios name = vbox2
   security = user
   map to guest = bad user
   dns proxy = no

[compartido]
   path = /srv/samba/compartido
   browsable = yes
   writable = yes
   guest ok = yes
   read only = no
   create mask = 0666
   directory mask = 0777
   force user = nobody
```

Por sencillez (entorno de laboratorio) la compartición usa acceso de invitado (`guest ok = yes`). En producción habría que añadir usuarios con `smbpasswd` y restringir `guest ok`. Tras escribir la configuración, el handler `Restart smbd` reinicia el servicio si fuera necesario.

El rol `samba_client`, aplicado a los cuatro hosts no-servidor (`vbox-gateway`, `vbox1`, `vbox-DNS` y `vbox-DNS2`), instala `cifs-utils`, crea el punto de montaje `/mnt/compartido` y añade la entrada al `fstab` con `ansible.posix.mount`:

```yaml
- name: fstab montaje samba
  ansible.posix.mount:
    path: "{{ samba_mount_point }}"
    src: "//{{ hostvars['vbox2'].ip }}/{{ samba_share_name }}"
    fstype: cifs
    opts: "guest,_netdev,uid=1000,gid=1000,iocharset=utf8,vers=3.0"
    state: mounted
```

El estado `mounted` realiza dos acciones a la vez: añade la línea al `/etc/fstab` (para que el montaje sobreviva a los reinicios) y monta inmediatamente. La opción `_netdev` indica que el montaje depende de la red, de modo que `systemd` espera a que la red esté lista antes de intentarlo. Tras aplicar `playbooks/08-samba-clients.yml`, los cuatro clientes tienen `/mnt/compartido` accesible en lectura y escritura, apuntando al mismo directorio físico de `vbox2`.

---

## 9. Monitor casero (CPU/RAM/Disco)

El rol `monitor` implementa una monitorización mínima sin depender de software adicional, repartiendo responsabilidades entre el gateway (master, que centraliza los logs) y el resto de nodos (agentes, que envían métricas). En el master crea el directorio `/var/log/monitor/` y un script `monitor-resumen.sh` que muestra las últimas cinco líneas de cada log de host, útil para inspecciones rápidas. En los agentes instala `/usr/local/bin/monitor.sh`, que cada minuto (cron `*/1 * * * *`) recoge la carga de CPU (`top -bn1`), el uso de RAM (`free -m`) y la ocupación del disco raíz (`df -h /`), y envía la línea por SSH al gateway para que se escriba en `/var/log/monitor/<host>.log`.

Para que esa copia por SSH funcione sin intervención manual, el playbook de bootstrap (`00-bootstrap-ssh.yml`) genera un par de claves para `root` en cada agente y autoriza la pública en el gateway. Este monitor convive con el rol `nagios_client`: el primero es un log centralizado simple, el segundo expone métricas estructuradas para un servidor Nagios externo, que es lo que se describe a continuación.

---

## 10. Monitorización con Nagios (clientes NRPE + configuración del servidor)

Para integrarnos con el servidor Nagios existente en `192.168.0.150` (la instalación inicial del servidor y el primer cliente de ejemplo `192.168.0.151` se hicieron manualmente en clase siguiendo el Laboratorio 6), Ansible se encarga de dos tareas complementarias: instalar el agente NRPE en los cinco nodos del dominio y, acto seguido, generar y desplegar en el servidor Nagios los ficheros de configuración necesarios para que esos nodos aparezcan monitorizados. Todo el proceso se lanza con un único comando.

### Inventario

En el inventario se añaden dos grupos. `nagios_clients` reúne a los cinco hosts que se van a monitorizar, mientras que `nagios_server` apunta a la máquina que aloja Nagios (192.168.0.150):

```yaml
    nagios_clients:
      hosts:
        vbox-gateway:
        vbox1:
        vbox2:
        vbox-DNS:
        vbox-DNS2:
    nagios_server:
      hosts:
        vbox-nagios:
          ansible_host: 192.168.0.150
          ip: 192.168.0.150
          iface: enp0s3
```

Como el servidor Nagios ya tiene una configuración propia hecha a mano siguiendo el lab (Apache, htpasswd, `nagios.cfg`, plugins, etc.), no queremos que el rol `common` lo sobrescriba. Por eso `playbooks/01-common.yml` apunta a `all:!nagios_server`, lo que mantiene a `vbox-nagios` fuera del rol base pero dentro del alcance del rol específico `nagios_server`. En `group_vars/all.yml` se define `nagios_server_ip: 192.168.0.150`, valor que consume el rol `nagios_client` para construir `allowed_hosts` en cada agente.

### Rol `nagios_client` (instalación de agentes NRPE)

El rol `nagios_client` define en `defaults/main.yml` el valor del servidor Nagios y la IP del cliente, que se resuelve automáticamente a `ip` (o `ip_int` en el caso del gateway) sin necesidad de definir variables nuevas por host:

```yaml
nagios_server_ip: 192.168.0.150
nrpe_cfg: /etc/nagios/nrpe.cfg
nrpe_local_cfg: /etc/nagios/nrpe_local.cfg
nagios_client_ip: "{{ ip | default(ip_int) }}"
```

Las tareas del rol instalan los paquetes `nagios-nrpe-server` y `monitoring-plugins`, ajustan `nrpe.cfg` para fijar `server_address` y `allowed_hosts`, despliegan `nrpe_local.cfg` con los comandos `check_*` desde plantilla, y dejan el servicio activo y habilitado en el arranque:

```yaml
- name: Instalar NRPE y plugins
  ansible.builtin.apt:
    name: [nagios-nrpe-server, monitoring-plugins]
    state: present
    update_cache: true

- name: nrpe.cfg - fijar server_address al IP del cliente
  ansible.builtin.lineinfile:
    path: "{{ nrpe_cfg }}"
    regexp: '^[#\s]*server_address\s*='
    line: "server_address={{ nagios_client_ip }}"
  notify: Restart nrpe

- name: nrpe.cfg - permitir host Nagios
  ansible.builtin.lineinfile:
    path: "{{ nrpe_cfg }}"
    regexp: '^[#\s]*allowed_hosts\s*='
    line: "allowed_hosts=127.0.0.1,::1,{{ nagios_server_ip }}"
  notify: Restart nrpe

- name: nrpe_local.cfg - comandos check_*
  ansible.builtin.template:
    src: nrpe_local.cfg.j2
    dest: "{{ nrpe_local_cfg }}"
    owner: root
    group: root
    mode: '0644'
  notify: Restart nrpe

- name: Servicio NRPE activo
  ansible.builtin.service:
    name: nagios-nrpe-server
    enabled: true
    state: started
```

La decisión clave aquí es por qué se usa `lineinfile` para `nrpe.cfg` y `template` para `nrpe_local.cfg`. El paquete Debian trae un `nrpe.cfg` con decenas de tunables que no queremos perder, así que solo se tocan quirúrgicamente las dos líneas que controlamos. En cambio, `nrpe_local.cfg` está pensado para overrides locales y lo controlamos íntegro, por lo que tiene sentido sobrescribirlo entero desde plantilla:

```
# Gestionado por Ansible - rol nagios_client
command[check_root]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /
command[check_ping]=/usr/lib/nagios/plugins/check_ping -H {{ nagios_client_ip }} -w 100.0,20% -c 500.0,60% -p 5
command[check_ssh]=/usr/lib/nagios/plugins/check_ssh -4 {{ nagios_client_ip }}
command[check_http]=/usr/lib/nagios/plugins/check_http -I {{ nagios_client_ip }}
command[check_apt]=/usr/lib/nagios/plugins/check_apt
```

El handler `Restart nrpe` reinicia el servicio si cualquiera de las tareas anteriores produjo un cambio. El playbook que aplica este rol es `playbooks/10-nagios-clients.yml`.

### Rol `nagios_server` (registro automático de hosts en Nagios)

Una vez los agentes están escuchando, el siguiente paso es que el servidor Nagios sepa de su existencia. En lugar de editar a mano un fichero `.cfg` por cada host en `/usr/local/nagios/etc/servers/` (como propone el Laboratorio 6), el rol `nagios_server` los genera automáticamente recorriendo el grupo `nagios_clients` del inventario. Los defaults del rol concentran las rutas:

```yaml
nagios_cfg_dir: /usr/local/nagios/etc
nagios_servers_dir: "{{ nagios_cfg_dir }}/servers"
nagios_commands_cfg: "{{ nagios_cfg_dir }}/objects/commands.cfg"
nagios_service: nagios
nagios_owner: nagios
nagios_group: nagios
```

Las tareas hacen tres cosas. Primero, garantizan que existe el directorio `servers/` con la propiedad correcta. Segundo, aseguran con `blockinfile` que el comando `check_nrpe` está definido en `commands.cfg` (el lab pide añadirlo a mano en T6.5; aquí queda automatizado y es idempotente gracias al marker). Tercero, recorren con un `loop` el grupo `nagios_clients` y, por cada host, generan un fichero `<host>.cfg` desde plantilla, consultando las variables del host via `hostvars`:

```yaml
- name: Asegurar directorio de configs por host
  ansible.builtin.file:
    path: "{{ nagios_servers_dir }}"
    state: directory
    owner: "{{ nagios_owner }}"
    group: "{{ nagios_group }}"
    mode: '0755'

- name: Asegurar definición del comando check_nrpe en commands.cfg
  ansible.builtin.blockinfile:
    path: "{{ nagios_commands_cfg }}"
    marker: "# {mark} ANSIBLE BLOCK check_nrpe"
    block: |
      define command {
          command_name check_nrpe
          command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
      }
  notify: Restart nagios

- name: Generar fichero .cfg por cada cliente Nagios
  ansible.builtin.template:
    src: host.cfg.j2
    dest: "{{ nagios_servers_dir }}/{{ item }}.cfg"
    owner: "{{ nagios_owner }}"
    group: "{{ nagios_group }}"
    mode: '0644'
  loop: "{{ groups['nagios_clients'] }}"
  vars:
    nagios_host: "{{ item }}"
    nagios_host_ip: "{{ hostvars[item].ip | default(hostvars[item].ip_int) }}"
    nagios_host_groups: "{{ hostvars[item].group_names }}"
  notify: Restart nagios

- name: Validar configuración de Nagios
  ansible.builtin.command: /usr/local/nagios/bin/nagios -v {{ nagios_cfg_dir }}/nagios.cfg
  register: nagios_verify
  changed_when: false
  failed_when: nagios_verify.rc != 0

- name: Servicio nagios activo y habilitado
  ansible.builtin.service:
    name: "{{ nagios_service }}"
    enabled: true
    state: started
```

La validación previa al reinicio (`nagios -v`) actúa como red de seguridad: si la plantilla produjese una sintaxis inválida, el playbook falla **antes** de reiniciar el servicio, evitando dejar Nagios caído. El handler `Restart nagios` solo se dispara si hubo cambios efectivos en algún `.cfg`.

La plantilla `host.cfg.j2` produce un `define host` y cuatro `define service` (PING, Check SSH, Check Root/Disk, Check APT Update) para todos los hosts, más un quinto `define service` (Check HTTP) que solo aparece si el host pertenece al grupo `web`, gracias a un `{% if 'web' in nagios_host_groups %}`. Así, añadir un host al inventario con su grupo correcto basta para que aparezca monitorizado con el set de checks adecuado, sin tocar nada en el servidor Nagios.

### Despliegue con un solo comando

Todo lo anterior se ejecuta con un único `ansible-playbook`, gracias al orden en `site.yml`:

```bash
ansible-playbook playbooks/site.yml
```

`site.yml` importa `10-nagios-clients.yml` (instala NRPE en los cinco nodos) y a continuación `11-nagios-server.yml` (genera los `.cfg` en el servidor y recarga el servicio). Si solo se quiere actuar sobre la monitorización, sin reaplicar el resto del ecosistema, se pueden lanzar los dos playbooks de Nagios de forma aislada:

```bash
ansible-playbook playbooks/10-nagios-clients.yml playbooks/11-nagios-server.yml
```

### Firewall y prerrequisitos

`192.168.0.150` está dentro de la red `192.168.0.0/24`, así que NRPE (TCP/5666) viaja por LAN sin pasar por el NAT del gateway: no hace falta tocar el firewall. Como prerrequisito, `vbox-nagios` debe permitir SSH desde el nodo de control con el usuario `osboxes` y sudo (idealmente NOPASSWD). Si todavía no se hizo, se ejecuta el bootstrap limitado a esta máquina:

```bash
ansible-playbook playbooks/00-bootstrap-ssh.yml --limit vbox-nagios --ask-pass --ask-become-pass
```

El cliente manual `192.168.0.151` (`vbox-Nagios-cliente`) creado en el lab queda al margen del inventario: se sigue gestionando a mano y su `.cfg` ya está en `/usr/local/nagios/etc/servers/`. Si en algún momento se quiere automatizar también, basta con añadirlo al grupo `nagios_clients` y reaplicar `site.yml`.

---

## 11. Caso práctico

### Despliegue completo

La primera vez que se despliega el ecosistema, hace falta autenticarse con la contraseña del usuario `osboxes` para ejecutar el bootstrap y, a continuación, lanzar el orquestador completo. Después se cambia a la fase final de DNS para que los clientes apunten a los BIND internos:

```bash
ansible-playbook playbooks/00-bootstrap-ssh.yml --ask-pass --ask-become-pass
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/01-common.yml -e dns_phase=final
```

A partir de ese momento, cualquier nuevo despliegue se hace sin contraseña simplemente con `ansible-playbook playbooks/site.yml`. Como todo es idempotente, si nada cambió todos los hosts saldrán con `changed=0`.

### Tareas comunes

Una de las grandes ventajas de Ansible es que no estamos limitados a los playbooks de configuración: podemos lanzar tareas ad-hoc sobre cualquier subconjunto de hosts desde la línea de comando, sin escribir un playbook nuevo. La sintaxis general es `ansible <selector> -b -m <módulo> -a "<argumentos>"`, donde `-b` equivale a `sudo`, `-m` indica el módulo y `-a` sus argumentos.

Por ejemplo, para instalar un paquete en uno o varios hosts:

```bash
# htop en vbox1
ansible vbox1 -b -m ansible.builtin.apt -a "name=htop state=present update_cache=true"

# tcpdump en master y slave de DNS
ansible 'dns_master:dns_slave' -b -m ansible.builtin.apt -a "name=tcpdump state=present"

# nmap en todos los hosts del ecosistema
ansible all -b -m ansible.builtin.apt -a "name=nmap state=present update_cache=true"
```

Para actualizar el sistema entero, equivalente a lo que hace el rol `common` en cada despliegue completo pero ejecutado bajo demanda:

```bash
ansible all -b -m ansible.builtin.apt -a "update_cache=true upgrade=dist"
```

Reiniciar (o recargar) servicios concretos es igual de directo:

```bash
ansible web -b -m ansible.builtin.service -a "name=apache2 state=restarted"
ansible 'dns_master:dns_slave' -b -m ansible.builtin.service -a "name=bind9 state=reloaded"
```

Si hay que reiniciar las propias máquinas, Ansible se encarga de esperar a que cada host vuelva a aceptar SSH antes de marcar la tarea como completa:

```bash
ansible all -b -m ansible.builtin.reboot -a "reboot_timeout=300"
```

Y para inspecciones rápidas, `shell` y `ping` son insustituibles:

```bash
ansible all -m ansible.builtin.shell -a "uptime"
ansible all -m ansible.builtin.shell -a "df -h /"
ansible all -m ansible.builtin.ping
```

### Añadir un nuevo host al ecosistema

Supongamos que queremos añadir `vbox3` (192.168.0.110) como segundo servidor web. El primer paso es declararlo en el inventario dentro de los grupos correspondientes (`web`, `samba_clients`, `nagios_clients`, etc.). Después se ejecuta el bootstrap solo para el nuevo host y se reaplica el orquestador:

```bash
ansible-playbook playbooks/00-bootstrap-ssh.yml --limit vbox3 --ask-pass --ask-become-pass
ansible-playbook playbooks/site.yml
```

A partir de aquí todo es automático. `vbox3` aparece en `/etc/hosts` de todos los nodos gracias al bucle de la plantilla, BIND9 genera las nuevas entradas A y PTR para `vbox3` al regenerar las zonas, `vbox3` monta `/mnt/compartido` por Samba al aplicarse el rol `samba_client`, y NRPE queda instalado y listo para que Nagios empiece a monitorizarlo. Toda esta cascada de cambios sale gratis precisamente porque el código de los roles se apoya en el inventario y no en valores hardcodeados.

### Verificación end-to-end

Para confirmar que todo el ecosistema está operativo, conviene ejecutar una batería de comprobaciones. La conectividad básica se verifica con un simple `ping` de Ansible. Para validar el DNS, se usa `dig` desde cualquier host:

```bash
ansible all -m ansible.builtin.ping
ansible all -a "dig +short vbox1.asr.es"
ansible all -a "dig +short -x 192.168.0.101"
```

Para verificar el servidor web, una petición HTTP local; para Samba, comprobar que el montaje aparece en `mount` y que un fichero creado desde un cliente es visible desde otro:

```bash
ansible web -m ansible.builtin.uri -a "url=http://localhost return_content=yes"
ansible samba_clients -a "mount | grep compartido"
ansible vbox-gateway -b -a "touch /mnt/compartido/desde-gateway.txt"
ansible vbox-DNS     -a "ls -la /mnt/compartido/"
```

El monitor casero se inspecciona leyendo los logs centralizados en el gateway, y los clientes NRPE confirmando que el servicio está activo y escuchando en el puerto 5666:

```bash
ansible vbox-gateway -b -a "tail -n 5 /var/log/monitor/vbox1.log"
ansible nagios_clients -b -a "systemctl is-active nagios-nrpe-server"
ansible nagios_clients -b -a "ss -ltn | grep 5666"
```

Finalmente, desde el servidor Nagios (`192.168.0.150`), una llamada directa a `check_nrpe` debe responder con un OK por cada métrica:

```bash
/usr/lib/nagios/plugins/check_nrpe -H 192.168.0.100 -c check_http
```

Si todo lo anterior responde correctamente, el ecosistema está operativo y monitorizado.

---

## 12. Bibliografía

1. Documentación oficial de Ansible — https://docs.ansible.com/
2. Módulos `ansible.builtin` — https://docs.ansible.com/ansible/latest/collections/ansible/builtin/
3. Colección `ansible.posix` (`mount`, `sysctl`, `authorized_key`) — https://docs.ansible.com/ansible/latest/collections/ansible/posix/
4. Colección `community.crypto` (`openssh_keypair`) — https://docs.ansible.com/ansible/latest/collections/community/crypto/
5. BIND9 — Administrator Reference Manual — https://bind9.readthedocs.io/
6. Samba — `smb.conf(5)` — https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html
7. vsftpd — Documentación — https://security.appspot.com/vsftpd.html
8. Netfilter/iptables — Tutorial Oskar Andreasson — https://www.netfilter.org/documentation/HOWTO/NAT-HOWTO.html
9. Netplan — Referencia YAML — https://netplan.io/reference
10. NRPE — Nagios Remote Plugin Executor — https://nagios.org/downloads/nagios-core-addons/
11. Laboratorio 6 ASR — Monitorización de servicios en red con Nagios (Universitat Jaume I, Grado en Ingeniería Informática)
12. Repositorio del proyecto — `inventory/hosts.yml`, `playbooks/site.yml`, `roles/*` (en este mismo directorio)
