# Guardando las reglas de iptables

Las reglas de iptables que escribimos en la línea de comandos se pasan
a memoria donde se ejecutan cuando corresponde para los diferentes
paquetes que llegan al equipo, pero como todo lo que está en memoria,
se borra al reiniciar el equipo.

Hay diferentes formas de conseguir que nuetras reglas de iptables
permanezcan activas tras reiniciar la máquina y vamos a presentar aquí
dos de las más comunes: utilización de un script como enfoque más
tradicional y utilización de iptables-save/iptables-restore como un
método más específico de iptables.

## Utilización de un script

En lugar de ir escribiendo una a una las reglas de iptables en la
línea de comandos, lo que hacemos en este caso es escribirlas en un
fichero y ejecutarlo como un script en el arranque del equipo.

```
#!/usr/bin/env bash

# Limpiamos las tablas
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
# Establecemos la política
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
# Reglas de INPUT y OUTPUT
iptables -A OUTPUT -o br0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i br0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A INPUT -i virbr1 -p icmp -s 192.168.100.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr1 -p icmp -d 192.168.100.0/24 -j ACCEPT
iptables -A INPUT -i virbr2 -p icmp -s 192.168.200.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr2 -p icmp -d 192.168.200.0/24 -j ACCEPT

# Nuevas cadenas
iptables -N LAN_A_INTERNET
iptables -N INTERNET_A_LAN
iptables -N A_DMZ
iptables -N DESDE_DMZ
iptables -N DMZ_A_INTERNET
iptables -N INTERNET_A_DMZ
iptables -N LAN_A_DMZ
iptables -N DMZ_A_LAN

# Enviamos todas las peticiones de la LAN a Internet a la cadena
# LAN_A_INTERNET
iptables -A FORWARD -i virbr1 -o br0 -s 192.168.100.0/24 -m state \
--state NEW,ESTABLISHED -j LAN_A_INTERNET
# Enviamos todas las respuestas de Internet de las peticiones hechas
# desde la LAN a la cadena INTERNET_A_LAN
iptables -A FORWARD -o virbr1 -i br0 -d 192.168.100.0/24 -m state \
--state ESTABLISHED -j INTERNET_A_LAN
# Reglas de LAN_A_INTERNET
iptables -A LAN_A_INTERNET -d 1.1.1.1/32 -p udp --dport 53 -j ACCEPT
iptables -A LAN_A_INTERNET -p tcp --dport 80 -j ACCEPT
iptables -A LAN_A_INTERNET -p tcp --dport 443 -j ACCEPT
# Reglas de INTERNET_A_LAN
iptables -A LAN_A_INTERNET -s 1.1.1.1/32 -p udp --sport 53 -j ACCEPT
iptables -A LAN_A_INTERNET -p tcp --sport 80 -j ACCEPT
iptables -A LAN_A_INTERNET -p tcp --sport 443 -j ACCEPT
# Limite de peticiones simultáneas desde Internet
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 25 \
-m connlimit --connlimit-above 2 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 80 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i br0 -o virbr2 -p tcp --syn --dport 443 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
# Peticiones desde Internet o LAN a servicios de la DMZ:
iptables -A FORWARD -o virbr2 -d 192.168.200.0/24 -m state \
--state NEW,ESTABLISHED -j A_DMZ
# Servicios accesibles en la LAN
iptables -A A_DMZ -p tcp --dport 25 -j ACCEPT
iptables -A A_DMZ -p tcp --dport 80 -j ACCEPT
iptables -A A_DMZ -p tcp --dport 443 -j ACCEPT
# Respuestas desde la DMZ a las peticiones desde Internet o LAN
iptables -A FORWARD -i virbr2 -s 192.168.200.0/24 -m state \
--state ESTABLISHED -j DESDE_DMZ
# Servicios accesibles en la DMZ
iptables -A DESDE_DMZ -p tcp --sport 25 -j ACCEPT
iptables -A DESDE_DMZ -p tcp --sport 80 -j ACCEPT
iptables -A DESDE_DMZ -p tcp --sport 443 -j ACCEPT
# Acceso entre las 12:00 y las 12:30 de la DMZ a Internet
iptables -A FORWARD -i virbr2 -o br0 -s 192.168.200.0/24 \
-m time --timestart 12:00 --timestop 12:30 -m state --state \
NEW,ESTABLISHED -j DMZ_A_INTERNET
# De la DMZ a Internet
iptables -A DMZ_A_INTERNET -p udp --dport 53 -j ACCEPT
iptables -A DMZ_A_INTERNET -p tcp --dport 80 -j ACCEPT
iptables -A DMZ_A_INTERNET -p tcp --dport 443 -j ACCEPT
# Respuestas entre las 12:00 y las 12:30 de Internet a DMZ
iptables -A FORWARD -o virbr2 -i br0 -d 192.168.200.0/24 \
-m time --timestart 12:00 --timestop 12:30 -m state \
--state ESTABLISHED -j INTERNET_A_DMZ
# De Internet a DMZ
iptables -A INTERNET_A_DMZ -p udp --dport 53 -j ACCEPT
iptables -A INTERNET_A_DMZ -p tcp --dport 80 -j ACCEPT
iptables -A INTERNET_A_DMZ -p tcp --dport 443 -j ACCEPT
# Peticiones y respuestas de la LAN a DMZ
iptables -A FORWARD -i virbr1 -o virbr2 -s 192.168.100.0/24 \
-d 192.168.200.0/24 -m state --state NEW,ESTABLISHED -j LAN_A_DMZ
# De la LAN a LA DMZ
iptables -A LAN_A_DMZ -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A LAN_A_DMZ -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A LAN_A_DMZ -p tcp --sport 3306 -j ACCEPT
# Peticiones y respuestas de la DMZ a la LAN
iptables -A FORWARD -o virbr1 -i virbr2 -d 192.168.100.0/24 \
-s 192.168.200.0/24 -m state --state NEW,ESTABLISHED -j DMZ_A_LAN
# De la DMZ a la LAN
iptables -A DMZ_A_LAN -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A DMZ_A_LAN -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A DMZ_A_LAN -p tcp --dport 3306 -j ACCEPT

# Añadimos aquí las reglas de NAT
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o br0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o br0 -m time \
--timestart 12:00 --timestop 12:30 -j MASQUERADE
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 80 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 443 -j DNAT --to 192.168.200.2
iptables -t nat -A PREROUTING -i br0 -p tcp --dport 25 -j DNAT --to 192.168.200.2
```

Aunque no es el caso anterior, es muy habitual que en el caso anterior
se utilicen variables y comentarios que hacen más cómoda la lectura y
modificación del script.

Ubicamos este fichero de script en un directorio adecuado y le damos
permisos de ejecución, por ejemplo:

```
/usr/local/bin/iptables.sh
chmod +x /usr/local/bin/iptables.sh
```

### Ejecutar el script automáticamente al arrancar la máquina

Hoy en día los sistemas GNU/Linux utilizan mayoritariamente systemd en
lugar de init System V, por lo que explicaremos brevemente cómo
definir una unidad de systemd que ejecute el script de iptables al
arrancar el equipo.

Creamos el fichero iptables.service en el directorio
/etc/systemd/system, con el siguiente contenido:

```
[Unit]
Description=Reglas de iptables
After=systemd-sysctl.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/iptables.sh

[Install]
WantedBy=multi-user.target
```

Tendremos que habilitar la unidad anterior para que se ejecute la
próxima vez y si queremos en este momento arrancarla:

```
systemctl enable iptables.service
systemctl start iptables.service
```

## Utilizar iptables-save e iptables-restore

iptables proporciona un mecanismo propio para almacenar las reglas
aplicadas, que es más cómodo que utilizar un script en un fichero,
sobre todo para configuraciones complejas. La forma de trabajar es
diferente, en este caso vamos creando las reglas en el equipo,
añadiendo y modificando lo que sea necesario, hasta que la
configuración sea la deseada. Una vez que la configuración sea la que
queremos, la guardamos con ayuda de iptables-save en un fichero
(típicamente algo como /etc/iptables/rules.v4, aunque no hay ubicación
por defecto):

```
iptables-save > /etc/iptables/rules.v4
```

El formato de iptables-save es como reglas de iptables a las que se
haya quitado el comando iptables (tal como aparecen con iptables -S) y
además se almacenan los contadores de cada cadena.

Ejecutar las reglas de iptables almacenadas con iptables-save se
realiza mediante iptables-restore:

```
iptables-restore /etc/iptables/rules.v4
```

### Ejecutar las reglas al arrancar el equipo

De forma similar al caso anterior vamos a crear una unidad de systemd
para que se encargue de ejecuta iptables-restore e iptables-save de
forma correcta:

```
[Unit]
Description=Reglas de iptables
After=systemd-sysctl.service

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables/rules.v4

[Install]
WantedBy=multi-user.target
```
