---
layout: post
title:  "Mejorando nuestro servidor de correo Zimbra (1/2): seguridad con Mailscanner"
date:    2016-03-10 00:00:00 +0200
tags: zimbra mailscanner
---


En este post veremos cómo realizar la instalación del **sistema de seguridad para servidores de correo electrónico [MailScanner](https://www.mailscanner.info/)**. MailScanner nos va a ayudar a protegernos contra spam, virus, phising y malware dado que va a analizar cada correo electrónico que pasa por él.

Teniendo en cuenta el **[principio KISS](https://es.wikipedia.org/wiki/Principio_KISS)**, muy habitual en el mundo del Software Libre, MailScanner no reinventa la rueda y hace uso de otras herramientas específicas para cada uno de los análisis que realiza sobre los mails. Por ejemplo, **Spamassassin** para la detección del spam, **ClamAV** para los virus…

Este post pretende ser la continuación de otros dos anteriores en los que instalamos Zimbra como servidor de correo ([Creando un sistema de correo profesional con Zimbra 1]({% post_url 2016-01-21-zimbra-1 %}) y [Creando un sistema de correo profesional con Zimbra 2]({% post_url 2016-02-09-zimbra-2 %})), ya que vamos a centrar la instalación en un servidor con **Zimbra 8.6 Open Source Edition**. Tened en cuenta que MailScanner se puede utilizar con cualquier otro MTA, como **Postfix** o **Exim**.

![mailscanner](/assets/images/mailscanner.png){: .w-50 }

## ¿Qué hace MailScanner?

Tal como se suele decir, una imagen vale más que mil palabras, así que vamos con ella:

![mailscanner](/assets/images/mailscanner-workflow.png)

He querido dejar en grande esta imagen dada su importancia, ya que es la mejor manera de ver cómo funciona y qué hace MailScanner. Es tan explicativa que creo que es mejor no decir más 😛

Tal como hemos puesto en la introducción, MailScanner hace uso de otras herramientas muy conocidas, por lo que será necesario que estén instaladas, pero el instalador nos facilitará esta tarea.

### Descargar MailScanner

Desde la página de [descargas de Mailscanner](https://www.mailscanner.info/downloads/) podemos elegir el paquete para la distribución de nuestro servidor. Como recordaréis, el servidor donde instalamos Zimbra es una **Ubuntu 14.04 LTS**, por lo que elegimos el enlace «**Debian / Ubuntu**«:

    root@mail:~# wget https://s3.amazonaws.com/mailscanner/release/v4/deb/MailScanner-4.85.2-3.deb.tar.gz
    --2016-01-19 11:33:14--  https://s3.amazonaws.com/mailscanner/release/v4/deb/MailScanner-4.85.2-3.deb.tar.gz
    Resolving s3.amazonaws.com (s3.amazonaws.com)... 54.231.2.200
    Connecting to s3.amazonaws.com (s3.amazonaws.com)|54.231.2.200|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 826212 (807K) \[application/x-gzip\]
    Saving to: ‘MailScanner-4.85.2-3.deb.tar.gz’

    100%\[======================================================================>\] 826.212      702KB/s   in 1,1s

    2016-01-19 11:33:15 (702 KB/s) - ‘MailScanner-4.85.2-3.deb.tar.gz’ saved \[826212/826212\]

    root@mail:~# tar -xzf MailScanner-4.85.2-3.deb.tar.gz 
    root@mail:~# cd MailScanner-4.85.2-3
    root@mail:~/MailScanner-4.85.2-3# ls
    ChangeLog  COPYING  install.sh  mailscanner-4.85.2-3-noarch.deb  README  UPGRADE

Como podeis ver, una vez descomprimido el fichero descargado, tenemos el típico fichero README (al que no está de más echarle un ojo 😉 ), un paquete **.deb** y el instalador **install.sh**

### Instalación de MailScanner

Tal como hemos visto, hay un instalador, así que vamos a ejecutarlo:

    root@mail:~/MailScanner-4.85.2-3# ./install.sh

    MailScanner Installation for Debian Based Systems


    This will INSTALL or UPGRADE the required software for MailScanner on Debian based systems
    via the Apt package manager. Supported distributions are Debian 6,7 and associated variants
    such as Ubuntu. Internet connectivity is required for this installation script to execute.



    You may press CTRL + C at any time to abort the installation. Note that you may see
    some errors during the perl module installation. You may safely ignore errors regarding
    failed tests for optional packages.

    When you are ready to continue, press return ...

    Tal como podemos leer, nos dice que va a realizar la **instalación de las dependencias** que necesita MailScanner y que, si queremos,     podemos cancelar la instalación en cualquier momento. Para la instalación de las dependencias nos hará una serie de preguntas. Así que    pulsamos **intro** para continuar.

    Do you want to install a Mail Transfer Agent (MTA)?

    I can install an MTA via the apt package manager to save you the trouble of having to do
    this later. If you plan on using an MTA that is not listed below, you will have install 
    it manually yourself if you have not already done so.

    1 - sendmail
    2 - postfix
    3 - exim
    N - Do not install

    Recommended: 1 (sendmail)

    Install an MTA? \[1\] : N

Como en nuestro servidor ya tenemos instalado Zimbra, que es nuestro servidor de correo (**MTA**), le decimos que **NO** queremos instalar ninguno.

![spamassassin](/assets/images/SpamAssassin_logo.png){: .w-25}

```
Do you want to install or update Spamassassin?

This package is recommended unless you have your own spam detection solution.

Recommended: Y (yes)

Install or update Spamassassin? [n/Y] : Y
```

Confirmamos que queremos instalar [Spamassassin](http://spamassassin.apache.org/), que va a ser el encargado de realizar la detección de spam, dándole a los mails una puntuación según las reglas heurísticas de las que dispone.

![clamav](/assets/images/clamav.png){: .w-25}

    Do you want to install or update Clam AV during this installation process?

    This package is recommended unless you plan on using a different virus scanner.
    Note that you may use more than one virus scanner at once with MailScanner.

    Even if you already have Clam AV installed you should select this option so I
    will know to check the clamav-wrapper and make corrections if required.

    Recommended: Y (yes)

    Install or update Clam AV? [n/Y] : Y

De nuevo, confirmamos la instalación de [Clam AV](http://www.clamav.net/), que es el antivirus utilizado por Mailscanner.

    Do you want to install missing perl modules via CPAN?

    I will attempt to install Perl modules via apt, but some may not be unavailable during the
    installation process. Missing modules will likely cause MailScanner to malfunction.

    Recommended: Y (yes)

    Install missing Perl modules via CPAN? [n/Y] : Y

Dado que MailScanner está escrito en el lenguaje [Perl](https://www.perl.org/), se necesitan varias dependencias. Por defecto, se intentará realizar la instalación a través del sistema de paquetes **apt**, pero en caso de no tenerlas, las instalará a través del sistema CPAN.

    Do you want to ignore MailScanner dependencies?

    This will force install the MailScanner .deb package regardless of missing
    dependencies. It is highly recommended that you DO NOT do this unless you
    are debugging.

    Recommended: N (no)

    Ignore MailScanner dependencies (nodeps)? [y/N] : N

    Confirmamos que NO queremos ignorar las dependencias. Tras este paso, comenzará la instalación como tal:

    Installation results are being logged to mailscanner-install.log

    [...]

    Installing the MailScanner .deb package ...
    Seleccionando el paquete mailscanner previamente no seleccionado.
    (Leyendo la base de datos ... 110549 ficheros o directorios instalados actualmente.)
    Preparing to unpack .../mailscanner-4.85.2-3-noarch.deb ...
    Unpacking mailscanner (4.85.2-3) ...
    Configurando mailscanner (4.85.2-3) ...
     Adding system startup for /etc/init.d/mailscanner ...
       /etc/rc0.d/K20mailscanner -> ../init.d/mailscanner
       /etc/rc1.d/K20mailscanner -> ../init.d/mailscanner
       /etc/rc6.d/K20mailscanner -> ../init.d/mailscanner
       /etc/rc2.d/S20mailscanner -> ../init.d/mailscanner
       /etc/rc3.d/S20mailscanner -> ../init.d/mailscanner
       /etc/rc4.d/S20mailscanner -> ../init.d/mailscanner
       /etc/rc5.d/S20mailscanner -> ../init.d/mailscanner
    Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
    Processing triggers for ureadahead (0.100.0-16) ...

    ----------------------------------------------------------
    Installation Complete

    See http://www.mailscanner.info for more information and
    support via the MailScanner mailing list.

Cuando la instalación comienza nos empieza a sacar por pantalla todo lo que va haciendo. Afortunadamente, si nos fijamos, nos indica que todo lo que va a hacer se va a loguear en el fichero **mailscanner-install.log**, dentro del directorio donde estamos ejecutando el instalador. Si abrís el fichero tras la instalación podréis ver más tranquilamente qué es lo que ha ido pasando. El resumen es:

- Realiza un «apt-get update» para actualizar los repositorios
- Instala las primeras dependencias para MailScanner
- Instala Spamassassin y ClamAV
- Instala las librerías y dependencias Perl por CPAN
- Añade los scripts de arranque/parada al runlevel

## Integración de Zimbra y MailScanner

En este punto ya tenemos MailScanner instalado, pero falta toda la tarea de configurar Zimbra para «pasarle» los mails a MailScanner y que los analice. Para ser más específico, Zimbra no pasa los mails, si no que vamos a hacer que los deje en la cola **HOLD** (hay que recordar que Zimbra está basado en Postfix), MailScanner coge los mails de esa cola, y tras los análisis realizados, en caso de no ser rechazados, los «**inyecta**» en la cola de salida para que Zimbra los entregue. Podéis volver a mirar la imagen del principio del post para recordar cómo funciona Mailscanner.

### Configurar Zimbra

Tal como hemos dicho, Zimbra es el que deja los mails en su cola **HOLD** para que posteriormente sean recogidos por MailScanner, por lo que tenemos que realizar la siguiente configuración.

- Vamos a «**Configurar → Configuración General → Archivos adjuntos**» y desmarcamos la opción «Enviar notificación de extensión bloqueada al destinatario».
- Tenemos que editar el fichero **/opt/zimbra/conf/postfix_header_checks.in añadiendo** cierta información, pero hay que darle permisos al fichero. Desde consola sería, siendo root:

        zimbra@mail:~$ zmprov mcf zimbraMtaBlockedExtensionWarnRecipient FALSE

        root@mail:~# chmod 666 /opt/zimbra/conf/postfix_header_checks.in

        root@mail:~# echo "/^Received:/ HOLD" >> /opt/zimbra/conf/postfix_header_checks.in

        root@mail:~# chmod 444 /opt/zimbra/conf/postfix_header_checks.in

        root@mail:~# /opt/zimbra/postfix/sbin/postconf -e header_checks=pcre:/opt/zimbra/conf/postfix_header_checks

- También tenemos que modificar el fichero ****/opt/zimbra/postfix/conf/master.cf.in**** y la línea donde se especifica **qmgr** quede tal que:

        qmgr      fifo  n       -       n       300     1       qmgr

Tras esto, hay que realizar una parada del zimbra y volver a levantarlo, esta vez como usuario zimbra:

![zimbra](/assets/images/zimbra-logo2.png){: .w-25}

    root@mail:~# su - zimbra
    zimbra@mail:~$ zmcontrol restart
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

Si ahora intentamos mandar un mail desde el interfaz web (por ejemplo, desde la cuenta **_admin@irontec.com_** a **_ruben@irontec.com_**) deberíamos ver lo siguiente en el log **/var/log/mail.log** Fijaros en la línea destacada cómo indica «**hold: header Received**«.

    Jan 19 15:34:47 mail postfix/smtpd[24125]: connect from mail.irontec.com[123.1.2.3]
    Jan 19 15:34:47 mail postfix/smtpd[24125]: NOQUEUE: filter: RCPT from mail.irontec.com[123.1.2.3]: <admin@irontec.com>: Sender address    triggers FILTER smtp-amavis:[127.0.0.1]:10026; from=<admin@irontec.com> to=<ruben@irontec.com> proto=ESMTP helo=<mail.irontec.com>
    Jan 19 15:34:47 mail postfix/smtpd[24125]: 44BC1261C9B: client=mail.irontec.com[123.1.2.3]
    Jan 19 15:34:47 mail postfix/cleanup[24129]: 44BC1261C9B: hold: header Received: from mail.irontec.com (mail.irontec.com [123.1.2.3])??by     mail.irontec.com (Postfix) with ESMTP id 44BC1261C9B??for <ruben@irontec.com>; Tue, 19 Jan 2016 15:34:47 +0100 (CET) from mail.irontec.com    [123.1.2.3]; from=<admin@irontec.com> to=<ruben@irontec.com> proto=ESMTP helo=<mail.irontec.com>
    Jan 19 15:34:47 mail postfix/cleanup[24129]: 44BC1261C9B: message-id=<2057556805.5.1453214087168.JavaMail.zimbra@irontec.com>
    Jan 19 15:34:47 mail postfix/smtpd[24125]: disconnect from mail.irontec.com[123.1.2.3]

Ya hemos conseguido lo que queríamos, todos los mails que llegan al Zimbra se queden en la cola HOLD, siendo el directorio: **/opt/zimbra/data/postfix/spool/hold**

### Configurar MailScanner

Ahora tenemos que hacer que MailScanner vaya a por los mails, cogiéndolos de**/opt/zimbra/data/postfix/spool/hold**, los analice y los deje en la cola correspondiente del Zimbra para que luego vaya a su destino final. Teniendo en cuenta la extensión del fichero de configuración de MailScanner, aún a pesar de venir bien documentado, es buena idea tener abierta en otra pestaña la [documentación de MailScanner](https://www.mailscanner.info/MailScanner.conf.index.html).

Toda la configuración de MailScanner se realiza en el directorio **/etc/MailScanner/** y la gran mayoría de la configuración en el fichero **/etc/MailScanner/MailScanner.conf** . Si lo abrís, veréis que es muy extenso, pero en su mayoría son comentarios explicativos acerca de cada una de las configuraciones.

Dado que la configuración por defecto de **MailScanner.conf** está bastante bien, pero también es muy extensa, os dejamos las modificaciones que realizamos en el fichero para tener un sistema base que funcione con Zimbra:

    %org-name% = Irontec
    %org-long-name% = Irontec S.L.
    %web-site% = www.irontec.com
    %report-dir% = /etc/MailScanner/reports/es
    Max Children = 1
    Run As User = postfix
    Run As Group = postfix
    Incoming Queue Dir = /opt/zimbra/data/postfix/spool/hold
    Outgoing Queue Dir = /opt/zimbra/data/postfix/spool/incoming
    MTA = postfix
    Incoming Work Group = clamav
    Incoming Work Permissions = 0666
    Quarantine Permissions = 0666
    Maximum Archive Depth = 3
    Virus Scanners = clamd
    Allow Password-Protected Archives = yes
    ClamAVmodule Maximum Recursion Level = 2
    Clamd Lock File = /var/run/clamav/clamd.pid
    Quarantine Whole Message = yes
    Always Include SpamAssassin Report = yes
    Sign Clean Messages = no
    Mark Unscanned Messages = no
    Deliver Cleaned Messages = no
    Notify Senders = no
    Notices To = admin@irontec.com
    Local Postmaster = postmaster@irontec.com
    Spam List =  spamhaus-ZEN spamcop.net
    Required SpamAssassin Score = 4
    High SpamAssassin Score = 8
    SpamAssassin User State Dir = /var/spool/MailScanner/spamassassin
    SpamAssassin Install Prefix = /var/spool/MailScanner/spamassassin

**Nota:** De estas modificaciones, más adelante cambiaremos «Max Children» poniéndolo a 10, pero durante la fase de pruebas es mejor tener sólo un hijo para ver mejor los logs.

Debido a la funcionalidad **apparmor** de Ubuntu, hay que hacer una modificación para evitar el siguiente error que nos puede aparecer más adelante:

    Jan 20 11:30:50 mail MailScanner\[18516\]: Clamd::ERROR:: UNKNOWN CLAMD RETURN ./lstat() failed: Permission denied. ERROR :: /var/spool/MailScanner/incoming/18516

Para ello, tenemos que modificar el fichero **/etc/apparmor.d/local/usr.sbin.clamd** y añadir lo siguiente:

    /var/spool/MailScanner/\*\* rw,
    /var/spool/MailScanner/incoming/\*\* rw,

Y tendremos que hacer un **restart** de la configuración de dicho servicio:

    root@mail:~# service apparmor restart
    * Reloading AppArmor profiles
    Skipping profile in /etc/apparmor.d/disable: usr.sbin.rsyslogd
      ...done.

Para hacer que MailScanner arranque, hay que modificar el fichero **/etc/default/mailscanner** y asegurar que aparece la siguiente línea descomentada:

    run_mailscanner=1

Arrancamos el servicio

    root@mail:/etc/MailScanner# /etc/init.d/clamav-daemon start
    * Starting ClamAV daemon clamd
      ...done.

root@mail:~# /etc/init.d/mailscanner start

Y en el log **/var/log/mail.log** deberíais ver algo parecido a lo siguiente:

    Jan 20 12:59:24 mail MailScanner[13669]: MailScanner E-Mail Virus Scanner version 4.85.2 starting...
    Jan 20 12:59:24 mail MailScanner[13669]: Reading configuration file /etc/MailScanner/MailScanner.conf
    Jan 20 12:59:24 mail MailScanner[13669]: Reading configuration file /etc/MailScanner/conf.d/README
    Jan 20 12:59:24 mail MailScanner[13669]: Read 868 hostnames from the phishing whitelist
    Jan 20 12:59:24 mail MailScanner[13669]: Read 5807 hostnames from the phishing blacklists
    Jan 20 12:59:25 mail MailScanner[13669]: Using SpamAssassin results cache
    Jan 20 12:59:25 mail MailScanner[13669]: Connected to SpamAssassin cache database
    Jan 20 12:59:25 mail MailScanner[13669]: Expired 14 records from the SpamAssassin cache
    Jan 20 12:59:26 mail MailScanner[13669]: Connected to Processing Attempts Database
    Jan 20 12:59:26 mail MailScanner[13669]: Found 0 messages in the Processing Attempts Database
    Jan 20 12:59:26 mail MailScanner[13669]: Using locktype = flock

Y si mandamos de nuevo un mail veremos el siguiente log:

    Jan 20 13:00:12 mail postfix/smtpd[14603]: connect from mail.irontec.com[123.1.2.3]
    Jan 20 13:00:12 mail postfix/smtpd[14603]: NOQUEUE: filter: RCPT from mail.irontec.com[123.1.2.3]: <admin@irontec.com>: Sender address triggers FILTER smtp-amavis:[127.0.0.1]:10026; from=<admin@irontec.com> to=<ruben@irontec.com> proto=ESMTP helo=<mail.irontec.com>
    Jan 20 13:00:12 mail postfix/smtpd[14603]: 210F0261C9D: client=mail.irontec.com[123.1.2.3]
    Jan 20 13:00:12 mail postfix/cleanup[14606]: 210F0261C9D: hold: header Received: from mail.irontec.com (mail.irontec.com [123.1.2.3])??by mail.irontec.com (Postfix) with ESMTP id 210F0261C9D??for <ruben@irontec.com>; Wed, 20 Jan 2016 13:00:12 +0100 (CET) from mail.irontec.com[123.1.2.3]; from=<admin@irontec.com> to=<ruben@irontec.com> proto=ESMTP helo=<mail.irontec.com>
    Jan 20 13:00:12 mail postfix/cleanup[14606]: 210F0261C9D: message-id=<1454317676.81.1453291212065.JavaMail.zimbra@irontec.com>
    Jan 20 13:00:12 mail postfix/smtpd[14603]: disconnect from mail.irontec.com[123.1.2.3]


    Jan 20 13:00:14 mail MailScanner[13669]: New Batch: Scanning 1 messages, 1863 bytes
    Jan 20 13:00:14 mail MailScanner[13669]: Virus and Content Scanning: Starting
    Jan 20 13:00:15 mail MailScanner[13669]: Requeue: 210F0261C9D.AFE5C to CDCFE261CA1
    Jan 20 13:00:15 mail MailScanner[13669]: Uninfected: Delivered 1 messages
    Jan 20 13:00:15 mail MailScanner[13669]: Deleted 1 messages from processing-database


    Jan 20 13:00:15 mail postfix/qmgr[29083]: CDCFE261CA1: from=<admin@irontec.com>, size=1124, nrcpt=1 (queue active)
    Jan 20 13:00:15 mail postfix/dkimmilter/smtpd[14616]: connect from localhost[127.0.0.1]
    Jan 20 13:00:15 mail postfix/dkimmilter/smtpd[14616]: 537AA261C9D: client=localhost[127.0.0.1]
    Jan 20 13:00:15 mail postfix/smtp[14613]: CDCFE261CA1: to=<ruben@irontec.com>, relay=127.0.0.1[127.0.0.1]:10026, delay=3.3, delays=3.1/0.01/0/0.14, dsn=2.0.0, status=sent (250 2.0.0 from MTA(smtp:[127.0.0.1]:10030): 250 2.0.0 Ok: queued as 537AA261C9D)
    Jan 20 13:00:15 mail postfix/cleanup[14606]: 537AA261C9D: hold: header Received: from localhost (localhost [127.0.0.1])??by mail.irontec.com (Postfix) with ESMTP id 537AA261C9D??for <ruben@irontec.com>; Wed, 20 Jan 2016 13:00:15 +0100 (CET) from localhost[127.0.0.1]; from=<admin@irontec.com> to=<ruben@irontec.com> proto=ESMTP helo=<localhost>
    Jan 20 13:00:15 mail postfix/cleanup[14606]: 537AA261C9D: message-id=<1454317676.81.1453291212065.JavaMail.zimbra@irontec.com>
    Jan 20 13:00:15 mail postfix/dkimmilter/smtpd[14616]: disconnect from localhost[127.0.0.1]
    Jan 20 13:00:15 mail postfix/qmgr[29083]: CDCFE261CA1: removed


    Jan 20 13:00:21 mail MailScanner[13669]: New Batch: Scanning 1 messages, 3036 bytes
    Jan 20 13:00:21 mail MailScanner[13669]: Virus and Content Scanning: Starting
    Jan 20 13:00:21 mail MailScanner[13669]: Requeue: 537AA261C9D.A7668 to 1F000261CA4
    Jan 20 13:00:21 mail MailScanner[13669]: Uninfected: Delivered 1 messages
    Jan 20 13:00:21 mail MailScanner[13669]: Deleted 1 messages from processing-database


    Jan 20 13:00:21 mail postfix/qmgr[29083]: 1F000261CA4: from=<admin@irontec.com>, size=2359, nrcpt=1 (queue active)
    Jan 20 13:00:22 mail postfix/amavisd/smtpd[14625]: connect from localhost[127.0.0.1]
    Jan 20 13:00:22 mail postfix/amavisd/smtpd[14625]: 3D621261C9D: client=localhost[127.0.0.1]
    Jan 20 13:00:22 mail postfix/cleanup[14606]: 3D621261C9D: message-id=<1454317676.81.1453291212065.JavaMail.zimbra@irontec.com>
    Jan 20 13:00:22 mail postfix/qmgr[29083]: 3D621261C9D: from=<admin@irontec.com>, size=3099, nrcpt=1 (queue active)
    Jan 20 13:00:22 mail postfix/amavisd/smtpd[14625]: disconnect from localhost[127.0.0.1]
    Jan 20 13:00:22 mail postfix/smtp[14613]: 1F000261CA4: to=<ruben@irontec.com>, relay=127.0.0.1[127.0.0.1]:10032, delay=6.9, delays=6.4/0/0.01/0.52, dsn=2.0.0, status=sent (250 2.0.0 from MTA(smtp:[127.0.0.1]:10025): 250 2.0.0 Ok: queued as 3D621261C9D)
    Jan 20 13:00:22 mail postfix/qmgr[29083]: 1F000261CA4: removed
    Jan 20 13:00:22 mail postfix/lmtp[14626]: 3D621261C9D: to=<ruben@irontec.com>, relay=mail.irontec.com[123.1.2.3]:7025, delay=0.2, delays=0/0.01/0.13/0.06, dsn=2.1.5, status=sent (250 2.1.5 Delivery OK)
    Jan 20 13:00:22 mail postfix/qmgr[29083]: 3D621261C9D: removed

Analizando el log, he hecho unos saltos de línea para verlo mejor, se puede resumir en:

- Recepción del mail por parte del Postfix (proceso **smtpd**) de Zimbra y deajdo en la cola **HOLD**
- MailScanner comprueba que existe un mensaje (**_de salida_**), lo analiza y lo deja en la cola correspondiente de Zimbra
- Postfix (proceso **qmgr**) recoge el mail de la cola, y se lo entrega al proceso encargado de firmar y comprobar la firma **DKIM**
- Postfix recibe de nuevo el correo y lo deja en el buzón del usuario correspondiente

He querido mostraros cómo funciona en el caso de un mail local, cómo es recibido, analizado y entregado posteriormente. En caso de que fuese un **mail a un servidor externo**, el mail sería analizado, y entregado al servidor remoto correspondiente. El servidor receptor del correo, si tiene MailScanner (o un sistema similar) analizaría el mail de nuevo y lo entregaría a su buzón correspondiente.

## Configuración avanzada, extras

Hasta ahora hemos dejado configurado Mailscanner de manera sencilla para asegurar que funciona, pero hay configuración extra que deberíamos añadir para mejorar nuestro sistema. Si en vuestras configuraciones tenéis opciones que no hemos añadido, no dudéis en dejarnos vuestras mejoras en los comentarios 😀

### Configuración spamassassin

La configuración de Spamassassin podemos realizarla en sus ficheros de configuración (local.cf por ejemplo), pero MailScanner nos facilita el fichero **/etc/MailScanner/****spam.assassin.prefs.conf** para ello. Realizamos un enlace simbólico del fichero en el directorio donde está la configuración de Spamassassin para que nos aseguremos que luego será cargada:

root@mail:/etc/MailScanner# ln -s /etc/MailScanner/spam.assassin.prefs.conf /etc/spamassassin/mailscanner.cf

Editamos el fichero **/etc/MailScanner/****spam.assassin.prefs.conf** para modificar las líneas donde aparece **_X-YOURDOMAIN-COM_** para que sea como la variable **%org-name%** del fichero Mailscanner.conf, y una serie de configuración extra y ejemplos de reglas propias:

    bayes_path /etc/MailScanner/bayes/bayes
    bayes_file_mode 0777

    bayes_ignore_header X-Irontec-MailScanner
    bayes_ignore_header X-Irontec-MailScanner-SpamCheck 
    bayes_ignore_header X-Irontec-MailScanner-SpamScore 
    bayes_ignore_header X-Irontec-MailScanner-Information

    ifplugin Mail::SpamAssassin::Plugin::Pyzor
    pyzor_path /usr/bin/pyzor
    endif

    use_razor2     1
    use_pyzor      1

    # reglas propias
    score SPF_FAIL          4
    score SPF_SOFTFAIL      3
    score SPF_NEUTRAL       2

    envelope_sender_header X-Irontec-MailScanner-From

    # para locales e idiomas
    ok_locales en
    loadplugin Mail::SpamAssassin::Plugin::TextCat
    ok_languages es eu

    # ejemplo de regla propia
    header   Irontec_CASINO       Subject =~ /casinos/i
    describe Irontec_CASINO       Casinos
    score    Irontec_CASINO       1.5 

    # para los paises
    loadplugin Mail::SpamAssassin::Plugin::RelayCountry
    add_header all Relay-Country _RELAYCOUNTRY_
    include Relay_Countries.cf

Tenemos que crear el directorio donde se guardará la base de datos Bayes, y podemos aprovechar la base de datos de Zimbra para utilizar lo aprendido hasta ahora por él:

    root@mail:/etc/MailScanner# mkdir /etc/MailScanner/bayes

    root@mail:/etc/MailScanner# cp /opt/zimbra/.spamassassin/bayes_* /etc/MailScanner/bayes/

    root@mail:/etc/MailScanner# chmod 777 -R /etc/MailScanner/bayes

    Para el plugin **RelayCountry** necesitamos instalar una dependencia de perl:

    root@mail:/etc/MailScanner# apt-get install libgeo-ip-perl

    [...]

    Processing triggers for man-db (2.6.7.1-1ubuntu1) ...
    Configurando libgeo-ip-perl (1.43-1) ...

Y por último, si os fijáis, hemos hecho un **include** del fichero Relay_Countries.cf  Esto es un fichero, que os podéis descargar de **[aquí](/assets/images/Relay_Countries.txt)** y moverlo a **/etc/spamassassin/Relay_Countries.cf** .

    root@mail:~# cd /etc/spamassassin/

    root@mail:/etc/spamassassin# wget https://blog.irontec.com/wp-content/uploads/2016/01/Relay_Countries.txt

    root@mail:/etc/spamassassin# mv Relay_Countries.txt Relay_Countries.cf

Este fichero lo que hace es **puntuar** a los mails que vienen desde ciertos países con más puntuación o menos, dependiendo de nuestro gusto:

    header          RELAYCOUNTRY_CN    X-Relay-Countries =~/\bCN\b/
    describe        RELAYCOUNTRY_CN    Relayed through China
    score           RELAYCOUNTRY_CN    4.0

Para la configuración de **Pyzor**, tenemos que crear un directorio en la home del usuario Postfix (que es el que lo arranca), y posteriormente lanzar Pyzor para actualizarlo:

    root@mail:/etc/spamassassin# mkdir /opt/zimbra/postfix/.pyzor

    root@mail:/etc/spamassassin# chown postfix:postfix /opt/zimbra/postfix/.pyzor/

    root@mail:/etc/spamassassin# su - postfix
    $ pwd
    /opt/zimbra/postfix

    $ pyzor --homedir /opt/zimbra/postfix/.pyzor discover
    downloading servers from http://pyzor.sourceforge.net/cgi-bin/inform-servers-0-3-x

### Whitelist y blacklist

A través de MailScanner, y su configuración, podemos forzar a que unos mails sean siempre considerados como lícitos o como spam. Estas son las configuraciones a modificar en **/etc/MailScanner/MailScanner.conf**:

    # fichero de configuración para las whitelist
    Is Definitely Not Spam = %rules-dir%/spam.whitelist.rules

    # fichero de configuración para las blacklist
    Is Definitely Spam = %rules-dir%/spam.blacklist.rules

Con esta configuración lo que estamos indicando es que para que un mail sea considerado directamente como **_whitelisted_** (es decir, que siempre va a ser considerado lícito) mire las reglas del fichero dentro del path **%rules-dir%**, que al principio del fichero vemos que es /etc/MailScanner/rules , es decir, **/etc/MailScanner/rules/spam.whitelist.rules** . Y para considerar que un mail sea directamente Spam que mire el fichero **/etc/MailScanner/rules/spam.blacklist.rules**

Ejemplo de configuración de unas reglas para considerar los mails lícitos:

    # al poner yes, especificamos que SI son lícitos
    From:       123.4.5.6      yes
    From:       \*@bizkaia.org  yes
    From:       direccion@hotmail.com yes

    # el resto de mails se analizarán y puntuarán
    FromOrTo:   default     no

Y aquí os dejo un ejemplo para reglas que siempre sean considerado Spam:

    # al poner yes, especificamos que SON SPAM
    From:     *@address.com   yes
    From:     *@*.jp          yes
    From:     222.78.150.40   yes

    FromOrTo:   default     no

Para ver todos los tipos de reglas que podemos crear y para más opciones condicionales de este estilo lo mejor es echar un ojo al fichero que MailScanner nos proporciona: **/etc/MailScanner/rules/EXAMPLES**

## Resumen

En este post hemos **instalado y configurado un servicio extra de seguridad para nuestro entorno Zimbra** que nos ayudará en la jungla que es hoy día internet en lo que respecta a los sistemas de correo electrónico como el spam, el phising…

¡Comentad y ayudadnos a mejorar este post **añadiendo vuestras reglas contra el _spam_**!
