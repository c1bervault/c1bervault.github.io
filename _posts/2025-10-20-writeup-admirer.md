---
title: "Resolución de la máquina admirer(NO TERMINADA)"
date: 2025-10-04 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_facil, linux, fuzzing]
description: "Máquina Admirer de Hack The Box "
image:
  path: /assets/img/posts/WriteUpAdmirer/admirer.png
  alt: "Información de Admirer"
---
## Introducción

En este post veremos la resolución de la maquina retirada apocalyst de Hack The Box , es una maquina linux de dificultad media y  en ella tocaremos los siguientes temas.

- Fuzzing

- MySql

- FTP

---

## 1 Reconocimiento

### 1.1 Escaneo de puertos

Para empezar con el reconocimiento inicial de esta máquina vamos a comenzar con un escaneo de los 65535 puertos que existen para ver cual de ellos tiene el estado open en la maquina, para ello, vamos a utilizar el comando:

`nmap -p- --open -sS --min-rate 500 -vvv -Pn -n -oG allPorts 10.10.10.187`

En este comando como siempre escaneamos los 65535 puertos con el comando -p- en el caso de que queramos escanear puertos concretos por ejemplo el 22 y el 80  podriamos hacer uso del mismo parametro pero indicandolo de la siguiente manera `-p22,80` asi el escaneo sera mas rapido pero solo escanearemos esos dos puertos y nos saltaremos informacion, seguimos con el comando --open para solamente reportar los puertos que tengan estado open ya que los demas estados que son filtered y closed no nos interesan de momento, continuamos con el parametro -sS para especificar que queremos hacer un SYN Scan , ya que es mas rapido que el clasico -sT ya que no completa el handshake por TCP.

Seguimos con el comando --min-rate 5000 para especificar que no queremos que se emitan menos de 5000 paquetes por segundo , esto garantiza rapidez pero es mas ruidoso que utilizar menos paquetes por segundo, continuamos con -vvv para triple verbose ya que con esto podemos ver información de los puertos segun se detectan , usamos el parametro -Pn elimina el descubrimiento de host ya que sabemos que el host esta activo y nos ahorramos el paquete que envia para descubrir si el host esta activo , continuamos con el parametro -n para eliminar resolución DNS y actuar solo con ips y para finalizar -oG para exportar la salida del comando a un archivo llamado allPorts en formato *grepeable* de cara al informe final y a la claridad de la información.

Descubrimos que tiene abierto el puerto 21 que corresponde a FTP , el puerto 22 que corresponde a SSH y el puerto 80 que corresponde a un servicio web en este caso Apache.

![1768763189775](/assets/img/posts/WriteUpAdmirer/escaneoinicial.png)

Para seguir con el reconocimiento vamos a continuar con un escaneo detallado de esos puertos utilizando scripts de reconocimiento de Nmap, para ello vamos a usar el comando:

`nmap -sCV -p21,22,80 -oN 10.10.10.187`

Aqui empleamos los scripts de reconocimiento de servicio y version con el parametro -s utilizando C para los scripts por defecto de nmap y V para detectar la versión del servicio que corre en cada puerto.Especificamos los puertos con el parametro -p y exportamos la salida con el parametro -oN en formato nmap para posterior analisis.

![1768763189775](/assets/img/posts/WriteUpAdmirer/escaneodetallado.png)

De este escaneo sacamos información importante:

- Servicio FTP

  - Versión anticuada --> 3.0.3
  - CVE existente para causar una denegación de servicio (CVE-2021-30047)
  - No reporta posibilidad de logearse con el usuario anonimo
- Servicio Web

  - Fichero robots.txt existente
  - Entrada deshabilitada al directorio /admin-dir
  - Versión desactualizada de Apache --> Apache/2.4.25 , cves existentes bastante graves.

Finalizado el escaneo de puertos procedemos al reconocimiento web.

### 1.2 Reconocimiento web

![1768763189775](/assets/img/posts/WriteUpAdmirer/web.png)

En cuanto a información destacable en la pagina web no obervamos nada relevante, procedemos a confirmar lo que vimos antes del directorio admin-dir en el robots.txt

![1768763189775](/assets/img/posts/WriteUpAdmirer/robots.png)

No solo lo confirmamos si no que sacamos un posible usuario pero todavia no esta validado --> waldo

### 1.3 Fuzzing

Vamos a empezar aplicando reconocimiento sobre el directorio oculto llamado /admin-dir, para ello , vamos a utilizar el diccionario de Seclists llamado **raft-medium-files.txt**

Como etapa inicial no encontramos nada destacable pero vamos a usar un diccionario mas grande llamado **raft-large-files.txt**

Para ello usamos el comando:

`gobuster dir -u http://10.129.46.201/admin-dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-files.txt  -t 50`


Usamos el parametro dir para especificar que queremos hacer una enumeración de directorios o archivos, continuamos con el parametro -w para especificar el diccionario que vamos a uar y para finalizar con el parametro -t 50 especificamos los hilos con los que queremos que lleve a cabo el escaneo , es decir , la potencia y rapidez del escaneo .


![1768763189775](/assets/img/posts/WriteUpAdmirer/fuzzing.png)

Del escaneo encontramos cosas bastantes curiosas como un archivo llamado credentials.txt expuesto y otro archivo llamado contacts.txt expuesto tambien al publico.

El archivo credentials.txt vemos que contiene credenciales de una cuenta interna de mail , una cuenta de ftp y una cuenta de Wordpress con admin.

![1768763189775](/assets/img/posts/WriteUpAdmirer/credentials.png)

El archivo expuesto contacts.txt contiene diferentes correos diferentes correos corporativos que pueden ser interesantes.

![1768763189775](/assets/img/posts/WriteUpAdmirer/contacts.png)


## 2 Explotación

### 2.1 Archivos FTP

De las credenciales expuestas vemos que la unica valida es la de ftp ya que la página no es un Wordpress y no tenemos acceso a un webmail.

En el servidor ftp vemos que hay dos archivos:
  - Archivo sql de una base de datos.
  
  - Archivo llamado html que esta comprimido.

![1768763189775](/assets/img/posts/WriteUpAdmirer/archivosftp.png)


A continuación descargamos los archivos con el comando:

`get dump.sql`

`get html.tar.gz`

![1768763189775](/assets/img/posts/WriteUpAdmirer/extraccionarchivos.png)


Del archivo sql no sacamos nada interesante , ya que no hay ninguna credencial pero sacamos el nombre de una base de datos "admirerdb".


![1768763189775](/assets/img/posts/WriteUpAdmirer/dumpsql.png)


En cuanto al archivo comprimido descubrimos que es todo el codigo fuente de la web y vemos que en el directorio /utility-scripts hay un script llamado dbadmin.php que contiene credenciales en texto claro de una base de datos 


![1768763189775](/assets/img/posts/WriteUpAdmirer/credencialessql.png)


### 2.2 Fuzzing al directorio descubierto

Hemos descubierto un directorio nuevo interesante llamado /utility-scripts asi que procedemos a hacer reconocimiento sobre este directorio nuevo 

En este caso utilizaremos un diccionario diferente ya que con el diccionario anterior no hay resultados, cabe destacar que , como vemos que son archivos php los del directorio descubierto incluiremos la extension.php a las palabras del diccionario , procedemos con este comando:

`gobuster dir -u http://10.129.47.187/utility-scripts -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-small-directories.txt -x .php -t 50`

Lo unico que cambia de los comandos anteriores de fuzzing es el parametro -x para indicar que queremos añadir la extensión .php a las palabras del diccionario.


![1768763189775](/assets/img/posts/WriteUpAdmirer/fuzzingnuevo.png)

Aparte de los archivos que hemos descubierto antes , descubrimos un archivo nuevo llamado adminer.php en el que podemos usar las credenciales obtenidas anteriormente.

![1768763189775](/assets/img/posts/WriteUpAdmirer/adminer.png)







