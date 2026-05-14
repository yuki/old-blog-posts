---
layout: post
title:  "Novedades en las nuevas versiones de Zimbra y Mailscanner"
date:    2017-04-05 00:00:00 +0200
tags: zimbra mailscanner
---


Como buenos [BOFHs](https://es.wikipedia.org/wiki/Bastard_Operator_from_Hell) tenemos que mantener nuestros sistemas al día y cada poco tiempo toca actualizar el software y/o nos toca hacer instalaciones con las versiones más recientes que hay en el momento.

Como algunos actores, parece que me empiezo a encasillar hablando de **Zimbra** (donde ya hicimos la [instalación]({% post_url 2016-01-21-zimbra-1%}) y [configuración]({% post_url 2016-04-07-mejorando-zimbra-2 %})), con la creación de un [sistema antispam basado en **Mailscanner**]({% post_url 2016-03-10-mejorando-zimbra-1 %}), al que [añadimos una aplicación web para controlarlo con **MailWatch**]({% post_url 2016-04-07-mejorando-zimbra-2 %}).

Hoy toca hablar de las últimas versiones de cada una de las piezas de nuestro puzzle.

## Antes de empezar: contraindicaciones

Seamos sinceros, a todos nos gusta probar las últimas versiones del software que usamos habitualmente. Da igual que sea de nuestro sistema de escritorio favorito (no voy a entrar en flames 😛 ), la última versión de nuestro mediacenter favorito ([Kodi](https://kodi.tv/)), o ya hablando en serio, de los servicios que tenemos en producción (léase [Docker]({% post_url 2018-10-29-introduccion-muy-breve-y-desenfadada-a-docker %}), [PHP](https://php.net), …)

Pero todos sabemos los problemas que conlleva hacer actualizaciones «a lo cowboy™» o instalar las últimas versiones sin dejar un poco de margen a que otros sean los atrevidos. Por eso siempre es recomendable tener un buen laboratorio de pruebas donde poder realizar pruebas hasta que podamos ponerlo en producción.

Por otro lado, las ventajas de tener nuestro sistema actualizado son mucho mayores al «si funciona no lo toques», entre las que podemos destacar correcciones de bugs, **arreglos de vulnerabilidades**, nuevas features…

## Ubuntu 16.04

Aunque no entra dentro del sistema de correo electrónico como tal, cabe destacar que desde hace ya un año contamos con la última versión **LTS** de Ubuntu, siendo la **16.04** o también conocida con su sobrenombre **[Xenial Xerus](https://wiki.ubuntu.com/XenialXerus/ReleaseNotes).** Como ya sabréis, las versiones LTS son consideradas estables durante más tiempo que las normales, y Canonical les da más soporte durante más tiempo.

![ubuntu xenial](/assets/images/ubuntu-xenial.png){: .w-25}

El trío Zimbra+Mailscanner+Mailwatch es «agnóstico» del SO, y podríamos instalarlo en otras distribuciones, pero dado que en mis post anteriores hablé de la instalación sobre Ubuntu, no está de más acordarse de esta «nueva» versión. Seguro que todos la tenéis instalada en algún servidor.

## Zimbra 8.7

Hace pocos días salió una actualización menor de esta gran suite de colaboración, [Zimbra 8.7.7](https://www.zimbra.org/download/zimbra-collaboration). No voy a explicar cómo realizar la actualización ni cómo realizar la instalación, ya que es muy parecido a la [explicada anteriormente]({% post_url 2016-01-21-zimbra-1 %}), pero durante la instalación la novedad radica en la siguiente pregunta:

    ...
    Use Zimbra's package repository \[Y\] Y

    Importing Zimbra GPG key

    Configuring package repository

    Checking for installable packages
    ...

![zimbra](/assets/images/zimbra-logo2.png){: .w-25}

Tal como se ve, ahora podemos hacer la instalación de Zimbra haciendo uso de sus propios repositorios y así usar sus propios paquetes. Esto facilita la actualización de los paquetes necesarios por parte de ellos, que en caso de haber una vulnerabilidad, no se hace necesario esperar a que las distribuciones hagan la actualización, si no que el paquete que se actualizará será de los propios repositorios de Zimbra.

Si después de hacer la instalación buscamos por los paquetes instalados que contienen el nombre **zimbra**, veremos un listado parecido al siguiente:

    root@mail:~# dpkg -l zimbra-a\*
    Deseado=desconocido(U)/Instalar/eliminaR/Purgar/retener(H)
    | Estado=No/Inst/ficheros-Conf/desempaqUetado/medio-conF/medio-inst(H)/espera-disparo(W)/pendienTe-disparo
    |/ Err?=(ninguno)/requiere-Reinst (Estado,Err: mayúsc.=malo)
    ||/ Nombre                 Versión          Arquitectura     Descripción
    +++-======================-================-================-=================================================
    ii  zimbra-altermime       0.3.20100505-1zi amd64            altermime Binaries
    ii  zimbra-amavis-logwatch 1.51.03-1zimbra8 amd64            amavis-logwatch Binaries
    ii  zimbra-amavisd         2.10.1-1zimbra8. amd64            amavisd Binaries
    ii  zimbra-apache          8.7.6.GA.1776.UB amd64            Best email money can buy
    ii  zimbra-apache-base     1.0.0-1zimbra8.7 all              Zimbra Apache Base
    ii  zimbra-apache-componen 1.0.0-1zimbra8.7 all              Zimbra components for apache package
    ii  zimbra-apr-lib:amd64   1.5.2-1zimbra8.7 amd64            Apache Portable Runtime Libraries
    ii  zimbra-apr-util-lib:am 1.5.4-1zimbra8.7 amd64            Apache Portable Runtime Libraries
    ii  zimbra-aspell          0.60.6.1-1zimbra amd64            Aspell Binaries
    ii  zimbra-aspell-ar       1.2.0-1zimbra8.7 amd64            Aspell Arabic Dictionary
    ii  zimbra-aspell-da       1.4.42.1-1zimbra amd64            Aspell Danish Dictionary
    ii  zimbra-aspell-de       20030222.1-1zimb amd64            Aspell German Dictionary
    ii  zimbra-aspell-en       7.1.0-1zimbra8.7 amd64            Aspell English Dictionary
    ii  zimbra-aspell-es       1.11.2-1zimbra8. amd64            Aspell Spanish Dictionary
    ...

Como consecuencia de esto, ya no aparecen los típicos enlaces simbólicos que existían antes en **/opt/zimbra.** Os dejo un ejemplo de un Zimbra versión 8.6 donde vemos cómo aparece el servicio **clamav** que apunta a la versión utilizada en el momento de la instalación, así como la de **postfix.**

    root@mail:/opt/zimbra# ls postfix\* clamav\* -dl
    lrwxrwxrwx 1 root root   25 oct  8  2015 clamav -> /opt/zimbra/clamav-0.98.4
    drwxr-xr-x 9 root root 4096 oct  8  2015 clamav-0.98.4
    lrwxrwxrwx 1 root root   29 oct  8  2015 postfix -> /opt/zimbra/postfix-2.11.1.2z
    drwxr-xr-x 8 root root 4096 oct  9  2015 postfix-2.11.1.2z

Si actualizáis de 8.6 a 8.7 esos enlaces no desaparecen, y la verdad es que se queda un poco feo, pero no hay problema en borrarlos, ya que si os fijáis, serán enlaces huérfanos…

El mayor cambio en el que están trabajando es que la actualización de Zimbra se pueda hacer como un paquete más del sistema (en nuestro caso, con «apt upgrade»), pero todavía no lo han terminado de implementar. Esto hará que la actualización del «core» de Zimbra sea más sencilla y rápida, lo que nos ayudará a tener el sistema actualizado y que no lo retrasemos en el tiempo. Además, desde la versión 8.7.2 están sacando nuevas versiones (con correcciones de bugs menores) cada dos semanas.

Dejando de lado las novedades a nivel de instalación, hay que destacar que la versión Zimbra 8.7 viene con **muchas novedades**, entre las que podríamos destacar:

- [Factor de doble autenticación](https://www.zimbra.com/email-server-software/twfactor-authentication/): para mejorar la seguridad en el acceso a nuestro buzón (sólo disponible en la versión Network Edition).
- [SSL SNI](https://www.zimbra.com/email-server-software/ssl-sni/): posibilidad de tener múiples certificados SSL en la misma IP.
- [Postscreen](https://www.zimbra.com/email-server-software/email-security-postscreen/): gracias a la actualización interna de postfix a la versión 3.1, se hace uso de [postscreen](http://www.postfix.org/POSTSCREEN_README.html), que se encarga de mantener a raya las conexiones no legítimas.

Si queréis ver todo el listado de modificaciones internas y bugs corregidos los podéis encontrar en su [wiki](https://wiki.zimbra.com/wiki/Zimbra_Releases/8.7.0).

## Mailscanner 5

Pocas semanas después del post sobre [instalar Mailscanner]({% post_url 2016-03-10-mejorando-zimbra-1 %}),  apareció la versión 5.0 de este gran software encargado de facilitarnos la vida contra la lucha del spam, virus y phising. En su [Changelog](https://github.com/MailScanner/v5/blob/master/changelog) podéis echar un ojo a las últimas novedades.

![mailscanner](/assets/images/mailscanner.png){: .w-25}

Como principal novedad (desde la v4.86), es que durante la instalación podemos especificar la creación de un sistema **ramdisk** para que el análisis de los mails sea más rápido. Podremos crearlo de distintos tamaños en la instalación (desde 256M hasta 8192M).

A nivel de configuración, el fichero principal (**/etc/Mailscanner/Mailscanner.conf**) no varía apenas y sigue la misma nomenclatura que en versiones anteriores, siendo la novedad el fichero secundario **/etc/Mailscanner/defaults** . Éste se encargará de tener la configuración para que Mailscanner arranque, las opciones de CRON, el permitir actualizar las reglas de Spamassassin, phising y alguna cosilla más.

Durante la instalación, también se **nos creará los fichero de CRON** en **/etc/cron.hourly/mailscanner** y **/etc/cron.daily/mailscanner** , que tal como he comentado, harán uso de la configuración de /etc/Mailscanner/defaults

Como última novedad, en la instalación nos creará varios scripts que empiezan con «**ms-«** que nos facilitarán la administración del sistema:

    root@mail:/etc/MailScanner# ms-
    ms-check              ms-cron               ms-msg-alert          ms-sa-cache           ms-update-sa          ms-upgrade-conf
    ms-clean-quarantine   ms-d2mbox             ms-peek               ms-update-bad-emails  ms-update-safe-sites
    ms-create-locks       ms-df2mbox            ms-perl-check         ms-update-bad-sites   ms-update-vs

Como veis, casi todos los nombres son bastante auto-explicativos, por lo que os dejo como deberes que les echéis un ojo, ya que son shell-scripts y fácil de entender las tareas que hacen.

Como adelanto, os enseño la salida de **ms-sa-cache**, que nos saca un resumen de la caché del SpamAssassin y de virus detectados:

    root@mail:/etc/MailScanner# ms-sa-cache
    --------- TOTALS ---------
    Total records:       8
    First seen (oldest): 3860 sec
    First seen (newest): 270 sec
    Last seen (oldest):  3860 sec
    Last seen (newest):  270 sec
    Cache Hit Rate       0%
    -------- NON-SPAM --------
    Total records:       6
    First seen (oldest): 2047 sec
    First seen (newest): 270 sec
    Last seen (oldest):  2047 sec
    Last seen (newest):  270 sec
    -------- LOW-SPAM --------
    Total records:       0
    First seen (oldest): 0 sec
    First seen (newest): 0 sec
    Last seen (oldest):  0 sec
    Last seen (newest):  0 sec
    ------- HIGH-SPAM --------
    Total records:       2
    First seen (oldest): 3860 sec
    First seen (newest): 2308 sec
    Last seen (oldest):  3860 sec
    Last seen (newest):  2308 sec
    -------- VIRUSES  --------
    Total records:       0
    First seen (oldest): 0 sec
    First seen (newest): 0 sec
    Last seen (oldest):  0 sec
    Last seen (newest):  0 sec
    ----- TOP 5 HASHES -------
    MD5                                     COUNT   FIRST   LAST

## MailWatch 1.2.2

Dentro de nuestro stack de servicios para la gestión del correo electrónico, MailWatch es el que menos se ha actualizado, pero también hay que recordar que su función es más limitada. Han realizado varios cambios internos, pero que a nivel visual/web no son apreciables. Entre los cambios, la creación de un fichero de configuración para la integración de MailScanner, llamado **MailWatch.pm** y que por fin la creación de las tablas de la base de datos hace uso del motor InnoDB, por lo que os ahorráis un paso en las explicaciones que os di en el post [Mejorando nuestro servidor Zimbra (2/2): MailWatch]({% post_url 2016-04-07-mejorando-zimbra-2 %})

También han centralizado las tareas CRON, haciendo que únicamente haya que copiar el fichero que viene en el zip **tools/Cron\_jobs/mailwatch** para dejarlo en **/etc/cron.daily/,** y los ficheros de CRON meterlos en **/usr/local/bin** . Leeros el fichero INSTALL que os darán más detalle de los cambios 😉

Para estar atentos a los cambios lo mejor es que vayáis a su [GitHub](https://github.com/mailwatch/MailWatch) . Esperemos que en breve hagan un cambio a nivel de interfaz, para darle un aspecto más renovado y moderno, aunque también es verdad que el cambio estético no es necesario si las tripas funcionan bien 😀
