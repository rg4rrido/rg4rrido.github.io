---
title: "VulNet:Internal TryHackMe"
date: 2026-10-01 15:28:04 +0200
categories: [CTF, TryHackMe]
tags: [smb, redis, rsync, portforwarding, eJPTv2]
---
- - -

## Introducción

![Logo](/assets/img/posts/2026-10-01-vulnet-internal/00-VulnetInternalLogo.png)

En este writeup vamos a resolver [VulnNet:Internal](https://tryhackme.com/room/vulnnetinternal), una máquina de TryHackMe centrada principalmente en la enumeración de servicios y abuso de servicios internos para escalar privilegios.

La máquina resulta especialmente interesante para practicar una metodología de pentesting realista, ya que durante el proceso tendremos que enumerar distintos servicios como SMB, NFS y Rsync, analizar información expuesta en configuraciones y utilizar credenciales encontradas para continuar avanzando. También trabajaremos con Redis, realizaremos un Local Port Forwarding mediante SSH para acceder a un servicio interno y, finalmente, aprovecharemos una funcionalidad de TeamCity para conseguir ejecución remota de comandos.

Por la variedad de técnicas que combina, esta máquina puede ser un buen complemento práctico para preparar la eJPTv2.

## Fase de Reconocimiento

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.130.152.106 -oN allPorts
```

```bash
grep '^[0-9]' allPorts  | cut -d '/' -f1 | xargs | sort | tr ' ' ','
```

```bash
nmap -p22,111,139,445,873,2049,6379,35811,40121,42505,45107,46197 -sCV 10.130.152.106 -oN targeted
```

![Escaneo de puertos](/assets/img/posts/2026-10-01-vulnet-internal/01-escaneo-de-puertos.png){: width="900" }

Tras el escaneo de puertos, detectamos varios servicios expuestos, entre ellos: SMB, SSH, NFS… Vamos a empezar enumerando el servicio SMB.

### Enumeración

#### Servicio SMB

Vamos a tratar de utilizar **Null Sessions** para enumerar los recursos compartidos a nivel de red con este servicio.

```bash
netexec smb 10.130.152.106 -u '' -p '' --shares
```

![Servicio SMB](/assets/img/posts/2026-10-01-vulnet-internal/02-servicio-smb.png)

Detectamos que tenemos permisos de lectura en el recurso “shares”, haciendo uso de *smbclient* podemos tratar de acceder a él.

```bash
smbclient //10.130.152.106/shares -N
```

![Servicio SMB](/assets/img/posts/2026-10-01-vulnet-internal/03-servicio-smb.png){: width="800" }

![Servicio SMB](/assets/img/posts/2026-10-01-vulnet-internal/04-servicio-smb.png){: width="800" }

En el contenido de estos archivos encontramos lo siguiente.

![Servicio SMB](/assets/img/posts/2026-10-01-vulnet-internal/05-servicio-smb.png){: width="1000" }

Bien, encontramos la primera flag del laboratorio, no parece que haya nada más destacable por aquí, así que podemos pasar a enumerar el siguiente servicio.

#### Servicio RPC (NFS)

Para enumerar este servicio podemos usar la herramienta *showmount*. `Showmount -e <IP>` consulta al servicio *mountd* de la máquina remota y te pide la lista de exports — los directorios que el servidor *NFS* tiene configurados para compartir por red. Es el equivalente NFS a listar los shares SMB con `smbclient -L`. No accede al contenido, solo te dice qué está disponible para montar y quién puede montarlo.

```bash
showmount -e 10.130.152.136
```

![Servicio RPC (NFS)](/assets/img/posts/2026-10-01-vulnet-internal/06-servicio-rpc-nfs.png){: width="300" }

El output nos indica que la carpeta compartida por NFS desde el servidor es */opt/conf* y que *cualquiera* (*asterisco*) en la red puede acceder a ella. En este punto podemos montarla en nuestro equipo atacante y enumerar su contenido:

```bash
mkdir /mnt/NFS
mount -t nfs 10.130.152.106:/opt/conf /mnt/NFS
```

![Servicio RPC (NFS)](/assets/img/posts/2026-10-01-vulnet-internal/07-servicio-rpc-nfs.png){: width="500" }

Encontramos el siguiente contenido en el recurso compartido en la red mediante *NFS*, entre los directorios, uno correspondiente a la base de datos *Redis*.

> **Base de datos Redis:**
> 
> Es una **base de datos NoSQL de código abierto** y almacén de estructura de datos en memoria, diseñado para ofrecer **altísimo rendimiento y baja latencia**. A diferencia de las bases de datos tradicionales que guardan información en disco, Redis almacena los datos directamente en la **memoria RAM**, lo que permite tiempos de respuesta en **milisegundos** o microsegundos.
> Destaca por su **modelo clave-valor**, almacena datos en pares de clave y valor.
{: .prompt-info }

Podemos tratar de encontrar credenciales hardcodeadas en el archivo de configuración:

```bash
cat redis/redis.conf | grep pass
```

![Servicio RPC (NFS)](/assets/img/posts/2026-10-01-vulnet-internal/08-servicio-rpc-nfs.png){: width="500" }

Efectivamente encontramos la credencial, `B65Hx562********`, con ella podemos conectarnos a la base de datos a través de su cliente de terminal *redis-cli*.

#### Enumeración de base de datos Redis

```bash
redis-cli -h 10.130.152.106 -p 6379 -a "B65Hx562********"
```

![Enumeración de Base de datos Redis](/assets/img/posts/2026-10-01-vulnet-internal/09-enumeracion-de-base-de-datos.png){: width="800" }

Con esto nos conectamos a la base de datos *Redis* y enumeramos sus claves almacenadas, entre ellas “internal flag”. Otra clave que nos debe llamar la atención es “authlist”, que podemos tratar de enumerar de la misma forma que “internal flag”.

```bash
get "authlist"
```

![Enumeración de Base de datos Redis](/assets/img/posts/2026-10-01-vulnet-internal/10-enumeracion-de-base-de-datos.png){: width="700" }

Nos da un error (WRONGTYPE) debido a que la clave “authlist” no es una string. Para listar el tipo de clave que es, podemos ejecutar:

```bash
type "authlist"
```

![Enumeración de Base de datos Redis](/assets/img/posts/2026-10-01-vulnet-internal/11-enumeracion-de-base-de-datos.png){: width="300" }

Vemos que es una lista. Para listar su contenido podemos hacerlo ejecutando el siguiente comando:

```bash
lrange authlist 0 -1
```

![Enumeración de Base de datos Redis](/assets/img/posts/2026-10-01-vulnet-internal/12-enumeracion-de-base-de-datos.png){: width="800" }

Nos lista una misma cadena en base64 varias veces, si la decodificamos para ver su contenido:

![Enumeración de Base de datos Redis](/assets/img/posts/2026-10-01-vulnet-internal/13-enumeracion-de-base-de-datos.png)

El contenido de la cadena corresponde a credenciales para el servicio *Rsync*, vamos a validar estas credenciales.

> **Rsync**
> 
> Rsync, o Remote Sync, es una herramienta gratuita de línea de comandos que permite transferir archivos y directorios a destinos locales y remotos. Rsync se utiliza para crear copias espejo, realizar copias de seguridad o migrar datos a otros servidores.
{: .prompt-info }

#### Enumeración de Rsync

Para enumerar los módulos expuestos (recursos compartidos del servidor), podemos hacerlo ejecutando:

```bash
rsync 10.130.152.106::
```

![Enumeración de Rsync](/assets/img/posts/2026-10-01-vulnet-internal/14-enumeracion-de-rsync.png){: width="400" }

Para listar los módulos de rsync también podemos hacer uso de nmap y uno de sus scripts:

```bash
nmap -p 873 --script rsync-list-modules 10.130.152.106 -oN rsyncModules
```

![Enumeración de Rsync](/assets/img/posts/2026-10-01-vulnet-internal/15-enumeracion-de-rsync.png){: width="800" }

De esta forma enumeramos el módulo files. El siguiente paso sería tratar de listar su contenido de la siguiente manera:

```bash
rsync --list-only rsync://rsync-connect@10.130.152.106/files
```

![Enumeración de Rsync](/assets/img/posts/2026-10-01-vulnet-internal/16-enumeracion-de-rsync.png){: width="400" }

Descubrimos varios directorios en el recurso compartido “files”. Para traérnoslos a nuestra máquina atacante, podemos ejecutar el siguiente comando:

```bash
rsync -avz rsync://rsync-connect@10.130.152.106/files/ files
```

Al traernos el contenido enumeramos lo siguiente:

![Enumeración de Rsync](/assets/img/posts/2026-10-01-vulnet-internal/17-enumeracion-de-rsync.png){: width="500" }

Encontramos la flag de usuario en “user.txt”. Otra cosa que nos debe llamar la atención son las carpetas ocultas “.ssh”. Con la herramienta rsync tenemos la posibilidad de subir archivos, por lo que podríamos tratar de subir un archivo authorized_keys con nuestra clave privada para ganar acceso a la máquina.

## Intrusión a la máquina víctima

Para ello podemos:

```bash
rsync -av /root/.ssh/id_ed25519.pub rsync://rsync-connect@10.130.152.106/files/sys-internal/.ssh/authorized_keys
```

![Intrusión a la máquina Víctima](/assets/img/posts/2026-10-01-vulnet-internal/18-intrusion-a-la-maquina-victima.png){: width="800" }

Una vez hecho, probamos el acceso a la máquina mediante SSH.

```bash
ssh sys-internal@10.130.152.106 
```

![Intrusión a la máquina Víctima](/assets/img/posts/2026-10-01-vulnet-internal/19-intrusion-a-la-maquina-victima.png){: width="400" }

Con esto ganamos acceso a la máquina y podemos empezar la enumeración local para escalar privilegios y conseguir la última flag.

## Enumeración local

```bash
grep -e sh$ /etc/passwd
```

![Enumeración local](/assets/img/posts/2026-10-01-vulnet-internal/20-enumeracion-local.png){: width="500" }

Enumerando manualmente la máquina, descubrimos un directorio en la raíz del sistema llamado “TeamCity”. En él encontramos varios archivos, entre ellos:

![Enumeración local](/assets/img/posts/2026-10-01-vulnet-internal/21-enumeracion-local.png)

Vemos que al parecer hay un servicio corriendo en el puerto 8111 de la máquina. Para validarlo, podemos usar el comando *ss*:

```bash
ss -tn | grep 8111
```

![Enumeración local](/assets/img/posts/2026-10-01-vulnet-internal/22-enumeracion-local.png){: width="600" }

Y efectivamente, en este punto deberíamos aplicar un Port Forwarding para obtener acceso a este servicio interno de la máquina víctima.

### Port Forwarding a servicio interno

```bash
ssh sys-internal@10.130.152.106 -L 8111:127.0.0.1:8111
```

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/23-port-forwarding-a-servicio-interno.png){: width="1000" }

Vemos que tenemos la opción de iniciar sesión como “Super user”. Al hacer clic en esta opción nos pide lo siguiente:

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/24-port-forwarding-a-servicio-interno.png){: width="300" }

Un token de autenticación. Podemos tratar de obtenerlo en los archivos de configuración en la carpeta TeamCity del sistema.

```bash
cd /TeamCity/logs
cat * | grep -i token
```

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/25-port-forwarding-a-servicio-interno.png){: width="1000" }

Vamos a ir probando todos hasta dar con el válido.

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/26-port-forwarding-a-servicio-interno.png){: width="1200" }

Tras probar con varios, logramos acceder con el token “45452211427380#####”. En el aplicativo web, vamos a echar un vistazo a las opciones que tenemos. Vemos que tenemos la opción de crear un proyecto:

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/27-port-forwarding-a-servicio-interno.png){: width="1000" }

Seguimos probando creando una “build”. Una vez creada, podemos editarla:

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/28-port-forwarding-a-servicio-interno.png){: width="1000" }

Una vez creada la build, podemos editar su configuración y vemos un apartado que pone “Build Step”. Al darle, nos deja elegir el tipo de “Runner”, en el que destaca el “*command_line*”. Con esto interpretamos que tenemos opción de ejecutar comandos en la máquina víctima con privilegios de administrador, que es quien desplegó este servicio interno.

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/29-port-forwarding-a-servicio-interno.png){: width="800" }

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/30-port-forwarding-a-servicio-interno.png)

Por ejemplo, vamos a probar a darle permisos *suid* a la shell bash. Ejecutamos la build:

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/31-port-forwarding-a-servicio-interno.png)

Desde la conexión SSH, vamos a validar que se haya ejecutado correctamente el comando.

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/32-port-forwarding-a-servicio-interno.png){: width="600" }

Efectivamente se ejecutó con éxito y podemos escalar privilegios.

```bash
/bin/bash -p
```

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/33-port-forwarding-a-servicio-interno.png)

![Port Forwarding a servicio interno](/assets/img/posts/2026-10-01-vulnet-internal/34-port-forwarding-a-servicio-interno.png)

De esta manera habríamos completado este CTF.
