# Glosario y uso básico de Ansible

| Campo | Detalle |
|---|---|
| Autor | Emmanuel Tello Saldívar |
| Contexto | Servicio social - Área de Supercómputo |
| Actividad | 4. Revisión de Ansible para aprovisionamiento de sistemas GNU/Linux |
| Producto | Glosario de conceptos y documentación de uso básico de Ansible |
| Última actualización | 15 de octubre de 2026 |

## Índice

1. [Qué es Ansible](#1-qué-es-ansible)
2. [Glosario de conceptos](#2-glosario-de-conceptos)
3. [Uso básico paso a paso](#3-uso-básico-paso-a-paso)
4. [Módulos más usados](#4-módulos-más-usados)
5. [Opciones frecuentes de la línea de comandos](#5-opciones-frecuentes-de-la-línea-de-comandos)
6. [Problemas frecuentes y solución](#6-problemas-frecuentes-y-solución)
7. [Prácticas realizadas](#7-prácticas-realizadas)

---

## 1. Qué es Ansible

Ansible es una herramienta de automatización que permite describir el **estado deseado** de uno o varios sistemas (paquetes instalados, archivos, servicios, usuarios, reglas de firewall) y aplicarlo de forma repetible.

Características principales:

- **Sin agente:** las máquinas administradas no necesitan un programa de Ansible instalado. Basta con SSH y Python.
- **Declarativo:** se describe *cómo debe quedar* el sistema, no los comandos para lograrlo.
- **Idempotente:** ejecutar el mismo playbook varias veces deja el sistema en el mismo estado final, sin repetir trabajo ni romper nada.
- **Legible:** los playbooks se escriben en YAML y sirven también como documentación.

Diferencia con un script de shell: un script ejecuta sus comandos a ciegas cada vez; Ansible primero compara el estado actual con el deseado y solo actúa si hay diferencias.

## 2. Glosario de conceptos

### 2.1 Arquitectura

| Término | Definición |
|---|---|
| **Nodo de control** | Máquina desde donde se ejecuta Ansible. Debe tener `ansible-core` instalado. |
| **Nodo administrado (managed node)** | Máquina que Ansible configura. Solo necesita SSH y Python. |
| **Inventario (inventory)** | Archivo que lista los nodos administrados y los organiza en grupos. |
| **Grupo** | Conjunto de hosts con un nombre (por ejemplo `[log_clients]`). Los playbooks y las variables se asignan por grupo. |
| **Conexión local** | Modo `ansible_connection=local`: Ansible ejecuta las tareas en la misma máquina, sin SSH. |
| **Facts** | Información que Ansible recopila de cada host al iniciar (sistema operativo, IPs, memoria). Se obtiene en la tarea "Gathering Facts". |

### 2.2 Estructura de trabajo

| Término | Definición |
|---|---|
| **Módulo** | Unidad de trabajo reutilizable (copiar un archivo, instalar un paquete, administrar un servicio). Ansible lo envía al destino, lo ejecuta y recibe el resultado. |
| **Tarea (task)** | Una llamada a un módulo con sus parámetros. |
| **Play** | Conjunto de tareas (o roles) que se aplican a un grupo de hosts. |
| **Playbook** | Archivo YAML con una lista de plays. Es la unidad que se ejecuta con `ansible-playbook`. |
| **Rol (role)** | Estructura de carpetas estándar que agrupa tareas, plantillas, variables y handlers de una función concreta, para reutilizarla. |
| **Handler** | Tarea especial que solo se ejecuta si otra tarea la notifica (`notify`) y reportó un cambio. Se ejecuta una vez, al final del play. |
| **Colección (collection)** | Paquete que distribuye módulos, roles y plugins (por ejemplo `ansible.posix`, `containers.podman`). |
| **Tag (etiqueta)** | Marca que se asigna a tareas para ejecutar solo una parte del playbook (`--tags`). |
| **Idempotencia** | Propiedad por la cual una tarea puede ejecutarse varias veces con el mismo resultado final. |

### 2.3 Estados de resultado de una tarea

| Estado | Significado |
|---|---|
| `ok` | La tarea se ejecutó y el sistema ya estaba en el estado deseado. No cambió nada. |
| `changed` | La tarea modificó el sistema. |
| `failed` | La tarea falló. Detiene el play en ese host. |
| `skipped` | La tarea se omitió por una condición. |
| `unreachable` | No se pudo conectar con el host. |

### 2.4 Variables y plantillas

| Término | Definición |
|---|---|
| **Variable** | Valor reutilizable referenciado con `{{ nombre }}`. |
| **`group_vars/`** | Carpeta cuyos archivos definen variables para un grupo del inventario (`group_vars/log_clients.yml`). |
| **`defaults/`** | Variables de un rol con la precedencia más baja; pensadas para sobrescribirse. |
| **Precedencia** | Orden en que se resuelve una variable definida en varios lugares. De menor a mayor: `defaults` del rol, `group_vars/all`, `group_vars/<grupo>`, `-e` en la línea de comandos. |
| **Jinja2** | Motor de plantillas que usa Ansible. `{{ }}` inserta un valor; `{% %}` ejecuta lógica (`if`, `for`). |
| **Plantilla (template)** | Archivo `.j2` con variables y lógica que el módulo `template` convierte en un archivo final. |
| **`register`** | Guarda el resultado de una tarea en una variable para usarlo después (por ejemplo, `containerfile is changed`). |
| **`loop`** | Repite una tarea por cada elemento de una lista; el elemento actual es `item`. |
| **`ansible_managed`** | Variable que inserta un comentario avisando que el archivo lo genera Ansible. |

### 2.5 Seguridad y privilegios

| Término | Definición |
|---|---|
| **`become`** | Escalada de privilegios (equivale a usar `sudo`). Puede ponerse a nivel de play o de tarea. |
| **`--ask-become-pass` / `-K`** | Pide la contraseña de sudo antes de ejecutar. |
| **Llave SSH** | Par de llaves (pública y privada) para autenticarse sin contraseña. La pública se copia al nodo con `ssh-copy-id`. |
| **`NOPASSWD`** | Opción de `sudoers` que permite usar sudo sin contraseña; se configura en `/etc/sudoers.d/` con `visudo`. |

### 2.6 Lenguaje YAML

| Elemento | Regla |
|---|---|
| Indentación | Se usan **espacios**, nunca tabuladores. Define la jerarquía. |
| Lista | Cada elemento inicia con `- `. Un playbook siempre es una lista de plays. |
| Diccionario | Pares `clave: valor`. |
| Cadenas | Entre comillas cuando contienen caracteres especiales o `{{ }}`. |
| Comentarios | Inician con `#`. |

## 3. Uso básico paso a paso

### 3.1 Instalación en el nodo de control

```bash
sudo dnf install -y ansible-core
ansible --version
```

Si un playbook usa módulos fuera del núcleo, se instalan las colecciones:

```bash
ansible-galaxy collection install ansible.posix containers.podman
```

### 3.2 Preparar el acceso por SSH con llave

```bash
ssh-keygen -t ed25519 -C "ansible-lab"
ssh-copy-id USUARIO@IP_DEL_NODO
ssh USUARIO@IP_DEL_NODO "hostname"    # no debe pedir contraseña
```

El `USUARIO` debe ser exactamente el nombre de usuario del nodo administrado (Linux distingue mayúsculas y minúsculas).

### 3.3 Crear el inventario

Archivo `inventory.ini`:

```ini
[rocky_clients]
rocky-vm ansible_host=IP_DEL_NODO ansible_user=USUARIO
```

### 3.4 Probar la conexión

```bash
ansible -i inventory.ini rocky_clients -m ping
```

Resultado esperado: `"ping": "pong"`. Este módulo no es un ping de red: comprueba que SSH y Python funcionan.

### 3.5 Configurar sudo sin contraseña en el nodo administrado

Si el playbook usa `become: true` y no se pasa `-K`, debe configurarse sudo sin contraseña para el usuario de Ansible:

```bash
sudo visudo -f /etc/sudoers.d/ansible-nopasswd
# contenido:
# USUARIO ALL=(ALL) NOPASSWD: ALL
```

`visudo` valida la sintaxis antes de guardar, lo que evita dejar el sistema sin sudo.

### 3.6 Escribir un playbook

Archivo `primer-playbook.yml`:

```yaml
- name: Primer playbook de prueba
  hosts: rocky_clients
  become: true

  tasks:
    - name: Asegurar que tree esté instalado
      ansible.builtin.dnf:
        name: tree
        state: present

    - name: Crear un archivo con contenido fijo
      ansible.builtin.copy:
        content: "Archivo creado por Ansible.\n"
        dest: /tmp/prueba-ansible.txt
        mode: '0644'
```

Puntos de sintaxis: el archivo inicia con `-`; `name`, `hosts`, `become` y `tasks` están alineados; cada tarea inicia con `-` y sus parámetros van un nivel más adentro.

### 3.7 Validar y ejecutar

```bash
ansible-playbook -i inventory.ini primer-playbook.yml --syntax-check
ansible-playbook -i inventory.ini primer-playbook.yml
```

### 3.8 Comprobar la idempotencia

Se ejecuta el mismo playbook una segunda vez sin modificarlo. La primera ejecución muestra `changed` y la segunda `ok`, lo que confirma que Ansible detecta que el sistema ya está en el estado deseado.

### 3.9 Handlers

```yaml
tasks:
  - name: Desplegar página principal
    ansible.builtin.template:
      src: templates/index.html.j2
      dest: /var/www/html/index.html
    notify: Reiniciar httpd

handlers:
  - name: Reiniciar httpd
    ansible.builtin.systemd:
      name: httpd
      state: restarted
```

El servicio se reinicia solo cuando la plantilla produjo un cambio real. Si el archivo no cambia, el handler no se ejecuta.

## 4. Módulos más usados

| Módulo | Función | Ejemplo de parámetros |
|---|---|---|
| `ansible.builtin.ping` | Prueba conexión SSH y Python | - |
| `ansible.builtin.dnf` | Instala o elimina paquetes | `name: tree`, `state: present` |
| `ansible.builtin.copy` | Copia un archivo o escribe contenido fijo | `src`, `dest`, `mode` |
| `ansible.builtin.template` | Genera un archivo desde una plantilla Jinja2 | `src`, `dest`, `owner`, `mode` |
| `ansible.builtin.file` | Crea directorios, ajusta permisos y dueño | `path`, `state: directory`, `owner` |
| `ansible.builtin.user` | Gestiona usuarios | `name`, `group`, `shell` |
| `ansible.builtin.group` | Gestiona grupos | `name`, `state: present` |
| `ansible.builtin.systemd` | Inicia, detiene o habilita servicios | `name`, `state: started`, `enabled: true` |
| `ansible.builtin.command` | Ejecuta un comando sin shell | `rsyslogd -N1` |
| `ansible.builtin.wait_for` | Espera a que un puerto o archivo esté disponible | `port: 514`, `timeout: 30` |
| `ansible.posix.firewalld` | Administra reglas de firewalld | `port: 514/tcp`, `permanent: true` |
| `ansible.posix.sysctl` | Gestiona parámetros del kernel | `name`, `value`, `sysctl_file` |
| `containers.podman.podman_image` | Construye o descarga imágenes | `name`, `path` |
| `containers.podman.podman_container` | Crea y gestiona contenedores | `name`, `image`, `publish`, `volume` |

Los módulos de `ansible.posix` y `containers.podman` pertenecen a colecciones que deben instalarse aparte.

## 5. Opciones frecuentes de la línea de comandos

| Opción | Función |
|---|---|
| `-i ARCHIVO` | Indica el inventario (innecesaria si `ansible.cfg` lo define). |
| `-m MODULO` | Con el comando `ansible`, ejecuta un solo módulo. |
| `-K` / `--ask-become-pass` | Solicita la contraseña de sudo. |
| `--syntax-check` | Valida la sintaxis sin ejecutar nada. |
| `--check` | Simula la ejecución sin aplicar cambios. Algunos módulos no admiten simulación. |
| `--diff` | Muestra las diferencias de los archivos que cambian. |
| `--tags` / `--skip-tags` | Ejecuta u omite tareas por etiqueta. |
| `--limit HOST` | Restringe la ejecución a ciertos hosts o grupos. |
| `-e '{"var": "valor"}'` | Sobrescribe variables desde la línea de comandos. |
| `-v`, `-vv`, `-vvv` | Aumenta el nivel de detalle para diagnosticar. |
| `--list-tasks` / `--list-hosts` | Lista tareas u hosts sin ejecutar. |

Diferencia entre comandos: `ansible` ejecuta **un módulo suelto** (pruebas rápidas); `ansible-playbook` ejecuta **un playbook completo**.

## 6. Problemas frecuentes y solución

| Síntoma | Causa | Solución |
|---|---|---|
| `Permission denied` al usar `ssh-copy-id` aunque la contraseña es correcta | El nombre de usuario no coincide (mayúsculas o nombre distinto) | Verificar el usuario real con `whoami` en el nodo y usarlo en `ssh-copy-id` y en `ansible_user` |
| `Missing sudo password` | El playbook usa `become` y el usuario no tiene sudo sin contraseña | Pasar `-K` o configurar `NOPASSWD` con `visudo` |
| Error de sintaxis en la primera línea | El playbook no inicia con `- ` | Iniciar el play con `- name:` |
| `hosts`, `become` o `tasks` no se reconocen | Indentación incorrecta tras agregar el guion | Alinearlos con `name`, con 2 espacios |
| La tarea falla al crear un archivo en `/temp` | Ruta inexistente; el directorio estándar es `/tmp` | Corregir la ruta |
| `couldn't resolve module/action` | Falta instalar una colección | `ansible-galaxy collection install ...` |
| Ansible ignora `ansible.cfg` | La carpeta tiene permisos de escritura para todos | `chmod o-w .` |
| `module` de firewalld no encontrado | `ansible.posix` no instalada | Instalar la colección |

## 7. Prácticas realizadas

1. **Conexión:** configuración de SSH con llave y prueba con el módulo `ping` contra la VM Rocky Linux.
2. **Primer playbook:** instalación de un paquete y creación de un archivo, con verificación de idempotencia (`changed` → `ok`).
3. **Playbook de servidor web:** creación de grupo y usuario, instalación de paquetes, despliegue de una página desde plantilla Jinja2, servicio con systemd, regla de firewalld y un handler.
4. **Proyecto con roles:** aprovisionamiento del contenedor rsyslog y de los clientes (ver `DocumentacionAnsibleRsyslog.md`).
