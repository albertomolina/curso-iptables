# Añadir extensiones al cortafuegos perimetral

Vamos a mejorar el cortafuegos perimetral incluyendo algunas opciones
adicionales mediante el uso de las extensiones de iptables, sin
incluir de momento el seguimiento de la conexión que se hará en el
siguiente apartado.

### Protocolo ICMP

Limitamos al ping las peticiones y respuestas que se aceptan desde el
equipo del cortafuegos al exterior:

```
iptables -A OUTPUT -o wlan0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i wlan0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
```

Entre las dos redes también limitamos el ICMP a las peticiones y
respuestas de ping:

```
iptables -A FORWARD -i virbr1 -o virbr2 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr1 -i virbr2 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A FORWARD -i virbr2 -o virbr1 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr2 -i virbr1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
```

### Límite de conexiones simultáneas desde el exterior

Limitamos a dos conexiones simultáneas al sermidor de correos, pero a
15 al servidor web:
```
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 25 \ 
-m connlimit --connlimit-above 2 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 80 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 443 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
```

### Multiports

Restringimos las peticiones desde fuera a puertos no privilegiados

```
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp -m multiports --sports 1024:65535 --dport 80 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp -m multiports --dports 1024:65535 --sport 80 -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp -m multiports --sports 1024:65535 --dport 443 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp -m multiports --dports 1024:65535 --sport 443 -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp -m multiports --sports 1024:65535 --dport 25 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp -m multiports --dports 1024:65535 --sport 25 -j ACCEPT
```

### Permitir conexión a Internet desde la DMZ a una hora concreta

Los equipos de la DMZ no pueden conectarse a Internet, esto es una
consideración de seguridad básica, para limitar el alcance de un
eventual ataque, pero también es necesario que esos equipos puedan
conectarse a Internet para realizar las actualizaciones de
seguridad. Una solución a este problema es limitar el acceso a
Internet a unos determinados equipos y a hacerlo a una hora
determinada:

```
iptables -A FORWARD -i virbr2 -o wlan0 -p udp --dport 53 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p udp --sport 53 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -p tcp --dport 80 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p tcp --sport 80 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -p tcp --dport 443 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p tcp --sport 443 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
```

## Configuración en un solo paso


```
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
iptables -A OUTPUT -o wlan0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i wlan0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A INPUT -i virbr1 -p icmp -s 192.168.100.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr1 -p icmp -d 192.168.100.0/24 -j ACCEPT
iptables -A INPUT -i virbr2 -p icmp -s 192.168.200.0/24 -j ACCEPT
iptables -A OUTPUT -o virbr2 -p icmp -d 192.168.200.0/24 -j ACCEPT
iptables -A FORWARD -i virbr1 -o virbr2 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr1 -i virbr2 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A FORWARD -i virbr2 -o virbr1 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A FORWARD -o virbr2 -i virbr1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A FORWARD -i virbr1 -o wlan0 -s 192.168.100.0/24 -d 1.1.1.1/32 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -o virbr1 -i wlan0 -d 192.168.100.0/24 -s 1.1.1.1/32 -p udp --sport 53 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -s 192.168.200.0/24 -d 1.1.1.1/32 -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -d 192.168.200.0/24 -s 1.1.1.1/32 -p udp --sport 53 -j ACCEPT
iptables -A FORWARD -i virbr1 -o wlan0 -s 192.168.100.0/24 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -o virbr1 -i wlan0 -d 192.168.100.0/24 -p tcp --sport 80 -j ACCEPT
iptables -A FORWARD -i virbr1 -o wlan0 -s 192.168.100.0/24 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -o virbr1 -i wlan0 -d 192.168.100.0/24 -p tcp --sport 443 -j ACCEPT
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 25 \ 
-m connlimit --connlimit-above 2 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 80 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -i wlan0 -o virbr2 -p tcp --syn --dport 443 \
-m connlimit --connlimit-above 15 -j REJECT --reject-with tcp-reset
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 80 -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 443 -j ACCEPT
iptables -A FORWARD -o virbr2 -d 192.168.200.2/32 -p tcp --dport 25 -j ACCEPT
iptables -A FORWARD -i virbr2 -s 192.168.200.2/32 -p tcp --sport 25 -j ACCEPT
iptables -A FORWARD -i virbr2 -o virbr1 -s 192.168.200.2/32 -d 192.168.100.2/32 -p tcp --dport 3306 -j ACCEPT
iptables -A FORWARD -o virbr2 -i virbr1 -d 192.168.200.2/32 -s 192.168.100.2/32 -p tcp --sport 3306 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -p udp --dport 53 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p udp --sport 53 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -p tcp --dport 80 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p tcp --sport 80 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -i virbr2 -o wlan0 -p tcp --dport 443 -m multiports \
--sports 1024:65535 -s 192.168.200.0/24 -d 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
iptables -A FORWARD -o virbr2 -i wlan0 -p tcp --sport 443 -m multiports \
--dports 1024:65535 -d 192.168.200.0/24 -s 1.1.1.1 -m time \
--time-start 12:00 --time-stop 12:30 -j ACCEPT
```

