# Instalación de WordPress usando contenedores Docker y Docker Compose

Para realizar esta práctica hay 3 archivos fundamentales. Por un lado, **el script para instalar docker y docker-compose**. Y por otro lado, el archivo **docker-compose.yml** y el archivo **.env** con las variables comunes.

## Script de instalación de Docker

- Lo primero que hay que hacer es definir una variable del nombre de usuario para llamarlo ubuntu:

```
USERNAME=ubuntu
```

- El siguiente paso, como siempre, es actualizar la lista de paquetes del repositorio:

```
apt update
```
- Se descarga el script de instalación de docker:

```
curl -fsSL https://get.docker.com -o get-docker.sh
```

- Tras ello, se ejecuta dicho script:

```
sh ./get-docker.sh
```

- Ahora es el turno de añadir el usuario de la máquina virtual de Amazon al grupo docker:

```
usermod -aG docker $USERNAME
```

- Lo siguiente es iniciar el servicio de Docker:

```
systemctl start docker
```

- El penúltimo paso es instalar el paquete de **docker-compose**:

```
apt install docker-compose -y
```

- Para finalizar esta primera parte, y ya fuera del script de instalación, hay que ejecutar el comando **newgrp docker** para no tener que reiniciar la máquina y que los cambios se efectúen. Pero es importantísimo hacerlo fuera del script.

----------
## Archivo docker-compose.yml

En este archivo es donde se configuran todos los servicios para levantar la aplicación de WordPress, que se componen de distintas imágenes:

- **WordPress**:

Voy a poner un copy-paste y se van explicando todos los apartados de este primer servicio, y luego el resto ya las cosas que le falten a este primero:

```
version: '3.3'

services:
  wordpress:
    image: wordpress:php8.0
    environment:
      - WORDPRESS_DB_HOST=${WORDPRESS_DB_HOST}
      - WORDPRESS_DB_USER=${WORDPRESS_DB_USER}
      - WORDPRESS_DB_PASSWORD=${WORDPRESS_DB_PASSWORD}
      - WORDPRESS_DB_NAME=${WORDPRESS_DB_NAME}
    restart: always
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - mysql
    networks:
      - frontend_network
      - backend_network
```
La **version** '3.3' es la versión del archivo .yml del Docker-compose. El apartado **services** es donde se engloban todas las instancias (wordpress, mysql, phpmyadmin, etc). **image** es la imagen de docker-hub que se escoge para la instalación. **environment** son las variables de configuración de la imagen necesarias para que arranque. Si atendemos al valor de dichas variables, vemos que no hay un valor, sino una referencia a otra variable (*${WORDPRESS_DB_HOST}*). Esto se configura en un archivo **.env**, que en esta práctica es el siguiente:

```
WORDPRESS_DB_HOST=mysql
WORDPRESS_DB_USER=wordpress_user
WORDPRESS_DB_PASSWORD=wordpress_password
WORDPRESS_DB_NAME=wordpress_db
MYSQL_ROOT_PASSWORD=password
```
Por lo tanto, desde las variables del archivo .yml se llama a las variables del .env.
**restart** es para que si la imagen se cae, se levante sola y no se pierdan los datos. **depends-on** se utiliza para que esa imagen no se arranque si no se arranca otra del docker-compose.yml. Finalmente, **networks** sirve para identificar a que niveles quieres que perteneza la imagen en sí.

- **mysql**:

```
  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${WORDPRESS_DB_NAME}
      - MYSQL_USER=${WORDPRESS_DB_USER}
      - MYSQL_PASSWORD=${WORDPRESS_DB_PASSWORD}
    restart: always
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend_network
```

- **phpmyadmin**:

```
  phpmyadmin:
    image: phpmyadmin:5
    restart: always
    ports:
      - 8080:80
    environment:
      - PMA_HOST=mysql
    depends_on:
      - mysql
    networks:
      - frontend_network
      - backend_network
```

En este caso se ve el apartado **ports**. Es para que las peticiones del puerto 80 de la máquina se pase al 8080 del phpmyadmin.

- **https-portal**:

```
  https-portal:
    image: steveltn/https-portal:1
    ports:
      - 80:80
      - 443:443
    environment:
      #DOMAINS: 'localhost -> http://wordpress:80 #local'
      DOMAINS: 'dem-dockerwp.ddns.net -> http://wordpress:80 #production'
    volumes:
      - ssl_certs_data:/var/lib/https-portal
    depends_on:
      - wordpress
    restart: always
    networks:
      - frontend_network

volumes:
  wordpress_data:
  mysql_data:
  ssl_certs_data:

networks:
  frontend_network:
  backend_network:
```
Esta imagen sirve para habilitar un certificado HTTPS en el dominio seleccionado.

