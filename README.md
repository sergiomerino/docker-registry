# Docker Registry v1 en Ubuntu 14.04

## Prerequisitos Globales
En cada server donde se pretende instanciar un registro de docker v1 es necesario llevar a cabo las siguientes tareas de preparación del entorno.

1. Configurar el servidor con salida a Internet para ser capaz de descargar actualizaciones e instalar paquetes (apt-get) en el fichero `/etc/environment`
    * no_proxy=".isbcloud.isban.corp"
    * http_proxy="http://user:pwd@IP:port"
    * https_proxy=https://user:pwd@IP:port
            
2. Instalar docker
    * `wget -qO- https://get.docker.com/ | sh`
	  
3. Descargar imágenes de Docker Hub. Para ello es preciso tocar en el fichero `/etc/default/docker`
    * export no_proxy=".isbcloud.isban.corp, .dev.corp"
    * export http_proxy="http://user:pwd@IP:port"
    * export https_proxy=https://user:pwd@IP:port

4. Crear los siguientes directorios:
    *  `/opt/external/registry-v1`, donde estará toda la configuración del front del registro de Docker. Como front se usará una imagen existente en Docker Hub.
    *  `/storage-docker/runtime/docker-registry-v1`, directorio a partir del cual se almacenarán las imagenes dockers del registro
    *  `/etc/docker/certs.d/nombremachine.isbcloud.isban.corp:443`, a modo de ejemplo: /etc/docker/certs.d/oclubunc002.isbcloud.isban.corp:443  

	
## Procedimiento
La imagen del registro existente en Docker Hub implementa exclusivamente las funciones core propias del registro dejando a un lado temas como autenticación de usuarios, UI, etc
Por tanto, parece necesario agregar al menos un componente sobre el registro que se encargue de la autenticación de usuarios. Tecnicamente dicho componente sera un nginx que hace las veces de proxy reverso con SSL y un mecanismo de autenticación de usuarios basica

Antes de levantar los contenedores Docker necesarios, es preciso llevar a cabo una serie de tareas de configuración en cada uno de los servidores que haran las veces de servidores de registro privados Docker.

### Generación de Certificados
1. Ejecutar el siguiente comando que generará un certificado para el host en cuestión (en caso de no existir, ejecutar apt-get install openssl-server). Lo mas importante es cumplimentar adecuamente el Common Name (CN) con el FQDN del servidor.

             `openssl req -x509 -newkey rsa:4086 -keyout nombrehost-key.pem -out nombrehost-cert.pem -days 3650 -nodes`

2. Hacer una copia del fichero `nombrehost-key.pem` a `nombrehost-key.crt`
            
            `cp nombrehost-key.pem  nombrehost-key.crt`

3. Copiar el fichero `nombrehost-key.crt` al directorio `/usr/local/share/ca-certificates`

4. Ejecutar el siguiente comando: 
            
        `update-ca-certificates`
  
### Generación del fichero htpasswd 
Para ello es preciso instalar previamente el paquete utils de apache2 (`apt-get install -y apache2-utils`)

La primera vez que se crea el fichero es preciso usar la opción `-c`.
        
        `htpasswd -c docker-registry.htpasswd user1`

Para los nuevos usuarios que se quiera añadir el comando a utilizar sería:
        
        `htpasswd docker-registry.htpasswd userN`
  
### Configuración adicional 
1. El directorio `/opt/external/registry-v1` previamente creado, debe tener los siguientes ficheros:
    * `docker-registry.conf`, existente en el repositorio.
    * `docker-registry.htpasswd`, previamente generado.
    * `nombremachine-cert.crt`, previamente generado.
    * `nombremachine-cert.pem`, previamente generado.
    * `nombremachine-key.pem`, previamente generado.

2. Copiar el certificado al directorio de dockers

    `cp nombrehost-cert.crt /etc/docker/certs.d/nombremachine.isbcloud.isban.corp:443`

3.   Copiar los ficheros al directorio del server que será montado en la imagen del front del registro.

    `cp nombrehost-cert.crt nombrehost-cert.pem nombrehost-key.pem /opt/external/registry-v1`
	
4. 	Revisar el fichero docker-registry.conf para ajustar los nombres de los certificados y claves que debe usar.

    `nano /opt/external/registry-v1/docker-registry.conf`
	
6.	Reiniciar el servicio de docker 

    `sudo service docker restart`
    

### Registro Docker  aplicaciones dockerizadas 

##### Arrancar el contenedor del registro:

    `docker run -d -p 5000:5000 --name registry-apps  -v /storage-docker/runtime/docker-registry-v1:/registry -e "SETTINGS_FLAVOR=dev" -e "STORAGE_PATH=/registry" registry`

##### Arrancar el front sobre el registro:

    `docker run -d -p 443:443 -v /opt/external/registry-v1/:/etc/nginx/external --link registry-apps:registry --name nginx-registryapps-proxy marvambass/nginx-registry-proxy`

##### Arrancar un UI básico que ofrece visibilidad de los contenedores e imagenes existentes en el host:

    `docker run -d -p 9000:9000 --name monitorBaseContainers --privileged -v /var/run/docker.sock:/var/run/docker.sock dockerui/dockerui`



### Registro Docker de imagenes base 

##### Arrancar el contenedor del registro:
    `docker run -d -p 5000:5000 --name registry-base -v /storage-docker/runtime/docker-registry-v1:/registry -e "SETTINGS_FLAVOR=dev" -e "STORAGE_PATH=/registry" registry:latest`

##### Arrancar el front sobre el registro:
    `docker run -d -p 443:443 -v /opt/external/registry-v1/:/etc/nginx/external  --link registry-base:registry --name nginx-registrybase-proxy marvambass/nginx-registry-proxy`

##### Arrancar el webui que permite visualizar el contenido de los dos repositorios.

##### Antes de lanzar el contenedor es preciso crear el keystore que albergue los dos certificados
    `keytool -importcert -file ca.crt -alias ca -keystore truststore -storepass password -noprompt`

##### Arrancando el Web UI que permite tener visibilidad sobre las imágenes existenes en ambos repositorios.
    `docker run --name index-basic -v /opt/external/truststore/truststore:/tmp/truststore \
         -e REG1=https://sergio:sergio@oclubunc016.isbcloud.isban.corp:443/v1/ \
         -e REG2=https://sergio:sergio@oclubunc017.isbcloud.isban.corp:443/v1/  \
         -e 'JAVA_OPTS=-Djavax.net.ssl.trustStore=/tmp/truststore -Djavax.net.ssl.trustStorePassword=password' \
         -d -p 8080:8080 atcol/docker-registry-ui`

##### Arrancar un UI básico que ofrece visibilidad de los contenedores e imagenes existentes en el host:
    `docker run -d -p 9000:9000 --name monitorBaseContainers --privileged -v /var/run/docker.sock:/var/run/docker.sock dockerui/dockerui`
