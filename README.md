<h1>Relatório de Projeto: Cloud Computing and Virtualization - Project Base App</h1>

<h3>Introdução</h3> 

Este relatório detalha o desenvolvimento e a implementação de um projeto dividido em duas versões distintas (A e B) de um aplicativo web. O objetivo central é analisar a arquitetura existente do aplicativo, propor uma nova arquitetura que utilize recursos de nuvem/distribuídos, e implementá-la, justificando as escolhas tecnológicas e abordagens adotadas, assim como suas eventuais limitações. 
<h2>Versão A: Implantação com Vagrant e VMs</h2> 
Implementação 

Configuração do Ambiente com Vagrant: 

1. Definição do Vagrantfile para especificar a configuração das VMs.
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define "webapp" do |node|
      node.vm.box = "bento/ubuntu-22.04"
      node.vm.hostname = "webapp"
      node.vm.network :private_network, ip: "192.168.44.22"
      node.vm.provider "virtualbox" do |v|
        v.name = "Project_O-webapp"
        v.memory = 2048
        v.cpus = 2
        v.linked_clone = true
      end
      node.vm.provision "shell", path: "./provision/web.sh"

      # the box generic/alpine might not have shared folders by default
      #node.vm.synced_folder "app/", "/var/www/html"
    end

end
```
Distribuição do Aplicativo: 

Utilização de um balanceador de carga para distribuir as requisições entre as VMs. 
```
Vagrant.configure("2") do |config|
    config.vm.define "proxy" do |proxy|
      proxy.vm.box = "bento/ubuntu-22.04"
      proxy.vm.network "private_network", ip: "192.168.44.10"
      
      # Sincroniza a pasta local ./public com /var/www/html na VM
      proxy.vm.synced_folder "./public", "/var/www/html", create: true, owner: "www-data", group: "www-data"
      
      # Provisione usando um script inline
      proxy.vm.provision "shell", inline: <<-SHELL
        # Atualiza os pacotes e instala o Nginx
        sudo apt-get update
        sudo apt-get install -y nginx
  
        # Configurações adicionais do Nginx, se necessário
        cat <<EOL | sudo tee /etc/nginx/sites-available/default
        server {
            listen 80 default_server;
            listen [::]:80 default_server;
  
            root /var/www/html;
            index index.html index.htm;
  
            server_name _;
  
            location / {
                try_files $uri $uri/ =404;
            }
        }
```

<h2>Versão B: Implantação com Docker e Serviços de Nuvem</h2>
Implementação 

1. Containerização com Docker: 

- Criação de Dockerfiles para cada componente do aplicativo.
```
  # Escolha a imagem base apropriada
FROM ubuntu:22.04

# Defina argumentos para a zona de tempo
ARG DEBIAN_FRONTEND=noninteractive

# Atualize a lista de pacotes e instale pacotes essenciais
RUN apt-get update && apt-get install -y \
    software-properties-common \
    && add-apt-repository ppa:ondrej/php \
    && apt-get update && apt-get install -y \
    apache2 \
    php8.1 \
    php8.1-cli \
    php8.1-common \
    php8.1-mysql \
    php8.1-pgsql \
    php8.1-pdo \
    php8.1-zip \
    php8.1-gd \
    php8.1-mbstring \
    php8.1-curl \
    php8.1-xml \
    php8.1-bcmath \
    zip \
    unzip \
    curl \
    git \
    && apt-get clean

# Instale o Composer
RUN curl -sS https://getcomposer.org/installer -o /tmp/composer-setup.php \
    && php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer

# Copie arquivos de configuração e scripts necessários
COPY ./provision/projectA.conf /etc/apache2/sites-available/

# Habilite o site e módulos do Apache
RUN a2dissite 000-default.conf \
    && a2ensite projectA.conf \
    && a2enmod rewrite

# Copie e instale as dependências do projeto
COPY ./ws /var/www/ws
COPY ./app /var/www/app

# Defina as permissões corretas para os arquivos e diretórios
RUN chown -R www-data:www-data /var/www/app/public_html \
    && chmod -R 755 /var/www/app/public_html

WORKDIR /var/www/ws
RUN composer install

WORKDIR /var/www/app
RUN composer install

# Atualize a data de deploy no arquivo .env
RUN ISO_DATE=$(TZ=Europe/Lisbon date -Iseconds) \
    && sed -i "s/^DEPLOY_DATE=.*/DEPLOY_DATE=\"$ISO_DATE\"/" .env

# Exponha a porta do Apache
EXPOSE 80

# Defina o comando padrão para iniciar o Apache
CMD apachectl -D FOREGROUND
```
- Utilização do Docker Compose para orquestrar os containers.
```
version: '3.8'

services:
  meuphp1:
    image: php1
    container_name: meuphp1
    ports:
      - "8001:80"

  meuphp2:
    image: php2
    container_name: meuphp2
    ports:
      - "8002:80"
  meuphp3:
    image: php3
    container_name: meuphp3
    ports:
      - "8003:80"

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - meuphp1
      - meuphp2
      - meuphp3
```
- Configuração do nginx
```
  worker_processes 1;

events {
    worker_connections 1024;
}

http {
    upstream projectb_backend {
        server meuphp1:80;
        server meuphp2:80;
        server meuphp3:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://projectb_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}```

