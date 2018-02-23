# Cluster MySQL + HAProxy Failover

Guía para montar un clúster de MySQL y HAProxy en Failover.

El clúster de MySQL estará formado por varios tipos de nodos:

 - Nodo de administración (**MGM**): Es el nodo que se va a encargar de manejar, controlar y coordinar al resto de nodos del clúster. Debido a esto tiene que ser el primero en iniciarse.

 - Nodo de datos (**NDB**): Este tipo de nodo almacena los datos del clúster de forma distribuida. De tal forma que, si uno de ellos cae, el resto de nodos pueden seguir sirviendo la información.

 - Nodo SQL (**MYSQLD**): Este tipo de nodo nos permite acceder a los datos del clúster.


Los servidores HAProxy se encargarán de balancear las peticiones/consultas que realicen los clientes entre los nodos MYSQLD. A su vez, el HAProxy junto con Keepalived formarán un failover entre sí para que en caso de que se produzca la pérdida de un servidor HAProxy, entre automáticamente el otro en funcionamiento.

En este caso montaremos los nodos MGM y MYSQLD juntos, de tal forma que, nos evitamos montarlo en 2 servidores por separado. Habrá un MGM Maestro y un Esclavo de modo que no estarán funcionando a la vez.

También dispondremos de 2 nodos NDB que serán replicas sincronizadas y los nodos MGM serán los encargados de administrar y gestionar estos.


Todos los servidores van a estar corriendo bajo **Ubuntu Server 16.04**.


# Esquema de RED

En el siguiente esquema podemos ver los diferentes servidores que van a formar nuestro sistema con sus correspondientes hostname y direcciones IP.

De cara al cliente que realice las peticiones solo dispondrá de una dirección IP virtual que en este caso será 192.168.1.230.

![Esquema de red para un cluster de MySQL y HAProxy](https://github.com/RafaMunoz/Cluster-MySQL-HAProxy/blob/master/img/esquema_red_cluster_mysql.png)

# Configuración basica

En este apartado veremos las configuraciones básicas que se tiene que realizar en cada servidor teniendo en cuenta sus direcciones IP y su hostname.

Actualizamos el listado de los repositorios y a continuación actualizaremos los paquetes.
 
	 sudo apt update
	 sudo apt upgrade
	    
Le pondremos a cada servidor una dirección IP fija.

     sudo nano /etc/network/interfaces
	    
Un ejemplo de la configuración  de unos de los servidores puede ser la siguiente:

    auto enp0s3
    iface enp0s3 inet static
    address 192.168.1.210
    gateway 192.168.1.1
    netmask 255.255.255.0
    network 192.168.1.0
    broadcast 192.168.1.255
    dns-nameservers 8.8.8.8 8.8.4.4

Si durante la instalación del sistema operativo no hemos puesto el nombre de hostname correctamente o hemos partido de un clon de una máquina virtual, lo cambiaremos por el que corresponda.

    sudo nano /etc/hostname

Por ejemplo, en uno de ellos podemos poner:

    MGM01
 
Seguidamente también lo cambiaremos en el archivo host:

    sudo nano /etc/hosts


Este es un ejemplo del archivo hosts del servidor MGM01:

    127.0.1.1       MGM01

Reiniciaremos los servidores para que cojan los cambios.

    sudo reboot
 

# Instalación clúster MySQL
Los siguientes tres comandos deberán ejecutarse en los 4 servidores que formarán el clúster MySQL.

 Necesitaremos instalar el paquete **libaio1**.

    sudo apt install libaio1

Lo siguiente será descargar de la web de [MySQL](https://dev.mysql.com/downloads/cluster/) el paquete de MySQL-Cluster que en este caso es la versión 7.5.9 para sistemas 64 bits.

    wget https://dev.mysql.com/get/Downloads/MySQL-Cluster-7.5/mysql-cluster-gpl-7.5.9-linux-glibc2.12-x86_64.tar.gz

Descomprimimos el paquete descargado.

    sudo tar xvfz mysql-cluster-gpl-7.5.9-linux-glibc2.12-x86_64.tar.gz

## Instalación MGM (Nodo de administración)

Lo primero que tenemos que hacer es instalar los nodos de MGM que se va a encargar de manejar, controlar y coordinar al resto de nodos del clúster.

Entramos dentro de la carpeta que hemos descomprimido.

    cd ./mysql-cluster-gpl-7.5.9-linux-glibc2.12-x86_64/

Copiamos todos los archivos ndb_mgm a /usr/local/bin.

    sudo cp bin/ndb_mgm* /usr/local/bin

Le damos permisos de ejecución a la carpeta que hemos copiado.

    sudo chmod +x /usr/local/bin/ndb_mgm*

Creamos la carpeta mysql-cluster en /var/lib.

    sudo mkdir /var/lib/mysql-cluster

Dentro de esta carpeta crearemos el archivo de configuración.

    sudo nano /var/lib/mysql-cluster/config.ini

Ejemplo config.ini (serán iguales en los dos nodos MGM):

    [ndbd default]
    # Opciones para los servidores ndb:
    NoOfReplicas=2
    DataMemory=80M
    IndexMemory=18M
    
	[ndb_mgmd]
	# Opciones de nodo de administración MGM01:
	HostName=192.168.1.210
	NodeId=1
	DataDir=/var/lib/mysql-cluster 
	
	[ndb_mgmd]
	# Opciones de nodo de administración MGM02:
	HostName=192.168.1.211
	NodeId=2
	DataDir=/var/lib/mysql-cluster 

	[mysqld]
	# Opciones para el nodo SQL 01:
	NodeId=3
	HostName=192.168.1.210

	[mysqld]
	# Opciones para el nodo SQL 02:
	NodeId=4
	HostName=192.168.1.211
	
	[ndbd]
	# Opciones para el nodo de datos NDB01:
	NodeId=5
	HostName=192.168.1.215
	DataDir=/usr/local/mysql/data
	
	[ndbd]
	# Opciones para el nodo de datos NDB02:
	NodeId=6
	HostName=192.168.1.216
	DataDir=/usr/local/mysql/data
	
	 
 Iniciamos el nodo de administración (MGM _01):

    sudo ndb_mgmd --ndb-nodeid=1 -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster/

  Iniciamos el nodo de administración (MGM _02):

    sudo ndb_mgmd --ndb-nodeid=2 -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster/

Agregamos el comando anterior al archivo /etc/rc.local para que se inicie cuando arranque el servidor.

> Hay que poner el comando antes **exit 0**

    sudo nano /etc/rc.local

 - Servidor MGM01:
  `ndb_mgmd --ndb-nodeid=1 -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster/`
  
  - Servidor MGM02:
  `ndb_mgmd --ndb-nodeid=2 -f /var/lib/mysql-cluster/config.ini --configdir=/var/lib/mysql-cluster/`


## Instalación de los nodos de datos (NDB)
Ahora instalaremos y configuraremos los dos nodos que almacenarán las bases de datos.

> Nota: Este proceso será igual para los dos nodos.

Entramos dentro de la carpeta que hemos descomprimido anteriormente.

    cd ./mysql-cluster-gpl-7.5.9-linux-glibc2.12-x86_64/

Copiamos la carpeta **bin/ndbd** a **/usr/local/bin/ndbmtd** y la carpeta **bin/ndbd** a **/usr/local/bin/ndbd**.

    sudo cp bin/ndbd /usr/local/bin/ndbd
    sudo cp bin/ndbmtd /usr/local/bin/ndbmtd

 
 Nos movemos al directorio donde hemos copiado las anteriores carpetas y damos permisos de ejecución a todos los archivos de ndb.

    cd /usr/local/bin
    sudo chmod +x ndb*

Dentro crearemos el archivo de configuración **my.cnf**.

    sudo nano /etc/my.cnf

Dentro del archivo my.cnf ponemos lo siguiente:

    [mysqld]
     # Opciones para el servicio mysqld:
     ndbcluster # Ejecutar el motor de almacenamiento del NDB
    
    [mysql_cluster]
     # Opciones para los procesos del NDB:
     ndb-connectstring=192.168.1.210,192.168.1.211  # Nodos MGM

Creamos las carpetas mysql y data.

    sudo mkdir /usr/local/mysql
    sudo mkdir /usr/local/mysql/data

Iniciamos el NDB para que se conecte con el MGM.

    sudo ndbd

Agregamos el comando anterior a /etc/rc.local para que se inicie cuando arranca el SO.

> Hay que poner el comando antes **exit 0**

    sudo nano /etc/rc.local

Quedaría de la siguiente forma:

    ndbd
    exit 0
 

## Instalación de los nodos SQL (MYSQLD)

Ahora instalaremos los nodos MYSQLD, que serán los encargados de recibir las consultas que se vayan a realizar a la base de datos. Recordad que estarán en los mismos servidores que los nodos MGM.

Creamos el grupo mysql y el usuario mysql.

	sudo groupadd mysql
    sudo useradd -g mysql mysql

Movemos la carpeta que hemos descargado al principio a /usr/local mysql.

    sudo mv mysql-cluster-gpl-7.5.9-linux-glibc2.12-x86_64/ /usr/local/mysql

Entramos al directorio que hemos movido y ejecutamos el script de instalación.
	
	cd /usr/local/mysql/bin
    sudo ./mysqld --initialize

Al final del proceso se genera una clave aleatoria que deberemos apuntar. En mi caso ha sido **rLQuaIEwm7*E**.

    2018-02-21T20:29:50.457419Z 1 [Note] A temporary password is generated for root@localhost: rLQuaIEwm7*E

Cambiamos el propietario de todos los archivos del directorio.

	cd ..
    sudo chown -R root:mysql .
    sudo chown -R mysql data

Copiamos el archivo del servicio para que se inicie automaticamente y le damos permisos de ejecución.

    sudo cp support-files/mysql.server /etc/init.d/
    sudo chmod +x /etc/init.d/mysql.server

Creamos el archivo de configuración para MYSQLD.

     sudo nano /etc/my.cnf

El contenido será el mismo que en los nodos NDB.

    [mysqld]
     # Opciones para el servicio mysqld:
     ndbcluster # Ejecutar el motor de almacenamiento del NDB
    
    [mysql_cluster]
     # Opciones para los procesos del NDB:
     ndb-connectstring=192.168.1.210,192.168.1.211  # Nodos MGM

Habilitamos el servicio de mysql, recargamos el systemctl e iniciamos el servicio.

    sudo systemctl enable mysql.server
    sudo systemctl daemon-reload
    sudo service mysql.server start

Para dar mayor seguridad a nuestro sistema ejecutamos el siguiente comando y lo primero que nos pedirá es la contraseña aleatoria que se nos ha generado antes.

    sudo bin/mysql_secure_installation

Podéis seleccionar las opciones a vuestro gusto, en mi caso han sido estas:

    Securing the MySQL server deployment.

	Enter password for user root: 
	The existing password for the user account root has expired. Please set a new password.

	New password: 

	Re-enter new password: 

	VALIDATE PASSWORD PLUGIN can be used to test passwords
	and improve security. It checks the strength of password
	and allows the users to set only those passwords which are
	secure enough. Would you like to setup VALIDATE PASSWORD plugin?

	Press y|Y for Yes, any other key for No: y

	There are three levels of password validation policy:

	LOW    Length >= 8
	MEDIUM Length >= 8, numeric, mixed case, and special characters
	STRONG Length >= 8, numeric, mixed case, special characters and dictionary                  file

	Please enter 0 = LOW, 1 = MEDIUM and 2 = STRONG: 0
	Using existing password for root.

	Estimated strength of the password: 100 
	Change the password for root ? ((Press y|Y for Yes, any other key for No) : n

	 ... skipping.
	By default, a MySQL installation has an anonymous user,
	allowing anyone to log into MySQL without having to have
	a user account created for them. This is intended only for
	testing, and to make the installation go a bit smoother.
	You should remove them before moving into a production
	environment.
	
	Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
	Success.

	Normally, root should only be allowed to connect from
	'localhost'. This ensures that someone cannot guess at
	the root password from the network.

	Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
	Success.

	By default, MySQL comes with a database named 'test' that
	anyone can access. This is also intended only for testing,
	and should be removed before moving into a production
	environment.

	Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
	 - Dropping test database...
	Success.

	 - Removing privileges on test database...
	Success.

	Reloading the privilege tables will ensure that all changes
	made so far will take effect immediately.

	Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
	Success.

	All done!

## Comprobaciones del Clúster
Ahora en los nodos MGM ejecutamos el siguiente comando para ver el estado del clúster.

    sudo ndb_mgm -e show

Si lo ejecutamos en los nodos MGM podemos observar como en este caso está haciendo la gestión el Nodo MGM01 (192.168.1.210).

	Connected to Management Server at: 192.168.1.210:1186
	Cluster Configuration
	---------------------
	[ndbd(NDB)]	2 node(s)
	id=5	@192.168.1.215  (mysql-5.7.21 ndb-7.5.9, Nodegroup: 0, *)
	id=6	@192.168.1.216  (mysql-5.7.21 ndb-7.5.9, Nodegroup: 0)

	[ndb_mgmd(MGM)]	2 node(s)
	id=1	@192.168.1.210  (mysql-5.7.21 ndb-7.5.9)
	id=2	@192.168.1.211  (mysql-5.7.21 ndb-7.5.9)

	[mysqld(API)]	2 node(s)
	id=3	@192.168.1.210  (mysql-5.7.21 ndb-7.5.9)
	id=4	@192.168.1.211  (mysql-5.7.21 ndb-7.5.9)

Y en el caso de que apaguemos el nodo MGM01 veremos cómo entra en funcionamiento el segundo nodo MGM02 (192.168.1.211) para realizar la gestión.

    Connected to Management Server at: 192.168.1.211:1186
	Cluster Configuration
	---------------------
	[ndbd(NDB)]	2 node(s)
	id=5	@192.168.1.215  (mysql-5.7.21 ndb-7.5.9, Nodegroup: 0, *)
	id=6	@192.168.1.216  (mysql-5.7.21 ndb-7.5.9, Nodegroup: 0)

	[ndb_mgmd(MGM)]	2 node(s)
	id=1 (not connected, accepting connect from 192.168.1.210)
	id=2	@192.168.1.211  (mysql-5.7.21 ndb-7.5.9)

	[mysqld(API)]	2 node(s)
	id=3 (not connected, accepting connect from 192.168.1.210)
	id=4	@192.168.1.211  (mysql-5.7.21 ndb-7.5.9)

Podéis jugar apagando y encendido los diferentes nodos para ver el comportamiento del sistema.

De momento hasta que configuremos los HAProxy podemos acceder al sistema y crear bases de datos entrando en los nodos SQL (MYSQL) y ejecutando el siguiente comando:

    /usr/local/mysql/bin/mysql -u root -p

# Instalación de HAProxy

Una vez tenemos preparado el clúster de MySQL vamos a proceder a crear el balanceo y failover de HAProxy.

## Instalación y configuración HAProxy
Instalamos haproxy.

    sudo apt-get install haproxy

Editamos la configuración del archivo **haproxy.cfg** que será igual para los dos servidores excepto en el apartado de estadísticas que configuraremos la dirección IP del servidor HAProxy que estamos configurando.

    sudo nano /etc/haproxy/haproxy.cfg

Esta configuración está formada por varios apartados:

    listen mysql_cluster 
		bind 0.0.0.0:3306
		mode tcp
		balance leastconn
		option mysql-check user check_haproxy 
		server MGM01 192.168.1.210:3306 check fall 3 rise 2
		server MGM02 192.168.1.211:3306 check fall 3 rise 2 
        
    listen stats
		bind 192.168.1.220:8181
		stats enable
		stats uri /
		stats realm Haproxy\ Statistics
		stats auth admin:admin
		stats refresh 5s

 - En **"mysql_cluster"** indicaremos que todo lo que se reciba por el puerto **3306** será balanceado a cada nodo MGM. 
La selección en este caso la va a marcar el comando **leastconn** que selecciona el servidor con la menor cantidad de conexiones y este se recomienda para sesiones largas como las de SQL. Los servidores también se rotan de manera round-robin.

	Estos nodos van a ser dados por inactivos si se producen tres comprobaciones erróneas seguidas y se volverá a tomar como activo al recibir dos comprobaciones correctas. Por defecto se realizan cada 2000ms pero es configurable.

	Para checkear los nodos que reciben las consultas de mysql usaremos el setting **option mysql-check [user "username"]**, donde username corresponde con el nombre de usuario que usamos únicamente para comprobar la conexión (no vale un usuario con contraseña).
	Para crear este usuario nos iremos a los 2 nodos en los que hemos instalado MYSQLD (MGM01 y MGM02) y ejecutaremos lo siguiente.

	    /usr/local/mysql/bin/mysql -u root -p
	    USE mysql;
	    INSERT INTO user (Host,User,ssl_cipher,x509_issuer,x509_subject) VALUES ('192.168.1.220', 'check_haproxy','','','');
	    INSERT INTO user (Host,User,ssl_cipher,x509_issuer,x509_subject) VALUES ('192.168.1.221', 'check_haproxy','','','');
	    FLUSH PRIVILEGES;

 3. Por último, en **listen stats** le diremos que todo lo que se pida a la IP del HAProxy que estamos configurando y al puerto 8181 será para mostrar las estadísticas de conexiones y balanceo de carga que está realizando. 
Esta información la podremos ver haciendo una petición a través del navegador web y con el usuario **admin** y contraseña **admin**  configurados en el setting **stats auth**.


## Instalación y configuración de Keepalived
Es hora de instalar Keepalived que será el software instalado en los servidores de HAProxy para crear el failover entre ellos y proporcionar una IP virtual para que los clientes realicen las peticiones.

Instalamos Keepalived.

    sudo apt-get install keepalived

Configuraremos el archivo **keepalived.conf** que tendrá una configuración diferente para cada servidor.

    sudo nano /etc/keepalived/keepalived.conf

Los settings más importantes de cada configuración son:

 - **interface**: donde indicamos la tarjeta de red de nuestro servidor.
 - **state**: le decimos cual va a ser el MASTER y cual el BACKUP.
 - **priority**: daremos más prioridad al servidor maestro de tal forma que en caso de que los dos servidores HAProxy estén iniciados, este tomará toda la carga de conexiones.
 - **virtual_router_id**: identificador numérico que tiene que ser igual en los dos servidores.
 - **auth_pass**: especifica la contraseña utilizada para autenticar los servidores en la sincronización de failover.
 - **virtual_ipaddress**: sera la dirección IP virtual que compartirán lo dos servidores y a la que tienen que realizar las peticiones los clientes.

Configuración servidor **MASTER**:

    vrrp_script chk_haproxy {
	    script "pidof haproxy"
	    interval 2
    }
    
	vrrp_instance VI_1 {
	    interface enp0s3
	    state MASTER
	    priority 200

	    virtual_router_id 33

	    authentication {
	        auth_type PASS
	        auth_pass 129348
	    }

	    virtual_ipaddress {
	        192.168.1.230/24
	    }

	    track_script {
	        chk_haproxy
	    }
	}

Configuración servidor **BACKUP**:

    vrrp_script chk_haproxy {
	    script "pidof haproxy"
	    interval 2
    }
    vrrp_instance VI_1 {
	    interface enp0s3
	    state BACKUP
	    priority 100

	    virtual_router_id 33

	    authentication {
	        auth_type PASS
	        auth_pass 129348
	    }

	    virtual_ipaddress {
	        192.168.1.230/24
	    }
    
	    track_script {
	        chk_haproxy
	    }
	}


Una vez realizados todos estos pasos creamos un usuario que usaremos para acceder a la base de datos y realizar todas las consultas que necesitemos.

En los dos nodos MYSQLD (MGM01 y MGM02) ejecutamos los siguientes comandos para crear el **usuario** test con **contraseña** test:

    /usr/local/mysql/bin/mysql -u root -p
	CREATE USER 'test' IDENTIFIED BY 'test';
	GRANT ALL PRIVILEGES ON * . * TO 'test'@'%';
	FLUSH PRIVILEGES;

Por último, con un cliente de MySQL, por ejemplo, [MySQL Workbench](https://dev.mysql.com/downloads/workbench/) podeis probar la conexión apuntando a la IP Virtual que hay creada entre los dos nodos HAProxy.

![Test de conexión cluster-mysql](https://github.com/RafaMunoz/Cluster-MySQL-HAProxy/blob/master/img/test_conexion_ipvirtual.png)

