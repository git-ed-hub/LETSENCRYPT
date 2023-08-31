

#Certificados AutoFirmados
Se necesita que el equipo cuente con openSSL
Se guardan con los nombres de:
    server.key
    server.crt
Ya que de esa manera los consume el servidor apache

###Para generarlos se siguen estos pasos:

###***Paso 1***
ejecutar el comando este genera el server.key
```
openssl genrsa -out server.key 2048
```

   - **genrsa** (generate an RSA private key)
   - tipo de cifrado -aes128|-aes192|-aes256|-aria128|-aria192|-aria256|-camellia128|-camellia192|-camellia256|-des|-des3|-idea

***Para un certificado Autofirmado Desplegado en ImagenDocker se recomienda no poner pass ya que te pedira ingresar el pass y no dejara desplegar la imagen***

![Error phasshare](/imag/errorPhass.png)
Aqui solo se comenta un cifrado de 2048bits

Se genera un archivo server.csr este archivo es el que generalmente se manda ala Entidad Certificadora para generar el certificado
```
openssl req -new -key server.key -out server.csr
```
Estos son campos que hay que rellenar para que se genere el CSR

    country name:   mx
    State or Province: mexico
    Locality: mexico
    Organization Name: 
    Organizational unit name: 
    Common Name FQDN: lets.test
    email:

Con este comando se genera el certificado autofirmado
    - el **x509** permite firmar el certificado
    - se agrega **-days**  el numero de dias que quedara activo
```
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```
Si corres este te saltas un paso anterior y solo generas el key y crt

###***Paso 2***
```
openssl req -new -x509 -nodes -sha1 -days 365 -key server.key -out server.crt -extensions usr_cert
```
Make sure the server.key file is only readable by root:

    $ chmod 400 server.key

Los certificados deben guardarse en el siguiente path
```
/usr/local/apache2/conf/
```
#Activar el uso de SSL en Apache


Se modifica los siguientes parametros para aceptar los certificados


- En el archivo httpd.conf
  - Se agrega el ServerName lets.test:80 o en su defecto se quita
  - En el area de los modulos se les quita el # esto para permitir el uso de SSL
    - LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
    - LoadModule ssl_module modules/mod_ssl.so
  - En la parte inferior se descomente la linea   
    - Include conf/extra/httpd-ssl.conf
  - Solo como se comenta la linea
    - LoadModule mime_module modules/mod_mime.so
    - este modulo se encarga de interpretar las extensiones de archivos
    - Pedia configuracion por eso se comento



El archivo se encuentra en el siguiente path

```
/usr/local/apache2/conf/httpd.conf
```

Por ultimo se configura el archivo httpd-ssl.conf


En este archivo se configura el uso de los certificados

Se configura el puerto y el nombre que generamos para el certificado

    <VirtualHost *:443>
    ServerName lets.test

El path donde se encuentra el archivo es el siguiente
```
/usr/local/apache2/conf/extra/httpd-ssl.conf
```

Monte un docker-compose para probar copiando las configuracion y los certificados ya generados
```
version: '3.3'
services:
    httpd:
        ports:
            - '8080:80'
            - '8443:443'
        volumes:
            - '/home/ubuntu/apache/certs:/usr/local/apache2/conf/'
            - '/home/ubuntu/apache/httpd-ssl.conf:/usr/local/apache2/conf/extra/httpd-ssl.conf'
            - '/home/ubuntu/apache/httpd.conf:/usr/local/apache2/conf/httpd.conf'
        image: 'httpd:2.4'
```
y tambien cree una imagen pasando los archivos
```
FROM httpd:2.4

COPY ./certs/ /usr/local/apache2/conf/

COPY httpd.conf /usr/local/apache2/conf/httpd.conf

COPY httpd-ssl.conf /usr/local/apache2/conf/extra/httpd-ssl.conf
```
Dando como resultado la muestra del CertificadoAutofirmado

![CertificadoAutofirmado](/imag/Cert.png)



Los archivos para mostrar se guardan en el siguiente path
```
/usr/local/apache2/htdocs
```


![CertificadoAutofirmado](/imag/htdocs.png)

![CertificadoAutofirmado](/imag/CertificadoAutofirmado.png)


Docker image httpd:2.4
OpenSSL 3.0.2 15 Mar 2022


----------------------------------------------------------------

##Letsencrypt

AutoridadCertificadora
 - Hacer renovaciones cada 90 dias
 - Utiliza el protocolo ACME
 - Se requiere un Cliente ACME en nuestro servidor
   - El mas usual es Cerbot
   - https://letsencrypt.org/es/docs/client-options/

![Descripción de la imagen](/imag/cerbot.png)

El objetivo de Let’s Encrypt y el protocolo ACME es hacer posible configurar un servidor HTTPS y permitir que este genere automáticamente un certificado válido para navegadores, sin ninguna intervención humana. Esto se logra ejecutando un agente de administración de certificados en el servidor web.

Para generar un certificate hay dos paso en este proceso.
- Primero, el agente le demuestra a la autoridad certificadora (CA) que el servidor controla el dominio. 
- Posteriormente, el agente puede solicitar, renovar y revocar certificados del dominio.
----------------------------------------------------------------

- Let’s Encrypt identifica al administrador del servidor mediante llave publica
- La primera vez que el software del agente interactúa con Let’s Encrypt, genera un nuevo par de llaves y demuestra al Let’s Encrypt CA que el servidor controla uno o más dominios.
- Para iniciar el proceso, el agent le pregunta al Let’s Encrypt CA lo que hay que hacer para demostrar que controla example.com. El Let’s Encrypt CA mirará el nombre de dominio que se solicita y emitirá uno o más conjuntos de retos. Estas son diferentes maneras que el agente puede demostrar control sobre el dominio.
- Por ejemplo, la AC puede darle al agente la opció de:

Provisionar un record DNS record bajo example.com, ó
Provisionar un recurso HTTP bajo un well-known URI en http://example.com/
Junto con los retos, el Let’s Encrypt CA también provee un nonce que el agente debe firmar con su par de llave privada para demostrar que controla el par de llaves.

![Descripción de la imagen](/imag/howitworks_challenge.png)

El software de agente completa uno de los conjuntos de retos proveidos. Digamos que es capaz de realizar la segunda tarea anterior: crea un archivo en un path especifico en el site http://example.com. El agente también firma el nonce proveído con su llave privada. Una vez el agente ha completado estos pasos, notifica la AC que está listo para completar la validación.

Luego, es el trabajo de la AC verificar los que retos han sido satisfechos. La AC verifica la firma en el nonce, e intenta descargar un archivo del servidor web y hacerse seguro que recibió el contenido esperado.

![Descripción de la imagen](/imag/howitworks_authorization.png)

Si la firma sobre el nonce es válida, y los retos son válidos, entonces el agente identificado por su llave pública está autorizado a realizar la gestión de certificados para example.com. Llamamos el par de llaves que el agente usó un “par de llaves autorizado” para example.com.

#Emisión y Revocación de Certificados
Una vez el agente tenga un par de llaves autorizado, solicitando, renovando, y revocando certificados es simple—solo envia mensajes de manejamiento de certificados y firmalos con el par de llaves autorizado.

Para obtener un certificado para un dominio, el agente construye un PKCS#10 Certificate Signing Request que le pregunta al AC Let’s Encrypt que emita un certificado para example.com con una llave pública espeficificada. Como siempre, el CSR incluye una firma por la llave privada correspondiente a la llave pública en el CSR. El agent también firma el CSR entero con la llave autorizada para example.com de manera que el Let’s Encrypt CA sepa que está autorizado.

Cuando el Let’s Encrypt CA recibe una solicitud, verifica ambas firmas. Si todo se ve bien, emite un certificado para example.com con la llave pública del CSR y lo devuelve al agente.

![Descripción de la imagen](/imag/howitworks_certificate.png)

Revocación funciona de una manera similar. El agent firma una solicitud de revocación con el par de llaves autorizado para example.com, y el Let’s Encrypt CA verifica que la solicitud es autorizada. Si lo es, publica información de revocación a los canales normales de revocación (i.e. OCSP), para que los confiados tales como navegadores pueden saber que no deben aceptar el certificado recovado.

![Descripción de la imagen](/imag/howitworks_revocation.png)


#Requisitos

    Tener un dominio

    Tener instalado un cliente ACME (cerbot)
        Apache
        Nginx
        Haproxy
        Plesk

    System
        Ubuntu
        Fedora
        Web Hosting Service
        Debian
        Centos
        RHEL8

    Intalar cerbot en el servidor Ubuntu

Install Certbot
Run this command on the command line on the machine to install Certbot.
```
sudo snap install --classic certbot
```
Prepare the Certbot command
Execute the following instruction on the command line on the machine to ensure that the certbot command can be run.
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

Generar un certificado para nginx
```
sudo certbot --nginx -d dominio
```
Llenaras una serie de configuraciones datos para que aparezcan en tu certificado

![Cerbot agregar dominio](/imag/001.png)


Elijes el tipo de configuracion si quieres que te redirija a https

![config](/imag/002.png)

Aqui muestra en que directorio queda guardado
![config](/imag/003.png)

En tu archivo de configuracion es donde ***Cerbot*** añade los parametros de validacion

![config](/imag/004.png)

Genera un ***cronjob*** para renovar el certificado

![config](/imag/006.png)


Aqui ya se ve el certificado en funcion y con expiracion de 90 dias

![config](/imag/007.png)

Este comando haria una comprobacion de como estan los certificados no modifica nada

```
sudo certbot renew --dry-run
```
![config](/imag/005.png)

El certificado ya se encuentra activo
