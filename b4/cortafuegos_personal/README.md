# Cortafuegos personal con iptables

Vamos a realizar los primeros pasos para implementar un cortafuegos en
un nodo de una red, aquel que se ejecuta en el propio equipo que trata
de proteger, lo que a veces se denomina un cortafuegos
personal.

## Parámetros del equipo

```
Interfaz de red: eth0
Dirección IP: 192.168.100.2/24
Puerta de enlace: 192.168.100.1
```

## Configuración paso a paso del cortafuegos

Implementaremos un cortafuegos con política por defecto DROP e iremos
de forma paulatina agregando reglas que permitan un uso elemental del
equipo

### Limpieza de las reglas previas

```
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
```

### Política por defecto

```
iptables -P INPUT DROP
iptables -P OUTPUT DROP
```

Comprobamos que el equipo no puede acceder a ningún servicio ni de
Internet ni de la red local, ya que la política lo impide.

### Peticiones y respuestas al ping

```
iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
```

Comprobamos su funcionamiento haciendo ping a una IP pública:

```
ping -c 1 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=64 time=0.485 ms

--- 1.1.1.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.485/0.485/0.485/0.000 ms
```

### Consultas y respuestas DNS

```
iptables -A OUTPUT -o eth0 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i eth0 -p udp --sport 53 -j ACCEPT
```

Comprobamos su funcionamiento con una consulta DNS:

```
dig @1.1.1.1 www.openwebinars.net

; <<>> DiG 9.10.3-P4-Debian <<>> @1.1.1.1 www.openwebinars.net
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 898
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.openwebinars.net.		IN	A

;; ANSWER SECTION:
www.openwebinars.net.	261	IN	CNAME	openwebinars.net.
openwebinars.net.	261	IN	A	82.196.7.188

;; Query time: 13 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Fri Aug 24 21:09:55 CEST 2018
;; MSG SIZE  rcvd: 79
```

### Tráfico http

```
iptables -A OUTPUT -o eth0 -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 80 -j ACCEPT
```

Comprobamos que funciona accediendo a un servicio http (! no https)

```
curl http://portquiz.net:80
Port 80 test successful!
Your IP: 37.35.211.73
```

### Tráfico https

```
iptables -A OUTPUT -o eth0 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 443 -j ACCEPT
```

Comprobamos que funciona abriendo un navegador y accediendo a
cualquier sitio web (hoy en día la mayoría son https).

## Configuración en un solo paso

Editamos un fichero y añadimos todas las reglas anteriores:

```
```
iptables -F
iptables -t nat -F
iptables -Z
iptables -t nat -Z
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
iptables -A OUTPUT -o eth0 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -i eth0 -p udp --sport 53 -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 80 -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 443 -j ACCEPT
```

