---
title: "Comandos basicos de Nmap"
date: 2026-2-02 20:00:00 +0100
categories: [Ciberseguridad,Manual,Herramienta]
tags: [Puertos,Redes]
description: "Manual acerca de los comandos basicos de la herramienta de escaneo de puertos Nmap "
image:
  path: /assets/img/posts/Manual/logo.png
  alt: "Nmap"
---

## Introducción


Nmap es una herramienta de escaneo de puertos , muy utilizada en ciberseguridad debido a sus amplios parametros y usos.


## Comandos basicos

En mi caso el escaneo de puertos que realizo a la mayoria de las maquinas consta de dos fases:


### 1 Fase general del escaneo

Esta fase es un reporte general de los servicios y puertos abiertos en el objetivo.

Primero debemos determinar los objetivos de esta fase:

- Rapidez --> En este escaneo solo queremos saber que puertos estan abiertos y que servicio corre (sin detalles).

- Ajustar el comando a las caracteristicas de nuestro objetivo --> establecer protocolo , presencia de resolución DNS etc...

- Extraer datos importantes para ser usados en la segunda fase.


El comando principalmente que uso en esta fase es:

    nmap -p- --open --min-rate=5000 -vvv -sS -n -Pn -oG allPorts IP


Destacamos los siguientes parametros:


- -p- --> Indicamos que queremos escanear los 65535 puertos disponibles.

- --open --> Solo indica puertos abiertos = No indicamos puertos cerrados o filtrados ya que esto no nos sirve.

- --min-rate=5000 --> No queremos que se envien menos de 5000 paquetes por segundo = Nos garantiza rapidez absoluta a coste de ser sigiloso pero esto en entornos controlados no tiene mucha relevancia , a menos paquetes por segundo seremos menos visibles frente a IDS o IPS, ya que estos suelen monitorizar conexiones a puertos diferentes y conexiones fallidas en poco tiempo.

- -vvv --> Muestra el resultado por consola a medida que va descubriendo puertos y nos ayuda a pensar vectores de enumeración que podamos usar.

- -sS --> Usamos un Stealh Scan o Half Open , este escaneo no completa la conexión TCP ya que si el cliente le manda SYN y recibe SYN/ACK  puerto esta abierto y  manda RST al servidor y no se completa el handshake.Si el cliente envia SYN y recibe RST esta el puerto cerrado .

    - Este escaneo frente a sistemas sin IDS es mas discreto.
    - Frente a IDS modernos ambos detectan los escaneos.
    - A nivel de logs del servicio -sS no usa la pila TCP por lo tanto no genera logs locales de conexión completa.
    - La mayor desventaja de este modo es que necesita permisos de ROOT para utilizarlo.

- -n --> No usamos resolución DNS, ganamos tiempo y discrección ya que si usamos resolución DNS nmap intentara resolver la IP a dominio por lo tanto generará trafico DNS y puede alertar a sistemas de monitoreo 

    - Solo conviene usarlo si queremos identificar nombres de host o mapear una infraestructura por nombres.

- -Pn --> Con este parametro le indicamos que el host esta activo por lo tanto nmap se ahorra el ping que hace para determinar si el host esta activo o inactivo.

- -oG --> Exportamos la salida a un archivo en formato grepeable el cual nos será util para la segunda fase del escaneo donde extraeremos los puertos con una función hecha con Regex.



De este escaneo sacamos una salida similar a esta:

![1771859116539](/assets/img/posts/Manual/fasegeneral.png)

Vemos el archivo que se ha generado con el nombre **allPorts** y los puertos abiertos con su servicio sin determinar versión o mas caracteristicas.


### 2 Fase especifica  del escaneo

Una vez obtenido ese archivo procedemos a extraer los puertos del archivo con la función extractPorts creada por **@s4vitar**

 ![1771859116539](/assets/img/posts/Manual/extractports.png)


 Obtenemos una salida similar a esta si ejecutamos el comando:

    extractPorts allPorts


   ![1771859116539](/assets/img/posts/Manual/puertosextraidos.png)



Una vez obtenidos los puertos abiertos procedemos a la segunda fase del escaneo de la cual destacamos los siguientes objetivos:

  - Precisión--> Queremos saber la maxima información del servicio incluyendo versión y demas caracteristicas 
  
  - Identificar información valiosa --> usuarios permitidos , directorios ocultos etc...

  - Facilidad para adjuntar la salida al informe que posteriormente se hará 

El comando principal de esta fase es:

    nmap -p(puertos a escanear) -sCV -oN targeted IP

Destacamos los siguientes parametros:

  - -p --> Indicamos que puertos queremos escanear , en este caso todos los que hemos extraido del primer paso.
  
  - -sCV --> En este comando pedimos que nmap ejecute los scripts que tiene por defecto para identificar:
    -  V --> Versión del servicio
  
    -  C --> Caracteristicas del servicio que se esta ejecutando

  - -oN --> Exportamos la salida del comando a un archivo llamado targeted en formato nmap para poder analizarlo posteriormente e incluirlo en el informe.
  
Obtendremos una salida similar a esta:


   ![1771859116539](/assets/img/posts/Manual/fase2.png)

Aqui podemos ver un ejemplo de puertos que estan corriendo con su servicio y versiones del mismo además de nombres de dominio etc...

