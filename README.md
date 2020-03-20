# DOCKER
**Apuntes sobre contenedores Docker**


## INTRODUCCIÓN

Actualmente existen 2 formas principales de virtualización:
- Máquinas virtuales
- Contenedores

Tienen algunos aspectos en común:
- Ambos métodos utilizan imágenes como "plantilla".
- Usando una imagen pueden generarse muchos sistemas virtualizados, llamados máquinas virtuales y contenedores respectivamente.
- Las imágenes son de sólo lectura, el sistema virtualizado no.

Y una diferencia principal:
- Un contenedor no tiene kernel del sistema operativo, sino que utiliza el del anfitrión. Por ello no realiza toda la comprobación de hardware e inicialización del sistema operativo que realiza una máquina virtual, lo cual permite que el inicio de cualquier servicio en un contenedor sea mucho más rápido que el mismo servicio sobre una máquina virtual.

Actualmente el estándar de facto en contenedores es `docker`, aunque existen otros sistemas como `LXC`, `LXD`, etc.

## USO DE DOCKER


### Pasos previos

**Instalamos Docker y añadimos usuario al grupo docker**

```bash
sudo  apt  install  docker.io  docker-compose
sudo  adduser  `id -un`  docker
```

El último comando añade el usuario actual al grupo docker. 
Esto es necesario para que dicho usuario pueda trabajar con contenedores de docker. 



Para que los cambios tengan efecto,  cerramos sesión del usuario y volvemos a iniciarla. Una solución también válida,aunque más drástica, es reiniciar el ordenador. 

> NOTA: Las imágenes se guardarán en `/var/lib/docker`.

### DockerHub

Es muy recomendable registrarse en el sitio [https://hub.docker.com/](https://hub.docker.com/). Así podremos subir nuestras imágenes a Internet y compartirlas con los demás.

Mis imágenes están en [https://hub.docker.com/r/jamj2000/](https://hub.docker.com/r/jamj2000/).

### Comandos básicos

Si tenemos cuenta en el DockerHub.

```bash
docker  login 
docker  pull  <imagen>    (en este caso no es necesario estar registrado)
docker  push  <imagen>
docker  logout
```


> NOTA: En Ubuntu 18.04 parece haber un bug en alguna parte que impide realizar `docker login`.
>
> Como `workaround` temporal podemos ejecutar los siguientes comandos:
> ```bash
> rm  ~/.docker/config.json
> sudo  mv  /usr/bin/docker-credential-secretservice  /usr/bin/docker-credential-secretservice.orig
> ```
> 
> Mas información: https://stackoverflow.com/questions/50151833/cannot-login-to-docker-account

**Ayuda**

```bash
docker  help
docker  help  image
docker  help  container
```


**Listar imágenes en equipo local**

```bash
docker  images
docker  images -a
docker  image  ls  
docker  image  ls  -a 
```

**Listar contenedores en equipo local**

```bash
docker  ps   
docker  ps -a
docker  container  ls 
docker  container  ls -a
```

**Eliminar imagenes**

```bash
docker  rmi  <imagen>
docker  rmi  <imagen>  -f 
docker  image  rm  <imagen>  
docker  image  rm  <imagen>  -f  
```

> NOTA: La opción `-f` fuerza el borrado. Se utiliza en el caso de que algún contenedor esté ejecutándose con dicha imagen.

**Eliminar contenedores**

```bash
docker  rm  <id_contenedor>
docker  rm  <id_contenedor>  -f 
```

> NOTA: La opción `-f` fuerza el borrado. Detiene el contenedor antes de elimarlo.


**Eliminar todos los contenedores**

```bash
docker rm $(docker ps -a -q) -f
```


**Ejecutar una imagen**

La imagen puede estar disponible en el equipo local o no. Si la imagen es remota, ésta se descarga previamente (`docker pull` implícito)

```bash
docker  run  <imagen>
docker  run  -it  imagen
docker  run  -d   imagen
```

**Ejemplo de ejecución de imagen de Apache**

```bash
docker run -d -p 80:80  httpd 
docker run -d -p 80:80  -v /var/www/html:/usr/local/apache2/htdocs  httpd 
```

**Ejemplo de ejecución de comando interno de un contenedor en ejecución**

```bash
docker  exec  -it  f44ad5467274  bash
```

> NOTA: `f44ad5467274` es el identificador del contenedor, que pueder verse con `docker ps`. 


**Ejemplo de parada, reinicio e inicio de un contenedor**

```bash
docker  stop     f44ad5467274 
docker  restart  f44ad5467274 
docker  start    f44ad5467274 
```

> NOTA: `f44ad5467274` es el identificador del contenedor, que pueder verse con `docker ps`. 
  

**Iniciar aplicación con docker-compose**

```bash
docker-compose  up  -d    
``` 

**Eliminar aplicación con docker-compose**

```bash
docker-compose  down
```

## EJEMPLOS SENCILLOS

```bash
# Tomcat 8
docker  run  -d  -p 8080:8080  --name tomcat8    tomcat:8.0-jre8

# Sonarqube 7.9 LTS 
docker  run  -d  -p 9000:9000  --name sonarqube  sonarqube:lts
```


## EJEMPLOS MÁS COMPLEJOS



```bash
# Eclipse CHE - Puerto 8080
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock -v /tmp:/data  eclipse/che:6.19.0   start
```

```bash
docker  network  create  red

docker  run  -d  -p 8081:8080  -v /var/lib/tomcat8/webapps:/usr/local/tomcat/webapps  -v /var/lib/tomcat8/lib/mysql-connector-java-5.1.21.jar:/usr/local/tomcat/lib/mysql-connector-java-5.1.21.jar   --network red  tomcat:8
docker  run  -d  -p 3307:3306  -e MYSQL_ALLOW_EMPTY_PASSWORD=yes  -e MYSQL_ROOT_HOST=%  -e MYSQL_DATABASE=planticas  --network red  --name mysql  mysql:8
```

Para conectar al servidor MySQL anterior, hacemos:

```bash
mysql  -u root -h 127.0.0.1  -P 3307
```


## CONSTRUCCIÓN DE NUEVA IMAGEN

Como ejemplo, vamos a crear una nueva imagen para la aplicación cuyo código es [`https://github.com/jamj2000/tienda0.git`](https://github.com/jamj2000/tienda0.git).
Hacemos 

```bash
git  clone  https://github.com/jamj2000/tienda0.git
cd tienda0
```

Y revisamos los 2 archivos siguientes:

- .dockerignore

  ```
  .git
  *Dockerfile*
  *docker-compose*
  node_modules
  snapshots
  ```

- Dockerfile

  ```
  FROM  node:10
  WORKDIR  /usr/src/app
  COPY . .

  RUN  npm  install
  EXPOSE  3000
  CMD  ["npm", "start"]
  ```

> NOTAS: 
> - `WORKDIR` es el directorio interior de la imagen donde se guardarán los archivos de la app
> - `COPY . .` copia todos los archivos del directorio local actual al directorio WORKDIR

para construir hacemos:

```bash
docker  build  -t jamj2000/tienda0_app  .   
```

**NOTA**: El punto final es importante. Indica el directorio donde se halla el archivo Dockerfile.

**Subida a DockerHub**

```bash
docker  push  jamj2000/tienda0_app
```

**Usando docker-compose**

Usamos `docker-compose` cuando deseamos desplegar varios servicios de una vez: típicamente **aplicación + base de datos**.

En el siguiente ejemplo, desplegamos el servicio `app` que contiene la aplicación y el servicio `mongo` que contiene la base de datos.

- docker-compose.yml

```yaml
version: '2'

services:
  app:
    container_name: app
    restart: always
    build: .
    ports: 
      - '80:3000'
    links:
      - mongo
  mongo:
    container_name: mongo
    image: mongo
    ports:
      - '27017:27017'
```

Para iniciar la aplicación, ejecutamos

```bash
docker-compose  up  -d
```

## TRABAJAR CON VOLÚMENES DE DATOS

Algunas aplicaciones hacen uso de volúmenes para proporcionar persistencia de datos entre distintas ejecuciones. Típicamente son usados por bases de datos como puede ser MySQL.

Por ejemplo para la aplicación [FP-RESULTADOS](https://github.com/jamj2000/fp-resultados), ejecutamos las siguientes sentencias para lanzar la aplicación con los datos:

```bash
git  clone  https://github.com/jamj2000/fp-resultados.git
cd fp-resultados.git
docker-compose  up  -d
docker  exec  fpresultados_bd_1  /data/database.sh
```

Si accedemos a la aplicación mediante http://localhost:8888/login, e iniciamos sesión con rol de administrador, podremos añadir, modificar o borrar datos de MySQL.

Si después eliminamos la aplicación con:

```bash
docker-compose  down
```

el volumen de datos `fpresultados_datos` seguirá existiendo. Podemos verlo con el comando:

```bash
docker  volume  ls
```

Para hacer una copia de seguridad de dicho volumen ejecutamos el siguiente comando:

**Exportar volumen fpresultados_datos** 


```bash
docker run --rm \
  -v fpresultados_datos:/source:ro \
  busybox tar -czC /source . > fpresultados_datos.tar.gz
```

Y obtendremos una copia de seguridad del volumen en el archivo `fpresultados_datos.tar.gz`.

Si por cualquier motivo perdemos la información del volumen, podremos volver a restaurar el volumen desde la copia de seguridad anterior. Imaginemos que eliminamos el volumen con:

```bash
docker  volume  rm  fpresultados_datos
```

Podemos restaurar los datos contenidos en la copia de seguridad e iniciar de nuevo la aplicación de la manera que se indica a continuación:

Para restaurar la copia de seguridad anterior de dicho volumen ejecutamos el siguiente comando:

**Importar volumne fpresultados_datos**

```bash
docker run --rm -i \
  -v fpresultados_datos:/target \
  busybox tar -xzC /target < fpresultados_datos.tar.gz
```

Para volver a iniciar la aplicación:

```bash
docker-compose  up  -d
```




## CREAR UN REGISTRO PRIVADO


Un registro es un servidor donde se almacenan imágenes de docker. Un registro privado puede montarse en una red local para ahorrar ancho de banda en las descargas desde Internet.

### Pasos previos

**Instalamos Docker y añadimos usuario al grupo docker**

Este paso debe realizarse tanto en el servidor que actuará como registro como en los clientes. 

```bash
sudo  apt  install  docker.io  docker-compose
sudo  adduser  `id -un`  docker
```

Cerramos sesión del usuario y volvemos a iniciarla.


### En el servidor

**Pasos para crear un registro local e incorporar imágenes**


#### Creamos directorio para el registro
```
mkdir  ~/registro
```

#### Descargamos imagen (registry:2) y ponemos en funcionamiento un contenedor de registro 

LLamaremos `registry` al contenedor, aunque su nombre no tiene mucha importancia.


```
docker run -d -p 5000:5000 \
              -v /home/`id -un`/registro/:/var/lib/registry \
              --restart always \ 
              --name registry \
              registry:2
```

**NOTA**: Observa la ironía. El servicio de registro es proporcionado por un contenedor, no por un servidor del sistema.


#### Incorporamos la imagen hello-world a nuestro registro con el nombre hola

```
docker pull hello-world
docker tag hello-world localhost:5000/hola
docker push localhost:5000/hola
```

#### (Opcional) Borramos caché

```
docker rmi hello-world
docker rmi localhost:5000/hola  # don't worry, no se borrará la imagen que posee el registro registry
```

#### (Opcional) Si necesitamos parar el registro
```
docker stop registry
docker rm -v registry
``` 


### En el cliente

**Pasos para descargar imágenes**


En los equipos cliente debemos tener instalado también el paquete `docker.io`, así como configurar el demonio para que permita la conexión a sitios "inseguros" (sin HTTPS).

#### Permitimos registros inseguros
```
nano /etc/docker/daemon.json
```

Añadir la siguiente línea:
```
{ "insecure-registries":["172.20.7.0:5000"] }
```

> NOTA: Colocar en lugar de `172.20.7.0` la dirección IP del servidor de registro

#### Reiniciamos daemon

```
sudo  systemctl  restart  docker
```

#### Usamos la imagen hola del registro privado

```
docker  run  172.20.7.0:5000/hola
```
> NOTA: Colocar en lugar de `172.20.7.0` la dirección IP del servidor de registro


### Referencias

- [Desplegar un Registro Privado](https://docs.docker.com/registry/deploying/)
- [Ejecutar travis-cli en Docker](https://500.keboola.com/run-any-binary-in-a-container-like-it-exists-on-your-computer-8f6205b8cd16)
- [Construir un entorno de integración continua con TravisCI](https://medium.com/google-developers/how-to-run-travisci-locally-on-docker-822fc6b2db2e)
- [Construir un entorno de integración continua con Jenkins y SonarQube](https://yeiei.net/es/como-construir-un-entorno-de-integracion-continua-con-jenkins-y-docker/)


