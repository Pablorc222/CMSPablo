# Infraestructura con Vagrant y OwnCloud

Este repositorio describe la infraestructura de servidores configurados usando **Vagrant** y **VirtualBox**. En esta infraestructura, se utiliza **OwnCloud** para gestionar almacenamiento en la nube, con un **Balanceador de Carga**, un **Servidor SGBD (Base de Datos)**, servidores **NFS**, y **Servidores Web**.

## Requisitos

- **Vagrant**: Para la gestión de máquinas virtuales.
- **VirtualBox**: Para la virtualización de las máquinas.
- **OwnCloud**: Para el almacenamiento en la nube.

## Infraestructura y Direccionamiento IP

La infraestructura consta de los siguientes componentes:

1. **Servidor SGBD (Base de Datos)**: Este servidor aloja **MariaDB**.
2. **Servidor NFS**: Se utiliza para compartir almacenamiento entre los servidores web.
3. **Servidores Web (2)**: Ejecutan **OwnCloud** para el acceso a almacenamiento en la nube.
4. **Balanceador de Carga**: Distribuye el tráfico entre los servidores web para alta disponibilidad.

### Direccionamiento IP

- **Servidor SGBD (Base de Datos)**: `192.168.56.10`
- **Servidor NFS**: `192.168.56.11`
- **Servidor Web 1**: `192.168.56.12`
- **Servidor Web 2**: `192.168.56.13`
- **Balanceador de carga**: `192.168.56.100` (con red pública)

## Instalación y Configuración

### 1. Vagrantfile

Este archivo configura las máquinas virtuales utilizando **Vagrant** y **VirtualBox**. Crea las máquinas con las direcciones IP privadas mencionadas anteriormente y define los scripts de provisión para cada máquina.

```ruby
Vagrant.configure("2") do |config|
  # Definir la caja base
  config.vm.box = "debian/bullseye64"

  # Red privada compartida para todas las máquinas
  private_net = "192.168.56."

  # Servidor SGBD (Base de datos)
  config.vm.define "serverSGBDPabloRodriguez" do |db|
    db.vm.hostname = "serverSGBD"
    db.vm.network "private_network", ip: "#{private_net}10"
    db.vm.provision "shell", path: "BDD.sh"
  end

  # Servidor NFS + PHP-FPM
  config.vm.define "serverNFSPabloRodriguez" do |nfs|
    nfs.vm.hostname = "serverNFS"
    nfs.vm.network "private_network", ip: "#{private_net}11"
    nfs.vm.provision "shell", path: "nfs.sh"
  end

  # Servidor Web 1
  config.vm.define "serverWeb1PabloRodriguez" do |web1|
    web1.vm.hostname = "serverWeb1"
    web1.vm.network "private_network", ip: "#{private_net}12"
    web1.vm.provision "shell", path: "webs.sh"
  end

  # Servidor Web 2
  config.vm.define "serverWeb2PabloRodriguez" do |web2|
    web2.vm.hostname = "serverWeb2"
    web2.vm.network "private_network", ip: "#{private_net}13"
    web2.vm.provision "shell", path: "webs.sh"
  end

  # Balanceador de carga (con acceso externo)
  config.vm.define "BalanceadorPabloRodriguez" do |lb|
    lb.vm.hostname = "Balanceador"

    # Red privada para comunicarse con los servidores internos
    lb.vm.network "private_network", ip: "#{private_net}100"

    # Red pública para acceso desde la LAN o Internet
    lb.vm.network "public_network"

    lb.vm.provision "shell", path: "balanceador.sh"
  end
end

```

###2. Script de Balanceador de Carga (`balanceador.sh`)

Este script configura **nginx** como balanceador de carga, distribuyendo el tráfico entre los servidores web.

```bash
#!/bin/bash

# Actualizar repositorios e instalar nginx
sudo apt-get update -y
sudo apt-get install -y nginx

# Verificar si nginx se instaló correctamente
if ! command -v nginx &> /dev/null; then
    echo "Error: Nginx no se instaló correctamente." >&2
    exit 1
fi

# Configuración de Nginx como balanceador de carga
cat <<EOF > /etc/nginx/sites-available/default
upstream backend_servers {
    least_conn; # Distribuye las solicitudes al servidor con menos conexiones activas
    server 192.168.56.12 max_fails=3 fail_timeout=30s; # serverWeb1PabloRodriguez
    server 192.168.56.13 max_fails=3 fail_timeout=30s; # serverWeb2PabloRodriguez
    keepalive 10;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_connect_timeout 10;
        proxy_send_timeout 10;
        proxy_read_timeout 10;
        send_timeout 10;
    }
}
EOF

# Verificar configuración de Nginx antes de reiniciar
nginx -t

# Habilitar y reiniciar Nginx
sudo systemctl enable nginx
sudo systemctl restart nginx





###2.1 Script de Base de Datos (`BDD.sh`)
#!/bin/bash

# Actualizar repositorios e instalar MariaDB
sudo apt-get update -y
sudo apt-get install -y mariadb-server

# Configurar MariaDB para permitir acceso remoto desde los servidores web
sed -i 's/bind-address.*/bind-address = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf

# Reiniciar el servicio de MariaDB
sudo systemctl restart mariadb

# Crear base de datos y usuario en MariaDB con permisos remotos
mysql -u root <<EOF
CREATE DATABASE owncloud;
CREATE USER 'owncloud'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON owncloud.* TO 'owncloud'@'%';
FLUSH PRIVILEGES;
EOF

# Habilitar MariaDB en el arranque
sudo systemctl enable mariadb

# Corregir posibles errores de red eliminando rutas por defecto incorrectas
if ip route | grep -q "default"; then
  sudo ip route del default
fi





###2.2 Script de Servidores Webs(`webs.sh`)
#!/bin/bash

# Actualizar repositorios e instalar nginx, nfs-common y PHP 7.4
sudo apt-get update -y
sudo apt-get install -y nginx nfs-common php7.4 php7.4-fpm php7.4-mysql php7.4-gd php7.4-xml php7.4-mbstring php7.4-curl php7.4-zip php7.4-intl php7.4-ldap mariadb-client

# Asegurarse de que el servicio nfs-common no esté enmascarado
sudo systemctl unmask nfs-common
sudo systemctl enable nfs-common
sudo systemctl start nfs-common

# Crear directorio para montar la carpeta compartida por NFS
sudo mkdir -p /var/www/html

# Montar la carpeta NFS desde el servidor NFS
sudo mount -t nfs 192.168.56.11:/var/www/html /var/www/html

# Verificar que el montaje fue exitoso
if mountpoint -q /var/www/html; then
    echo "Montaje NFS exitoso en /var/www/html"
else
    echo "Error: No se pudo montar NFS en /var/www/html" >&2
    exit 1
fi

# Configuración de Nginx para OwnCloud
cat <<EOF | sudo tee /etc/nginx/sites-available/default
server {
    listen 80;

    root /var/www/html/owncloud;
    index index.php index.html index.htm;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 192.168.56.11:9000;  # IP del servidor PHP-FPM
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README) {
        deny all;
    }
}
EOF

# Verificar configuración de Nginx antes de reiniciar
sudo nginx -t

# Reiniciar nginx
sudo systemctl restart nginx





###2.3 Script de Servidor NFS(`nfs.sh`)
#!/bin/bash

# Actualizar repositorios e instalar NFS y PHP 7.4
sudo apt-get update -y
sudo apt-get install -y nfs-kernel-server php7.4 php7.4-fpm php7.4-mysql php7.4-gd php7.4-xml php7.4-mbstring php7.4-curl php7.4-zip php7.4-intl php7.4-ldap unzip

# Crear carpeta compartida para OwnCloud
sudo mkdir -p /var/www/html
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

# Configurar NFS para compartir la carpeta
echo "/var/www/html 192.168.56.12(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
echo "/var/www/html 192.168.56.13(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports

# Aplicar configuración de exportaciones
sudo exportfs -a
sudo systemctl restart nfs-kernel-server



