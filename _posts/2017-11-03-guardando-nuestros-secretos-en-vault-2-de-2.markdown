---
layout: post
title:  "Guardando nuestros secretos en Vault (2 de 2)"
date:    2017-11-03 00:00:00 +0200
tags: vault passwords
---

Hoy vamos a seguir hablando de [secretos y Vault]({% post_url 2017-10-17-guardando-nuestros-secretos-en-vault-1-de-2 %}), realizando una **configuración real del servidor y afianzando parte de lo que hablamos en el [anterior post.]({% post_url 2017-10-17-guardando-nuestros-secretos-en-vault-1-de-2 %})**

## Haciendo Vault persistente

Tal como comentamos en el post anterior, **el modo «-dev» de Vault sirve para hacer pruebas**, y cualquier secreto y configuración que hagamos se pierde cuando el servicio se para. Por lo tanto, vamos a realizar una configuración persistente del servicio.

### Antes de empezar

Como ya vimos, de momento **no existen paquetes para Vault**, por lo que tenemos que instalar el binario, y elegimos dejarlo en **/usr/local/sbin** . Ahora necesitamos crear los directorios donde se guardará el fichero de configuración y los propios datos cifrados:

    root@vault:~# mkdir /etc/vault

    root@vault:~# mkdir -p /var/lib/vault/secrets

![vault-logo](/assets/images/vault-logo.png){: .w-25}

### Cifrando la comunicación con Vault

Toda comunicación con Vault **tiene que ir cifrada,** ya que no tendría mucho sentido que fuese en texto plano a través de la red. Vamos a generar un certificado auto-firmado para el servidor de Vault, el cual tiene que incluir **SubjectAlternativeNames** (SAN), algo que exige modificaciones del fichero **/etc/ssl/openssl.cnf** y que haremos creando un fichero aparte en **/etc/vault/openssl.cnf**

    [ req ]
    default_bits = 4096
    distinguished_name = req_distinguished_name
    req_extensions = req_ext

    [ req_distinguished_name ]
    countryName = Country Name (2 letter code)
    stateOrProvinceName = State or Province Name (full name)
    localityName = Locality Name (eg, city)
    organizationName = Organization Name (eg, company)
    commonName = Common Name (e.g. server FQDN or YOUR name)

    [ req_ext ]
    subjectAltName = @alt_names

    [alt_names]
    DNS.1 = vault.irontec.com
    DNS.2 = vault.irontec.vpn
    IP.1 = 66.66.66.66
    IP.2 = 192.168.1.10

Como podéis apreciar, en el apartado «**\[alt\_names\]**» hemos puesto los DNS y las IPs con las que accederemos al servicio.

Generamos la CA para Vault y el certificado dentro del directorio **/etc/vault/ssl**:

    root@vault:~# mkdir /etc/vault/ssl ; cd /etc/vault/ssl

    root@vault:/etc/vault/ssl# openssl genrsa -out vaultca.key 4096

    root@vault:/etc/vault/ssl# openssl req -x509 -new -nodes -key vaultca.key -sha256 -days 9999 -out vaultca.crt

    root@vault:/etc/vault/ssl# openssl req -out vault.csr -newkey rsa:4096 -nodes -keyout vault.key -config /etc/vault/openssl.cnf

    root@vault:/etc/vault/ssl# openssl x509 -req -in vault.csr -CA vaultca.crt -CAkey vaultca.key -CAcreateserial -out vault.crt -days 9998 -extfile /etc/vault/openssl.cnf -extensions req\_ext

Y podemos ver cómo se han añadido al certificado los **SubjectAlternativeNames:**

    root@vault:/etc/vault/ssl# openssl x509 -in vault.crt -text -noout | grep DNS

        DNS:vault.irontec.com, DNS:vault.irontec.vpn, IP Address:66.66.66.66, IP Address:192.168.1.10

### Configuración de Vault

Ahora es cuando vamos a crear la configuración que utilizará Vault cuando arranquemos el servicio. El fichero de configuración es de tipo [HCL](https://github.com/hashicorp/hcl), un formato muy parecido a JSON. Creamos el fichero **/etc/vault/vault-server.hcl** con el siguiente contenido:

    cluster_name = "ironvault"
    disable_mlock = 0
    disable_cache = 0
    cache_size = "32000"
    default_lease_ttl = "3h"
    max_lease_ttl = "3h"

    storage "file" {
    path = "/var/lib/vault/secrets"
    }

    listener "tcp" {
    address = ":8200"
    cluster_address = ":8201"
    tls_disable = 0
    tls_cert_file = "/etc/vault/ssl/vault.crt"
    tls_key_file = "/etc/vault/ssl/vault.key"
    tls_min_version = "tls12"
    tls_prefer_server_cipher_suites = 0
    }

Vamos a destacar:

- **storage**: vamos a hacer uso del tipo de storage «**file**«, para almacenar los secretos en el disco duro del servidor. Existen más [tipos de almacenamientos donde guardar los secretos](https://www.vaultproject.io/docs/configuration/storage/index.html), como por ejemplo, MySQL, Consul, AWS…
- **listener**: todo lo referente al puerto de escucha del servicio y los certificados generados en el paso anterior.

### Servicio de arranque

Como no hemos instalado Vault por paquetes, no tenemos la manera de arrancarlo automáticamente en caso de reinicio del servidor, por lo que vamos a generar el fichero de arranque para Systemd en **/lib/systemd/system/vault-server.service**.

    [Unit]
    Description=HashiCorp's Vault Secrets Store
    After=network.target

    [Service]
    Type=simple
    PIDFile=/var/run/vault-server.pid
    ExecStart=/usr/local/sbin/vault server -config=/etc/vault/vault-server.hcl

    [Install]
    WantedBy=multi-user.target

Y para que sea efectivo tenemos que indicar que el servicio debe **arrancar automáticamente**:

    root@vault:~# systemctl daemon-reload

    root@vault:~# systemctl enable vault-server.service

    root@vault:~# systemctl start vault-server.service

    root@vault:~# systemctl status vault-server.service

El último comando nos debería mostrar cómo el servicio ha arrancado de manera correcta.

## Inicializando Vault

En el post anterior explicamos cómo funciona Vault de manera interna, y cómo es necesario generar la clave de cifrado, junto con la clave maestra y se generan varias claves compartidas. Como hicimos uso del modo «-dev» todos estos pasos nos los ahorramos, ya que se generaban automáticamente, pero ahora **es el momento de hacerlo**.

![vault-logo](/assets/images/vault-shamir-secret-sharing.png){: .w-50}

Tenemos que realizar la exportación de un par de variables. **La primera es para indicar en qué IP** y puerto está escuchando **Vault**. La segunda sirve para ignorar que el certificado es auto-firmado y no verifica si está firmado por una empresa de certificación:

    root@vault:~# export VAULT_ADDR='https://192.168.1.10:8200'

    root@vault:~# export VAULT_SKIP_VERIFY=1

    root@vault:~# vault init

Y nos sacará un mensaje como el siguiente:

    Unseal Key 1: cJ+8LiXEiUf1o8R/IS/CbNi9Wk3LT3hX4PRHPzJdXwIu
    Unseal Key 2: 2FTImdF7+3RC1mXDVLtl7e/0I1rcDxpDyt9SQ7GPgwmH
    Unseal Key 3: ejECIhqXSgo1ODh/cVCwI1QUnBbTfpFlIqPloXVrvEmz
    Unseal Key 4: tczJhmugMOLgiccqn2rK6C3WZSDkLDCUCsS7FCxSFD79
    Unseal Key 5: fcFgmbQKYntnnHQgnVKBQBhWqPowTQBCM0zrebXHp048

    Initial Root Token: f80db773-e8e6-c698-025f-d20d1b050173

    Vault initialized with 5 keys and a key threshold of 3. Please
    securely distribute the above keys. When the vault is re-sealed,
    restarted, or stopped, you must provide at least 3 of these keys
    to unseal it again.

    Vault does not store the master key. Without at least 3 keys,
    your vault will remain permanently sealed.

Acabamos de hacer lo que se muestra en la imagen anterior, generar la clave de cifrado, se ha generado la clave maestra y nos muestra las 5 claves compartidas que **deberemos guardar.** Tal como nos dicen, **serán necesarias 3 para desprecintar Vault**. Es importante guardar estas claves de manera correcta, porque sin ellas, si Vault se sella, no podremos volver a acceder a los datos.

También nos muestra un token de Root para que podamos hacer configuraciones mediante la API o empezar a crear secretos como ya vimos.

### Desprecintando Vault

Si ahora **comprobamos el «status»** de Vault veremos que sigue sellado. Por lo tanto vamos a repetir 3 veces el siguiente comando introduciendo cualquiera de las 5 «Unseal Keys» que tenemos:

    root@vault:~# vault unseal
    Key (will be hidden):

    Y al introducir la tercera si miramos el estado:

    root@vault:~# vault status

    Sealed: false
    Key Shares: 5
    Key Threshold: 3
    Unseal Progress: 0
    Unseal Nonce:
    Version: 0.8.3
    Cluster Name: ironvault
    Cluster ID: ce60ba47-584d-e274-a977-114da06f6a95

    High-Availability Enabled: false

Ya tenemos **Vault desellado** a la espera de introducir secretos, pero para ello nos tendremos que **autenticar con el token de root**:

    root@vault:~# vault auth

    Token (will be hidden):

    Successfully authenticated! You are now logged in.
    token: f80db773-e8e6-c698-025f-d20d1b050173
    token_duration: 0
    token_policies: [root]

### Auditando los accesos

Para tener controlados los accesos podemos activar el [sistema de auditoría](https://www.vaultproject.io/docs/audit/index.html), el cual crea logs de todos los accesos realizados. Vamos a hacer que nos genere logs en **/var/log/vault\_audit.log**

    root@vault:~# vault audit-enable file file_path=/var/log/vault_audit.log

## Autenticación y políticas de acceso

Hasta ahora, todo lo que hemos realizado es con el token de root, pero como todos sabemos: «un gran poder lleva una gran responsabilidad» (TM), por lo que vamos a añadir un sistema de autenticación para usuarios, con sus correspondientes políticas de acceso.

Al igual que con los backends de almacenamiento, **en Vault existen varias alternativas** para el sistema de autenticación: [LDAP](https://www.vaultproject.io/docs/auth/ldap.html), [usuario y password](https://www.vaultproject.io/docs/auth/userpass.html), [Github](https://www.vaultproject.io/docs/auth/github.html)…

Para este post vamos a usar la autenticación de «**usuarios y contraseñas**«, por ser el más sencillo y no depender de ningún servicio externo, ya que en este caso todo se guarda en Vault. En la siguiente imagen se explica cómo funciona la autenticación contra un LDAP, pero el workflow es el mismo para cualquiera de los sistemas permitidos.

![vault workflow](/assets/images/vault-auth-workflow.png)

### Creando backend de autenticación

Vamos a crear dos usuarios para las pruebas de concepto, pero primero tenemos que habilitar el sistema de autenticación:

    root@vault:~# vault auth-enable userpass
    Successfully enabled 'userpass' at 'userpass'!

    root@vault:~# vault write auth/userpass/users/user1 password=h4ck3r1 policies=admins
    Success! Data written to: auth/userpass/users/user1

    root@vault:~# vault write auth/userpass/users/user2 password=h4ck3r2 policies=users
    Success! Data written to: auth/userpass/users/user2

Como se puede ver, aparte de crear los usuarios, a cada uno le hemos dado un parámetro «policies» o política de acceso que todavía no hemos creado. Luego veremos cómo funcionan, pero primero vamos a loguearnos con un usuario:

    root@vault:~# vault auth -method=userpass username=user1 password=h4ck3r1

    Successfully authenticated! You are now logged in.
    The token below is already saved in the session. You do not
    need to "vault auth" again with the token.
    token: 2278ee98-dab6-4b48-fecd-9d5882f8337b
    token_duration: 10799
    token_policies: [admins default]

El mensaje que nos muestra es bastante autoexplicativo:

- Nos hemos logueado de manera correcta.
- Nos ha generado un token y nos lo ha guardado en la sesión, por lo que no tenemos que autenticarnos de nuevo con el token. Ahora mismo, en esta shell, somos **user1**.
- La duración del token es de **10799 segundos**. Esto son 3 horas, que es lo que hemos puesto en el fichero de configuración más arriba: **default\_lease\_ttl = «3h»** . Esto sirve para que los tokens expiren y no sean perpetuos. Entra en vuestro criterio si es mucho tiempo o poco.
- **Políticas del token**: siempre nos añade a la política «default», y aparte a la de «admins» que es la que hemos añadido nosotros al crear el usuario.

Si quisiésemos autenticarnos con curl:

    root@vault:~# curl -k https://192.168.1.10:8200/v1/auth/userpass/login/user1 -d '{ "password": "h4ck3r1" }'

    {
      "request_id": "8d97aa6c-06ac-057d-bac0-131e22b20f6c",
      "lease_id": "",
      "renewable": false,
      "lease_duration": 0,
      "data": null,
      "wrap_info": null,
      "warnings": null,
      "auth": {
        "client_token": "aaf7f710-b0e9-b438-48fb-4ac092f1a19d",
        "accessor": "3037ff52-059f-37f0-91b1-ca761ab99a3f",
        "policies": [
          "admins",
          "default"
        ],
        "metadata": {
          "username": "user1"
        },
        "lease_duration": 10800,
        "renewable": true
      }
    }

Como ya comentamos en el anterior post, **todo en Vault se puede acceder por la API**, y la autenticación no iba a ser menos. Al autenticarnos, hemos generado un nuevo token, con sus permisos correspondientes.  Este no invalida el otro, pero cada uno tiene su tiempo de expiración propio.

### Políticas de acceso (policies)

Hasta ahora no hemos comentado, pero **en Vault todo está basado en «paths»**. Como se puede ver, el sistema de autenticación está dentro del path «auth/», en el anterior post vimos que los secretos los guardamos en «secret/»… **Las «policies» es la manera de dar permisos a los paths dentro de Vault**. Existen los siguientes permisos:

- **create**: Permite crear datos dentro del path indicado.
- **read**: Permite leer datos en el path.
- **update**: Permite actualizar datos. En gran parte del sistema de Vault no existe distinción entre «update» y «create», por lo que si no existen los datos al hacer «update», locrea.
- **delete**: Permite borrar los datos del path especificado.
- **list**: Permite listar los datos dentro de un path. Especialmente útil para listar secretos dentro de un «directorio».
- **sudo**: Tal como pasa en Linux, permite acceso a los paths que están protegidos sólo para root. El token debe de tener permisos de sudo.
- **deny**: Deniega el acceso. Siempre tiene precedencia sobre el resto y es el permiso por defecto para todos los paths.

Con esto, vamos a crear las dos políticas expuestas antes en los usuarios. Para ello, vamos a crear un fichero para cada una de las políticas de acceso. En **/etc/vault/policies/admins.hcl**

    path "secret/*" {
       capabilities = ["create", "read", "update", "delete", "list"]
    }

Y la política para usuarios normales en **/etc/vault/policies/users.hcl** :

    path "secret/*" {
      capabilities = ["read", "update", "list"]
    }

    path "secret/admins/*" {
      capabilities = ["deny"]
    }

Y ahora vamos a cargar las políticas, pero acuérdate que tienes que estar autenticado como root, no como user1 😉

    root@vault:~# vault policy-write admins /etc/vault/policies/admins.hcl
    Policy 'admins' written.

    root@vault:~# vault policy-write users /etc/vault/policies/users.hcl
    Policy 'users' written.

Os dejo como deberes que hagáis las siguientes pruebas teniendo en cuenta este post y el [anterior]({% post_url 2017-10-17-guardando-nuestros-secretos-en-vault-1-de-2 %}) para crear secretos:

1. Crear un secreto en «**secret/secreto1**» y en «**secret/admins/privado1**» con los valores que queráis.
2. Loguearos con el token de **root** y leer ambos secretos.
3. Loguearos con el token de «**user1**» y leer ambos secretos.
4. Autenticaros con el «**user2**» e intentad leer «**secret/admins/privado1**» o listar dentro de «secret/admins». Deberíais tener el siguiente error:

``` bash
root@vault:~# vault read secret/admins/privado1
Error reading secret/admins/privado1: Error making API request.

URL: GET https://192.168.1.10:8200/v1/secret/admins/privado1
Code: 403. Errors:

* permission denied
```

En la [documentación oficial](https://www.vaultproject.io/docs/concepts/policies.html) se explican más opciones acerca de las políticas de acceso, y con ejemplos a través de la API.

Por cierto, tras todos estos pasos, deberíais echar un ojo a **/var/log/vault\_audit.log**, que tal como hemos comentado, nos sirve para auditar los accesos que ha habido.

## Resumen

Con este post hemos querido mostraros un poco más el mundo de Vault, pero la verdad es que nos hemos dejado bastantes cosas en el tintero, como son la creación de distintos **backends** de autenticación tan interesantes como el **de SSH**, montar un sistema en **alta disponibilidad**, ver alguno de los **interfaces gráficos** que existen…

Si tenéis alguna duda o queréis que hagamos un nuevo post ahondado más en el tema dejadnos un comentario 😀 y nos pondremos a ello.