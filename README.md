# Requisitos previos
- Vagrant 2.4.9
- VirtualBox 7.2.16
# Configurar VM
Nos vamos al directorio ~/Documetns/LAB/NGINX en donde vamos aprovisionar una VM
con vagrant.
# Vagrantfile
```
cat << EOF > Vagrantfile 
Vagrant.configure("2") do |config|
  # Usa una imagen ligera de Ubuntu 22.04
  config.vm.box = "ubuntu/jammy64"
  # Configuración de red (accesible via IP)
  config.vm.network "private_network", ip: "192.168.56.32"
  # Cambiar hostname
  config.vm.hostname = "jorhak"
end
EOF
```
## Iniciamos VM
```
vagrant up
```
## Ingresamos a VM
```
vagrant ssh
```
# Decargar recursos
## Descargar OpenSSL
```
wget https://github.com/openssl/openssl/releases/download/OpenSSL_1_1_1t/openssl-1.1.1t.tar.gz
```
## Descargar Nginx
```
wget https://nginx.org/download/nginx-1.26.1.tar.gz
```
# Empaquetar Nginx con Openssl
# Instalar dependencias
```
sudo apt update
sudo apt install -y gcc make perl libpcre3 libpcre3-dev zlib1g zlib1g-dev
```
# Crear directorio
```
mkdir nginx-custom_1.26.1_amd64
```
# Descomprimir OpenSSL
```
tar -xzf openssl-1.1.1t.tar.gz && cd openssl-1.1.1t
```
## Compilar OpenSSL
```
./config --prefix=/opt/openssl --openssldir=/usr/local/ssl
make
cd ..
```
# Descomprimir Nginx
```
tar -xzf nginx-1.26.1.tar.gz && cd nginx-1.26.1
```
### Modificar codigo fuente Nginx
```
vi src/core/nginx.h
```

```
#define NGINX_VERSION      "1.0.0"
#define NGINX_VER          "AWS UG Santa Cruz/" NGINX_VERSION 
```

```
vi src/http/ngx_http_header_filter_module.c
```

```
static u_char ngx_http_server_string[] = "Server: AWS UG Santa Cruz" CRLF;
```

```
vi src/http/ngx_http_special_response.c
```

```
static u_char ngx_http_error_full_tail[] =
"<hr><center> "AWS UG SANTA CRUZ" </center></body>" CRLF
"</html>" CRLF;

static u_char ngx_http_error_tail[] =
"<hr><center>AWS UG SANTA CRUZ</center></body>" CRLF
"</html>" CRLF;
```

```
vi html/index.html
```

```
<!DOCTYPE html>
<html>
<head>
<title>Bootcamp Creadores de Contenido</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; padding:1.em; line-height: 1.6; }
h1 { text-align: center; margin-bottom: 0.5em; }
p { margin: 0.5em 0; }
img {
    display: block;
    margin: 2em auto 0 auto;
    max-width: 100%;
    height: auto;
    border-radius: 8px; /* opcional: esquinas redondeadas */
}
</style>
</head>
<body>
<h1>Bootcamp para Creadores de contenido</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
<img src="https://prod-assets.cosmic.aws.dev/a/3IIIFr7bqtVcdcjZujONsgGq91T/aws-.webp" alt="Descripción de la imagen" style="max-width:100%;">
</body>
</html>
```

## Compilar Nginx
```
./configure --prefix=/usr/share/nginx \
--sbin-path=/usr/sbin/nginx \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--pid-path=/run/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--with-http_ssl_module \
--with-http_sub_module \
--with-stream \
--with-openssl=/home/vagrant/openssl-1.1.1t
make
make install DESTDIR=/home/vagrant/nginx-custom_1.26.1_amd64
cd ..
```
### Modificar el fichero nginx.conf
```
vi nginx-custom_1.26.1_amd64/etc/nginx/nginx.conf
```

```
user nginx;
pid /run/nginx.pid 

# Descomentar
error_log  /var/log/nginx/error.log;
error_log  /var/log/nginx/error.log  notice;
error_log  /var/log/nginx/error.log  info;

#Descomentar
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
```
## Crear control
```
cd nginx-custom_1.26.1_amd64
mkdir DEBIAN 
cat<< EOF > DEBIAN/control
Package: nginx-custom
Version: 1.26.1-1
Architecture: amd64
Maintainer: Pierluigi Vaca <pierrevaca@gmail.com>
Section: httpd
Priority: optional
Description: Servidor Nginx 1.26.1 personalizado compilado estaticamente con OpenSSL 1.1.1t.
EOF
```
## Crear Script de Servicio
```
mkdir -p lib/systemd/system
cat<< EOF > lib/systemd/system/nginx.service
[Unit]
Description: Un servidor HTTP de alto rendimiento y servidor proxy inverso
After: network-online.target remote-fs.target nss-lookup.target
Wants: network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx.conf
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
PrivateTmp=true

[Service]
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
```
## Crear Script Post Instalacion
```
cat<< EOF > DEBIAN/postinst
#!/bin/sh
set -e

# Crear el usuario y grupo nginx si no existen
if ! getent passwd nginx > /dev/null 2>&1; then
    adduser --system --quiet --group --no-create-home nginx
fi

# Ajustar permisos de los directorios de logs y ejecución
chown -R nginx:nginx /var/log/nginx
chown -R nginx:nginx /usr/share/nginx
chmod 755 -R /usr/share/nginx
chown -R nginx:nginx /etc/nginx
chmod 755 -R /etc/nginx 
# Recargar systemd, habilitar e iniciar el servicio
systemctl daemon-reload
systemctl enable nginx
systemctl start nginx || true

exit 0
EOF
```
### Asignar permisos para el script
```
chmod 755 DEBIAN/postinst
```
# Construir paquete final
```
cd ..
dpkg-deb --build nginx-custom_1.26.1_amd64
```
# Instalar
```
sudo apt install ./nginx-custom_1.26.1_amd64.deb 
```
# Desinstalar
```
sudo apt remove nginx-custom
sudo apt purge nginx-custom
```
# Detener servicio
```
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl daemon-reload
```
# Crear certificados autofirmados
```
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/prueba.key \
  -out /etc/nginx/ssl/prueba.crt \
  -subj "/C=ES/ST=State/L=City/O=Organization/CN=localhost"
```
## Configurar SSL en Nginx
Si estamos utilizando otro bloque lo debemos comentar. De lo contrario nos 
va dar un error. Solo debe estar este bloque habilitado.
```
sudo vi /etc/nginx/nginx.conf
```

```
server {
    listen              443 ssl;
    server_name         localhost;

    # Rutas a los archivos que acabamos de generar
    ssl_certificate     /etc/nginx/ssl/prueba.crt;
    ssl_certificate_key /etc/nginx/ssl/prueba.key;

    # Protocolos soportados por tu OpenSSL 1.1.1t
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        root   html;
        index  index.html index.htm;
    }
}
```
## Redireccionando a servicio 
```
sudo vi /etc/nginx/nginx.conf 
```

```
server {
    listen              443 ssl;
    server_name         localhost;

    # Rutas a los archivos que acabamos de generar
    ssl_certificate     /etc/nginx/ssl/prueba.crt;
    ssl_certificate_key /etc/nginx/ssl/prueba.key;

    # Protocolos soportados por tu OpenSSL 1.1.1t
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://192.168.0.5:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Cabeceras para verificar en la API qué TLS y Cipher se negociaron en el Handshake
        proxy_set_header X-SSL-Protocol $ssl_protocol;
        proxy_set_header X-SSL-Cipher $ssl_cipher;
    }
}
```
## Reiniciamos servicio
```
sudo systemctl reload nginx
```
## Listar metodos criptograficos
```
nmap --script ssl-enum-ciphers -p 443 127.0.0.1
```
# Detener VM
```
vagrant halt
```
