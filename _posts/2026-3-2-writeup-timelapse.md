---
title: "Resolución de la máquina Timelapse"
date: 2026-3-01 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_facil, Windows,smb_enumeration]
description: "Máquina Timelapse de Hack The Box "
image:
  path: /assets/img/posts/WriteUpTimelapse/timelapse.png
  alt: "Información de Timelapse"
---
## Introducción

Hoy traigo el writeup de la maquina **Timelapse** de Hack The Box, en este caso Windows ya que quiero centrarme mas en Active Directory.
En la maquina trataremos estos aspectos:

- Enumeración SMB

- LAPS_Readers


##  1 Reconocimiento

### 1.1 Escaneo de puertos


Primero dado que Hack The Box solo nos da la ip de la maquina , empezaremos con un escaneo de los 65535 puertos que hay en total para ver cual de ellos esta en estado abierto en esta maquina, para ello utilizaremos este comando:

    nmap -p- --open --min-rate=5000 -vvvv -n -Pn -oG allPorts 10.129.227.113

Respecto a todas las opciones de la herramienta tengo un post donde explico para que sirve cada parámetro de la herramienta que solemos usar en los escaneos tipicos.

![1772438373523](/assets/img/posts/WriteUpTimelapse/escaneogeneral.png)

Una vez ejecutado este comando obtenemos una información muy general de la máquina , asi que procedemos a escanear detalles y versión del servicio que esta corriendo en cada puerto.

    nmap -p(puertos a escanear) -sCV -oN targeted 10.129.227.113

![1772438373523](/assets/img/posts/WriteUpTimelapse/escaneodetallado.png)

De este escaneo principalmente sacamos dos cosas relevantes:

-  Puerto 445 abierto --> Comenzaremos por enumeración SMB
-  Puerto 5986 abierto --> WinRM sobre TLS/SSL dado que el puerto de WinRM que usa HTTP es 5985
-  El puerto 5986 de WinRM nos devuelve dc01.timelapse.htb asi que lo mas seguro es que estemos ante un Domain Controller (DC)

### 1.2 Enumeración del servicio SMB

![1772438373523](/assets/img/posts/WriteUpTimelapse/escaneosmbprincipal.png)

Usando la herramienta netexec para enumerar mas la ip confirmamos de nuevo que nos encontramos ante un DC, ahora procedemos a probar la autenticación con el usuario **Guest** para ver si esta habilitada en SMB.

![1772438373523](/assets/img/posts/WriteUpTimelapse/usuarioguesthabilitado.png)

Vemos que esta habilitado asi que esto nos abre la puerta a dos cosas:

- Listado de recursos habilitados en SMB con conexión Nula
  
- Enumeración de usuarios en el sistema por ataques de fuerza bruta al RID de todos los SID de los objetos del dominio.


Primero comenzamos con un ataque de fuerza bruta al RID de los SID con este comando


    nxc smb 10.129.227.113 -u "Guest" -p "" --rid-brute

![1772438373523](/assets/img/posts/WriteUpTimelapse/usuariosdelsistema.png)

Con esto sacamos todos los usuarios del sistema  , nos puede ser util en un futuro.

Una vez hecho esto procedemos a listar recursos validos del sistema de SMB.

Usamos el comando smbclient con el parametro -L para listar recursos y -N para conexión nula es decir usuario **Guest** habilitado.

    smbclient -L //10.129.227.113/ -N 


![1772438373523](/assets/img/posts/WriteUpTimelapse/recursossmb.png)

Vemos que hay un recurso llamado **Shares** asi que vamos a intentar iniciar sesión en ese recurso para ver si tenemos algo de valor.

    smbclient  //10.129.227.113/Shares -N

![1772438373523](/assets/img/posts/WriteUpTimelapse/recursoshares.png)

Vemos el directorio **Dev** ya que HelpDesk no tiene nada interesante.

Nos encontramos un zip asi que lo descargamos a nuestra maquina atacante.

![1772438373523](/assets/img/posts/WriteUpTimelapse/recursozip.png)


## 2 Explotación

### 2.1 Crackeo clave privada 

Intentando descomprimirlo vemos que nos pide una contraseña, tambien vemos que dentro contiene un archivo de autenticación de un usuario llamado **legacyy**.


![1772438373523](/assets/img/posts/WriteUpTimelapse/pidecontraseña.png)

Como no tenemos contraseña lo primero va a ser intentar crackear la contraseña si es debil, por lo tanto , primero obtenemos el hash del archivo zip con la herramienta **zip2john**, para ello usamos este comando.

    zip2john archivo.zip > zip.hash


![1772438373523](/assets/img/posts/WriteUpTimelapse/hashzip.png)

Ahora vamos a usar la herramienta **John** y el usuario rockyou para intentar crackearla


![1772438373523](/assets/img/posts/WriteUpTimelapse/contraseñacrackeada.png)


Vemos que ha obtenido la contraseña  , probamos a usarla en el zip y funciona.

### 2.2  Crackeo certificado

Conseguimos un archivo **pfx** que normalmente estan cifrados por contraseña y contienen claves privadas asi que procedemos a repetir el proceso que hemos hecho antes , extraemos hash y crackeamos.

En este caso extraemos el hash con la herramienta **pfx2john** .


![1772438373523](/assets/img/posts/WriteUpTimelapse/pfx2hash.png)


Una vez extraido el hash vamos a crackearlo.

Ha conseguido la contraseña por lo tanto vamos a extraer la clave del archivo.

![1772438373523](/assets/img/posts/WriteUpTimelapse/crackeohashpfx.png)


Para extraer las claves usaremos este comando:


    openssl pkcs12 -in archivo.pfx -nocerts -nodes

- pkcs12 --> Le decimos a openssl que trabaje con PKCS#12 que es --> .pfx
  
- in --> le indicamos a openssl el archivo  .pfx que queremos usar

- nocerts --> no extraemos certificados , solo clave privada

- nodes --> no ciframos la clave privada al exportarla
  

![1772438373523](/assets/img/posts/WriteUpTimelapse/sacamosclave.png)

Guardamos la clave en un archivo key.pem y procedemos a sacar la clave publica del archivo **pfx** también.


![1772438373523](/assets/img/posts/WriteUpTimelapse/cert.png)

### 2.3 Shell como legacyy

Usamos mismos parametros que antes solo añadiendo el parametro  -nokeys para extraer solo el certificado y -out para especificar en que archivo queremos volcar la salida.

Una vez extraidos los dos archivos procedemos a intentar conectarnos por WinRM con la herramienta evil-winrm.

Para ello usamos el comando 

    evil-winrm -i 10.129.227.113 -c cert.pem -k key.pem -S 

Usamos el parametro -c para especificar el certificado,  -k para emplear la clave privada y -S para decir que use SSL porque solo esta abierto el puerto 5986 que es el puerto de WinRM sobre SSL.

![1772438373523](/assets/img/posts/WriteUpTimelapse/winrm.png)

Vemos que nos conecta como el usuario **legacy**


## 3 Escalada de privilegios


Una de las cosas que primero hay que ver es el historial de comandos dado que es un vector infravalorado, muchas veces se dejan pistas o accciones que no deberían ser visibles.

Para ello vamos a usar este comando:

     type $env:APPDATA\Microsoft\Windows\Powershell\PSReadLine\ConsoleHost_history.txt

Este comando usa:

- type --> para mostrar contenido 

- $env --> es una variable de entorno que apunta a la ruta C:\Users\Usuario\AppData\Roaming




![1772438373523](/assets/img/posts/WriteUpTimelapse/historial.png)

### 3.1 Shell como svc_deploy

Vemos hay un comando que convierte una contraseña a **SecureString** esto es porque para crear usuarios por powershell la contraseña no se puede poner como texto plano y antes hay que convertir la contraseña a texto seguro para guardarlo en una variable en memoria y asi poder indicar la contraseña.

Una vez con la contraseña del usuario **svc_deploy** procedemos a intentar conectarnos como ese usuario.

Usamos la herramienta evil-winrm pero indicando usuario , contraseña y el parametro -S ya que seguimos estando por SSL.

![1772438373523](/assets/img/posts/WriteUpTimelapse/svcdeploy.png)


Una vez como este usuario, no tenemos permisos de administrador pero podemos ver los grupos en los que esta  con el comando:

    net user svc_deploy

![1772438373523](/assets/img/posts/WriteUpTimelapse/svcdeploygroups.png)

Esta en el grupo LAPS_Readers , investigando un poco , veo que los integrantes de ese grupo tienen permisos para leer contraseñas de administrador local.

Para ello utilizamos este comando:

    Get-ADComputer DC01 -property 'ms-mcs-admpwd'

Este comando utiliza los siguientes parametros:

- Get-ADCcomputer --> llama al modulo RSAT para consultar objetos del dominio

- DC01 --> especificamos el 'nombre' del objeto que queremos consultar en este caso es DC01 ya que ese es el nombre del equipo en el que estamos

- property --> por defecto Get-ADComputer solo devuelve propiedades basicas asi que con este parametro especificaremos una propiedad ademas de las basicas que queremos consultar

- ms-mcd-admpwd --> atributo de LAPS donde se guarda la contraseña del administrador local del equipo

![1772438373523](/assets/img/posts/WriteUpTimelapse/contraseña-admin.png)

### 3.2 Shell como Administrador 

Una vez con la contraseña del administrador local procedemos a conectarnos como evil-winrm

![1772438373523](/assets/img/posts/WriteUpTimelapse/winrmadmin.png)

Vemos que estamos como Administrador


![1772438373523](/assets/img/posts/WriteUpTimelapse/whoamiadmin.png)

La flag no está en el Desktop de Administrator por lo tanto vemos que hay un directorio llamado **TRX** , entramos y vemos la flag de root en su escritorio.


![1772438373523](/assets/img/posts/WriteUpTimelapse/flagroot.png)


## 4 Mitigación de vectores de ataque

### 4.1 Vectores usados en Reconocimiento

- Primero para obtener el archivo zip hemos utilizado el usuario Guest en SMB ya que estaba habilitado , tener este usuario habilitado es muy peligroso ya que hemos podido enumerar todos los usuarios y hemos podido entrar al recurso gracias a ello, por lo tanto , lo ideal no es tenerlo habilitado.

- También hemos podido obtener acceso al zip porque hemos podido descargarlo y leer sobre directorios , asi que en el caso de que el usuario Guest tenga que estar habilitado , no debe tener permisos ni de lectura ni escritura.


### 4.2 Vectores usados en explotación

- Para obtener la clave privada hemos podido porque el zip tenia una clave debil , en el caso de tener archivos con contraseña es obligatorio usar contraseñas fuertes con mayusculas,minusculas,numeros,caracteres especiales y que no tengan sentido para anular el vector de ataque por fuerza bruta, este consejo se podria aplicar también al archivo pfx ya que este también tenía contraseña debil.


- Para obtener la shell hemos podido porque el usuario legacyy estaba en el grupo Windows Remote Management Users, este es un grupo muy peligroso ya que cualquier persona que tenga las credenciales de algún usuario puede conectarse como el mismo remotamente, cuantos menos usuarios estén en este grupo mejor.


- Por ultimo hemos conseguido acceder al usuario svc_deploy ya que en el historial estaban las contraseñas , lo ideal es que al final de cada sesión se elimine el historial para evitar estos vectores de ataque.


### 4.3 Vectores usados en escalada de privilegios


- El usuario svc_deploy estaba en el grupo LAPS_Readers , este grupo al igual que WinRM no deberia haber muchas personas ya que permite directamente leer las contraseñas del Administrador.


