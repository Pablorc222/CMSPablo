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

## Instalación y Configuración

### 1. Script de Balanceador de Carga (`balanceador.sh`)

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

