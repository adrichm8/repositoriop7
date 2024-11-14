# repositoriop7
# Configuración de DNS en Docker (Bind9)
Para configurar un servidor DNS con Bind9 en Docker, creamos una estructura sencilla en docker-compose.yml que incluye dos contenedores: uno para el servidor DNS y otro para el cliente que hará las consultas. El servidor DNS se conecta a una red personalizada con IPs estáticas, y los volúmenes montados permiten gestionar la configuración y las zonas de forma flexible.
```
version: "3.9"
services:
  bind9:
    container_name: dnsserver
    image: internetsystemsconsortium/bind9:9.18
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    networks:
      dnsnet:
        ipv4_address: 192.168.1.1
    volumes:
      - ./config:/etc/bind
      - ./zones:/var/lib/bind
    restart: always

  cliente:
    container_name: cliente
    image: alpine
    tty: true
    stdin_open: true
    dns:
      - 192.168.1.1
    networks:
      dnsnet:
        ipv4_address: 192.168.1.2

networks:
  dnsnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.254
```
# Explicación:
Servidor DNS (bind9):

Imagen: internetsystemsconsortium/bind9:9.18.
Puertos: 53 TCP y UDP.
Volúmenes: Se mapean las carpetas locales config y zones al contenedor.
Red: Está en la subred 192.168.1.0/24 con la IP 192.168.1.1.
Cliente:
Imagen: alpine.
Red: Configurado en la misma subred que el servidor, con IP 192.168.1.2.

# Archivos de configuración de Bind
Dentro de config, los archivos clave para Bind son el archivo named.conf.local:
```
zone "midominio.local" {
    type master;
    file "/etc/bind/db.midominio.local";
};
```
El archivo named.conf.options
```
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    listen-on { any; };
    listen-on-v6 { any; };
};
```
Y el archivo named.conf
```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```
db.midominio.local
```
$TTL 604800
@       IN      SOA     ns.midominio.local. admin.midominio.local. (
                    1   ; Serial
                    604800   ; Refresh
                    86400    ; Retry
                    2419200  ; Expire
                    604800 ) ; Negative Cache TTL
@       IN      NS      ns.midominio.local.
ns      IN      A       192.168.1.1
www     IN      A       192.168.1.3
```
# Comprobación de Configuración
Iniciar los contenedores:
```
docker-compose up -d
```
Acceder al contenedor del cliente:
```
docker exec -it cliente /bin/sh
```
Instalar dig:
```
apk update
apk add bind-tools
```
Consultar el DNS:
```
dig @192.168.1.1 www.midominio.local
```
Si todo está correcto, deberías obtener la respuesta con la IP configurada para www.midominio.local.
