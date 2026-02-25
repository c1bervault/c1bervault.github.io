---
title: "Resolución de la máquina Driver"
date: 2025-10-04 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_facil, Windows,  ]
description: "Máquina Driver de Hack The Box "
image:
  path: /assets/img/posts/WriteUpDriver/driver.png
  alt: "Información de Driver"
---
## Introducción

En este post veremos la resolución de la maquina retirada Driver de Hack The Box , es una maquina linux de dificultad facil y  en ella tocaremos los siguientes temas.

- Archivos scf
- Crackeo de Hashes NTLM v2


## 1 Reconocimiento

### 1.1 Escaneo de puertos

 Para empezar iniciaremos con un escaneo de los puertos abiertos de la maquina con el comando 

     nmap -p- --open  --min-rate=5000 -sS -vvv -Pn -n -oG allPorts 10.129.4.59

 Los diferentes parámetros de nmap vienen explicados en el post de  [comandos basicos sobre nmap](/posts/manual-nmap/) 


![1771859116539](/assets/img/posts/WriteUpDriver/allPorts.png)


 De este escaneo inicial sacamos en conclusión que el puerto 80 de la máquina esta alojando una web de la cual no tenemos mucha información de momento, el puerto 135 que es del protocolo RPC esta abierto pero en este caso no es relevante, el puerto 445 que es del protocolo SMB esta abierto lo cual puede ser una via potencial de ataque, el puerto 5985 que es del protocolo WinRM lo cual puede ser interesante en una etapa mas avanzada y por ultimo tiene abierto el puerto 7680 que es de entrega de actualizaciones por lo tanto no sera vital aqui.

 Ahora procederemos a hacer un escaneo mas detallado de los puertos abiertos lanzando los scripts de reconocimiento de servicio y versión de Nmap con este comando 

    nmap -sCV -p80,135,445,5985,7680 -oN targeted 10.129.4.59

![1771859116539](/assets/img/posts/WriteUpDriver/targeted.png)

 Del escaneo detallado sacamos las siguiente conclusión:

 - El puerto 80 aloja una web en la que la que hay una autenticación y podemos sacar en claro de que hay un usuario llamado admin


![1771859116539](/assets/img/posts/WriteUpDriver/admin.png)



### 1.2 Reconociminento Web

Si ponemos la ip de la maquina en el navegador vemos que tenemos el login anterior,ya sabemos que hay un usuario llamado admin, con lo cual lo proximo sería averiguar la contraseña, podriamos usar el diccionario rockyou e Hidra para hacer un ataque de fuerza bruta, pero probé la contraseña **admin** y dió login valido por lo tanto no hizo falta.

![1771859116539](/assets/img/posts/WriteUpDriver/login.png)


Nos encontramos un panel de administración de impresoras, en el cual nos da la opción de subir archivos y lo mas importante nos dice que el firmware que subamos se subira a su servidor SMB.


![1771859116539](/assets/img/posts/WriteUpDriver/subidaarchivos.png)

Esto nos da una pista ya que hay ciertos archivos en Windows llamados **.scf** que contienen comandos para el explorador de Windows.Si pudiesemos subir un archivo modificado con una ruta de de un recurso en red en nuestra maquina , el servidor SMB de la maquina victima se conectaría a nuestra maquina con su consecuente hash NTLM v2 del usuario actual y si empleasemos el uso de la herramienta **responder** 

Dicho esto crearemos el archivo **.scf** con la ruta de un archivo imaginario en nuestra maquina.


![1771859116539](/assets/img/posts/WriteUpDriver/archivoscf.png)


En este archivo lo importante es el apartado **Iconfile=** donde indicamos la ip de nuestra maquina atacante y el recurso que no tiene porque existir.

Si subimos el archivo **scf** y nos ponemos en escucha con la herramienta **Responder** recibiremos el hash NTLM v2 del usurio actual de la maquina victima.

Para ponernos en escucha con la herramienta **Responder** usaremos el comando

    sudo responder -I tun0


Recibimos el hash NTLM v2.


![1771859116539](/assets/img/posts/WriteUpDriver/hashntlm.png)


## 2 Explotación

### 2.1 Crackeo del hash NTLM v2 


Con el hash NTLM v2 podemos ver que el usuario se llama **tony**

A continuación procederé a crackear el hash offline ya que si es una contraseña poco segura se puede adivinar, para ello usare la herramienta hashcat.

Para empezar guardaremos el hash en un archivo llamado **hash.txt** para poder crackearlo


A continuación usaremos el siguiente comando para poder crackear el hash usando el diccionario rockyou.

El modo 5600 es usado para hashes NTLM v2

    hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

Una vez se ejecute y crackee la contraseña añadimos el parametro --show para que enseñe la contraseña crackeada


    hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --show

Vemos que la contraseña del usuario es **liltony**



![1771859116539](/assets/img/posts/WriteUpDriver/contraseña.png)

### 2.2 Shell como tony


Una vez tenemos las credenciales `tony` y `liltony` lo primero que probaré sera si son credenciales validas del dominio, para ello usare la herramienta netexec con el comando:

    nxc smb driver.htb -u tony -p "liltony"


![1771859116539](/assets/img/posts/WriteUpDriver/credencialesvalidas.png)


Vemos que nos marca que las credenciales son validas con lo cual podremos empezar a enumerar con ese usuario


Lo primero que se me ocurrió fue ver si el usuario estaba en el grupo **Remote management users** ya que si esta en este grupo podremos obtener una shell como este usuario.


Para ello usaremos la herramienta netexec con el comando:


    nxc winrm driver.htb -u tony -p "liltony"



![1771859116539](/assets/img/posts/WriteUpDriver/winrm.png)


Vemos que esta en el grupo por lo tanto podremos obtener una shell como el usuario tony con la herramienta  **evil-winrm** , usaremos el siguiente comando:

    evil-winrm -i driver.htb -u tony -p 'liltony' 


![1771859116539](/assets/img/posts/WriteUpDriver/shell.png)


La flag esta en C:\Users\tony\desktop/user.txt

## 3 Escalada de privilegios

### 3.1 CVE-2021-1675

Buscando por Internet vemos que hay una vulnerabilidad famosa CVE-2021-1675  que esta relacionada con la elevación de privilegios.

Esta vulnerabilidad fue muy peligrosa en 2021 ya que afectaba al Windows Print Spooler.

Esta vulnerabilidad permitía que un usuario con pocos privilegios pueda instalar un driver de impresora malicioso, que el servicio Spooler lo cargue y como el Spooler corre como SYSTEM pueda ejecutar codigo como administrador.


### Explotación de la CVE


Para empezar clonaremos el repositorio del script malicioso en nuestra maquina atacante con el comando:

    git clone https://github.com/calebstewart/CVE-2021-1675.git

Cuando ya lo tengamos copiado tendremos que subir el script a la maquina victima con el domando:

    upload /home/user/Desktop/htb/labs/easy/driver/content/CVE-2021-1675/CVE-2021-1675.ps1


![1771859116539](/assets/img/posts/WriteUpDriver/scriptsubido.png)

La idea es importar el script como modulo pero si lo hacemos nos dira que la política de ejecución de scripts esta deshabilitada por defecto , por lo tanto primero tendriamos que activarla con el comando:

    Set-ExecutionPolicy Unrestricted -Scope Currentuser

![1771859116539](/assets/img/posts/WriteUpDriver/politicascripts.png)

Ahora si lo importamos vemos que no nos da ningún error.


![1771859116539](/assets/img/posts/WriteUpDriver/importacion.png)


Ahora procedemos a crear el usuario Administrador con el modulo del script malicioso importado

En este caso crearemos un usuario que se llama **prueba** con la contraseña **prueba123**, usaremos el comando:

    Invoke-Nightmare -NewUser "prueba" -NewPassword "prueba123"


![1771859116539](/assets/img/posts/WriteUpDriver/usuariocreado.png)

Ya tendremos nuestro usuario con privilegios de Administrador creado, ahora,  nos conectaremos como hemos hecho antes con el usuario **tony** pero con nuestro usuario

    evil-winrm -i driver.htb -u prueba -p 'prueba123'

![1771859116539](/assets/img/posts/WriteUpDriver/conexionpriv.png)

Si vemos los grupos del usuario que hemos creado vemos que estamos en el grupo **Administrators**

![1771859116539](/assets/img/posts/WriteUpDriver/grupopriv.png)



## 4 Mitigación de vectores de ataque

Si esta maquina de Hack The Box hubiese sido un entorno real podriamos haber tomado las siguientes medidas para haber evitado un ataque como este:

#### Fase de reconocimiento

- En cuanto a reconocimiento lo más importante seria no haber expuesto el panel de login al publico , lo ideal seria que para acceder a el se necesitase alguna vpn.

- Tambien no se deberia haber usado la contraseña admin ya que es muy probable que esa sea la primera contraseña que prueben los atacantes.
    
- Como idea complementaria , hemos conocido el usuario admin porque en los headers de la solicitud http esta el texto Please enter password for admin, una indicación mas generica por ejemplo,       Please enter password for your user seria mejor ya que no conoceriamos el usuario admin 

- Para finalizar las recomendaciones para esta fase , cambiar el usuario admin por un usuario que tenga un nombre menos descriptivo y de menos indicaciones sobre los permisos de dicho usuaurio seria más recomendable.
  

#### Fase de explotación


- En la fase de explotación hemos ganado acceso al sistema porque la web nos dejaba subir archivos .scf , una idea seria blacklistear dichas extensiones o mas facilmente, solo aprobar extensiones de archivos que no sean maliciosos.

- Dado que hemos descubierto la contraseña del usuario crackeandola , una contraseña con simbolos y numeros nos hubiese hecho imposible crackearla y no hubiese sido posible ese vector de ataque.

- Como ultima recomendación en esta fase , hemos podido conseguir la shell gracias a que el usuario tony estaba en el grupo Remote Management Users , en este grupo es recomendable que solo esten los menos usuarios posibles ya que con herramientas como Evil-Winrm nos podemos conectar solo sabiendo usuario y contraseña.


### Escalada de privilegios

- Lo mas crucial para haber evitado este paso hubiese sido que el usuario tony no hubiese tenido permisos para modificar la politica de ejecución de scripts , asi no hubiesemos podido cambiarla y no habriamos podido importar el módulo.



