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

Cerramos sesión del usuario y volvemos a iniciarla.

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
docker  images     ls -a
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
  

**Construcción de nueva imagen**

Como ejemplo, vamos a crear una nueva imagen para la aplicación cuyo código es `[ttps://github.com/jamj2000/tienda0.git]https://github.com/jamj2000/tienda0.git)`.
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

Usamos `docker-compose` cuando deseamos desplegar varios servicios de una vez: típicamente aplicación + base de datos.

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


