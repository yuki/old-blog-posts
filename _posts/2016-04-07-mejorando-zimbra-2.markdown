---
layout: post
title:  "Mejorando nuestro servidor de correo Zimbra (2/2): Mailwatch"
date:    2016-04-07 00:00:00 +0200
tags: zimbra mailwatch
---


En nuestra serie de posts donde explicamos cómo tener un sistema de correo electrónico seguro y profesional, ya vimos [cómo instalar Zimbra 8.6]({% post_url 2016-01-21-zimbra-1 %}), [cómo configurarlo]({% post_url 2016-02-09-zimbra-2 %}), y por último [cómo instalar MailScanner]({% post_url 2016-03-10-mejorando-zimbra-1 %}). Hoy toca mejorar el post anterior para poder hacerlo más visible y usable. Para ello vamos a realizar la instalación de **[MailWatch](http://mailwatch.org/)**, que es un **interfaz web para MailScanner**.

En este post modificaremos parte de la configuración de MailScanner, por lo que es importante haber seguido el post anterior y tenerlo bien instalado. Os recordamos que nuestro servidor es **Ubuntu 14.04 LTS**, pero los pasos son similares en el resto de distribuciones.

![mailwatch](/assets/images/mailwatch-logo.png)

Tal como he dicho en la entrada, para la instalación y configuración de MailWatch vamos a realizar modificaciones en la configuración de MailScanner, por lo que hay que estar un poco atento sobre qué ficheros estamos modificando; pero, sobre todo, ver para qué son dichas modificaciones. Así que al lío 😀

## ¿Qué hace MailWatch?

Ya he comentado que **MailWatch es un interfaz web para MailScanner**. Pero, para que funcione, tenemos que hacer que **MailScanner guarde información en una base de datos**. Es decir, tras los análisis realizados a los mails, vamos a indicar a MailScanner que sus conclusiones las meta en una base de datos MySQL y **MailWatch será el encargado de visualizarnos estas conclusiones**, haciendo un análisis de los resultados globales.

## Pasos previos

Dado que MailWatch es un interfaz web, vamos a necesitar de un servidor web (usaremos **[Apache](https://httpd.apache.org/)**), junto con las dependencias **[PHP](http://www.php.net/)** que necesita, así como de una base de datos **[MySQL](https://www.mysql.com/)**. Como Zimbra cuenta con su propio servidor web, tenemos que tener cuidado a la hora de instalar otro, para que no haya conflicto de puertos. Para ello, modificaremos el puerto del Apache que vamos a instalar. Para evitar cualquier problema durante la instalación de las dependencias, vamos a hacer una parada de Zimbra y después lo volveremos a levantar.

    root@mail:~# su - zimbra
    zimbra@mail:~$ zmcontrol stop
    Host mail.irontec.com
            Stopping vmware-ha...skipped.
                    /opt/zimbra/bin/zmhactl missing or not executable.
            Stopping zmconfigd...Done.
            Stopping zimlet webapp...Done.
            Stopping zimbraAdmin webapp...Done.
            Stopping zimbra webapp...Done.
            Stopping service webapp...Done.
            Stopping stats...Done.
            Stopping mta...Done.
            Stopping spell...Done.
            Stopping snmp...Done.
            Stopping cbpolicyd...Done.
            Stopping archiving...Done.
            Stopping opendkim...Done.
            Stopping amavis...Done.
            Stopping antivirus...Done.
            Stopping antispam...Done.
            Stopping proxy...Done.
            Stopping memcached...Done.
            Stopping mailbox...Done.
            Stopping logger...Done.
            Stopping dnscache...Done.
            Stopping ldap...Done.

### Instalación de las dependencias

Tal como hemos dicho, vamos a realizar la instalación del servidor web **Apache**, junto con la base de datos **MySQL** (en Ubuntu podemos instalar la versión 5.5 o la 5.6, por lo que cogemos la última) y el lenguaje de programación **PHP** junto con los módulos para Apache y MySQL:

    root@mail:~# apt-get install apache2 libapache2-mod-php5  php5-mysql php5-gd mysql-server-5.6
    Leyendo lista de paquetes... Hecho
    Creando árbol de dependencias       
    Leyendo la información de estado... Hecho
    Se instalarán los siguientes paquetes extras:

    [...]

    ¿Desea continuar? [S/n] S

    [...]

Durante la instalación nos va a pedir que introduzcamos una contraseña de administración de MySQL.

### Configuración puerto del Apache

Vamos a modificar Apache para que en lugar de que escuche en el puerto 80, pase a escuchar en el **puerto 81**. Los ficheros de configuración a modificar son **/etc/apache2/ports.conf** y **/etc/apache2/sites-available/000-default.conf** , y en ellos buscamos donde pone 80 **para substituirlo por 81**.

Tras esto, realizamos un reinicio del servicio y nos aseguramos que ya está escuchando en el puerto 81:

![apache](/assets/images/apache.png)

    root@mail:/etc/apache2# service apache2 restart
     * Restarting web server apache2
       ...done.

    root@mail:/etc/apache2# netstat -naput | grep apache
    tcp6       0      0 :::81                   :::\*                    LISTEN      2575/apache2

Todo bien, así que podemos seguir 😀

### Configuración MySQL

Vamos a crear una base de datos junto con un usuario para uso exclusivo de MailScanner y MailWatch:

![mysql](/assets/images/mysql-logo.png)

    root@mail:~# mysql -u root -p
    Enter password: 
    Welcome to the MySQL monitor.  Commands end with ; or \g.
    Your MySQL connection id is 45
    Server version: 5.6.27-0ubuntu0.14.04.1 (Ubuntu)

    Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

Y creamos la base de datos y el usuario para acceder a ella.

    mysql> create database mailscanner;
    Query OK, 1 row affected (0,00 sec)

    mysql> GRANT ALL ON mailscanner.* TO mailwatch@localhost IDENTIFIED BY 'password';
    Query OK, 0 rows affected (0,00 sec)

    mysql> GRANT FILE ON *.* TO mailwatch@localhost IDENTIFIED BY 'password';
    Query OK, 0 rows affected (0,00 sec)

    mysql> flush privileges;
    Query OK, 0 rows affected (0,00 sec)

Guardaremos el usuario **MailWatch** y la contraseña introducida para utilizarla posteriormente

## Instalación de MailWatch

Durante la instalación de MailWatch vamos a realizar una serie de _hacks_ para mejorar el rendimiento del mismo, ya que, por defecto, no vienen aplicados. Tal como hemos dicho al principio, haremos varios saltos entre la configuración de MailWatch y MailScanner, por lo que tenemos que estar atentos a qué ficheros vamos tocando y con qué finalidad.

### Descargar MailWatch

El código fuente de MailWatch está almacenado en su [github](https://github.com/mailwatch/1.2.0).

    root@mail:~# wget https://github.com/mailwatch/1.2.0/archive/master.zip -O mailwatch-1.2.zip

    --2016-01-22 11:37:00--  https://github.com/mailwatch/1.2.0/archive/master.zip
    Resolving github.com (github.com)... 192.30.252.130
    Connecting to github.com (github.com)|192.30.252.130|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://codeload.github.com/mailwatch/1.2.0/zip/master [following]
    --2016-01-22 11:37:00--  https://codeload.github.com/mailwatch/1.2.0/zip/master
    Resolving codeload.github.com (codeload.github.com)... 192.30.252.163
    Connecting to codeload.github.com (codeload.github.com)|192.30.252.163|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 4739930 (4,5M) [application/zip]
    Saving to: ‘mailwatch-1.2.zip’

    100%[============================================================>] 4.739.930   5,13MB/s   in 0,9s   

    2016-01-22 11:37:01 (5,13 MB/s) - ‘mailwatch-1.2.zip’ saved [4739930/4739930]

    root@mail:~# unzip mailwatch-1.2.zip
    [...]

    root@mail:~# cd 1.2.0-master/

    root@mail:~/1.2.0-master# ls
    CHANGELOG.md                INSTALL      MailScanner_perl_scripts  upgrade.php
    CONTRIBUTING.md             LICENSE.md   README.md                 UPGRADING
    create.sql                  luser        Remote_DB.txt             USER_FILTERS
    fix_quarantine_permissions  mailscanner  tools

Hemos descargado el código, descomprimido el fichero y, por último, podemos ver los ficheros que contiene. Tal como se puede ver, existe un fichero **INSTALL** que nos indica los pasos a dar 😉

### Configuración de MailWatch

MailWatch nos provee de un fichero para la creación de tablas de la base de datos, pero que vamos a modificar para que las tablas sean creadas con el motor InnoDB de MySQL en lugar del antiguo MyISAM:

    root@mail:~/1.2.0-master# sed -ie "s/MyISAM/InnoDB/" create.sql

    root@mail:~/1.2.0-master# mysql -u root -p < create.sql
    Enter password:

Con esto ya tenemos el esquema de la base de datos creado y ahora vamos a generar un usuario **admin** para poder acceder posteriormente con él a través del navegador web. Fijaros que ahora vamos a conectarnos a la base de datos con el usuario **mailwatch** que hemos creado más arriba. El usuario **admin** y la contraseña que pongo a continuación lo guardaremos para acceder a través del navegador:

    root@mail:~# mysql -u mailwatch -p mailscanner
    Enter password:

    [...]

    mysql> INSERT INTO users SET username = 'admin', password = md5('webpassword'), fullname = 'Admin', type ='A';
    Query OK, 1 row affected (0,00 sec)

Instalamos dependencias de Perl necesarias por MailWatch, que es el módulo **Encoding::FixLatin** :

    root@mail:~# cpan -i Encoding::FixLatin
    Reading '/root/.cpan/Metadata

    [...]

    Appending installation info to /usr/local/lib/perl/5.18.2/perllocal.pod
      GRANTM/Encoding-FixLatin-1.04.tar.gz
      /usr/bin/make install  -- OK

Vamos a editar el fichero **MailScanner\_perl\_scripts/MailWatch.pm** que luego usará MailScanner para introducir sus resultados en la base de datos usada por MailWatch. Tenemos que asegurar que cambiamos lo siguiente

    # Modify this as necessary for your configuration
    my($db_name) = 'mailscanner';
    my($db_host) = 'localhost';
    my($db_user) = 'mailwatch';
    my($db_pass) = 'password';

Y movemos este fichero a **/etc/MailScanner/custom/** . Si os fijáis, este directorio es un enlace simbólico a **/usr/share/MailScanner/MailScanner/CustomFunctions/** :

    root@mail:~/1.2.0-master# mv  MailScanner_perl_scripts/MailWatch.pm  /etc/MailScanner/custom/

    root@mail:~/1.2.0-master# ls -lh /etc/MailScanner/custom
    lrwxrwxrwx 1 root root 51 ene 19 12:34 /etc/MailScanner/custom -> /usr/share/MailScanner/MailScanner/CustomFunctions/

Hasta ahora hemos estado con la base de datos, pero todavía falta de instalar la parte del código web al que posteriormente accederemos vía navegador. El interfaz propiamente dicho está en el directorio **mailscanner**, dentro del fichero descargado de mailwatch, y lo vamos a dejar en el directorio correspondiente al que Apache accede por defecto, en el caso de Ubuntu 14.04 es **/var/www/html/** . También tenemos que asegurar que el usuario de apache (**www-data**) tiene permisos para acceder a unos directorios:

    root@mail:~/1.2.0-master# mv mailscanner/ /var/www/html/

    root@mail:~/1.2.0-master# cd /var/www/html/mailscanner/

    root@mail:/var/www/html/mailscanner# chown root:www-data images

    root@mail:/var/www/html/mailscanner# chmod ug+rwx images

    root@mail:/var/www/html/mailscanner# chown root:www-data images/cache

    root@mail:/var/www/html/mailscanner# chmod ug+rwx images/cache

    root@mail:/var/www/html/mailscanner# chown www-data:www-data temp

Copiamos el fichero de configuración de ejemplo para crear el fichero **conf.php** donde realizaremos la configuración de MailWatch:

    root@mail:/var/www/html/mailscanner# cp conf.php.example conf.php

Y nos aseguramos que modificamos en el fichero las variables para que queden tal que:

    define('DB_USER', 'mailwatch');
    define('DB_PASS', 'password');
    define('DB_HOST', 'localhost');
    define('DB_NAME', 'mailscanner');

    define('TIME_ZONE', 'Europe/Madrid');

    define('QUARANTINE_FROM_ADDR', 'admin@irontec.com');

Y ya podemos ir al navegador web, **http://[SERVER]:81/mailscanner** (donde SERVER es la IP o la resolución DNS de nuestro servidor). Nos pedirá el usuario **admin** con la contraseña que hemos introducido en MySQL previamente y veremos el siguiente interfaz::

![mailwatch](/assets/images/mailwatch.png)

### Configurar MailScanner

Como ya hemos visto, tenemos el interfaz web funcionando y, del post anterior, el MailScanner también; pero ahora falta realizar la integración entre ambos. Vamos a realizar modificaciones en el fichero de configuración de MailScanner **/etc/MailScanner/MailScanner.conf**  y nos aseguramos que las siguientes directivas están como os mostramos ahora:

    Always Looked Up Last = &MailWatchLogging

    Detailed Spam Report = yes

    Quarantine User = postfix
    Quarantine Group = www-data
    Quarantine Permissions = 0666

    Quarantine Whole Message = yes
    Quarantine Whole Messages As Queue Files = no

    Include Scores In SpamAssassin Report = yes

    Spam Actions = store deliver header "X-Spam-Status: Yes"
    High Scoring Spam Actions = store delete

    Is Definitely Not Spam = &SQLWhitelist
    Is Definitely Spam = &SQLBlacklist

Las últimas dos líneas son para poder tener las **whitelist** y **blacklist** en la base de datos, en lugar de tenerlo en fichero, como os enseñamos en el post anterior. Para ello, **en el directorio descargado de mailscanner** modificamos el fichero **MailScanner\_perl\_scripts/SQLBlackWhiteList.pm** para modificar los acceso a la base de datos, buscamos y dejamos las líneas así:

    my($db_name) = 'mailscanner';
    my($db_host) = 'localhost';
    my($db_user) = 'mailwatch';
    my($db_pass) = 'password';

Y movemos el fichero al directorio **custom** de la configuración del MailScanner:

    root@mail:~/1.2.0-master# mv MailScanner_perl_scripts/SQLBlackWhiteList.pm /etc/MailScanner/custom/

Reiniciamos Mailscanner:

    root@mail:~# /etc/init.d/mailscanner restart
     * Restarting mail spam/virus scanner MailScanner

       ...done.
    root@mail:~#

Volvemos a iniciar Zimbra

    root@mail:~# su - zimbra
    zimbra@mail:~$ zmcontrol start
    Host mail.irontec.com
            Starting ldap...Done.
            Starting zmconfigd...Done.
            Starting dnscache...Done.
            Starting logger...Done.
            Starting mailbox...Done.
            Starting memcached...Done.
            Starting proxy...Done.
            Starting amavis...Done.
            Starting antispam...Done.
            Starting antivirus...Done.
            Starting opendkim...Done.
            Starting snmp...Done.
            Starting spell...Done.
            Starting mta...Done.
            Starting stats...Done.
            Starting service webapp...Done.
            Starting zimbra webapp...Done.
            Starting zimbraAdmin webapp...Done.
            Starting zimlet webapp...Done.

Y si nos mandamos un mail de una cuenta a otra ya veremos cómo aparece en el MailWatch:

![mailwatch](/assets/images/mailwatch-mails.png){: .border}

Si hacemos click en el **#** de la izquierda de cada mail veremos toda la información del mail, sus cabeceras, la puntuación del SpamAssassin…

## Configuración final

Tras los pasos dados hasta ahora, ya tendríamos MailScanner y MailWatch integrados y funcionando, pero vamos a realizar una serie de configuración extra para poder mejorarlo.

### CRONs de mantenimiento

Dado que actualmente estamos metiendo información de los correos electrónicos a la base de datos, en caso de tener mucho tráfico, nuestra base de datos irá cogiendo cierto tamaño, lo que puede repercutir en el espacio utilizado en el servidor, así como en la velocidad de carga del interfaz web MailWatch. En el fichero **INSTALL** de MailWatch recomiendan la instalación de varios scripts que nos facilitan, para que sean ejecutados por el dominio **CRON**. Estos scripts se encuentran en el directorio **tools/Cron\_jobs** del directorio descomprimido al principio de este post.

Os dejo un resumen de los scripts  que pueden ser de utilidad, pero os dejamos a vuestro criterio la instalación de cada uno de ellos:

- **db_clean.php** : Borra de la base de datos de los mails anteriores a una fecha. Para ello, tiene en cuenta un par de variables del fichero de configuración de MailWatch **conf.php** , que por defecto son 60 días.
- **quarantine_maint.sh** : Dado que los mails en cuarentena son guardados en disco, llegará un momento en el que estemos ocupando espacio con mails no liberados que es absurdo seguir manteniendo. Este script  borra de la base de datos y de disco los correos que llevan en cuarentena más de 30 días.
- **quarantine_report.php** : Tiene en cuenta varias variables de **conf.php** para la generación y envío de un correo electrónico con la información recopilada de la base de datos. Este correo nos informa de los mails que han llegado al servidor que son spam, virus, phishing,…

### Configuración GeoIP

Para conocer los países de los servidores emisores necesitamos contar con una base de datos en la que se relacione IP con país. Para ello vamos a la pestaña **Tools/Links**, pinchamos en el enlace «**Update GeoIP Database**» y hacer click en el botón **Run now**«. Este proceso se descargará dos ficheros (**GeoIP.dat** y **GeoIPv6.dat**) que son guardados en **/var/www/html/mailscanner/temp** y que contienen una base de datos interna. Con estos ficheros, al ir a visualizar las cabeceras de un correo procesado nos informará del país procedente del correo.

![mailwatch](/assets/images/mailscanner-geoip.png){: .border}

### Actualizamos reglas SpamAssassin

Dado que cada cierto tiempo SpamAssassin crea nuevas reglas, para que MailWatch las conozca hay que realizar una actualización de las mismas. Para ello vamos a la pestaña **Tools/Links**, pinchamos en el enlace «**Update SpamAssassin Rule Descriptions**» y hacemos click sobre el botón «**Run now**«. Veremos el nombre y la descripción de las reglas que son usadas por SpamAssassin.

### Listas blancas/negras

En el post anterior os explicamos cómo generar ficheros para la creación de listas blancas y negras para MailScanner. Con MailWatch, y la configuración que hemos realizado en el MailScanner, la creación de listas blancas y negras va a través del interfaz web. Por lo tanto, esta información estará guardada en la base de datos. Si os fijáis, hay una pestaña que se llama «**B/W LISTS**«. Pinchando sobre ella, nos muestra un formulario para añadir direcciones de correo electrónico, o un dominio completo como lista blanca o negra.

### Visualización de cola de correos

MailWatch puede mostrarnos cuantos mails hay en la cola de correos, pero debido a los permisos de los directorios (creados por Zimbra), el usuario **www-data** no tiene permisos para acceder a ellos, y por tanto esta información no nos la muestra. El usuario www-data es el encargado de ejecutar Apache en nuestro servidor. Vamos a dar permisos para que pueda acceder cualquier usuario a los directorios de las colas:

![status](/assets/images/mailscanner-status.png){: .border}

    root@mail:~# chmod 755 /opt/zimbra/data/postfix/spool/hold/

    root@mail:~# chmod 755 /opt/zimbra/data/postfix/spool/incoming/

Y tenemos que modificar el fichero de MailWatch **/var/www/html/mailscanner/postfix.inc** para asegurar que aparece lo siguiente:

    function  postfixinq()
    {
        $handle = opendir('/opt/zimbra/data/postfix/spool/hold/');

    [...]

Con esto ya podríamos ver la información del estado de las colas, que tal como se puede ver en la imagen, ahora mismo es de cero mails en las colas.

## Resumen

Tras este post, podríamos dar por terminado la **instalación y la configuración del servidor de correo electrónico** donde podemos ver y controlar los mails que llegan y se envían, controlar las listas blancas y negras, obtener estadísticas…

¿Cómo controláis vosotros el flujo de mails de vuestros servidores? ¿Habéis hecho algún hack al MailScanner o al MailWatch?
