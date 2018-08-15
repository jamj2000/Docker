# DOCKER
**Apuntes sobre contenedores Docker**


## USO DE DOCKER


### Pasos previos

**Instalamos Docker y añadimos usuario al grupo docker**

```bash
sudo  apt  install  docker.io
sudo  adduser  `id -un`  docker
```

Cerramos sesión del usuario y volvemos a iniciarla.

> NOTA: Las imágenes se guardarán en `/var/lib/docker`.

### DockerHub

Es muy recomendable registrarse en el sitio [https://hub.docker.com/](https://hub.docker.com/). Así podremos subir nuestras imágenes a Internet y compartirlas con los demás.

Mis imágenes están en [https://hub.docker.com/r/jamj2000/](https://hub.docker.com/r/jamj2000/).

### Comandos básicos

Si tenemos cuenta en el DockerHub.

```bash
docker  login 
docker  pull     (en este caso no es necesario estar registrado)
docker  push
```

**Ayuda**

```bash
docker  help
docker  help  image
docker  help  container
```


**Ejemplos de comandos más usados**

- Ver imágenes en equipo local

```bash
docker  images
docker  images -a
docker  image  ls  
docker  image  ls  -a 
```

- Ver contenedores en equipo local

```bash
docker  ps   
docker  ps -a
docker  container  ls 
docker  images     ls -a
```

- Eliminar imagenes

```bash
docker  rmi  <imagen>
docker  rmi  <imagen>  -f 
docker  image  rm  <imagen>  
docker  image  rm  <imagen>  -f  
```

> NOTA: La opción `-f` fuerza el borrado. Se utiliza en el caso de que algún contenedor esté ejecutándose con dicha imagen.

- Eliminar contenedores

```bash
docker  rm  <id_contenedor>
docker  rm  <id_contenedor>  -f 
```

> NOTA: La opción `-f` fuerza el borrado. Detiene el contenedor antes de elimarlo.


- Ejecutar una imagen (Puede ser local o remota)

Si la imagen es remota, ésta se descarga previamente (git pull implícito)

```bash
docker  run  <imagen>
docker  run  -it  imagen
docker  run  -d   imagen
```

**Ejecución de Apache**

```bash
docker run -d -p 80:80  httpd 
docker run -d -p 80:80  -v /var/www/html:/usr/local/apache2/htdocs  httpd 
```

**Ejecutar un comando en un contenedor en ejecución**

```bash
docker  exec  -it  f44ad5467274  bash
```

> NOTA: `f44ad5467274` es el identificador del contenedor, que pueder verse con `docker ps`. 


**Parada, inicio y eliminación del contenedor**
```bash
docker  stop   f44ad5467274 
docker  start  f44ad5467274 
docker  rm     f44ad5467274 -f
```

> NOTA: `f44ad5467274` es el identificador del contenedor, que pueder verse con `docker ps`. 
  

**Construcción de nueva imagen**

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
>  `WORKDIR` es el directorio interior de la imagen donde se guardarán los archivos de la app
>  `COPY . .` copia todos los archivos del directorio local actual al directorio WORKDIR

para construir hacemos:

```bash
docker build  -t jamj2000/tienda0_app  .
```

**Subida a DockerHub**

```bash
docker push  jamj2000/tienda0_app
```

**Usando docker-compose**

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

Ejecutamos

```bash
docker-compose  up  -d
```


## CREAR UN REGISTRO PRIVADO


Un registro es un servidor donde se almacenan imágenes de docker. Un registro privado puede montarse en una red local para ahorrar ancho de banda en las descargas desde Internet.

### Pasos previos

**Instalamos Docker y añadimos usuario al grupo docker**

Este paso debe realizarse tanto en el servidor que actuará como registro como en los clientes. 

```bash
sudo  apt  install  docker.io
sudo  adduser  `id -un`  docker
```

Cerramos sesión del usuario y volvemos a iniciarla.


### En el servidor

__Pasos para crear un registro local e incorporar imágenes__


#### Creamos directorio para el registro
```
mkdir  ~/registro
```

#### Descargamos contenedor de registro (registry:2)

```
docker run -d -p 5000:5000 \
              -v /home/`id -un`/registro/:/var/lib/registry \
              --restart always \ 
              --name registry \
              registry:2
```

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
__Pasos para descargar imágenes__


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


