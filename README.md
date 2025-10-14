# Tutorial de como configurar wireguard

En esta guía vamos a configurar wireguard para que corra en una Maquina Virtual con Alpine Linux.

## Mi Setup

- **Host**: PC con Arch Linux
- **Guest**: Vm con Alpine Linux

## Paso 1) Descargar un Hipervisor.

Para crear máquinas virtuales, necesitamos un Hipervisor, para este proyecto voy a usar VirtualBox.

- [Descargar VirtualBox para Arch](https://wiki.archlinux.org/title/VirtualBox)
- [Descargar VirtualBox para otros Sistemas Linux](https://www.virtualbox.org/wiki/Linux_Downloads)
- [Descargar VirtualBox para Windows](https://www.virtualbox.org/wiki/Downloads)

## Paso 2) Descargar imagen de Alpine Linux

En la [Página oficial de Alpie Linux](https://www.alpinelinux.org/downloads/) encontramos las Imágenes de esta distribución.

Para nuestro caso de uso, usaramos la versión **x86_64** de la sección **"VIRTUAL"**.

![Descarga de Alpine Linux - Sección Virtual](images/image.png)


## Paso 3) Crear una Máquina Virtual en el Hipervisor

Creamos la máquina virtual con las siguientes especificaciones

![Configuracion VM 1](images/ConfiguracionVM1.png)

![Configuracion VM 2](images/ConfiguracionVM2.png)

![Configuracion VM 3](images/ConfiguracionVM3.png)

![Configuracion Network](images/ConfiguracionVM4.png)

## Paso 4) Configuraciones básica en Alpine.

### Iniciar Sesión
Si no configuramos ninguna credencial de inicio de sesión, se nos darán las siguientes credenciales como default

**user**: root (Sin passwword)


### Setup-Alpine

Para una configuración guiada de alpine, vamos a ejecutar el comando

```bash
setup-alpine
```

Las configuraciones que yo seleccione fueron:

- KeyboardLayout: latam-deadtile
- Hostname: vpnalpine
- Interface: 
    - dhcp
    - n
- Root Password: ***********
- TimeZone: America/Argentina/Buenos_Aires
- Proxy: none
- Network Time Protocol: busybox
- Apk Mirror: **r** (Use Random Mirror)
- User: porky
    - Which ssh server: openssh
- Disk & Install: 
    - Which disk would you like to use: sda (O el disco que aparezca)
    - How would you like to use it: sys
    - Erease the above disk and continue? (y/n): y

Apagamos la máquina Virtual y desde la interfaz visual seleccionamos la siguiente opción.

Vm -> Configuración -> Almacenamiento -> Controller: IDE -> Remove Attachement. 

[TODO : Agregar una foto de esto]

Esto hará que la VM no boote desde la iso, sino que use el almacenamiento que le configuramos

### Actualizar Paquetes

Alpine cuenta con su propio package manager llamado [apk](https://wiki.alpinelinux.org/wiki/Alpine_Package_Keeper), en ese link se encuentra una lista de todos los comandos de apk. 
Para actualizar los paquetes debemos ejecutar los siguientes comandos: 

```bash
apk update
apk upgrade
```

### Instalar Sudo

Para ejecutar comandos sin la necesidad de tener que usar su, vamos a:
- Instalar sudo
- Agregar un usuario al grupo wheel
- Modificar el archivo sudoers para que todos los miembros de wheel puedan ejecutar sudo

```bash
su
apk add sudo
adduser porky wheel
visudo
## Uncomment to allow members of group wheel to execute any command     
%wheel ALL=(ALL:ALL) ALL
```

De ahora en más, no necesitaremos ejecutar los comandos como root, podemos ejecutarlos con sudo.

### Conectar mediante SSH
[Fuente](https://wiki.alpinelinux.org/wiki/Setting_up_a_SSH_server)

Como pueden haber notado, es bastante incomdo ejecutar comandos desde la interfaz Visual de VirtualBox. Por eso, vamos a conectarnos mediante SSH a nuestra máquina virtual de alpine. 

El paquete openssh ya lo instalamos en **setup-alpine**.

Ejecutamos el comando

```bash
ip a
```

Y nuestra ip privada será la que está debajo de la interfaz de red (No la primera, esa es el loopback).

![Obtener Ip Privada](images/Ip_Privada.png)

En mi caso, mi ip privada es **192.168.0.228**

Desde nuestro host, ejecutamos el siguiente comando

```bash
ssh porky@192.168.0.228
```
Nos va a pedir la contraseña que configuramos y ya estariamos conectandonos a nuestra VM mediante ssh.

A partir de ahora, recomiendo ejecutar los comandos en el Host mediante ssh, ya que en mi opinión es más comodo. 

## Paso 5) Instalar Docker en Alpine
[Fuente](https://wiki.alpinelinux.org/wiki/Repositories)
[Fuente](https://virtualzone.de/posts/alpine-docker-rootless/)

Vamos a ejecutar wireguard y todos nuestros servicios en docker, para facilitar la configuración y mantenerlos aislados. 

Primero, debemos habilitar los repositorios de la comunidad puesto que el paquete de docker se encuentra allí. 

```bash
setup-apkrepos -c
apk add docker docker-cli-compose
rc-update add docker default
service docker start
addgroup ${USER} docker
```

Ejecutar docker sin root ni sudo: 

Instalamos los prerequisitos

```bash
apk add shadow-uidmap fuse-overlayfs iproute2 curl
```

Agregamos al archivo /etc/apk/repositories

```bash
sudo vi /etc/apk/repositories
# Agregamos esta línea
#https://dl-cdn.alpinelinux.org/alpine/v3.20/community

apk update
apk upgrade
```

Habilitamos cgroups v2  

```bash
vi /etc/rc.conf 
# Descomentamos esta línea
rc_cgroup_mode="unified"
#
rc-update add cgroups && rc-service cgroups start
```

Instalamos docker rootles extas

```bash
apk add docker-rootless-extras
```

Habilitamos el modulo iptables

```bash
su
echo "ip_tables" >> /etc/modules
modprobe ip_tables
```

Instalamos docker rootless:

```bash
sudo sh -c 'echo "porky:100000:65536" >> /etc/subuid'
sudo sh -c 'echo "porky:100000:65536" >> /etc/subgid'
curl -fsSL https://get.docker.com/rootless | sh
```

Crear Init script en /etc/init.d/docker-rootless

```bash
vi /etc/init.d/docker-rootless

#!/sbin/openrc-run

name=$RC_SVCNAME
description="Docker Application Container Engine (Rootless)"
supervisor="supervise-daemon"
command="/home/porky/bin/dockerd-rootless.sh"
command_args=""
command_user="porky"
supervise_daemon_args=" -e PATH=\"/home/porky/bin:/sbin:/usr/sbin:$PATH\" -e HOME=\"/home/porky\" -e XDG_RUNTIME_DIR=\"/home/porky/.docker/run\""

reload() {
    ebegin "Reloading $RC_SVCNAME"
    /bin/kill -s HUP \$MAINPID
    eend $?
}
```

Hacer el script ejecutable

```bash
chmod +x /etc/init.d/docker-rootless
rc-update add docker-rootless
rc-service docker-rootless start
```

Crear archivo .profile

```bash
vim .profile

#Añadir
export XDG_RUNTIME_DIR="$HOME/.docker/run"
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
export PATH="/home/<USER>/bin:/sbin:/usr/sbin:$PATH"
```

## Paso 6) Instalar WireGuard en docker
[Fuente](https://wg-easy.github.io/wg-easy/latest/examples/tutorials/basic-installation/)
[Wireguard Github](https://github.com/wg-easy/wg-easy)
[Wireguard Config Tool](https://www.wireguardconfig.com/)
[Configurar Wireguard Sin Proxy](https://github.com/wg-easy/wg-easy)

Cargamos los siguientes módulos al Kernel:

```bash
modprobe wireguard
modprobe ip6_tables
modprobe iptable_nat
```

Vamos a usar un docker compose para levantar wireguard. 

```bash
mkdir -p services/wireguard
mkdir -p /srv/wireguard
cd services/wireguard
docker compose up -d
```

Importante: 
Vamos a configurar la web sin proxy. Entonces debemos poner en el environment la siguiente línea:

```bash
environment:
    - INSECURE=true
```

Con esto, podemos acceder a **http://192.168.0.228:51821** y veremos una interfaz web para usar wireguard.
Creamos un usuario con contraseña y ya vamos a poder manejar la configuración de nuestros usuarios.

