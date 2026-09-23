# Centralización de logs con rsyslog

## rsyslog con Podman y Rocky Linux

En este documento se explica cómo se realizó una centralización básica de logs usando rsyslog y Podman.

## Índice

| Sección | Contenido |
|---|---|
| Descripción del sistema | [Qué hace este sistema](#qué-hace-este-sistema) |
| Configuración del servidor | [Instalación del servidor](#instalación-del-servidor) |
| Configuración del cliente | [Configuración del cliente](#configuración-del-cliente-rocky-linux) |
| Identificación de fallos | [Diagnóstico rápido](#diagnóstico-rápido) |
| Administración básica | [Operación diaria](#operación-diaria) |

## Información general

| Campo | Detalle |
|---|---|
| Autor original | Emmanuel Tello Saldívar |
| Contexto | Servicio social - Área de Supercómputo |
| Estado | Prototipo funcional |
| Servidor | Podman rootless + rsyslog |
| Cliente | Rocky Linux + rsyslog nativo |
| Transporte recomendado | TCP, puerto 514 |

## Qué hace este sistema

El cliente toma los mensajes de `/var/log/messages` y los manda por red al servidor. El servidor los recibe y los guarda separados por equipo y por programa.

```text
Cliente Rocky Linux                 Servidor Linux

/var/log/messages
        |
        v
rsyslog nativo -- TCP:514 --> Podman + rsyslog
                                      |
                                      v
                    /srv/syslog_data/<host>/<programa>.log
```

De esta forma, los logs de varios equipos se pueden revisar desde una sola máquina. Al agregar un cliente nuevo, el servidor crea su carpeta automáticamente.

## Conceptos básicos

| Término | Explicación sencilla |
|---|---|
| Log | Archivo que registra eventos de un sistema o programa. |
| rsyslog | Servicio de Linux que recibe, procesa, guarda o reenvía logs. |
| Cliente | Equipo que envía sus logs. |
| Servidor | Equipo que recibe y almacena los logs. |
| Podman rootless | Forma de ejecutar contenedores con un usuario normal y no directamente como root. |
| Bind mount | Carpeta del host que también se puede usar dentro del contenedor. |
| SELinux | Sistema de seguridad que controla el acceso a archivos y procesos. |

## Requisitos

- Una máquina host con Podman instalado.
- Una VM o equipo cliente con Rocky Linux y `rsyslog` activo.
- Privilegios `sudo` en ambos equipos.
- Conexión de red entre cliente y servidor. En una VM se usa el modo Bridge o puente para que tenga una IP en la misma red que el host.
- `firewalld` disponible en el servidor.

**Importante:** `IP_DEL_SERVIDOR` debe sustituirse por la dirección IP real del servidor. Las direcciones de ejemplo no funcionan fuera del entorno donde fueron usadas.

## Arquitectura

| Componente | Tecnología | Función |
|---|---|---|
| Servidor | Podman rootless + rsyslog | Escucha el puerto 514 y escribe los logs recibidos. |
| Cliente | rsyslog de Rocky Linux | Lee `/var/log/messages` y lo reenvía. |
| Transporte | TCP 514 | Se usó TCP para reducir la pérdida de mensajes. |
| Persistencia | `/srv/syslog_data` | Guarda los logs fuera del contenedor. |
| Red | Bridge / puente | Da a la VM una IP accesible desde el host. |

## Instalación del servidor

Esta parte se realiza en la máquina que va a recibir los logs. Primero se comprueba que el servidor funcione y después se configura el cliente.

### 1. Crear el directorio persistente

Los logs se guardan fuera del contenedor para no perderlos cuando se cambie o elimine la imagen.

```bash
sudo mkdir -p /srv/syslog_data
sudo chmod 755 /srv/syslog_data
sudo chown -R "$(whoami)":"$(whoami)" /srv/syslog_data
```

El directorio debe pertenecer al usuario que ejecuta Podman. Esto permite que Podman aplique la etiqueta de SELinux al volumen.

### 2. Crear la configuración de rsyslog del servidor

```bash
sudo mkdir -p /etc/rsyslog-server
sudo tee /etc/rsyslog-server/rsyslog.conf > /dev/null <<'EOF'
module(load="imtcp")
input(type="imtcp" port="514")

# Descomenta estas dos líneas si también recibirás UDP.
# module(load="imudp")
# input(type="imudp" port="514")

template(name="RemoteLogs" type="string"
         string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")

*.* ?RemoteLogs
& stop
EOF

sudo chown "$(whoami)":"$(whoami)" /etc/rsyslog-server/rsyslog.conf
```

La plantilla `RemoteLogs` guarda los archivos con esta estructura:

```text
/var/log/remote/nombre-del-cliente/nombre-del-programa.log
```

Por ejemplo, los mensajes del equipo `rocky-vm` con la etiqueta `rocky-messages` se guardan en:

```text
/srv/syslog_data/rocky-vm/rocky-messages.log
```

### 3. Construir la imagen

Se crea una carpeta de trabajo con un archivo llamado `Containerfile`:

```bash
mkdir -p ~/rsyslog-container
cd ~/rsyslog-container
```

> El archivo de definición completo está disponible en [`Containerfile`](./Containerfile), en la raíz de este repositorio.

```Dockerfile
FROM rockylinux:9
RUN dnf install -y rsyslog && dnf clean all
EXPOSE 514/tcp 514/udp
CMD ["/usr/sbin/rsyslogd", "-n", "-f", "/etc/rsyslog.conf"]
```

Construye la imagen:

```bash
podman build -t rsyslog-server:latest .
```

La opción `-n` evita que rsyslog se ejecute en segundo plano. Así el contenedor se mantiene activo.

### 4. Permitir el puerto 514 en Podman rootless

El puerto 514 necesita permisos especiales porque es menor a 1024. Si Podman muestra un error de permisos al iniciar, se permite usar el puerto desde 514:

```bash
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=514
```

Para mantener este cambio después de reiniciar el equipo, se crea este archivo:

```bash
echo 'net.ipv4.ip_unprivileged_port_start=514' | sudo tee /etc/sysctl.d/99-rootless-syslog.conf
sudo sysctl --system
```

### 5. Iniciar el contenedor

```bash
podman run -d \
  --replace \
  --name rsyslog-server \
  --restart unless-stopped \
  -p 514:514/tcp \
  -v /srv/syslog_data:/var/log/remote:Z \
  -v /etc/rsyslog-server/rsyslog.conf:/etc/rsyslog.conf:Z,ro \
  rsyslog-server:latest
```

| Opción | Para qué sirve |
|---|---|
| `-d` | Ejecuta el contenedor en segundo plano. |
| `--replace` | Reemplaza un contenedor anterior con el mismo nombre. |
| `--restart unless-stopped` | Lo reinicia si falla o si el host se reinicia, salvo que se haya detenido manualmente. |
| `-p 514:514/tcp` | Publica el puerto TCP del contenedor en el host. |
| `:Z` | Reetiqueta el contenido para SELinux y permite el acceso del contenedor. |
| `ro` | Monta la configuración como solo lectura. |

### 6. Abrir el firewall del servidor

```bash
sudo firewall-cmd --permanent --add-port=514/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

En este proyecto se usa TCP. Si se quiere usar UDP también se debe abrir `514/udp` y habilitar `imudp` en la configuración.

### 7. Comprobar el servidor

```bash
podman ps
podman logs rsyslog-server
ss -tulnp | grep 514
logger -n 127.0.0.1 -P 514 -T 'prueba desde el host'
find /srv/syslog_data -type f
```

El contenedor debe aparecer como `Up`, el puerto 514 debe estar en escucha y el comando `find` debe mostrar un archivo de prueba.

## Configuración del cliente Rocky Linux

Esta parte se realiza en la VM o equipo que enviará sus logs al servidor.

### 1. Verificar la red

En la VM se revisa que pueda comunicarse con el host:

```bash
ip addr show
ping -c 3 IP_DEL_SERVIDOR
```

Si el ping falla, hay que revisar la red. En VirtualBox se configura la red como **Adaptador puente** y se selecciona la interfaz correcta del host.

### 2. Crear la regla de reenvío

Se crea el archivo `/etc/rsyslog.d/forward-messages.conf` con esta configuración:

```bash
sudo tee /etc/rsyslog.d/forward-messages.conf > /dev/null <<'EOF'
module(load="imfile" PollingInterval="10")

ruleset(name="forwardMessages") {
    action(type="omfwd" target="IP_DEL_SERVIDOR" port="514" protocol="tcp")
}

input(type="imfile"
      File="/var/log/messages"
      Tag="rocky-messages"
      Severity="info"
      Facility="local0"
      ruleset="forwardMessages")
EOF

sudo systemctl restart rsyslog
```

Se usa un `ruleset` separado para enviar solamente `/var/log/messages`. Así no se mandan todos los mensajes que procesa rsyslog.

### 3. Enviar una prueba

```bash
logger 'mensaje de prueba desde el cliente'
sudo journalctl -u rsyslog -n 10 --no-pager
```

En el servidor se revisa el archivo generado con estos comandos:

```bash
find /srv/syslog_data -type f
tail -f /srv/syslog_data/NOMBRE_DEL_CLIENTE/rocky-messages.log
```

## Operación diaria

### Iniciar, detener y reiniciar

```bash
podman start rsyslog-server
podman stop rsyslog-server
podman restart rsyslog-server
podman logs rsyslog-server
```

Para que el contenedor rootless pueda iniciar después de reiniciar el host, se habilita el servicio de usuario y el linger:

```bash
systemctl --user enable --now podman-restart.service
sudo loginctl enable-linger "$(whoami)"
```

### Añadir otro cliente

1. Verifica que rsyslog esté instalado y activo en el nuevo cliente.
2. Crea una regla igual a `forward-messages.conf`, usando la IP del mismo servidor.
3. Reinicia rsyslog y envía un mensaje con `logger`.
4. Comprueba que aparezca una carpeta nueva dentro de `/srv/syslog_data`.

No se necesita cambiar el servidor. La plantilla crea una carpeta para cada hostname.

### Mantenimiento mínimo

- Se puede agregar `logrotate` para que `/srv/syslog_data` no crezca sin límite.
- Los respaldos se realizan sobre `/srv/syslog_data`, ya que ahí se guardan los logs.
- Para actualizar la imagen, se vuelve a ejecutar `podman build` y `podman run` con `--replace`.
- En un entorno de producción se recomienda usar TLS, porque TCP no cifra la información.

## Diagnóstico rápido

| Síntoma | Causa probable | Comprobación | Solución inicial |
|---|---|---|---|
| `No route to host` | Firewall del host bloquea el puerto. | `firewall-cmd --list-ports` en el servidor. | Abre `514/tcp` y recarga firewalld. |
| `Connection refused` | El host responde, pero no hay proceso escuchando. | `podman ps -a` y `ss -tulnp \| grep 514`. | Inicia o revisa el contenedor. |
| `Permission denied` al publicar el puerto | Podman rootless no puede usar 514. | Revisa el error de `podman run`. | Ajusta `net.ipv4.ip_unprivileged_port_start`. |
| `lsetxattr: operation not permitted` | Podman no puede reetiquetar un volumen SELinux. | `ls -ld /srv/syslog_data` y `ls -l` del archivo de configuración. | Se asigna la propiedad al usuario que ejecuta Podman y se usa `:Z`. |
| No llega ningún archivo | Red, firewall o regla del cliente. | Ping, `journalctl -u rsyslog` y `podman logs`. | Sigue la lista de verificación inferior. |
| Lectura o configuración inválida | Archivo con typo o caracteres extraños. | `sudo rsyslogd -N1` y `cat -A` del archivo. | Corrige el nombre y recrea el archivo. |

## Lista de verificación

- [ ] El contenedor aparece como `Up` en `podman ps`.
- [ ] El servidor escucha TCP 514.
- [ ] Firewalld permite `514/tcp`.
- [ ] El cliente alcanza la IP del servidor.
- [ ] `rsyslog` no reporta errores en el cliente.
- [ ] El archivo de log aparece en `/srv/syslog_data`.

## Seguridad

Este prototipo envía los logs por TCP sin cifrado. Para usarlo en un entorno real se deben considerar estas medidas:

- Habilitar TLS entre cliente y servidor.
- Limitar con firewall las IP que pueden conectarse al puerto 514.
- Revisar los archivos enviados, ya que los logs pueden contener usuarios, rutas, IP o datos de aplicaciones.
- Definir retención, rotación y respaldos según las necesidades del sistema.

## Resultado esperado

Al terminar la configuración, cada cliente crea archivos separados en el servidor:

```text
/srv/syslog_data/
├── rocky-vm-01/
│   └── rocky-messages.log
└── rocky-vm-02/
    └── rocky-messages.log
```

Con esta estructura los logs de varios equipos se pueden revisar y respaldar desde un solo lugar. Los archivos no se pierden al recrear el contenedor.
