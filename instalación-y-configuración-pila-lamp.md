---
title: "Instalación y configuración pila LAMP"
tags: ""
---

# Instalació servidor Apache

```bash
sudo apt-get install apache2
```

## Instalació Mysql

### Per a instalar la versio de MariaDB compatible en mysql

```bash
sudo apt-get instal mysql-server
systemctl status mysql
sudo mysql_secure_installation
```

## Instalació PHP

```bash
sudo apt-get install php7.4 libapache2-mod-php7.4 php7.4-mysql
sudo a2enmod php7.4
sudo systemctl restart apache2
sudo systemctl status apache2
php -version
```

Para veriricar creamos una pagina php "info.php"

```bash
sudo nano /var/www/html/info.php
<?php phpinfo(); ?>
```

guardamos cambios y queda accesible desde "localhost/info.php", deberia mostrarnos una web con info sobre versión php;

## Instalación phpmyadmin

La herramiendta phpmyadmin requiere acceso a la base de datos por lo que se deben realizar diversas configuraciones. En ciertas versiones de ubuntu se produce un error de autentificacion derivado del modulo de seguridad, por lo que es mejor desactivarlo antes, por el momento instalaremos algunas dependencias

```bash
sudo apt install php-mbstring php-zip php-gd php-json php-curl
```

Y ahora si realizamos la desactivacion del componente, aqui nos encontraremos el primer problema, y es que mysql suele dar por culo con la autentificacion de root, esta suele quedar configurada como auth_socket, lo que impide el acceso, debe cambiarse a caching_sha2_password de la siguiente forma,  donde 12345678 es la contraseña elegida:

```SQL
ALTER USER 	'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '12345678';
FLUSH PRIVILEGES;
```

Ahora si podemos deshabilitar el componente;

```bash
sudo mysql -u root -p
UNISTALL COMPONENT "file://component_validate_password";
exit
```

Una vez desactivado podemos proceder con la instalación. Hay un par de opciones que tenemos que tener en cuenta, marcar apache2 como servidor y configurar la base de datos para phpmyadmin dejando la contraseña en blanco.

```bash
sudo apt-get install phpmyadmin
```

A continuacion creamos algun usuario;

```SQL
sudo mysql -u root -p
CREATE USER 'jesus'@'localhost' IDENTIFIED WITH caching_sha2_password by 'jesus';
GRANT ALL PRIVILEGES ON *.* TO 'jesus'@'localhost' WITH GRANT OPTION;
exit;
```

Y ya podemos volver a instalar el modulo de seguridad:

```sql
sudo mysql -u root -p
INSTALL COMPONENT "file://component_validate_password";
exit;
```

## Instalación servidor FTP

Dado que va a ser una maquina virtual, o remota dado el caso, necesitamos un servidor FTP para subir las paginas web a dicho servidor, instalaremos el pure-ftpd:

```bash
sudo apt-get install pure-ftpd
```

Usaremos nuestra cuenta del systema para acceder por lo que tendremos que darle privilegios sobre la carpeta donde se alojan las paginas "var/www/html", crearemos un grupo, le añadiremos nuestro usuario y le daremos permisos sobre la carpeta:

```bash
$ sudo addgroup dev
Adding group `dev` (GID 1001) ...
Done
$ sudo adduser jesus dev
Adding user `jesus` to group `dev` ...
Adding user jesus to group dev
Done
$ cd /var/www
$ sudo chown root.dev html
$ sudo chmod g+w html
```

Ya podremos usar cualquier cliente ftp, por ejemplo alguno instalado en vscode como "ftp-simple", para acceder a nuestra carpeta remota a traves de ftp y volcar nuestras webs.  

## Instalacion servidor TOMCAT

El comando sería:

```bash
sudo apt install -y tomcat9 tomcat9-admin
```

Para iniciar, reiniciar y detener el servidor seria desde services:

```bash
sudo service tomcat9 start
sudo service tomcat9 stop
sudo service tomcat9 restart
```

### Configuración básica

Debemos cambiar el puerto de escucha para que no interfiera com el servidor apache, lo meteremos a escuchar al 8080, los archivos de configuración estan en "etc/tomcat9" en el fichero server.xml cambiaremos lo siguiente:

    <Connector port="8080" protocol="HTTP/1.1"
     connectionTimeout="20000"
     redirectPort="8443" />

Cabe añadir que de las 2 veces que lo he hecho no ha sido necesario alterarlo en ninguna.
Instalaremos la docu y los ejemplos:

```bash
sudo apt install -y tomcat9-docs tomcat9-examples
```

Y crearemos roles de usuario para acceder a los diferentes apartados del servidor, esto se hace en tomcat-users.xml, alli simplemente podemos descomentar el ejemplo que hay y cambiar los password. Habra que añadir el rol "admin-gui" y ponerselo a un usuario, por ejemplo "tomcat" para poder acceder al administrado web.

# VirtualHost

Tal como se indica en la documentacio debemos crear un nuevo fichero con el nombre de nuestro host virtual en "/etc/apache2/sites-available", por ejemplo "misitio.com.conf", quedaria asi 

```bash
sudo nano /etc/apache2/sites-available/misitio.com.conf
```

La configuracion basica seria la indicada:

```
<VirtualHost *:80>
 ServerAdmin webmaster@misitio.com
 DocumentRoot "/var/www/html/misitio.com"
 ServerName misitio.com
 ServerAlias www.misitio.com
 ErrorLog "misitio.com-error.log"
 CustomLog "misitio.com-access.log" combined
</VirtualHost>

```

**MUCHO OJO CON LA SINTAXIS, SI NOS EQUIVOCAMOS EN LO MAS MINIMO APACHE NO ARRANCARA**  

Luego tenemos que activar el sitio con `sudo a2ensite misitio.com`, lo que nos crea un enlace en sites-enabled. Es imprescindible reiniciar el servidor apache para que carge estos cambios `sudo service apache2 restart`.  

Para redireccionar a nuestro equipo local, desde el mismo equipo o bien desde el cliente, debemos editar el archio '/etc/hosts' añadiendo la ip del equipo y el nombre del host, por ejemplo desde mi equipo cliente y teniendo en cuenta que la ip del server es 192.168.1.24:
`192.168.1.24	misitio.com`
Si fuera desde el mismo server seria con la direccio de echo 127.0.0.1

Una vez echo esto ya podemos hacer ping al host virtual, si responde pasamos al siguiente punto, crear la carpeta y la web. Para ello vamos a la direccion que le hemos dado en DocumentRoot `/var/www/html/misitio.com`, y creamos un index.html en su interior . si todo esta correcto deberia visualizarse esta web tanto en el navegador local como en el mismo server.

Para terminar personalizaremos las paginas de error, en el archivo .conf del sistio 'misitio.com.conf' añadiremos las etiquetas siguientes:

    ErrorDocument 404 /error_404.html
    ErrorDocument 500 /error_500.html 

Estas paginas las crearemos en la raiz de la carpeta de nuestro host `/var/www/html/misitio.com`, cuando intentemos visualizar un contenido que no exista en el host nos deberia sacar la pagina de error correspondiente.

## Autenticación HTTP

### Metodo 1 htpasswd

Metodo para proteger carpetas dentro de nuestro servidor web, hay que instalar el paquete apache2-utils

```bash
sudo apt-get install apache2-utils
```

El comando que usaremos es htpasswd, con el crearemos un usuario y almacenamos el fichero en la ruta de configuración de apache, añadimos el -c por que el fichero aun no existe:

```bash
sudo htpasswd -c /etc/apache2/.htpasswd jesus
```

Con `bash cat /etc/apache2/.htpasswd`nos deberia mostrar el usuario y su contraseña encryptada.
A continuación asignamos la propiedad del nuevo fichero al usuario y grupo de apache:

```bash
chown www-data:www-data /etc/apache2/.htpasswd
```

Ahora para proteger nuestro host virtual con este nuevo usuario editamos su archivo de configuración, en este caso "etc/apache2/sites-available/misitio.com.conf" para añadir las siguientes lineas:

    <Directory "/var/www/html/misitio.com/usuarios">
     AuthType Basic
     AuthName "Acceso Restringido a Usuarios"
     AuthUserFile /etc/apache2/.htpasswd
     Require valid-user
     </Directory>

Ahora podemos crear ese directorio en a carpeta de misitio.com y provar a acceder desde el navegador con su ruta, deberia salirnos una ventana para login y con nuestro nuevo usuario nos deberia permitir visualizar la carpeta.  

Lo siguiente seria impedir que se puedan listar los ficheros de cualquier directorio por un visitante, simplemente tenemos que añadir dentro de la etiqueta ` <Directory>` la siguiente linea ` Options -Indexes`  

### Metodo2 htaccess

No detallare este metodo aqui, puesto que no lo he usado, mirar en apuntes "UD 02 Gestion de servidores web.pdf"

### Limitacion del ancho de banda y tamaños de archivo

Paginas 16/17 de apuntes

### Resscritura de URLs

Pagina 17/18  

# Seguridad

## Configurar firewall

```bash
sudo ufw enable
sudo ufw status
sudo ufw app list
sudo ufw allow 'Apache'
sudo ufw allow 'Apache Secure'
sudo ufw allow 'Apache Full'

```

## Configurar SSL/TLS en Apache

Para ello tendremos que generar un certificado autofirmado y alojarlo en la carpeta /etc/apache2/certs. Este no sera un certificado valido, puesto que debe ir firmado por una entidad autorizadora, pero aun asi nos vale para pruebas.

```bash
 sudo mkdir /etc/apache2/certs
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048
 -keyout /etc/apache2/certs/apache2.key
 -out /etc/apache2/certs/apache2.crt
```

Ya lo tenemos, ahora activamos el modulo en apache

```bash
sudo a2enmod ssl
```

Y ahora configuramos el VistualHost para soportar a conexion segura.

```bash
 SSLEngine On
 SSLCertificateFile /etc/apache2/certs/apache2.crt
 SSLCertificateKeyFile /etc/apache2/certs/apache2-key
 SSLProtocol All -SSLv3
```

Y finalmente lo habilitamos:

```bash
sudo a2ensite default-ssl.conf
```

## Datos instalacion:

-   Servidor: **zeus**  
-   ip Server: 192.168.1.24:80  
-   Usuario: txus, 123  
-   Pass mysql: 12345678

## Problemas:

-   phpmyadmin, error notfound:  
    `sudo nano /etc/apache2/apache2.conf`  
      Añadimos esta linea al final:  
      `Include /etc/phpmyadmin/apache.conf` 
-   Reconfigurar phpmyadmin:  
    `sudo dpkg-reconfigure phpmyadmin`  
      _Hay que desinstalar el modulo de seguridad para hacer esto, ya 		que se debe reinstalar la base de datos mysql_

```bash
    $ sudo mysql -u root -p
	mysql> UNINSTALL COMPONENT "file://component_validate_password";
	mysql> exit
```

_Luego lo volemos a instalar:_  

```bash
	$ sudo mysql -u root -p
	mysql> INSTALL COMPONENT "file://component_validate_password";
	mysql> exit
```

### Links:

-   [phpmyadmin seguro en ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-on-ubuntu-20-04-es)
-   [ftp visual studio](https://soporte.planisys.net/general/subir-y-bajar-archivos-por-ftp-desde-visual-studio/)  
-   [problema phpmyadmin not found](https://www.daniellucia.es/solucion-phpmyadmin-not-found-ubuntu-derivados.html)
