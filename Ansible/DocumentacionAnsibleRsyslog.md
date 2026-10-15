# Aprovisionamiento con Ansible del servidor de logs (rsyslog + Podman)

| Campo | Detalle |
|---|---|
| Autor | Emmanuel Tello Saldívar |
| Contexto | Servicio social - Área de Supercómputo |
| Actividad | 7. Aprovisionamiento con Ansible de la configuración del contenedor de bitácoras |
| Base | Contenedor documentado en `DocumentacioLogs.md` (actividad 6) |
| Conceptos | Ver `GlosarioAnsible.md` (actividad 4) |
| Última actualización | 15 de octubre de 2026 |
| Estado | Sintaxis y plantillas validadas; falta registrar la ejecución en el entorno real |

## Índice

1. [Objetivo](#1-objetivo)
2. [Arquitectura](#2-arquitectura)
3. [Requisitos](#3-requisitos)
4. [Estructura del proyecto](#4-estructura-del-proyecto)
5. [Variables](#5-variables)
6. [Funcionamiento de los roles](#6-funcionamiento-de-los-roles)
7. [Puesta en marcha](#7-puesta-en-marcha)
8. [Verificación](#8-verificación)
9. [Operación y cambios comunes](#9-operación-y-cambios-comunes)
10. [Diagnóstico](#10-diagnóstico)
11. [Limitaciones conocidas](#11-limitaciones-conocidas)
12. [Seguridad](#12-seguridad)

---

## 1. Objetivo

Automatizar con Ansible los pasos que en la actividad 6 se hicieron manualmente:

- Preparar el servidor: parámetro del kernel, directorios, configuración de rsyslog, imagen, contenedor Podman y firewall.
- Configurar el reenvío de `/var/log/messages` en los clientes Rocky Linux.

Beneficios: el procedimiento es repetible, se versiona en Git, es idempotente y permite agregar clientes o cambiar el puerto modificando solo variables.

## 2. Arquitectura

```text
 Máquina principal (nodo de control + servidor de logs)
 ┌──────────────────────────────────────────────────────────┐
 │ ansible-playbook site.yml                                │
 │                                                          │
 │ Play 1 ── conexión local ──► esta misma máquina          │
 │           sysctl, directorios, Podman, firewalld         │
 │                                                          │
 │ Play 2 ── SSH con llave ───► VM Rocky Linux (cliente)    │
 │           rsyslog + regla de reenvío                     │
 └──────────────────────────────────────────────────────────┘

 Flujo de un log:
 /var/log/messages (VM) → imfile → omfwd → TCP 514 → firewalld
   → contenedor rsyslog (imtcp) → plantilla RemoteLogs
   → /srv/syslog_data/<host>/<programa>.log
```

| Componente | Máquina | Conexión de Ansible |
|---|---|---|
| Servidor de logs (contenedor Podman rootless) | Máquina principal | `local` |
| Cliente (rsyslog nativo) | VM Rocky Linux | SSH con llave |

## 3. Requisitos

**Nodo de control (máquina principal):**

- `ansible-core` instalado.
- Colecciones `ansible.posix` y `containers.podman`.
- Podman instalado y firewalld activo.
- Se ejecuta con el **usuario normal** (no con `sudo ansible-playbook`) y con `-K` para la contraseña de sudo.

**Nodo administrado (VM):**

- Acceso SSH con llave para el usuario indicado en el inventario.
- Sudo sin contraseña para ese usuario (`/etc/sudoers.d/`) o ejecución con `-K`.
- Python 3.

```bash
sudo dnf install -y ansible-core
ansible-galaxy collection install -r requirements.yml
```

## 4. Estructura del proyecto

```text
ansible-proj/
├── ansible.cfg                  Configuración del proyecto
├── inventory.ini                Servidor (local) y clientes (VM)
├── requirements.yml             Colecciones necesarias
├── site.yml                     Playbook principal
├── group_vars/
│   ├── all.yml                  IP del servidor, puerto, protocolos
│   ├── log_servers.yml          Usuario de Podman
│   └── log_clients.yml          Etiqueta y archivo a reenviar
└── roles/
    ├── rsyslog_server/
    │   ├── defaults/main.yml    Nombres, rutas y puertos publicados
    │   ├── files/Containerfile  Definición de la imagen
    │   ├── templates/rsyslog.conf.j2
    │   ├── handlers/main.yml    Recrea el contenedor si cambia la config
    │   └── tasks/main.yml
    └── rsyslog_client/
        ├── defaults/main.yml
        ├── templates/forward-messages.conf.j2
        ├── handlers/main.yml    Reinicia rsyslog
        └── tasks/main.yml
```

### 4.1 `ansible.cfg`

```ini
[defaults]
inventory           = inventory.ini
roles_path          = roles
retry_files_enabled = False
interpreter_python  = auto_silent
```

### 4.2 `inventory.ini`

```ini
[log_servers]
log-server ansible_connection=local

[log_clients]
rocky-vm ansible_host=IP_DE_LA_VM ansible_user=USUARIO_DE_LA_VM
```

### 4.3 `site.yml`

```yaml
- name: Servidor central de logs (contenedor Podman rootless)
  hosts: log_servers
  roles:
    - rsyslog_server

- name: Clientes que reenvían /var/log/messages al servidor
  hosts: log_clients
  become: true
  roles:
    - rsyslog_client
```

El play del servidor **no usa `become` a nivel de play**: Podman rootless debe ejecutarse como el usuario normal. Solo las tareas que requieren root lo declaran individualmente.

## 5. Variables

| Variable | Archivo | Valor por defecto | Función |
|---|---|---|---|
| `rsyslog_server_ip` | `group_vars/all.yml` | IP de la máquina principal | Destino al que reenvían los clientes |
| `rsyslog_port` | `group_vars/all.yml` | `514` | Puerto de escucha y de reenvío |
| `rsyslog_protocols` | `group_vars/all.yml` | `[tcp]` | Protocolos habilitados (`tcp`, `udp`) |
| `podman_user` | `group_vars/log_servers.yml` | usuario que ejecuta Ansible | Dueño de directorios y del contenedor |
| `rsyslog_client_tag` | `group_vars/log_clients.yml` | `rocky-messages` | Etiqueta (determina el nombre del archivo en el servidor) |
| `rsyslog_client_file` | `group_vars/log_clients.yml` | `/var/log/messages` | Archivo que se reenvía |
| `container_name` | `defaults` del rol servidor | `rsyslog-server` | Nombre del contenedor |
| `image_name` / `image_tag` | `defaults` del rol servidor | `rsyslog-server` / `latest` | Imagen del contenedor |
| `data_dir` | `defaults` del rol servidor | `/srv/syslog_data` | Directorio persistente de logs |
| `conf_dir` | `defaults` del rol servidor | `/etc/rsyslog-server` | Directorio de la configuración |
| `build_dir` | `defaults` del rol servidor | `~/rsyslog-container` | Contexto de construcción de la imagen |
| `rsyslog_publish` | `defaults` del rol servidor | derivada | Lista de puertos publicados (`514:514/tcp`) |

Precedencia, de menor a mayor: `defaults` del rol → `group_vars/all` → `group_vars/<grupo>` → `-e`.

`rsyslog_publish` se calcula a partir de `rsyslog_protocols`:

```yaml
rsyslog_publish: "{{ rsyslog_protocols | map('regex_replace', '^', rsyslog_port ~ ':' ~ rsyslog_port ~ '/') | list }}"
```

Con `[tcp]` produce `["514:514/tcp"]`; con `[tcp, udp]` produce `["514:514/tcp", "514:514/udp"]`.

## 6. Funcionamiento de los roles

### 6.1 Rol `rsyslog_server`

| # | Tarea | Módulo | Root | Problema que resuelve |
|---|---|---|---|---|
| 1 | Permitir el puerto 514 a Podman rootless | `ansible.posix.sysctl` | Sí | `Permission denied` al publicar el puerto |
| 2 | Crear `data_dir` y `conf_dir` con el usuario de Podman como dueño | `ansible.builtin.file` | Sí | `lsetxattr: operation not permitted` |
| 3 | Crear `build_dir` | `ansible.builtin.file` | No | Orden |
| 4 | Desplegar `rsyslog.conf` desde plantilla | `ansible.builtin.template` | Sí | Configuración del servidor (notifica el handler) |
| 5 | Copiar el `Containerfile` | `ansible.builtin.copy` | No | Definición de la imagen |
| 6 | Construir la imagen | `containers.podman.podman_image` | No | `podman build` |
| 7 | Iniciar el contenedor | `containers.podman.podman_container` | No | `podman run` |
| 8 | Abrir el puerto en firewalld | `ansible.posix.firewalld` | Sí | `No route to host` |
| 9 | Esperar a que el puerto escuche | `ansible.builtin.wait_for` | No | Verificación |

Detalles de diseño:

- **Reconstrucción condicional:** la tarea 5 usa `register: containerfile` y la tarea 6 usa `force: "{{ containerfile is changed }}"`. La imagen solo se reconstruye si el `Containerfile` cambió.
- **Recreación condicional:** la tarea 7 usa `recreate: "{{ imagen is changed }}"`. El contenedor se recrea si la imagen cambió.
- **Handler:** al cambiar `rsyslog.conf` se notifica `Recrear contenedor rsyslog`, que lo vuelve a crear con la nueva configuración.
- **Plantilla `rsyslog.conf.j2`:** incluye los módulos `imtcp` e `imudp` según `rsyslog_protocols`, y la plantilla de rutas `RemoteLogs`.

```jinja
{% if 'tcp' in rsyslog_protocols %}
module(load="imtcp")
input(type="imtcp" port="{{ rsyslog_port }}")
{% endif %}
```

### 6.2 Rol `rsyslog_client`

| # | Tarea | Módulo | Función |
|---|---|---|---|
| 1 | Asegurar rsyslog instalado | `ansible.builtin.dnf` | Dependencia |
| 2 | Habilitar e iniciar rsyslog | `ansible.builtin.systemd` | Servicio activo |
| 3 | Desplegar `forward-messages.conf` | `ansible.builtin.template` | Regla de reenvío (notifica el handler) |
| 4 | Validar la sintaxis | `ansible.builtin.command` (`rsyslogd -N1`) | Evita reiniciar con una configuración inválida |
| handler | Reiniciar rsyslog | `ansible.builtin.systemd` | Se ejecuta solo si hubo cambio |

La validación (tarea 4) se ejecuta **antes** del handler: si la configuración es inválida, la tarea falla y rsyslog no se reinicia. El archivo defectuoso queda en disco y debe corregirse antes de reiniciar el servicio manualmente.

### 6.3 Etiquetas (tags)

| Tag | Tareas |
|---|---|
| `server` / `client` | Todo el rol correspondiente |
| `sistema` | sysctl y directorios |
| `imagen` | Containerfile e imagen |
| `configuracion` | Plantillas (servidor y cliente) |
| `contenedor` | Creación del contenedor |
| `firewall` | Reglas de firewalld |
| `verificacion` | Espera del puerto y validación de rsyslog |
| `paquetes`, `servicio` | Instalación y servicio del cliente |

## 7. Puesta en marcha

1. Clonar o copiar el proyecto en la máquina principal.
2. En `inventory.ini`, escribir la IP y el usuario reales de la VM.
3. En `group_vars/all.yml`, confirmar que `rsyslog_server_ip` es la IP **actual** de la máquina principal.
4. Instalar las colecciones:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```
5. Validar y ejecutar:
   ```bash
   ansible-playbook site.yml --syntax-check
   ansible-playbook site.yml --ask-become-pass
   ```
6. Ejecutar una segunda vez para comprobar la idempotencia.

Si `ansible.cfg` es ignorado porque la carpeta tiene permisos de escritura para todos, ejecutar `chmod o-w .`.

## 8. Verificación

**Servidor (máquina principal):**

```bash
podman ps
podman logs rsyslog-server
ss -tulnp | grep 514
sudo firewall-cmd --list-ports
```

**Cliente (VM):**

```bash
logger "prueba desde Ansible"
sudo journalctl -u rsyslog -n 10 --no-pager
```

**Resultado en el servidor:**

```bash
find /srv/syslog_data -type f -newer /tmp -mmin -5
```

Debe existir `/srv/syslog_data/<hostname-de-la-vm>/rocky-messages.log` con el mensaje.

**Idempotencia esperada:**

| Ejecución | Resultado |
|---|---|
| Primera | Varias tareas en `changed` |
| Segunda, sin modificar nada | `changed=0` |
| Tras modificar una variable | Cambian solo las tareas afectadas |

## 9. Operación y cambios comunes

### 9.1 Cambiar el puerto

1. Editar `rsyslog_port` en `group_vars/all.yml`.
2. Ejecutar `ansible-playbook site.yml -K`.

Cambian la plantilla del servidor, el sysctl, el contenedor (puertos publicados), firewalld y la regla de los clientes.

### 9.2 Habilitar UDP

```yaml
rsyslog_protocols:
  - tcp
  - udp
```

El servidor publica y abre ambos protocolos. Los clientes siguen usando el **primer protocolo** de la lista.

### 9.3 Cambiar la IP del servidor

Editar `rsyslog_server_ip` y ejecutar de nuevo. Solo cambia la regla de los clientes.

### 9.4 Agregar un cliente

1. Agregar una línea en `[log_clients]` del inventario:
   ```ini
   rocky-vm-02 ansible_host=IP ansible_user=USUARIO
   ```
2. Configurar SSH con llave y sudo en el nuevo nodo.
3. Ejecutar solo sobre el nuevo cliente:
   ```bash
   ansible-playbook site.yml --limit rocky-vm-02
   ```
   El play del servidor se omite porque ningún host coincide con el límite.

El servidor no requiere cambios: la plantilla `RemoteLogs` crea una carpeta por cada hostname.

### 9.5 Ejecutar solo una parte

```bash
ansible-playbook site.yml -K --tags configuracion
ansible-playbook site.yml -K --tags client
```

## 10. Diagnóstico

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Missing sudo password` | Falta `-K` en la máquina principal | Ejecutar con `-K` |
| `couldn't resolve module/action 'containers.podman...'` | Colecciones no instaladas | `ansible-galaxy collection install -r requirements.yml` |
| `Permission denied (publickey...)` | `ansible_user` incorrecto | Corregir el inventario |
| `lsetxattr: operation not permitted` | El dueño de los directorios no es el usuario de Podman (por ejemplo, se ejecutó con `sudo`) | Ejecutar como usuario normal y revisar `podman_user` |
| El contenedor no aparece en `podman ps` | Se creó bajo otro usuario (root) | Ejecutar sin `sudo`; revisar con `sudo podman ps` |
| `No route to host` desde la VM | El puerto no está abierto en firewalld | Revisar `firewall-cmd --list-ports` |
| `Connection refused` desde la VM | El contenedor no está corriendo | `podman ps -a` y `podman start rsyslog-server` |
| Falla `rsyslogd -N1` en el cliente | Plantilla o variables incorrectas | Revisar `/etc/rsyslog.d/forward-messages.conf` |
| No llegan logs aunque todo está en `ok` | IP del servidor desactualizada (DHCP) | Actualizar `rsyslog_server_ip` |

Para más detalle de cualquier tarea: `ansible-playbook site.yml -K -vv`.

## 11. Limitaciones conocidas

1. **Doble recreación en la primera ejecución:** el contenedor se crea en la tarea 7 y el handler lo recrea al final porque la plantilla cambió. Es redundante pero no pierde logs, que viven en `data_dir`. Puede evitarse con `meta: flush_handlers` tras desplegar la configuración.
2. **`wait_for` se ejecuta antes de los handlers**, por lo que no verifica el estado tras una recreación por handler.
3. **Los clientes usan solo el primer protocolo** de `rsyslog_protocols`.
4. **La IP del servidor se escribe a mano.** Con DHCP puede cambiar; conviene una reserva en el router o una IP estática.
5. **Sin cifrado:** el transporte es TCP sin TLS.
6. **Sin rotación de logs** en `data_dir`.
7. **Sin ejecución verificada en el entorno real** hasta registrar la salida de las corridas de la sección 7.

## 12. Seguridad

- No guardar contraseñas ni llaves privadas en el repositorio. Si se usan secretos, emplear Ansible Vault.
- `NOPASSWD: ALL` en la VM es adecuado para un laboratorio; en producción debe limitarse a los comandos necesarios.
- En la máquina principal se recomienda mantener `-K` en lugar de sudo sin contraseña.
- Habilitar TLS entre clientes y servidor antes de usar el sistema fuera de un entorno de pruebas.
- Restringir con firewalld las IP que pueden conectarse al puerto 514.
- Los logs pueden contener usuarios, rutas e IP: definir retención y permisos de `data_dir`.
- Sugerencia de `.gitignore`: archivos `*.retry`, llaves privadas y cualquier archivo de contraseñas de Vault.
