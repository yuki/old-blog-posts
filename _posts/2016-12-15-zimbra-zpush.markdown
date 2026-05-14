---
layout: post
title:  "Sincronización «Exchange ActiveSync» con Z-Push y Zimbra"
date:    2016-12-15 00:00:00 +0200
tags: zimbra zpush
---

Nuevo capítulo en nuestra travesía por Zimbra. Tras [instalar]({% post_url 2016-01-21-zimbra-1 %}), [configurar]({% post_url 2016-02-09-zimbra-2 %}), [securizar contra el spam]({% post_url 2016-03-10-mejorando-zimbra-1 %}) y [facilitar la administración]({% post_url 2016-04-07-mejorando-zimbra-2 %}) de nuestro Zimbra, hoy vamos a ver cómo mejorar la experiencia de usuario final. Para ello vamos a realizar la instalación y configuración del sistema **z-push** para la correcta sincronización de nuestro móvil contra **Zimbra**.

![z-push](/assets/images/zimbra-zpush.png){: .w-50}

## Qué es «Exchange ActiveSync»

**[Z-push](http://z-push.org/)** es una implementación **Open Source** del **protocolo «Exchange ActiveSync»**, originalmente desarrollado por Microsoft. El protocolo, creado en 1996, permite la sincronización de dispositivos con servidores que hablen este protocolo, básicamente para los correos electrónicos, contactos, notas y calendarios.

Es un protocolo que **se ha ido extendiendo** y hoy día los dispositivos móviles con **sistema operativo iOS, Android o WindowsPhone** lo implementan de serie.

### Z-Push y su implementación

Tal como hemos  comentado, Z-push implementa el protocolo Exchange ActiveSync, convirtiéndose en una capa intermedia entre el cliente (un móvil) y el servidor backend que contiene los datos, que en nuestro caso es **Zimbra**.

En la imagen de la web de Z-Push se explica claramente cómo actúa Z-Push, de intermediario entre el cliente y el backend.

![z-push arch](/assets/images/zpush_arch.png)

Como suele ser habitual en el mundo del **Software Libre**, aunque el programa inicialmente no contaba con la posibilidad de comunicarse con Zimbra, veremos que esto se sí se puede conseguir gracias a un plugin externo.

## Descarga de z-push

Dado que z-push es un sistema PHP, es necesaria la utilización de un servidor web, el cual debe estar securizado a través de un certificado SSL. Puesto que nuestro servidor Zimbra ya hace uso del puerto 443, vamos a realizar la instalación en otro servidor, aunque podríamos utilizar otro puerto independiente, tal como hicimos con la instalación del entorno web para Mailwatch.

### Pasos prévios

Para la instalación de z-push necesitamos un sistema GNU/Linux, y en nuestro caso la instalación la vamos a realizar sobre una **Debian 8.5** (aka **Jessie**). En este post no vamos a entrar en los detalles sobre la instalación, ya que lo único que hemos hecho es una instalación base, junto con el servicio openssh y la configuración base que realizamos en nuestros servidores.

### Instalar dependencias

Tal como hemos dicho, **z-push** es un sistema basado en **PHP**, por lo que necesitamos realizar la instalación de un servidor web junto con los módulos de PHP necesarios para el correcto funcionamiento de la aplicación.

    apt-get install apache2 php5 libapache2-mod-php5 php5-curl php-soap

Descargamos z-push, lo descomprimimos y lo movemos al directorio final donde lo configuraremos después en Apache:

    root@zpush:~# wget http://download.z-push.org/final/2.2/z-push-2.2.9.tar.gz
    --2016-04-05 12:02:11--  http://download.z-push.org/final/2.2/z-push-2.2.9.tar.gz
    Resolviendo download.z-push.org (download.z-push.org)... 62.245.227.250
    Conectando con download.z-push.org (download.z-push.org)[62.245.227.250]:80... conectado.
    Petición HTTP enviada, esperando respuesta... 200 OK
    Longitud: 467725 (457K) [application/x-gzip]
    Grabando a: “z-push-2.2.9.tar.gz”

    z-push-2.2.9.tar.gz   100%\[=========================>\] 456,76K  --.-KB/s   en 0,09s

    2016-04-05 12:02:12 (4,81 MB/s) - “z-push-2.2.9.tar.gz” guardado \[467725/467725\]


    root@zpush:~# tar -xzf z-push-2.2.9.tar.gz

    root@zpush:~# mv z-push-2.2.9 /var/www/zpush

## Configurar Apache

Tal como hemos dicho, **el servidor web debe escuchar en el  puerto 443**, securizado por SSL, por lo que vamos a realizar la correcta **configuración de un VirtualHost** para nuestro dominio. Para ello debemos realizar la **carga del módulo SSL**.

    root@zpush:/etc/apache2# a2enmod ssl
    Considering dependency setenvif for ssl:
    Module setenvif already enabled
    Considering dependency mime for ssl:
    Module mime already enabled
    Considering dependency socache_shmcb for ssl:
    Enabling module socache_shmcb.
    Enabling module ssl.
    See /usr/share/doc/apache2/README.Debian.gz on how to configure SSL and create self-signed certificates.
    To activate the new configuration, you need to run:
      service apache2 restart

Creamos el fichero **/etc/apache2/sites-available/010-zpush.conf** donde crearemos nuestra configuración de VirtualHost:

    <VirtualHost *:443>
        ServerAdmin sistemas@irontec.com
        ServerName zpush.irontec.com

        DocumentRoot /var/www/zpush
        Alias /Microsoft-Server-ActiveSync /var/www/zpush/index.php

        SSLEngine on
        SSLCertificateFile /etc/ssl/irontec/certificado.crt
        SSLCertificateKeyFile /etc/ssl/irontec/clave.key
        SSLCertificateChainFile /etc/ssl/irontec/cadena-certificado.crt

        <Directory /var/www/zpush>
            AllowOverride All
        
            php_flag magic_quotes_gpc off
            php_flag register_globals off
            php_flag magic_quotes_runtime off
            php_flag short_open_tag on
        </Directory>

        <DirectoryMatch \.svn>
                Order allow,deny
                Deny from all
        </DirectoryMatch>

        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/sync_access-ssl.log combined
        ErrorLog ${APACHE_LOG_DIR}/sync_error-ssl.log

    </VirtualHost>

Es muy importante que nos aseguremos de que no hemos escrito ningún fallo en la configuración. Para ello, ejecutaremos la comprobación de la sintaxis de la configuración de Apache, cargaremos el VirtualHost en la configuración final y haremos un reinicio del servicio

    root@zpush:/etc/apache2/sites-available# apache2ctl -t
    Syntax OK

    root@zpush:/etc/apache2/sites-available# a2ensite 010-theinit.conf 
    Enabling site 010-theinit.
    To activate the new configuration, you need to run:
      service apache2 reload

    root@zpush:/etc/apache2/sites-available# systemctl restart apache2

Tras esto, ya podremos ver a través del navegador cierta información si vamos a https://zpush.irontec.com/

![z-push web](/assets/images/z-push-web.png){: .border}

## Integrando Z-push con Zimbra

Como ya hemos dicho, Z-push nos permite la sincronización mediante ActiveSync, pero para que funcione con nuestro Zimbra **necesitamos que ambos se comuniquen.**  Este es el motivo por el que necesitamos el módulo **[ZimbraBackend](https://sourceforge.net/projects/zimbrabackend/files/)**. Para ello sólo hay que pinchar en el enlace anterior y descargarse en la última versión que aparece, en nuestro caso la **versión 64**. Descomprimimos y lo movemos al directorio final.

    root@zpush:~# tar -xzf zimbra64.tgz

    root@zpush:~# mv zimbra64/z-push-2/ /var/www/zpush/backend/zimbra

### Configuración z-push

Ahora ya sólo nos falta realizar la configuración del Z-push y que haga uso del módulo **zimbrabackend**. Para ello es necesario editar el fichero **/var/www/zpush/config.php**, y asegurarse de dejar la configuración del siguiente modo:

    define('TIMEZONE', 'Europe/Madrid');

    define('USE_FULLEMAIL_FOR_LOGIN', true);

    define('PROVISIONING', false);

    define('BACKEND_PROVIDER', 'BackendZimbra');

Hay que crear un par de directorios para que z-push guarde los logs y su estado:

    root@izsync:~# mkdir /var/lib/z-push/

    root@izsync:~# mkdir /var/log/z-push/

    root@izsync:~# chmod 777 /var/lib/z-push/  /var/log/z-push/

### Configuración ZimbraBackend

Modificamos el fichero **/var/www/zpush/backend/zimbra/config.php** y nos aseguramos de que quede configurado contra la URL del Zimbra:

    define('ZIMBRA_URL', 'https://mail.irontec.com');

    define('ZIMBRA_TIMEZONE', 'Europe/Madrid');

### Configuración Zimbra

Debido a que z-push va a estar realizando peticiones SOAP al servidor Zimbra, tenemos que hacer que nuestro Zimbra no se tome dichas peticiones como un ataque, ya que en ese caso bloquearía la IP. Para ello debes cambiar la IP 66.77.88.99 por **tu IP:**

    zimbra@iz:~$ zmprov mcf zimbraHttpThrottleSafeIPs 66.77.88.99

### Configurar AutoDiscover

Modificamos el fichero **autodiscover/config.php**

    define('TIMEZONE', 'Europe/Madrid');

    define('SERVERURL', 'https://zpush.irontec.com/Microsoft-Server-ActiveSync');

    define('USE_FULLEMAIL_FOR_LOGIN', true);

    define('BACKEND_PROVIDER', 'BackendZimbra');

### Confirmar que funciona

La mejor manera para confirmar que todo está funcionando de manera correcta es hacerlo a través de nuestro móvil, pero antes de eso **podemos hacer la prueba en el navegador**. Nos pedirá el usuario (el e-mail) y la contraseña que tenemos en el Zimbra. Si la autenticación funciona, nos aparecerá lo siguiente:

![z-push web](/assets/images/z-push-web-zimbra.png){: .border}

Tal como dice el mensaje, nuestro navegador «no sabe hablar» el protocolo ActiveSync. De ahí el mensaje. Esto significa que la comunicación con Zimbra es correcta, ya que funciona la autenticación, por lo que sólo quedaría probarlo en nuestro móvil.

## Extra

### Configurar logrotate

Dado que vamos a generar ficheros de logs, deberíamos realizar un rotado de los mismos para que no se llenen eternamente. Para ello crearemos el fichero **/etc/logrotate.d/z-push** con la siguiente configuración:

    /var/log/z-push/*.log {
        daily
        missingok
        rotate 14
        compress
        delaycompress
        notifempty
    }

### Para nota

Este post lo teníamos en la recámara desde hace tiempo, y desde entonces **Z-Push se ha actualizado** **permitiendo instalar la aplicación a través de repositorio**. Pero no os preocupéis. Podéis estar tranquilos porque **en breves realizaremos otro post donde veremos cómo instalarlo** y haremos alguna modificación en la configuración del backend de Zimbra. Hasta entonces, id probando esta configuración.
