# Practica-1.11

# Objetivos

El objetivo de la siguiente práctica consiste en la creación de un NFS Server (Network File System) y la incorporación del mismo a una estructura de dos frontends, un backend y loadbalancer. Gracias a este servidor, podrá crearse una carpeta que, alojada en nuestro servidor, se compartirá entre los dos frontends. Esto nos permite contar con dos frontends idénticos que no se ven comprometidos con una diferencia de datos tras actualizar uno de ellos, de la mism a forma, tampoco requerirá la actualización manual de ambos.

Para ello necesitaremos los siguientes scripts ya creados en anteriores prácticas:
1. install_lamp_backend.sh
2. install_lamp_frontend.sh
3. setup_loadbalancer.sh
4. setup_letsencrypt_certificate.sh
5. deploy_wordpress_backend.sh
6. deploy_wordpress_frontend.sh

Incorporaremos además los scripts nfs-server.sh y nfs-client.sh, los cuales se lanzarán antes de wordpress tanto en el servidor NFS como en ambos clientes.

# NFS Server.

Las primeras líneas del script corresponden al hashbang, importación de variables y la orden para montar toda la ejecución en el termina. Además. incluye en el `apt update` para actualizar el sistema.
```
#!/bin/bash
set -ex
source .env
apt update
```

Tras ello procederemos a la instalación de la herramienta para servidores nfs kernel server y de las carpeta que, definida en las variables, compartiremos con nuestros clientes.

```
apt install nfs-kernel-server -y

mkdir -p $WORDPRESS_DIRECTORY
```

Es necesario cambiar la propiedad de las carpetas a `nobody:nogroup` para evitar que un solo usuario sea propietario de la carpeta, reduciendo de tal forma conflictos con los permisos de los clientes y permitiéndoles a estos escribir en ésta.

```
sudo chown nobody:nogroup $WORDPRESS_DIRECTORY
```

Crearemos nuestro archivo `/etc/exports`, éste contendrá las variables que identificarán las carpetas a compartir y con quién se compartirá. Se recomienda compartir con una red  en concreto en lugar con dispositivos aislados, esto aumentará la escalabilidad del servidor.

```
WORDPRESS_DIRECTORY Network(rw,sync,no_root_squash,no_subtree_check) 
```
Copiado en nuestro servidor este archivo, remplazaremos con `sed` las variables. Aunque podemos llamarlas variables, en realidad, no son variables como tal que puedan ser sustituidas con nuestro .env, ya que los archivos de configuración no tienen variables.
```
cp ../exports /etc/exports
sed -i "s#CLIENT_IP_1#$Network#" /etc/exports
sed -i "s#WORDPRESS_DIRECTORY#$WORDPRESS_DIRECTORY#" /etc/exports
```

Por último será precios reiniciar el servidor para aplicar los cambioss.

```
systemctl restart nfs-kernel-server
```

# NFS Client

El script para la instalación de NFS en los clientes tiene gran parecido al anterior, compartiendo unas primeras líneas iniciales y cambiando únicamente entre éstas la versión de NFS, optando esta vez por la versión para clientes.

```
#!/bin/bash
set -ex
source .env
apt update
apt install nfs-common -y
```

A continuación, deberemos crear la carpeta que vamos a montar, pues ha de existir también en nuestro sistema para poder montarlas. Una vez montada con `df -h` podremos comprobarlo.

```
mkdir $WORDPRESS_DIRECTORY
sudo mount $SERVER_IP:$WORDPRESS_DIRECTORY $WORDPRESS_DIRECTORY
df -h
'''

Para finalizar, nos aseguramos de que el montaje es automático tras cada reinicio del sistema.

```
echo "$SERVER_IP:$WORDPRESS_DIRECTORY $WORDPRESS_DIRECTORY nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" >> /etc/fstab
```
